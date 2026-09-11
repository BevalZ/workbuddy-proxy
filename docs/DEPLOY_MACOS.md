# macOS 部署指南

在 macOS 上部署 workbuddy-proxy 插件到 CLIProxyAPI。

> macOS 上插件产物是 `.dylib`(动态库),CPA 会自动识别 `plugins/` 目录下的 `.dylib` 文件。

## 前置要求

| 依赖 | 版本 | 说明 |
|---|---|---|
| CLIProxyAPI | v7.2.x | 带 CGO / 插件支持 |
| Go | 1.26+ | 编译插件 |
| Xcode Command Line Tools | 最新 | 提供 clang/gcc |
| 网络 | 国内可达 | 国内版直连;国际版需 7890 代理 |

安装依赖:

```bash
# Xcode Command Line Tools(如未装)
xcode-select --install

# Go(如未装, 推荐 Homebrew)
brew install go

go version    # go1.26+
gcc --version # clang 可用
```

## 1. 克隆本仓库

```bash
git clone https://github.com/BevalZ/workbuddy-proxy.git
cd workbuddy-proxy
```

## 2. 编译插件

### 国内版(workbuddy.dylib)

```bash
cd domestic
CGO_ENABLED=1 GOOS=darwin GOARCH=arm64 go build -buildmode=c-shared -o workbuddy.dylib .
```

### 国际版(workbuddy-int.dylib)

```bash
cd ../international
CGO_ENABLED=1 GOOS=darwin GOARCH=arm64 go build -buildmode=c-shared -o workbuddy-int.dylib .
```

> **架构注意**:
> - Apple Silicon(M1/M2/M3/M4):`GOARCH=arm64`
> - Intel Mac:`GOARCH=amd64`
> - 必须与 CPA 运行架构一致,否则 `dlopen` 失败。

## 3. 部署到 CPA 插件目录

```bash
# 找到 CPA 的 plugins 目录(通常在 CPA 安装目录下)
cp domestic/workbuddy.dylib            <cpa>/plugins/
cp international/workbuddy-int.dylib   <cpa>/plugins/
ls <cpa>/plugins/
# workbuddy-int.dylib  workbuddy.dylib
```

## 4. 配置 config.yaml

编辑 CPA 的 `config.yaml`:

```yaml
plugins:
  enabled: true
  dir: "plugins"
  configs:
    workbuddy: { enabled: true, priority: 100 }
    workbuddy-int: { enabled: true, priority: 100 }
```

## 5. 重启 CPA 并验证

```bash
# 重启方式取决于你怎么跑的 CPA(launchd / 手动 / 容器)
# launchd 示例:
launchctl kickstart -k gui/$(id -u)/<label>

# 验证日志
# CPA 日志应出现:
#   plugin registered plugin_id=workbuddy
#   plugin registered plugin_id=workbuddy-int

# 验证模型
curl -s http://127.0.0.1:8317/v1/models -H "Authorization: Bearer <api-key>" | python3 -m json.tool | grep -E "gpt-5|hy3|deepseek"
```

## 6. 添加凭据(登录)

1. 打开管理面板 `http://127.0.0.1:8317/management.html`
2. **国内版**:添加 `workbuddy` → 浏览器扫码登录
3. **国际版**:添加 `workbuddy-int` → Google/GitHub 登录(开代理访问 www.workbuddy.ai)

登录后凭据存于 `~/.cli-proxy-api/workbuddy-<uid>.json` / `workbuddy-int-<uid>.json`。

## 7. 客户端接入

```bash
# Claude Code
export ANTHROPIC_BASE_URL=http://127.0.0.1:8317
export ANTHROPIC_API_KEY=<your-cpa-api-key>
export ANTHROPIC_MODEL=hy3
claude
```

## launchd 服务化(可选)

```xml
<!-- ~/Library/LaunchAgents/com.cliproxy.api.plist -->
<?xml version="1.0" encoding="UTF-8"?>
<!DOCTYPE plist PUBLIC "-//Apple//DTD PLIST 1.0//EN" "http://www.apple.com/DTDs/PropertyList-1.0.dtd">
<plist version="1.0">
<dict>
    <key>Label</key><string>com.cliproxy.api</string>
    <key>ProgramArguments</key>
    <array>
        <string>/Users/<you>/CLIProxyAPI/cli-proxy-api</string>
    </array>
    <key>WorkingDirectory</key><string>/Users/<you>/CLIProxyAPI</string>
    <key>RunAtLoad</key><true/>
    <key>KeepAlive</key><true/>
</dict>
</plist>
```

```bash
launchctl load ~/Library/LaunchAgents/com.cliproxy.api.plist
```

## 故障排查

| 症状 | 原因 | 解决 |
|---|---|---|
| `dlopen: no suitable image found` | 架构不匹配 | 检查 GOARCH 与 CPA 架构 |
| `plugin not loaded` | 未启用 | 检查 config.yaml |
| `dyld: Library not loaded` | 依赖缺失 | 确认 Go 编译环境完整(重新 `go build`) |
| 国际版超时 | 代理没起 | 确认 Clash 在 127.0.0.1:7890 |
| `11102 service info not found` | 模型无权限 | 换账号或改白名单 |
