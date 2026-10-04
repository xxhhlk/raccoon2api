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
| 积分服务 | `/api/web/points/v1` |
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
| 默认模型 | `raccoon-8c4485`；客户端别名 `raccoon-chat-ml-5-5`（路由到默认） |
| 上下文 / 输出 | contextWindow 默认 180000；maxTokens 官方托管默认 80000 |

### 请求形状

```
POST {base}/chat/completions
Authorization: Bearer <access_token>      # 只认这个头（token / X-Access-Token / Cookie 均 401）
Content-Type: application/json

body: { model, messages, stream, temperature, max_tokens, ... }
```

### 响应形态

- 非流式：标准 OpenAI 形状（`choices[].message.content` + `usage`）；另带
  `choices[].message.provider_specific_fields`、`usage.completion_tokens_details` / `prompt_tokens_details`
- 流式：真增量 SSE（`data: {...}` 分片，末尾 `data: [DONE]`）

### 错误信封（LiteLLM 风格）

```
401 {"code":200001,"message":"authorization_empty_error"}     # 缺 token
401 {"code":200003,"message":"authorization_verify_error"}    # 校验失败
400 {"error":{"code":"400","message":"litellm.BadRequestError: ..."}}
```

### 未知模型

上游对未知模型名**不报错，而是静默回落到默认模型并返回 200** —— 调用方必须本地校验模型名，
否则会以为在用 A 模型、实际消耗默认模型额度。

### 模型表（快照，以 `GET /model_catalog` 为准）

`visible:false` 的内部模型 UI 不展示但**可调用**。

| model_name | 说明 | visible | ctx | max_out | 标签 |
| --- | --- | --- | --- | --- | --- |
| `raccoon-8c4485` | Raccoon-Work（默认） | false | 1000000 | 100000 | vision / reasoning |
| `raccoon-19b265` | Raccoon-Work-260817-A | false | 1000000 | 100000 | auto |
| `raccoon-405a1c` | Raccoon-Work-260817-B | false | 1000000 | 100000 | fast |
| `sn-sensenova-6-8-flash-lite` | SenseNova-6.8-Flash-Lite | true | 256000 | 63999 | vision / fast |
| `sn-glm-5-3` | GLM-5-3 | true | 1000000 | 100000 | reasoning |
| `sn-kimi-k3` | Kimi-K3 | true | 1000000 | 100000 | reasoning |
| `sn-glm-5-3-flash` | GLM-5-3-Flash | true | 1000000 | 100000 | lite |
| `sn-deepseek-v4-1-flash` | DeepSeek-V4.1-Flash | true | 1000000 | 100000 | reasoning / auto |

## 鉴权

- 请求头：`Authorization: Bearer <access_token>`
- 凭据文件：`%USERPROFILE%\.box-agent\config\auth.json`（可用 `BOX_AGENT_CONFIG_DIR` 覆盖），字段为
  `access_token` / `refresh_token` / `office_identity` / `office_org_name` / `office_org_role`。
- `access_token` 约 2 小时有效；`refresh_token` 约 30 天。

### 刷新

```
POST https://xiaohuanxiong.com/api/electron/auth/v1/refresh
     # 亦可用 /api/web/auth/v1/refresh
body: {"refresh_token": "<refresh_token>"}
resp: {"data":{"access_token":"...","refresh_token":"..."}}
```

- **refresh_token 会轮换**，响应里的新值必须一并落盘，否则下次刷新失败。
- access 仅约 2 小时，调用方需在过期前自行刷新（客户端窗口为过期前 300 秒）。

## 积分

`GET /api/web/points/v1/balance`（鉴权同推理）→
`{"code":0,"data":{"available_points","daily_points","monthly_points","reward_points","topup_points","topup_frozen"}}`

## 行为观察

- **思考档位**：上游**默认档即最深思考**。向网关下发档位字段不会得到更深的思考，反而可能削弱 —— 调用方应主动剥离 `reasoning_effort`。
- **流式**：长回答必须用**无总超时的 client**；带总超时的 client 会把长回答从中间掐断。

## 最小调用示例

仅示意协议形状（导入 → 调用 → 解析 SSE → 刷新），非可部署实现。

```python
# 凭据文件（BOX_AGENT_CONFIG_DIR 可覆盖）：~/.box-agent/config/auth.json
#   {"access_token": "...", "refresh_token": "...", ...}
import json, urllib.request

def chat(base_url, access_token, model, prompt):
    body = json.dumps({"model": model, "stream": True,
                       "messages": [{"role": "user", "content": prompt}]}).encode()
    req = urllib.request.Request(base_url + "/chat/completions", data=body, method="POST")
    req.add_header("Authorization", f"Bearer {access_token}")   # 只认这个头
    req.add_header("Content-Type", "application/json")
    with urllib.request.urlopen(req) as resp:
        for raw in resp:                      # 流式：逐行读 data: 分片
            line = raw.decode("utf-8", "replace").strip()
            if not line.startswith("data:"):
                continue
            payload = line[5:].strip()
            if payload == "[DONE]":
                break
            delta = json.loads(payload)["choices"][0].get("delta", {})
            print(delta.get("content", ""), end="", flush=True)

def refresh(auth_origin, refresh_token):
    # access 仅 2h：换新 access+refresh（refresh 会轮换，务必落盘）
    body = json.dumps({"refresh_token": refresh_token}).encode()
    req = urllib.request.Request(auth_origin + "/api/electron/auth/v1/refresh",
                                 data=body, method="POST")
    req.add_header("Content-Type", "application/json")
    with urllib.request.urlopen(req) as resp:
        return json.load(resp)["data"]        # {"access_token", "refresh_token"}
```

## 范围与免责

- 仅用于互操作与研究。请遵守目标服务的使用条款。
- 本仓库不提供、也不描述绕过鉴权或校验的方法。

## License

MIT
