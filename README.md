# workbuddy-proxy

把 **腾讯 CodeBuddy**(国内版)和 **WorkBuddy International**(国际版,`www.workbuddy.ai`)封装成 [CLIProxyAPI](https://github.com/router-for-me/CLIProxyAPI)(CPA)动态插件,让任何支持 OpenAI / Anthropic 协议的客户端(Claude Code、Cursor、Cline、Kelivo、SDK……)都能直接调用两个平台背后的模型。

**仓库结构**:

```
workbuddy-proxy/
├── domestic/          # 国内版插件 → 腾讯 CodeBuddy (copilot.tencent.com)
├── international/     # 国际版插件 → WorkBuddy.ai (www.workbuddy.ai)
├── docs/
│   ├── ARCHITECTURE.md    # 架构与工作原理
│   ├── DEPLOY_LINUX.md    # Linux 部署
│   ├── DEPLOY_MACOS.md    # macOS 部署
│   └── DEPLOY_WINDOWS.md  # Windows 部署
└── README.md
```

## 功能特性

| 能力 | 国内版 domestic | 国际版 international |
|---|---|---|
| 上游 | `copilot.tencent.com` | `www.workbuddy.ai` |
| 登录方式 | 浏览器扫码 | Google / GitHub OAuth 链接登录 |
| 代理 | 直连(国内) | 默认走 `http://127.0.0.1:7890`(Clash) |
| 模型来源 | 动态同步 `/v3/config` + 白名单 | 动态同步 + 白名单(实测可用) |
| 账号隔离 | 多账号独立文件 | 多账号独立文件 |
| 模型自动刷新 | 24h ticker | 24h ticker |

## 快速开始(3 步)

### 前置要求

- 运行中的 **CLIProxyAPI v7.2.x**(带 CGO / 插件支持,默认端口 8317)
- **Go 1.26+** 和 **gcc**(编译插件用)
- 一个 CodeBuddy 账号(国内版)或 WorkBuddy 账号(国际版)

### 1. 编译插件

```bash
# 国内版
cd domestic
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -buildmode=c-shared -o workbuddy.so .

# 国际版
cd international
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -buildmode=c-shared -o workbuddy-int.so .
```

> macOS 用 `GOOS=darwin` 产物是 `.dylib`;Windows 用 `GOOS=windows` 产物是 `.dll`。
> 架构需与 CPA 实例一致(amd64 / arm64)。

### 2. 部署插件

把编译产物放入 CPA 的 `plugins/` 目录,并在 `config.yaml` 启用:

```yaml
plugins:
  enabled: true
  dir: "plugins"
  configs:
    workbuddy: { enabled: true, priority: 100 }
    workbuddy-int: { enabled: true, priority: 100 }
```

重启 CPA,日志出现 `plugin registered plugin_id=workbuddy` / `plugin_id=workbuddy-int` 即成功。

### 3. 添加凭据(登录)

- 打开管理面板 `http://<host>:8317/management.html`
- **国内版**:添加 `workbuddy` 凭据 → 浏览器打开弹出的二维码 URL → 用 CodeBuddy App 扫码
- **国际版**:添加 `workbuddy-int` 凭据 → 浏览器打开弹出的链接 → 用 **Google / GitHub** 登录(建议开代理访问 `www.workbuddy.ai`)

登录成功后凭据自动保存,`GET /v1/models` 即可看到模型。

## 客户端接入

| 协议 | Base URL |
|---|---|
| OpenAI | `http://<host>:8317/v1` |
| Anthropic | `http://<host>:8317`(不带 `/v1`,走 `x-api-key`) |

```bash
# Claude Code 示例
export ANTHROPIC_BASE_URL=http://localhost:8317
export ANTHROPIC_API_KEY=<your-cpa-api-key>
export ANTHROPIC_MODEL=hy3
claude
```

```bash
# curl / OpenAI 示例(国内版 hy3)
curl http://localhost:8317/v1/chat/completions \
  -H "Authorization: Bearer <your-cpa-api-key>" -H "Content-Type: application/json" \
  -d '{"model":"hy3","messages":[{"role":"user","content":"你好"}],"stream":true}'
```

## 模型

### 国内版(domestic)

`glm-5.3` · `glm-5.2` · `glm-5.3-flash` · `deepseek-v4-pro` · `deepseek-v4-flash` · `deepseek-v4.1-flash` · `kimi-k3-1` · `kimi-k2.7` · `hy3` · `hy4-preview` · `hy4-preview-x` · `minimax-m3` · `hunyuan-chat` 等(具体可用性以 CodeBuddy 账号权限为准)

### 国际版(international)

`gpt-5.5` · `gpt-5.4` · `gpt-5.3-codex` · `gpt-5.6-terra` · `gpt-5.6-luna` · `gemini-3.1-pro` · `gemini-3.5-flash` · `deepseek-v4.1-flash` · `glm-5.3` · `glm-5.2` · `hy3` · `hy4-preview` · `hy4-preview-x` · `kimi-k3` · `minimax-m3` 等(实测可用白名单)

> 模型列表每 24 小时自动从上游 `/v3/config` 同步刷新,无需手动更新。
> 新模型需确认账号有权限后,加入对应插件 `main.go` 里的 `modelSpecs` 白名单。

## 已知限制与注意事项

- **国际版需代理**:`www.workbuddy.ai` 在国内需走代理,插件默认已配置 `http://127.0.0.1:7890`;如代理端口不同,修改 `international/main.go` 中 `sharedHTTPClient()` 的 `proxyURL`。
- **国内版直连**:`copilot.tencent.com` 国内直连,插件不走代理。
- **内容审核**:国内版 CodeBuddy 会把 Claude Code 的固定 system 模板句加入黑名单,插件已自动做最小改写绕过(见 ARCHITECTURE.md)。
- **思考模式**:国内版 `hy3` 系列自动开最大思考(`reasoning_effort=high`),思考内容走 SSE 的 `delta.reasoning_content`。
- 上游可能随时调整模型权限,白名单需按账号实际情况维护。

## License

MIT
