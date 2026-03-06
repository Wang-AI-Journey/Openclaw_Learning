# OpenClaw飞书扩展源码修改说明

## 概述

本目录包含实现多智能体多轮对话功能的核心源码文件。这些修改使得Agent之间可以通过@互相唤醒，被@的Agent以自己的飞书身份在群里回复，形成多轮对话链。

## 修改文件清单

| 文件 | 修改点数量 | 作用 |
|------|-----------|------|
| bot.ts | 4处 | 痛点1修复 + synthetic事件转发核心逻辑 |
| reply-dispatcher.ts | 3处 | 保存回复文本供@检测使用 |

## bot.ts 修改详解

### 修改点1：痛点1修复 — 注释掉mentionTargets传递（约第504行）

**位置**：`handleFeishuMessage` 函数中，`isMentionForwardRequest` 判断块内

**修改前**：
```typescript
if (isMentionForwardRequest(event, botOpenId)) {
  const mentionTargets = extractMentionTargets(event, botOpenId);
  if (mentionTargets.length > 0) {
    ctx.mentionTargets = mentionTargets;  // 会导致Bot回复时自动@其他人
    const allMentionKeys = (event.message.mentions ?? []).map((m) => m.key);
    ctx.mentionMessageBody = extractMessageBody(content, allMentionKeys);
  }
}
```

**修改后**：
```typescript
if (isMentionForwardRequest(event, botOpenId)) {
  const mentionTargets = extractMentionTargets(event, botOpenId);
  if (mentionTargets.length > 0) {
    // ctx.mentionTargets = mentionTargets;  // 痛点1修复：不传递其他@目标
    const allMentionKeys = (event.message.mentions ?? []).map((m) => m.key);
    ctx.mentionMessageBody = extractMessageBody(content, allMentionKeys);
  }
}
```

**作用**：防止Bot回复时自动@消息中提到的其他Bot（痛点1）

---

### 修改点2：synthetic消息检测 + 强制mentionedBot（约第541行）

**位置**：`parseFeishuMessageEvent` 调用之后

**添加代码**：
```typescript
let ctx = parseFeishuMessageEvent(event, botOpenId);
const isGroup = ctx.chatType === "group";

// 痛点3修复v5：检测synthetic消息（由其他Agent的@转发产生）
// synthetic消息的message_id以"synthetic_"开头
// 强制mentionedBot=true，绕过requireMention检查
const isSyntheticForward = event.message.message_id.startsWith("synthetic_");
if (isSyntheticForward) {
  ctx = { ...ctx, mentionedBot: true };
  log(`feishu[${account.accountId}]: synthetic forward detected, forcing mentionedBot=true`);
}
```

**作用**：让synthetic消息绕过`requireMention`检查，确保被@的Agent能正常处理消息

---

### 修改点3：synthetic消息不传replyToMessageId（约第948行）

**位置**：`createFeishuReplyDispatcher` 调用处

**修改前**：
```typescript
const { dispatcher, replyOptions, markDispatchIdle, getLastDeliveredText } =
  createFeishuReplyDispatcher({
    cfg,
    agentId: route.agentId,
    runtime: runtime as RuntimeEnv,
    chatId: ctx.chatId,
    replyToMessageId: ctx.messageId,
    mentionTargets: ctx.mentionTargets,
    accountId: account.accountId,
  });
```

**修改后**：
```typescript
const { dispatcher, replyOptions, markDispatchIdle, getLastDeliveredText } =
  createFeishuReplyDispatcher({
    cfg,
    agentId: route.agentId,
    runtime: runtime as RuntimeEnv,
    chatId: ctx.chatId,
    // 痛点3 v5 hotfix: synthetic消息不传replyToMessageId，避免飞书API报错
    replyToMessageId: isSyntheticForward ? undefined : ctx.messageId,
    mentionTargets: ctx.mentionTargets,
    accountId: account.accountId,
  });
```

**作用**：避免飞书API用假的message_id调用typing indicator和reply接口报错

---

### 修改点4：dispatch complete后检测@并构造synthetic事件转发（约第986行）

