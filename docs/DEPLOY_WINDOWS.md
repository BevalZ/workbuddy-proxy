# Windows 部署指南

在 Windows 上部署 workbuddy-proxy 插件到 CLIProxyAPI。

> Windows 上插件产物是 `.dll`(动态链接库),CPA 会自动识别 `plugins/` 目录下的 `.dll` 文件。

## 前置要求

| 依赖 | 版本 | 说明 |
|---|---|---|
| CLIProxyAPI | v7.2.x | 带 CGO / 插件支持 |
| Go | 1.26+ | 编译插件 |
| gcc | MinGW-w64 | cgo 编译必需 |
| 网络 | 国内可达 | 国内版直连;国际版需 7890 代理 |

### 安装 MinGW-w64

Windows 上 cgo 编译**必须**有 gcc。两种方式:

**方式 A(推荐):MSYS2**
```powershell
# 下载安装 https://www.msys2.org/
# 在 MSYS2 终端:
pacman -S mingw-w64-x86_64-gcc
# 把 C:\msys64\mingw64\bin 加入 PATH
```

**方式 B:TDM-GCC**
下载 https://jmeubank.github.io/tdm-gcc/ 安装,把 bin 目录加入 PATH。

验证:
```powershell
gcc --version
go version   # go1.26+
```

## 1. 克隆本仓库

```powershell
git clone https://github.com/BevalZ/workbuddy-proxy.git
cd workbuddy-proxy
```

## 2. 编译插件

> 在 **MSYS2/MinGW shell**(或配置好 gcc 的 PowerShell)中执行。

### 国内版(workbuddy.dll)

```powershell
cd domestic
$env:CGO_ENABLED="1"; $env:GOOS="windows"; $env:GOARCH="amd64"
go build -buildmode=c-shared -o workbuddy.dll .
```

### 国际版(workbuddy-int.dll)

```powershell
cd ..\international
$env:CGO_ENABLED="1"; $env:GOOS="windows"; $env:GOARCH="amd64"
go build -buildmode=c-shared -o workbuddy-int.dll .
```

> **架构注意**:`GOARCH` 与 CPA 实例一致(一般 Windows 都是 `amd64`)。
> MSYS2 bash 环境语法:`CGO_ENABLED=1 GOOS=windows GOARCH=amd64 go build -buildmode=c-shared -o workbuddy.dll .`

## 3. 部署到 CPA 插件目录

```powershell
# 找到 CPA 的 plugins 目录
copy domestic\workbuddy.dll          <cpa>\plugins\
copy international\workbuddy-int.dll <cpa>\plugins\
dir <cpa>\plugins\
# workbuddy-int.dll  workbuddy.dll
```

## 4. 配置 config.yaml

编辑 CPA 的 `config.yaml`(与 CPA 主程序同目录):

```yaml
plugins:
  enabled: true
  dir: "plugins"
  configs:
    workbuddy: { enabled: true, priority: 100 }
    workbuddy-int: { enabled: true, priority: 100 }
```

## 5. 重启 CPA 并验证

```powershell
# 重启 CPA(视部署方式而定)
# 若为 Windows 服务:
Restart-Service <service-name>
# 若为前台运行:关掉重开

# 验证日志(CPA 控制台/stdout):
#   plugin registered plugin_id=workbuddy
#   plugin registered plugin_id=workbuddy-int

# 验证模型
curl.exe -s http://127.0.0.1:8317/v1/models -H "Authorization: Bearer <api-key>"
```

## 6. 添加凭据(登录)

1. 打开管理面板 `http://127.0.0.1:8317/management.html`
2. **国内版**:添加 `workbuddy` → 浏览器扫码
3. **国际版**:添加 `workbuddy-int` → Google/GitHub 登录(开代理)

凭据存于 `C:\Users\<you>\.cli-proxy-api\workbuddy-<uid>.json` / `workbuddy-int-<uid>.json`。

## 7. 客户端接入

```powershell
# Claude Code
$env:ANTHROPIC_BASE_URL="http://127.0.0.1:8317"
$env:ANTHROPIC_API_KEY="<your-cpa-api-key>"
$env:ANTHROPIC_MODEL="hy3"
claude

# curl 测试
curl.exe http://127.0.0.1:8317/v1/chat/completions `
  -H "Authorization: Bearer <api-key>" -H "Content-Type: application/json" `
  -d '{\"model\":\"hy3\",\"messages\":[{\"role\":\"user\",\"content\":\"你好\"}],\"stream\":true}'
```

## Windows 服务化(可选)

用 NSSM 把 CPA 注册为 Windows 服务:

```powershell
# 下载 https://nssm.cc/
nssm install CLIProxyAPI "<cpa>\cli-proxy-api.exe"
nssm set CLIProxyAPI AppDirectory "<cpa>"
nssm start CLIProxyAPI
```

## 故障排查

| 症状 | 原因 | 解决 |
|---|---|---|
| `gcc: command not found` | MinGW 未装/未在 PATH | 安装 MSYS2 gcc,加入 PATH |
| `build failed: cgo: C compiler` | gcc 环境问题 | 确认在 MSYS2 shell 编译 |
| `dlopen ... The specified module could not be found` | 缺 DLL 依赖/架构不符 | 检查 GOARCH;确认 Go 工具链完整 |
| 插件不加载 | config.yaml 未配置 | 检查 plugins.configs |
| 国际版超时 | 代理没起 | 确认 Clash 在 127.0.0.1:7890 |
| `11102 service info not found` | 模型无权限 | 换账号或改白名单 |

## Windows 常见坑

1. **PATH 顺序**:PowerShell 里 `gcc` 可能被系统自带命令遮蔽,优先把 MinGW bin 放最前。
2. **防火墙**:CPA 监听 8317 时,Windows 防火墙可能拦截外部访问,放行 TCP 8317。
3. **代理软件冲突**:国际版走 7890 代理,确保代理软件(Clash)允许本地回环连接(允许 LAN)。
