# workbuddy-int(国际版)插件

把 **WorkBuddy International**(`www.workbuddy.ai`)封装成 CLIProxyAPI(CPA)动态插件。任何支持 OpenAI / Anthropic 协议的客户端都能调用国际版 WorkBuddy 背后的模型(GPT-5.x、Gemini、DeepSeek、GLM、Kimi 等)。

基于国内版(腾讯 CodeBuddy)插件的同一套登录协议逆向——国际版与国内版共用代码库,`/v2/plugin/auth/state` 等 CLI 插件登录端点一致,仅上游域名不同。

## 与国际版国内版的关键差异

| | 国内版 domestic | 国际版 international |
|---|---|---|
| 上游 | `copilot.tencent.com` | `www.workbuddy.ai` |
| 登录 | 扫码 | Google / GitHub 链接登录 |
| 代理 | 直连 | **默认走 `http://127.0.0.1:7890`** |
| 模型 | 国内模型(glm/deepseek/hy3...) | 国际模型(gpt-5.x/gemini/deepseek-v4.1/hy4...) |

## 功能

- **OAuth 链接登录**:Google / GitHub 账号登录(上游走 Keycloak OIDC)
- **账号隔离**:多账号独立凭据(`workbuddy-int-<uid>.json`)
- **模型动态同步 + 白名单**:24h 从 `/v3/config` 同步,白名单为实测可用模型
- **自动代理**:所有上游请求走 `127.0.0.1:7890`(Clash),绕过 GFW
- **真流式**:SSE 逐块转发

## 模型(实测可用白名单)

`gpt-5.5` · `gpt-5.4` · `gpt-5.3-codex` · `gpt-5.6-terra` · `gpt-5.6-luna` · `gemini-3.1-pro` · `gemini-3.5-flash` · `deepseek-v4.1-flash` · `glm-5.3` · `glm-5.2` · `hy3` · `hy4-preview` · `hy4-preview-x` · `kimi-k3` · `kimi-k2.6` · `kimi-k2.5` · `minimax-m3` · `default-model` 系列等

> ⚠️ 有些模型在上游路由层存在但账号无权限(`11102 service info not found`),如 `gpt-6` 系列。白名单只保留实测可用的。

## 编译

```bash
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -buildmode=c-shared -o workbuddy-int.so .
```

产物放入 CPA `plugins/`:

```yaml
plugins:
  enabled: true
  dir: "plugins"
  configs:
    workbuddy-int: { enabled: true, priority: 100 }
```

## 登录

管理面板添加 `workbuddy-int` 凭据 → 浏览器打开弹出的 `www.workbuddy.ai/login?platform=CLI&state=...` 链接 → **Google / GitHub 登录** → 自动完成。

> ⚠️ 访问 www.workbuddy.ai 需代理。插件默认请求走 `127.0.0.1:7890`;浏览器登录时也要开代理。

## 代理配置

`sharedHTTPClient()` 默认 `http://127.0.0.1:7890`。如你的代理端口不同:

```go
proxyURL, _ := url.Parse("http://127.0.0.1:7890")  // 改成你的代理
```

重新编译即可。

## 使用

```bash
# Claude Code
export ANTHROPIC_BASE_URL=http://localhost:8317
export ANTHROPIC_API_KEY=<api-key>
export ANTHROPIC_MODEL=hy4-preview
claude
```

```bash
# curl
curl http://localhost:8317/v1/chat/completions \
  -H "Authorization: Bearer <api-key>" -H "Content-Type: application/json" \
  -d '{"model":"gpt-5.4","messages":[{"role":"user","content":"hello"}],"stream":true}'
```

## 模型白名单维护

`main.go` 的 `modelSpecs` 是权威白名单。新增模型流程:
1. 用已登录凭据探测:`POST /v2/chat/completions`(stream=true,带 system 消息)
2. 返回 `11102` → 账号无权限,不加
3. 正常回复 → 加入 `modelSpecs`

## License

MIT
