# Changelog

本仓库所有值得记录的变更。格式遵循 [Keep a Changelog](https://keepachangelog.com/zh-CN/1.1.0/)，
版本号遵循 [语义化版本](https://semver.org/lang/zh-CN/)。

## [v1.0.0] - 2026-09-14

首个正式发布：腾讯 CodeBuddy（国内版）+ WorkBuddy International（国际版）的
CLIProxyAPI 动态插件双端版本。

### 新增
- **domestic/**：国内版插件，上游 `copilot.tencent.com`，浏览器扫码登录，直连（国内网络）
- **international/**：国际版插件，上游 `www.workbuddy.ai`，Google/GitHub OAuth 链接登录，
  默认走 `http://127.0.0.1:7890`（Clash）代理
- 双端均支持：多账号 UID 隔离（独立凭据文件）、模型动态同步（`/v3/config` + 白名单）、
  24h 自动刷新
- 插件元数据加入 WorkBuddy logo（管理面板展示）

### 修复
- **international tool calling**：修复流式响应中 `tool_calls` 增量片段被误删的问题
  （首块携带 id/name、空 arguments，后续块追加片段；删除空字段会破坏工具调用流，
  导致 Claude Code 等经 Anthropic 翻译路径的客户端工具调用失效）
- 保留 top-level 空字段清理逻辑，但显式排除 `tool_calls` 分支

### 变更
- `-cn` 后缀方案（domestic 模型 ID 消歧义）经实测回滚：原名更利于路由兼容，恢复原名

### 文档
- `docs/ARCHITECTURE.md`：架构与工作原理
- `docs/DEPLOY_LINUX.md` / `DEPLOY_MACOS.md` / `DEPLOY_WINDOWS.md`：三平台部署指南
- `README.md`：快速开始（3 步）、功能对比表、常见问题

[v1.0.0]: https://github.com/BevalZ/workbuddy-proxy/releases/tag/v1.0.0
