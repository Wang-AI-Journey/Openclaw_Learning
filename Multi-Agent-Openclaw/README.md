# OpenClaw 飞书多Agent模板 — 对虾智能调控机制研究课题组

基于 [OpenClaw](https://github.com/nicepkg/openclaw) 实现的飞书多Agent协作模板。
三个AI学生（博士/硕士/本科）在同一个飞书群里协作，各司其职。

## 目录结构

```
.
├── openclaw.json          # 主配置（agents + bindings + feishu accounts）
├── workspace-chao/        # 王振潮（博士生）工作区
│   ├── SOUL.md            # 人设文件
│   ├── AGENTS.md          # 团队通讯录 + 协作策略
│   └── USER.md            # 导师信息
├── workspace-hang/        # 林晓杭（硕士生）工作区
│   ├── SOUL.md
│   ├── AGENTS.md
│   └── USER.md
└── workspace-yun/         # 杨若云（本科生）工作区
    ├── SOUL.md
    ├── AGENTS.md
    └── USER.md
```

## 快速开始

### 1. 在飞书开放平台创建3个应用

每个应用需要：
- 开启「机器人」能力
- 事件订阅 → 长连接模式 → 添加 `im.message.receive_v1`
- 申请接收群消息权限
- 创建版本并发布上线
- 将机器人添加到目标群聊

### 2. 替换配置中的占位符

编辑 `openclaw.json`，把以下占位符替换为你自己的值：

| 占位符 | 说明 |
|--------|------|
| `YOUR_API_KEY` | 阿里云百炼 API Key（或其他模型提供商） |
| `YOUR_APP_ID_CHAO` | 王振潮机器人的飞书 App ID |
| `YOUR_APP_SECRET_CHAO` | 王振潮机器人的飞书 App Secret |
| `YOUR_APP_ID_HANG` | 林晓杭机器人的飞书 App ID |
| `YOUR_APP_SECRET_HANG` | 林晓杭机器人的飞书 App Secret |
| `YOUR_APP_ID_YUN` | 杨若云机器人的飞书 App ID |
| `YOUR_APP_SECRET_YUN` | 杨若云机器人的飞书 App Secret |
| `YOUR_USERNAME` | 你的 Linux 用户名 |

同时替换各 `AGENTS.md` 中的 `YOUR_GROUP_ID` 为你的飞书群聊 ID。

### 3. 复制 workspace 目录

```bash
cp -r workspace-chao ~/.openclaw/workspace-chao
cp -r workspace-hang ~/.openclaw/workspace-hang
cp -r workspace-yun  ~/.openclaw/workspace-yun
mkdir -p ~/.openclaw/workspace-chao/memory
mkdir -p ~/.openclaw/workspace-hang/memory
mkdir -p ~/.openclaw/workspace-yun/memory
```

### 4. 合并 openclaw.json

把本模板的 `openclaw.json` 中的关键字段合并到你现有的 `~/.openclaw/openclaw.json`：
- `agents.list`（追加3个学生Agent）
- `bindings`（追加3条路由规则）
- `tools.agentToAgent`
- `tools.sessions`
- `session.agentToAgent`
- `channels.feishu.accounts`（追加3个飞书账户）

### 5. 重启 Gateway

```bash
openclaw gateway stop
openclaw gateway
```

## 核心机制说明

### Agent 创建与路由（bindings）

每个飞书机器人对应一个独立 Agent，通过 `bindings` 路由：
@王振潮 的消息 → `accountId: chao` → `agentId: chao`

### sessions_send：Agent 间通信

阿潮可以通过 `sessions_send` 给小杭/小云分配任务，实现串行协作：
```
sessionKey 格式：agent:<目标agentId>:<发起agentId>
例：agent:hang:chao  （阿潮发给小杭）
```

### sessions_history：读取群聊历史

飞书每个机器人只能收到 @自己 的消息。通过 `sessions_history` 可以读取其他 Agent 的群聊记录：
```
sessionKey 格式：agent:<agentId>:feishu:group:<群聊ID>
例：agent:chao:feishu:group:YOUR_GROUP_ID
```

需要在 `openclaw.json` 中配置 `tools.sessions.visibility: "all"` 才能生效。

## 参考文章

- [OpenClaw安装教程（Windows + WSL）](https://cloud.tencent.com/developer/article/2626160)
- [OpenClaw + 飞书多Agent实现教程 | 3个AI学生到岗，我终于体验了当导师的感觉](#)（公众号文章）
