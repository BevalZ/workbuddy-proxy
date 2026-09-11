# 架构与工作原理

本文档解释 workbuddy-proxy 两个插件的完整工作原理,帮助开发者理解代码结构、登录流程、模型同步和转发逻辑。

## 总体架构

```
┌─────────────┐    OpenAI/Anthropic 协议    ┌──────────────────┐
│  客户端      │ ──────────────────────────> │  CLIProxyAPI      │
│ Claude Code │    http://host:8317         │  (8317)           │
│ Cursor/Cline│                            │  ├─ plugins/      │
│ Kelivo/SDK  │                            │  │  ├─ workbuddy.so    (国内)
└─────────────┘                            │  │  └─ workbuddy-int.so (国际)
                                           └────────┬─────────┘
                                                    │ plugin ABI (c-shared)
                                    ┌───────────────┴───────────────┐
                                    │                               │
                          ┌─────────▼─────────┐          ┌──────────▼─────────┐
                          │ domestic 插件      │          │ international 插件  │
                          │ copilot.tencent.com│          │ www.workbuddy.ai    │
                          │ 直连              │          │ 7890 代理           │
                          └───────────────────┘          └─────────────────────┘
```

**核心**:每个插件是一个 `-buildmode=c-shared` 的 Go 动态库,通过 C ABI 与 CPA 通信。CPA 以 `plugin_id` 区分插件(即 `.so` 文件名),每个插件注册一个 provider。

## 插件 ABI 与生命周期

### 导出符号

```go
//export cliproxy_plugin_init       // 插件加载时调用,注册回调表
//export cliproxyPluginCall         // CPA → 插件的 RPC 入口
//export cliproxyPluginFree         // 释放响应缓冲区
//export cliproxyPluginShutdown     // 插件卸载时调用
```

### 消息分发

`cliproxyPluginCall` 接收 method 名 + JSON 请求,`handleMethod` 按 method 分发:

| Method | 用途 |
|---|---|
| `plugin.register` / `plugin.reconfigure` | 注册 provider 元数据、能力声明 |
| `model.static` / `model.for_auth` | 返回模型列表 |
| `auth.identifier` / `auth.parse` | 凭据识别/解析 |
| `auth.login.start` / `auth.login.poll` | 登录流程(启动/轮询) |
| `auth.refresh` | token 刷新 |
| `executor.execute` / `executor.execute_stream` | 转发聊天请求(非流式/流式) |

## 登录流程

### 国内版(workbuddy,扫码)

```
1. CPA 调用 auth.login.start
2. 插件 POST copilot.tencent.com/v2/plugin/auth/state?platform=CLI
   → {state, authUrl}
3. 插件把 authUrl(二维码登录链接)返回给 CPA 面板
4. 用户浏览器/App 打开链接扫码确认
5. CPA 轮询 auth.login.poll:
   插件 GET /v2/plugin/auth/token?state=<state> 直到拿到 accessToken
6. 插件 GET /v2/plugin/login/account?state=<state> 拿账号信息(uid/nickname)
7. 凭据封装为 AuthData 返回,CPA 持久化为 workbuddy-<uid>.json
```

### 国际版(workbuddy-int,OAuth 链接登录)

登录协议与国内版**完全相同**(同一套代码库),只是上游域名换成 `www.workbuddy.ai`:

```
1. POST www.workbuddy.ai/v2/plugin/auth/state?platform=CLI
   → {state, authUrl: https://www.workbuddy.ai/login?platform=CLI&state=...}
2. 用户浏览器打开该链接 → 页面走 Keycloak OIDC → 选 Google / GitHub 登录
3. 登录完成后轮询 token(同上)
```

> 国际版的 `authUrl` 指向 Keycloak 认证页(`/auth/realms/copilot/protocol/openid-connect/auth?client_id=console`),
> Google/GitHub 是 Keycloak 的 identity provider(支持 `kc_idp_hint=google|github`)。
> 探测结论:标准 OAuth 授权码交换需要 client secret 且本地回调被白名单拒绝,
> 因此**复用官网的 CLI 插件登录协议**是最简路径——这也是本项目采用的方式。

## 账号隔离

```go
// toAuthData 里,每个账号用 UID 生成独立 ID 和文件名:
uid := strings.TrimSpace(sa.Account.UID)
if uid == "" { uid = securityHash(sa.Auth.RefreshToken) }
id := providerName + "-" + uid
FileName: id + ".json"
```

