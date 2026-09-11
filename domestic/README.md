# workbuddy(国内版)插件

把**腾讯 CodeBuddy**(`copilot.tencent.com`)封装成 CLIProxyAPI(CPA)动态插件。任何支持 OpenAI / Anthropic 协议的客户端都能调用 CodeBuddy 背后的模型。

对 [Sliverkiss/cpa-plugin](https://github.com/Sliverkiss/cpa-plugin) 公开 `workbuddy.so` 的 clean-room 逆向重写,补齐了源码与 x86_64 支持;原设计归属 Sliverkiss,本仓库由 BevalZ 维护。

## 功能

- **扫码登录**:浏览器/App 扫码完成 CodeBuddy 登录,token 自动刷新
- **账号隔离**:多账号独立凭据文件(`workbuddy-<uid>.json`)
- **模型动态同步**:每 24h 从 `/v3/config` 拉取最新模型(白名单合并)
- **真流式**:SSE 逐块实时转发
- **审核绕过**:自动改写 Claude Code 固定 system 模板,避免被 CodeBuddy 内容审核拒答
- **直连**:国内网络直连 `copilot.tencent.com`,不走代理

## 模型

`glm-5.3` · `glm-5.2` · `glm-5.3-flash` · `deepseek-v4-pro` · `deepseek-v4-flash` · `deepseek-v4.1-flash` · `kimi-k3-1` · `kimi-k2.7` · `kimi-k2-thinking` · `hy3` · `hy4-preview` · `hy4-preview-x` · `minimax-m3` · `hunyuan-chat` 等(以账号权限为准)

## 编译

```bash
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -buildmode=c-shared -o workbuddy.so .
```

产物 `.so`(Linux)/ `.dylib`(macOS)/ `.dll`(Windows)。放入 CPA `plugins/` 目录:

```yaml
plugins:
  enabled: true
  dir: "plugins"
  configs:
    workbuddy: { enabled: true, priority: 100 }
```

重启 CPA,管理面板添加 workbuddy 凭据 → 扫码登录。

## 使用

```bash
# Claude Code
export ANTHROPIC_BASE_URL=http://localhost:8317
export ANTHROPIC_API_KEY=<api-key>
export ANTHROPIC_MODEL=hy3
claude
```

```bash
# curl
curl http://localhost:8317/v1/chat/completions \
  -H "Authorization: Bearer <api-key>" -H "Content-Type: application/json" \
  -d '{"model":"hy3","messages":[{"role":"user","content":"你好"}],"stream":true}'
```

## 模型同步与白名单

插件启动后后台同步 `/v3/config`(24h 刷新)。`wbModelsFallback()` 中的 `specs` 是**白名单**(实测可用模型),同步结果与白名单合并——白名单模型永远保留,v3/config 补充 context 信息。新增模型需确认权限后加入白名单。

## 已知限制

- CodeBuddy 拒绝非流式请求(`code 11101`),插件内部强制 `stream=true` 再聚合
- `hy3` 系列自动 `reasoning_effort=high`(CodeBuddy 只认 high 档)
- 审核黑名单是猫鼠游戏,`rewriteSystemForUpstream` 需随上游更新

## License

MIT
