# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

Grok 账号批量注册自动化工具，面向 Windows 平台。使用 Tkinter GUI + DrissionPage 浏览器自动化 + curl_cffi HTTP 客户端实现完整的注册流程。

## 常用命令

```bash
# 安装/同步依赖
uv sync

# 运行主程序
uv run python3 grok_register_ttk.py

# 运行邮箱调试工具
uv run python3 cf_mail_debug.py

# 添加新依赖
uv add <package>
```

## 代码架构

### 核心文件

- `grok_register_ttk.py` - 主程序（约 2300 行），包含完整的注册流程和 GUI
- `cf_mail_debug.py` - Cloudflare 邮箱 API 调试工具
- `config.example.json` - 配置示例模板

### 主程序结构 (grok_register_ttk.py)

**配置层** - `load_config()` / `save_config()` 读写 config.json，GUI 启动时加载

**邮箱提供商抽象** - 三个独立的提供商实现，各有 5 个核心函数：
- DuckMail: `get_domains`, `create_account`, `get_token`, `get_messages`, `get_message_detail`
- Cloudflare: `cloudflare_*` 前缀函数，支持自定义 API Base 和鉴权模式
- YYDS: `yyds_*` 前缀函数，支持 API Key 和 JWT 两种鉴权

**注册流程** - 顺序执行的浏览器自动化步骤：
1. `open_signup_page` → `click_email_signup_button` 打开注册页
2. 邮箱提供商创建临时邮箱
3. `fill_email_and_submit` 提交邮箱
4. `fill_code_and_submit` 轮询验证码并提交
5. `fill_profile_and_submit` 填写用户资料
6. `wait_for_sso_cookie` 提取 SSO token

**HTTP 工具层** - `http_get` / `http_post` 封装 curl_cffi，统一代理和超时配置

**grok2api 入池** - `add_token_to_grok2api_pools` 支持本地文件和远程接口两种入池方式

**GUI** - `GrokRegisterGUI` 类，Tkinter 实现，支持多线程并发注册

### 关键设计

- 所有网络请求支持代理（通过 `get_proxies()` 从配置读取）
- 浏览器实例全局单例，通过 `restart_browser()` 重建
- 注册流程支持取消（`cancel_callback` + `raise_if_cancelled`）
- 并发控制使用 `queue.Queue` + 工作线程池模式

## 配置说明

运行前需创建 `config.json`（参考 `config.example.json`）：
- `email_provider` - 邮箱服务商：`duckmail` / `yyds` / `cloudflare`
- `cloudflare_*` - Cloudflare 专用配置（API Base、Key、鉴权模式、路径）
- `proxy` - 代理地址，如 `http://127.0.0.1:7890`
- `grok2api_*` - token 入池配置

## 输出文件

- `accounts_YYYYMMDD_HHMMSS.txt` - 成功账号（`email----password----sso` 格式）
- `mail_credentials.txt` - 邮箱凭证记录（调试用）

## 平台限制

- 仅支持 Windows（依赖 DrissionPage 的 Chromium 控制）
- Python 3.12/3.13 推荐
