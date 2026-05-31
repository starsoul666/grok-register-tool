# grok-register-tool

## QQ 交流群

如需交流使用经验、参数配置、排障思路，可加入：

- **QQ 群：1060714372**

---

一个面向 Windows 的 Grok 注册自动化工具，采用 `Tkinter + DrissionPage + curl_cffi` 实现可视化批量流程。

项目目标是把“打开注册页 -> 创建临时邮箱 -> 轮询验证码 -> 填写注册资料 -> 获取 sso”这一整套链路做成可配置、可并发、可落盘、可复现的本地 GUI 工具，方便长期稳定运行与排障。

## 当前版本特性（真实状态）

- GUI 可视化配置与运行日志（实时滚动）
- 支持邮箱提供商：
  - DuckMail
  - YYDS
  - Cloudflare Temp Mail（自定义 API Base / 鉴权模式 / 路径）
- 支持并发注册（线程数可调，带启动节流）
- 注册成功后自动提取并保存：
  - 邮箱地址
  - 注册密码
  - `sso` token
- 支持 grok2api 自动入池：
  - 本地 token 文件入池
  - 远端接口入池（可选）
- 保留详细调试日志，便于定位验证码、邮箱 API、浏览器连接等问题

> 说明：本仓库版本已移除 NSFW 自动开启模块与教程弹窗/教程按钮。

## 运行环境

- 系统：Windows
- Python：3.12 / 3.13（推荐）
- 浏览器：本地 Chromium 内核环境（由 DrissionPage 驱动）

## 安装

```bash
# 方式一：uv（推荐）
uv sync

# 方式二：pip
pip install DrissionPage curl_cffi
```

## 启动

```bash
# uv
uv run python3 grok_register_ttk.py

# pip
python grok_register_ttk.py
```

首次启动后，工具会自动在当前目录生成 `config.json`。点击「开始注册」时也会自动保存 GUI 中的配置。

## 配置详解

### 1. 邮箱服务商

| 配置项 | 说明 |
|--------|------|
| `email_provider` | 三选一：`cloudflare`（推荐）、`duckmail`、`yyds` |

#### Cloudflare（推荐）

使用自部署的 Cloudflare Temp Mail 服务。需要填写以下配置：

```json
{
  "email_provider": "cloudflare",
  "cloudflare_api_base": "https://your-domain.pages.dev",
  "cloudflare_api_key": "your-key-if-needed",
  "cloudflare_auth_mode": "x-admin-auth",
  "cloudflare_path_domains": "/api/domains",
  "cloudflare_path_accounts": "/admin/new_address",
  "cloudflare_path_token": "/api/token",
  "cloudflare_path_messages": "/api/mails",
  "defaultDomains": "example.com"
}
```

**各字段说明：**

- **`cloudflare_api_base`**：你的临时邮箱服务地址，末尾不带 `/`
  - 先访问你的邮箱服务，调用 `/open_api/settings` 或 `/api/settings` 接口查看 `api_base` 字段
  - 示例：`https://xxxx.pages.dev`

- **`cloudflare_api_key`**：鉴权密钥
  - 如果你的邮箱服务 `needAuth=false`，留空即可
  - 如果 `needAuth=true`，填 `admin_password` 或 `api_key` 对应的值

- **`cloudflare_auth_mode`**：鉴权模式
  - `x-admin-auth`（默认）：将 key 放在 `x-admin-auth` 请求头中
  - `none`：不鉴权

- **`cloudflare_path_*`**：四个 API 路径，必须与你的邮箱后端实际路由一致
  - 不同版本的 cloudflare_temp_email 路径不同，常见对照：

  | 配置项 | 新版路径 | 旧版路径 |
  |--------|----------|----------|
  | `cloudflare_path_domains` | `/api/domains` | `/domains` |
  | `cloudflare_path_accounts` | `/admin/new_address` | `/api/new_address` |
  | `cloudflare_path_token` | `/api/token` | `/token` |
  | `cloudflare_path_messages` | `/api/mails` | `/messages` |

- **`defaultDomains`**：注册时使用的邮箱域名
  - 单域名：`example.com`
  - 多域名逗号分隔，会自动轮换：`a.com,b.com,c.com`
  - 域名必须在你的邮箱服务中已配置，可通过 `domains` 接口查询可用域名

#### DuckMail