**位置**：`dispatch complete`日志之后

**添加代码**：
```typescript
log(`feishu[${account.accountId}]: dispatch complete (...)`);

// 痛点3修复v5：Bot互相@唤醒
// 飞书不会把Bot消息推送给其他Bot，所以在dispatch complete后
// 检测回复中的@agentName，构造synthetic事件直接触发对应Agent
// 轮次控制：synthetic消息最多转发MAX_SYNTHETIC_TURNS轮
const MAX_SYNTHETIC_TURNS = 2;
if (isGroup && getLastDeliveredText) {
  try {
    const replyText = getLastDeliveredText();
    if (replyText) {
      // 从当前message_id提取轮次（synthetic_xxx_turn_N）
      let currentTurn = 0;
      const turnMatch = ctx.messageId.match(/synthetic_.*_turn_(\d+)/);
      if (turnMatch) currentTurn = parseInt(turnMatch[1], 10);

      if (currentTurn >= MAX_SYNTHETIC_TURNS) {
        log(`feishu[${account.accountId}]: synthetic turn limit reached (${currentTurn}/${MAX_SYNTHETIC_TURNS}), stopping chain`);
      } else {
        const allAgentIds = ["luqi","andrew","harrison","shunyu","oriol","junyang","tianqi","peter","yejin","junjie"];
        const mentionedAgents: string[] = [];
        const lowerReply = replyText.toLowerCase();
        for (const aid of allAgentIds) {
          if (aid !== account.accountId && lowerReply.includes(`@${aid}`)) {
            mentionedAgents.push(aid);
          }
        }
        if (mentionedAgents.length > 0) {
          const nextTurn = currentTurn + 1;
          log(`feishu[${account.accountId}]: detected @mentions in reply: [${mentionedAgents.join(", ")}], turn ${nextTurn}/${MAX_SYNTHETIC_TURNS}`);
          for (const targetAccountId of mentionedAgents) {
            const syntheticEvent: FeishuMessageEvent = {
              sender: {
                sender_id: {
                  open_id: "",  // 不传cross-app的open_id
                  user_id: "",
                },
                sender_type: "user",
              },
              message: {
                message_id: `synthetic_${ctx.messageId}_${targetAccountId}_turn_${nextTurn}_${Date.now()}`,
                chat_id: ctx.chatId,
                chat_type: "group",
                message_type: "text",
                content: JSON.stringify({ text: `${account.accountId} 说：${replyText}` }),
                mentions: [],
              },
            };
            log(`feishu[${account.accountId}]: forwarding to ${targetAccountId} via synthetic event (turn ${nextTurn})`);
            handleFeishuMessage({
              cfg,
              event: syntheticEvent,
              botOpenId: undefined,
              runtime: runtime as RuntimeEnv,
              chatHistories,
              accountId: targetAccountId,
            }).catch((e) => {
              log(`feishu[${account.accountId}]: forward to ${targetAccountId} failed: ${String(e)}`);
            });
          }
        }
      }
    }
  } catch (e) {
    // 静默失败，不影响主流程
  }
}
```

**作用**：
1. 检测Agent回复中的`@agentName`
2. 构造synthetic事件直接调用目标Agent的`handleFeishuMessage`
3. 轮次控制防止无限对话（MAX_SYNTHETIC_TURNS=2）
4. 被@的Agent以自己的飞书身份回复到群里

**关键设计**：
- `allAgentIds`数组需要包含所有Agent的accountId
- 使用小写匹配（`lowerReply.includes(\`@${aid}\`)`）支持大小写不敏感
- synthetic消息的message_id格式：`synthetic_{原始ID}_{目标Agent}_turn_{轮次}_{时间戳}`
- sender open_id设为空，避免跨应用错误

---

## reply-dispatcher.ts 修改详解

### 修改点1：添加闭包变量（约第83行）

**位置**：`createFeishuReplyDispatcher` 函数开头

**添加代码**：
```typescript
// 痛点3：保存最后的回复文本，供dispatch complete后检测@
let _lastDeliveredText = "";

let streaming: FeishuStreamingSession | null = null;
```

