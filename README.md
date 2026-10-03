# raccoon2api

商汤小浣熊（`xiaohuanxiong.com`）桌面客户端 API 的**接口结构参考**，供构建兼容客户端 / 网关使用。

> 本仓库只收录**接口结构与端点清单**：端点、鉴权形状、请求/响应结构。
> 不含客户端内嵌常量原值、凭据值、账号标识，也不含任何可复现的攻击面细节。

## 端点

| 用途 | 值 |
| --- | --- |
| 主站 / API | `https://xiaohuanxiong.com` |
| AI / LLM baseURL | `https://xiaohuanxiong.com/api/web/llm/v2` |
| 桌面认证前缀 | `/api/electron/auth/v1`（web 面 `/api/web/auth/v1`） |
| 基础 / 业务 API | `/api/web`、`/api/web/office/v3` |
| 组织 | `/api/web/org`、`/api/web/org/user` |
| 团队服务 | `https://xiaohuanxiong.com/agentapi` |
| 协作长连 | `wss://collab-server.xiaohuanxiong.com` |
| 控制台 / 文档 | `https://office-console.xiaohuanxiong.com`、`https://doc.xiaohuanxiong.com` |

## 推理

| 项 | 值 |
| --- | --- |
| 模型目录 | `GET {base}/model_catalog`（base = `/api/web/llm/v2`） |
| 对话 | `POST {base}/chat/completions` |
| Anthropic 模式 | 同 base 走 `/messages` |
| 图像生成 | `POST {base}/images/gen` |
| 默认模型 | `raccoon-chat-ml-5-5` |
| 上下文 / 输出 | contextWindow 默认 180000；maxTokens 官方托管默认 80000 |

## 鉴权

- 请求头：`Authorization: Bearer <access_token>`
- 凭据文件：`%USERPROFILE%\.box-agent\config\auth.json`（可用 `BOX_AGENT_CONFIG_DIR` 覆盖），字段为
  `access_token` / `refresh_token` / `office_identity` / `office_org_name` / `office_org_role`。
- `access_token` 约 2 小时有效；`refresh_token` 约 30 天，可续期。

## 行为观察

- **思考档位**：上游**默认档即最深思考**。向网关下发档位字段不会得到更深的思考。
- **流式**：长回答必须用**无总超时的 client**；带总超时的 client 会把长回答从中间掐断。

## 范围与免责

- 仅用于互操作与研究。请遵守目标服务的使用条款。
- 本仓库不提供、也不描述绕过鉴权或校验的方法。

## License

MIT