```json
{
  "email_provider": "duckmail",
  "duckmail_api_key": "your-duckmail-api-key"
}
```

#### YYDS

```json
{
  "email_provider": "yyds",
  "yyds_api_key": "your-api-key",
  "yyds_jwt": "your-jwt-token"
}
```

`yyds_api_key` 和 `yyds_jwt` 二选一即可。

### 2. 注册参数

| 配置项 | 说明 | 示例 |
|--------|------|------|
| `register_count` | 本轮注册总数量 | `1` |
| `register_threads` | 并发线程数，建议先 1-3 测试稳定后再提升 | `1` |
| `proxy` | 代理地址，留空为直连 | `http://127.0.0.1:7890` |
| `user_agent` | 浏览器 UA，一般无需修改 | - |

### 3. grok2api 入池（可选）

注册成功后自动将 SSO token 写入 [grok2api](https://github.com/your-grok2api) 池中。

#### 本地入池

```json
{
  "grok2api_auto_add_local": true,
  "grok2api_local_token_file": "/path/to/grok2api/data/token.json",
  "grok2api_pool_name": "ssoBasic"
}
```

- `grok2api_local_token_file`：grok2api 的 token.json 文件的绝对路径
- `grok2api_pool_name`：`ssoBasic`（普通池）或 `ssoSuper`（高级池）

#### 远端入池

```json
{
  "grok2api_auto_add_remote": true,
  "grok2api_remote_base": "http://your-grok2api:8080/admin/api",
  "grok2api_remote_app_key": "your-admin-key",
  "grok2api_pool_name": "ssoBasic"
}
```

- `grok2api_remote_base`：grok2api 的管理 API 地址，**必须包含 `/admin/api` 路径**
- `grok2api_remote_app_key`：grok2api 配置的 admin key

### 4. 完整配置示例

```json
{
  "email_provider": "cloudflare",
  "cloudflare_api_base": "https://xxxx.pages.dev",
  "cloudflare_api_key": "",
  "cloudflare_auth_mode": "x-admin-auth",
  "cloudflare_path_domains": "/api/domains",
  "cloudflare_path_accounts": "/admin/new_address",
  "cloudflare_path_token": "/api/token",
  "cloudflare_path_messages": "/api/mails",
  "defaultDomains": "example.com",
  "proxy": "",
  "register_count": 1,
  "register_threads": 1,
  "grok2api_auto_add_local": false,
  "grok2api_local_token_file": "",
  "grok2api_pool_name": "ssoBasic",
  "grok2api_auto_add_remote": false,
  "grok2api_remote_base": "",
  "grok2api_remote_app_key": "",
  "duckmail_api_key": "",
  "yyds_api_key": "",
  "yyds_jwt": ""
}
```

## 快速自检

首次使用建议：

1. 设置注册数量 = 1，并发线程 = 1
2. 点击「开始注册」，观察日志是否依次出现：
   - `已创建邮箱: xxx@你的域名`
   - `Cloudflare 本轮邮件数量: ...`
   - `从邮件中提取到验证码: ...`
3. 如果第一步就失败，优先检查 API Base、CF 路径、鉴权模式

## 输出文件

- `accounts_YYYYMMDD_HHMMSS.txt`
  - 成功账号导出（`email----password----sso`）
- `mail_credentials.txt`
  - 邮箱地址与邮箱侧 token/JWT 记录（用于验证码链路排查）

## 常见问题

| 现象 | 排查方向 |
|------|----------|
| 验证码收不到 | 检查 `defaultDomains` 是否在邮箱服务的可用域名列表中；检查 CF 路径是否正确 |
| HTTP 401 | `cloudflare_api_key` 或 `cloudflare_auth_mode` 不匹配，确认邮箱服务的鉴权方式 |
| HTTP 404 | CF 路径配置错误，确认邮箱后端实际路由（新旧版本路径不同） |
| 浏览器连接失败 | 降低并发线程数，或重启工具重试 |
| 代理超时 | 先去掉代理直连测试，确认是否为代理问题 |
| grok2api 入池 404 | 远端 Base 需包含 `/admin/api`，如 `http://host:8080/admin/api` |

## 安全与分发建议

发布给他人前请清理以下敏感文件：

- `config.json`
- `accounts_*.txt`
- `mail_credentials.txt`

推荐分发时仅保留示例配置（如 `config.example.json`）。

