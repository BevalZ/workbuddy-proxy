# Linux 部署指南

在 Linux 上部署 workbuddy-proxy 插件到 CLIProxyAPI。

## 前置要求

| 依赖 | 版本 | 说明 |
|---|---|---|
| CLIProxyAPI | v7.2.x | 带 CGO / 插件支持 |
| Go | 1.26+ | 编译插件 |
| gcc | 任意新版本 | cgo 依赖 |
| 网络 | 国内可达 | 国内版直连;国际版需 7890 代理 |

检查依赖:

```bash
go version        # go1.26+
gcc --version     # gcc 可用
# CLIProxyAPI 应已运行在 8317
curl -s http://127.0.0.1:8317/v1/models -H "Authorization: Bearer <api-key>" | head
```

## 安装 CLIProxyAPI(如未安装)

```bash
git clone https://github.com/router-for-me/CLIProxyAPI.git
cd CLIProxyAPI
go build -o cli-proxy-api ./cmd/server
# 首次运行生成 config.yaml
./cli-proxy-api
```

或参考 [CLIProxyAPI 官方文档](https://github.com/router-for-me/CLIProxyAPI)。

## 1. 克隆本仓库

```bash
git clone https://github.com/BevalZ/workbuddy-proxy.git
cd workbuddy-proxy
```

## 2. 编译插件

### 国内版(workbuddy.so)

```bash
cd domestic
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -buildmode=c-shared -o workbuddy.so .
```

### 国际版(workbuddy-int.so)

```bash
cd ../international
CGO_ENABLED=1 GOOS=linux GOARCH=amd64 go build -buildmode=c-shared -o workbuddy-int.so .
```

> **架构注意**:`GOARCH` 必须与 CPA 运行实例一致。检查 CPA 架构:`file $(which cli-proxy-api)`。
> 若 CPA 是 arm64(如树莓派):`GOARCH=arm64`。

## 3. 部署到 CPA 插件目录

```bash
# 找到 CPA 的 plugins 目录(默认在 CPA 安装目录下)
cp domestic/workbuddy.so        <cpa>/plugins/
cp international/workbuddy-int.so <cpa>/plugins/
ls <cpa>/plugins/
# workbuddy-int.so  workbuddy.so
```

## 4. 配置 config.yaml

编辑 CPA 的 `config.yaml`,添加插件配置:

```yaml
plugins:
  enabled: true
  dir: "plugins"
  configs:
    workbuddy: { enabled: true, priority: 100 }
    workbuddy-int: { enabled: true, priority: 100 }
```

> **⚠️ 已知坑(btrfs/文件系统)**:如果 `cp` 后插件加载报 `dlopen ... cannot read file data`(I/O error),
> 说明目标目录有坏块。改用 `dd` 复制并校验:
> ```bash
> dd if=domestic/workbuddy.so of=<cpa>/plugins/workbuddy.so bs=1M
> md5sum <cpa>/plugins/workbuddy.so domestic/workbuddy.so
> ```

## 5. 重启 CPA 并验证

```bash
# systemd 方式(如用 systemd 管理)
systemctl restart cliproxyapi   # 或你的服务名

# 验证插件加载
journalctl -u cliproxyapi --no-pager | grep -i "plugin registered"
# 应看到:
#   plugin registered plugin_id=workbuddy
#   plugin registered plugin_id=workbuddy-int

# 验证模型出现
curl -s http://127.0.0.1:8317/v1/models -H "Authorization: Bearer <api-key>" | python3 -m json.tool | grep -E "gpt-5|hy3|deepseek"
```

## 6. 添加凭据(登录)

1. 打开管理面板 `http://127.0.0.1:8317/management.html`
2. **国内版**:Auth 管理 → 添加 `workbuddy` → 浏览器打开弹出的链接 → CodeBuddy App 扫码
3. **国际版**:Auth 管理 → 添加 `workbuddy-int` → 浏览器打开弹出的链接 → Google/GitHub 登录
   - ⚠️ 访问 `www.workbuddy.ai` 需代理。确保本机有 Clash 在 `127.0.0.1:7890`(插件默认走它);
   - 如代理端口不同,改 `international/main.go` 的 `sharedHTTPClient()` 后重新编译。

登录成功后凭据保存为 `~/.cli-proxy-api/workbuddy-<uid>.json` / `workbuddy-int-<uid>.json`。

## 7. 客户端接入

```bash
# Claude Code
export ANTHROPIC_BASE_URL=http://127.0.0.1:8317
export ANTHROPIC_API_KEY=<your-cpa-api-key>
export ANTHROPIC_MODEL=hy3        # 或 hy4-preview / gpt-5.4 等
claude

# curl 测试
curl http://127.0.0.1:8317/v1/chat/completions \
  -H "Authorization: Bearer <api-key>" -H "Content-Type: application/json" \
  -d '{"model":"hy3","messages":[{"role":"user","content":"你好"}],"stream":true}'
```

## 系统服务化(可选)

用 systemd 管理 CPA:

```ini
# ~/.config/systemd/user/cliproxyapi.service
[Unit]
Description=CLIProxyAPI
After=network.target

[Service]
Type=simple
WorkingDirectory=/home/<user>/CLIProxyAPI
ExecStart=/home/<user>/CLIProxyAPI/cli-proxy-api
Restart=on-failure

[Install]
WantedBy=default.target
```

```bash
systemctl --user daemon-reload
systemctl --user enable --now cliproxyapi
```

## 故障排查

| 症状 | 原因 | 解决 |
|---|---|---|
| `dlopen ... Input/output error` | 文件系统坏块 | `dd` 重新复制 + `md5sum` 校验 |
| `plugin not loaded` | 插件未启用 | 检查 config.yaml plugins.configs |
| `model registrar ... deadline exceeded` | 注册超时(旧版插件) | 确认使用本仓库最新代码 |
| 登录链接打不开 | 网络问题 | 国际版开代理;国内版直连 |
| `11102 service info not found` | 模型无权限 | 换有权限的账号,或改白名单 |
| 国际版超时 | 代理没起 | 确认 Clash 运行在 7890 |