**作用**：创建闭包变量保存Agent的回复文本

---

### 修改点2：在deliver回调中保存文本（约第147行）

**位置**：`deliver` 回调函数开头

**修改前**：
```typescript
deliver: async (payload: ReplyPayload, info) => {
  const text = payload.text ?? "";
  if (!text.trim()) {
    return;
  }
  // ... 后续发送逻辑
```

**修改后**：
```typescript
deliver: async (payload: ReplyPayload, info) => {
  const text = payload.text ?? "";
  _lastDeliveredText = text; // 痛点3：保存回复文本
  if (!text.trim()) {
    return;
  }
  // ... 后续发送逻辑不变
```

**作用**：每次发送回复时保存文本内容

---

### 修改点3：在return中暴露getter（约第219行）

**位置**：`createFeishuReplyDispatcher` 函数返回值

**修改前**：
```typescript
return {
  dispatcher,
  replyOptions: {
    // ...
  },
  markDispatchIdle,
};
```

**修改后**：
```typescript
return {
  dispatcher,
  getLastDeliveredText: () => _lastDeliveredText,  // 新增
  replyOptions: {
    // ...
  },
  markDispatchIdle,
};
```

**作用**：暴露getter函数供bot.ts在dispatch complete后读取回复文本

---

## 工作原理

### 完整消息流转

```
用户在飞书群 @Harrison "你想跟谁交流？"
        ↓
飞书推送事件给所有Bot → Harrison的Bot检测到被@
        ↓
Harrison的Agent生成回复："我想问 @tianqi 一个问题..."
        ↓
reply-dispatcher保存回复文本 → 通过飞书API以Harrison身份发送
        ↓
bot.ts dispatch complete后检测到 @tianqi
        ↓
构造synthetic事件 → 直接调用handleFeishuMessage({ accountId: "tianqi" })
        ↓
tianqi的Bot检测到synthetic消息 → 强制mentionedBot=true
        ↓
tianqi的Agent生成回复 → 以tianqi的飞书身份发送到群里
        ↓
继续检测@mentions → 如果有则继续转发（turn 2）
        ↓
达到MAX_SYNTHETIC_TURNS=2 → 停止对话链
```

### 为什么用synthetic event？

飞书平台限制：Bot A发的消息不会触发Bot B的`im.message.receive_v1`事件订阅。因此必须在OpenClaw内部完成转发，完全绕过飞书的@机制。

### 为什么目标Agent能以自己的身份回复？

`handleFeishuMessage`接收`accountId: targetAccountId`参数，内部通过`resolveFeishuAccount`解析出目标Agent的飞书配置（appId、appSecret），`createFeishuReplyDispatcher`使用该配置创建飞书客户端，最终以目标Agent的身份发送消息。

---

## 应用这些修改

### 方法1：直接替换文件

```bash
# 备份原文件
cp ~/.openclaw/extensions/feishu/src/bot.ts ~/.openclaw/extensions/feishu/src/bot.ts.backup
cp ~/.openclaw/extensions/feishu/src/reply-dispatcher.ts ~/.openclaw/extensions/feishu/src/reply-dispatcher.ts.backup

# 复制修改后的文件
cp ./src/feishu/bot.ts ~/.openclaw/extensions/feishu/src/
cp ./src/feishu/reply-dispatcher.ts ~/.openclaw/extensions/feishu/src/

# 重启OpenClaw Gateway
openclaw gateway stop
openclaw gateway
```

### 方法2：手动应用修改

参考上述修改详解，在对应位置手动添加或修改代码。

---

## 注意事项

1. **allAgentIds数组**：bot.ts第986行的`allAgentIds`数组需要包含你所有Agent的accountId
2. **MAX_SYNTHETIC_TURNS**：默认为2，可根据需要调整（建议1-3之间）
3. **@格式**：Agent回复中使用`@accountId`格式（如`@tianqi`），不需要全名
4. **轮次控制**：对话链会在达到最大轮次后自动停止，防止无限循环
5. **错误处理**：synthetic转发逻辑包含try-catch，失败不会影响主流程

---
