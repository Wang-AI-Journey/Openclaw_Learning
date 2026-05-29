# Hermes Agent 配置文件包

用于 Windows + Docker Desktop 部署 [Hermes Agent](https://github.com/NousResearch/hermes-agent) 的即用配置文件包。

## 包含文件

| 文件 | 说明 |
|------|------|
| `.env` | 环境配置模板（API Key / 飞书 / 微信，占位符需替换） |
| `config.yaml` | Hermes 主配置（已配 DeepSeek 模型） |
| `docker-compose.yml` | Docker 编排文件（含 Windows 权限修复 + 飞书 SDK 自动安装） |
| `feishu.py` | 飞书表格渲染补丁（Markdown 表格 → CardKit v2 原生表格） |
| `SOUL.md` | Agent 人格定义（可自定义） |
| `VERSION` | 版本标记 |

## 快速使用

1. 修改 `.env` 中的 API Key 和平台凭证
2. 确保 `hermes-agent-main-amd64.tar` 镜像已通过 `docker load` 加载
3. `docker compose up -d` 启动

详细教程见配套部署文档。