- 国内版凭据文件:`~/.cli-proxy-api/workbuddy-<uid>.json`
- 国际版凭据文件:`~/.cli-proxy-api/workbuddy-int-<uid>.json`
- 不同账号互不覆盖,可同时登录多个账号

## 模型同步机制

```
插件加载
  └─ modelsSyncLoop() 启动(goroutine)
       ├─ 立即 refresh() 一次
       └─ 每 24h ticker refresh()
            └─ loadModelsFromUpstream()
                 ├─ 扫描 ~/.cli-proxy-api/workbuddy-*.json 拿第一个有效 token
                 ├─ GET <upstream>/v3/config (带 Bearer)
                 └─ 与 modelSpecs 白名单合并
                      ├─ 白名单模型永远保留(即使 v3/config 没列出)
                      └─ v3/config 补充 name/context/maxOutput 信息
```

**关键设计**:

1. **注册不阻塞**:`wbModels()` 返回缓存或白名单,绝不阻塞等待上游——避免 CPA 注册超时(`context deadline exceeded`)导致模型列表不更新。
2. **白名单 = 权威**:`modelSpecs` 是实测可用的模型清单。有些模型(如 `deepseek-v4.1-flash`、`hy4-preview`)**可调用但不在 v3/config 列表里**,所以白名单是 source of truth,v3/config 只补充上下文信息。
3. **注册失败兜底**:无凭据时返回硬编码白名单,保证模型列表始终可用。

## 请求转发(Executor)

### 非流式

```
executor.execute
  ├─ forceStreamBody(): 强制 stream=true
  │    (CodeBuddy 上游拒绝非流式, code 11101)
  ├─ rewriteSystemForUpstream(): 对 Claude Code 的固定 system 模板做最小改写
  │    (绕过内容审核黑名单, 见下)
  ├─ POST <upstream>/v2/chat/completions
  └─ aggregateCompletion(): 聚合 SSE 流为单个 chat.completion
```

### 流式

```
executor.execute_stream
  ├─ 有 StreamID → 异步: goroutine pumpUpstreamStream() 逐块 emit 给 CPA
  └─ 无 StreamID → 同步: collectUpstreamStream() 收集全部块返回
```

### 请求头

- `Authorization: Bearer <accessToken>`
- `X-User-Id` / `X-Enterprise-Id`(账号上下文)
- `X-Refresh-Token`(用于上游刷新)
- User-Agent 伪装为 `CLI/2.63.2 CodeBuddy/2.63.2`

## 内容审核绕过(国内版)

腾讯 CodeBuddy 的内容审核把 Claude Code 的两句固定 system 模板**逐字加入黑名单**,命中即拒答:

- `You are Claude Code, Anthropic's official CLI for Claude.`
- `Main branch (you will usually use this for PRs)`

插件在转发前自动做**最小改写**(精确匹配绕过,非语义审核):
`CLI` → `CLI tool`、`Main branch` → `Default branch`。语义不变,Claude Code 照常工作。

> 这是猫鼠游戏:腾讯哪天加新模板句,需要同步更新 `rewriteSystemForUpstream` / `sanitizeBlockedTemplates`。

## 代理配置(国际版)

`international/main.go` 的 `sharedHTTPClient()` 默认把所有上游请求(登录/模型同步/chat)走 `http://127.0.0.1:7890`(Clash 等本地代理):

```go
proxyURL, _ := url.Parse("http://127.0.0.1:7890")
Transport: &http.Transport{ Proxy: http.ProxyURL(proxyURL), ... }
```

国内版 `domestic/main.go` 的 `sharedHTTPClient()` **无代理**(copilot.tencent.com 国内直连)。

## 关键文件

| 文件 | 说明 |
|---|---|
| `main.go` | 全部逻辑(注册、登录、模型同步、执行器) |
| `workbuddy.h` / `workbuddy-int.h` | cgo 生成的头文件(ABI 声明) |
| `go.mod` | 依赖 `github.com/router-for-me/CLIProxyAPI/v7` |

## 调试

- 插件加载/注册日志:CPA 日志里搜 `pluginhost`
- 插件内日志:标准输出,CPA 日志可见
- 常见错误:
  - `dlopen ... cannot read file data` → 插件文件损坏/权限问题
  - `model registrar ... context deadline exceeded` → 注册超时(本仓库已修复:注册不阻塞)
  - 上游 `code 11102 service info not found` → 模型存在但账号无权限
  - 上游 `code 11128 first message is not system prompt` → 请求需带 system 消息
