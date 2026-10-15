

# 次元星域登录授权中心

<img src="https://img.107211.xyz/file/img/yP9XFpEh.webp" width="450" height="124.5" alt="绘里酱" align="right" style="max-width:45%;height:auto;margin:8px 0 12px 24px;border-radius:12px" />

> 以网页为载体，实现 OAuth 2.0 第三方登录与邮箱验证码登录
> 
> 内置 10 家第三方登录方式，含设备 / 系统 / 浏览器 / 归属地检测
> 
> 自研 SMTP 发信（基于 cloudflare:sockets）
>
> 支持主流邮件通道(QQ/163/Gmail/Resend)，主题与 HTML 均可自定义
> 
> 内含管理面板，配置、发信、授权全部在页面完成，无需改代码
> 
> 支持混淆前端内联脚本、AES-GCM 加密存储敏感字段

> ![Cloudflare Workers](https://img.shields.io/badge/Cloudflare-Workers-f38020?logo=cloudflare&logoColor=white)
> ![Deploy wrangler](https://img.shields.io/badge/deploy-wrangler-f38020?logo=cloudflare&logoColor=white)
> ![License AGPL-3.0](https://img.shields.io/badge/license-AGPL--3.0-lightgrey)
> ![Code worker.js](https://img.shields.io/badge/code-worker.js%20%C2%B7%20single%20file-7c3aed)

---

## 目录结构

```
oauth/
├── worker.js       # 全部业务代码：页面、路由、OAuth、邮箱验证、发信、管理面板
├── wrangler.toml   # Cloudflare Workers 部署配置（名称、入口、路由、KV 绑定、变量）
├── LICENSE         # AGPL-3.0
├── README.md
└── .wrangler/      # wrangler 自动生成的本地缓存与调试状态（不要提交）
```

## 部署

仨方法：

- 一键部署
- Wrangler 部署
- Worker 部署

### 一键部署
点击以下按钮自动完成部署
[![Deploy to Cloudflare](https://deploy.workers.cloudflare.com/button)](https://deploy.workers.cloudflare.com/?url=https://github.com/nino-natsume/Oauth)

### Wrangler部署

前置条件：Cloudflare 账号；Node.js（仅用于运行 wrangler CLI，建议 ≥ 18）。

```bash
# 1. 安装并登录 wrangler
npm i -g wrangler        # 或直接使用 npx wrangler
wrangler login

# 2. 创建配置 / 验证码持久化用的 KV，把返回的 id 填进 wrangler.toml 的 [[kv_namespaces]]
wrangler kv namespace create STORE

# 3. 部署
wrangler deploy           # 本地调试：wrangler dev
```

- KV 绑定名必须是 `STORE`（代码按 `env.STORE` 读取）。未绑定时配置与验证码只存内存，Worker 重启即丢失，管理面板会显示「仅内存」警告。
- （可选）登录历史：再建一个 KV 并在 `wrangler.toml` 中以 `binding = "LOG_KV"` 绑定，登录记录会写入其中，供 `/api/logs`、`/api/status` 读取。
- 第三方登录的 Client ID / Secret 用 secret 下发（部署时会自动保留，不必每次重填）：

```bash
wrangler secret put GITHUB_CLIENT_SECRET
wrangler secret put GOOGLE_CLIENT_SECRET
wrangler secret put MICROSOFT_CLIENT_SECRET
wrangler secret put QQ_APP_SECRET
# ……其余见下方环境变量表
```

- `wrangler.toml` 中已配置自定义域名路由 `oauth.107211.xyz/*`。代码里还内置了 IP 查询页，若要启用需自行在 `routes` 中追加该域名。

### Worker部署
将[worker.js](worker.js)内的代码复制进 Worker 内，待部署完成后在变量处配置必要变量即可

## 环境变量与绑定

**KV 绑定**

| 绑定名 | 必需 | 用途 |
| --- | --- | --- |
| `STORE` | 是 | 站点配置、验证码、限流、视图 cookie、intent 等 |
| `LOG_KV` | 否 | OAuth 登录历史 |

**`wrangler.toml` → `[vars]`**

| 变量 | 说明 |
| --- | --- |
| `SECRET` | 加密密钥：开启「加密敏感字段」时用于 AES-GCM 加密；同时作为 OAuth state / 会话签名密钥（缺省回退到配置里的签名密钥）。建议改为 `wrangler secret put SECRET` |
| `HOME_KEY` | 主页隐藏入口密钥：输入其值可进入管理面板 |
| `BACK` | 主页隐藏入口：输入其值切回介绍版界面 |
| `MICROSOFT_TENANT` | Microsoft 租户，默认 `common` |
| `QQ_APP_ID` | QQ 应用 ID（非敏感，放 vars） |

**第三方登录密钥**（配置了 ID 的登录方式才会出现在 `/oauth` 页面，Secret 一律用 `wrangler secret put` 下发）

需注意：以下：`wechat`、`weibo`、`qq`均需完成人工审核才可开 Oauth，未审核或审核未过在登录时会报错

| 登录方式 | ID 变量 | Secret 变量 | Oauth 申请站点 |
| --- | --- | --- | --- |
| `google` | `GOOGLE_CLIENT_ID` | `GOOGLE_CLIENT_SECRET` | [Google Cloud Console](https://console.cloud.google.com/apis/credentials)：「API 和服务 → 凭据」创建 OAuth 客户端 ID，类型选 **Web 应用** |
| `github` | `GITHUB_CLIENT_ID` | `GITHUB_CLIENT_SECRET` | [GitHub → Settings → Developer settings → OAuth Apps](https://github.com/settings/developers/apps)：新建 **OAuth App** |
| `microsoft` | `MICROSOFT_CLIENT_ID` | `MICROSOFT_CLIENT_SECRET` | [Microsoft Entra 管理中心](https://entra.microsoft.com/) →「应用注册」：账户类型按需含「个人 Microsoft 账户」，API 权限勾选 openid / email / profile / User.Read |
| `microsoft_msa` | `MICROSOFT_MSA_CLIENT_ID` | `MICROSOFT_MSA_CLIENT_SECRET` | 同上「应用注册」：账户类型**仅**选「仅限个人 Microsoft 账户」（对应 consumers 租户） |
| `X(twitter)` | `TWITTER_CLIENT_ID` | `TWITTER_CLIENT_SECRET` | [X Developer Portal](https://developer.x.com/en/portal/dashboard)：创建 App，User authentication settings 中 Type 选 **Web App**，开启 OAuth 2.0 |
| `gitlab` | `GITLAB_CLIENT_ID` | `GITLAB_CLIENT_SECRET` | [GitLab → 用户设置 → 应用程序](https://gitlab.com/-/user_settings/applications)：新建应用，Scope 勾选 `read_user` |
| `discord` | `DISCORD_CLIENT_ID` | `DISCORD_CLIENT_SECRET` | [Discord Developer Portal](https://discord.com/developers/applications)：新建 Application →「OAuth2」页获取 Client ID / Secret |
| `wechat` | `WECHAT_APP_ID` | `WECHAT_APP_SECRET` | [微信开放平台](https://open.weixin.qq.com/)：登录后「网站接入 → 网站应用」创建并交由审核 |
| `weibo` | `WEIBO_CLIENT_ID` | `WEIBO_CLIENT_SECRET` | [微博开放平台](https://open.weibo.com/)：「网站接入」申请成为开发者并创建网站应用 |
| `qq` | `QQ_APP_ID` | `QQ_APP_SECRET` | [QQ 互联](https://connect.qq.com/)：「网站应用 → 创建应用」并完成审核 |

各平台的回调地址统一为：`https://<你的域名>/callback/<登录方式 id>`，如 `https://oauth.ciallo.cc/callback/github`

## 用法

### 页面

| 路径 | 说明 |
| --- | --- |
| `GET /` | 主页。默认介绍版；带 `redirect` / `ref` 参数时切换为「Oauth / Email」两按钮对外登录视图 |
| `GET /oauth?ref=<url>` | 登录方式列表（仅列出已配置的提供商） |
| `GET /email?redirect=<url>` | 邮箱验证码页（`/mail` 等价），其余查询参数会随回跳透传 |
| `GET /auth/<provider>` | 发起第三方登录（支持 PKCE 的平台自动带上 `code_challenge`） |
| `GET /callback/<provider>` | 第三方平台回跳，成功后写入 24 小时会话 cookie 并跳回 `ref` |
| `GET /XXX(<HOME_KEY> X 某个数)` | 管理面板（使用`HOME_KEY` 的值计算得出路由） |
| `GET /me` / `GET /logout` | 查询当前登录态 / 退出登录 |
| `GET /favicon.ico` | 204 |

### 主页隐藏入口

在主页（介绍版 / 自助版）不点击任何输入框，直接用键盘敲入 `HOME_KEY` 的值，停顿 600ms 或按回车 → 跳转管理面板；敲入 `BACK` 的值 → 原地切换介绍版。命中后写入签名 cookie `mg_view`（有效期 7 天），视图切换状态与路径无关；未命中则静默清空，并有按 IP 的尝试限流（10 分钟 20 次）。

### 管理面板

三个页签：

- **授权**：站点白名单表格，添加 / 移除授权站点（`example.com`、`https://example.com`、`*.example.com`）。
- **SMTP & 人机验证**：发信通道（SMTP / Resend）、验证码规则（位数、有效期、重发间隔、尝试次数、ticket 有效期）、页面与邮件（站点名、主题、HTML 正文）、人机验证（算术题 / Cloudflare Turnstile）、签名密钥；附「发送测试邮件」。
- **其他**：自定义 API 文本配置、混淆内联脚本开关、加密敏感字段开关、清理缓存 / 邮件记录。

### 来源站点接入

使用统一入口时，一定要确保格式正确，两种入口，任选其一：

1. **统一入口**：`GET https://<授权中心>/api/auth?redirect=https://your-site/callback` —— 直接渲染对外登录页，用户选 Oauth 或 Email，成功后携登录结果跳回 `redirect`。
2. **分别指定**：`/oauth?ref=<url>`（第三方登录）或 `/email?redirect=<url>`（邮箱验证码）。

邮箱流程会把 `redirect` 上的自定义参数（如 `state`）原样带回；OAuth 流程回跳只附加登录结果参数。

回跳参数：

| 参数 | 说明 |
| --- | --- |
| `oauth_success=1` | 登录 / 验证成功标志 |
| `provider` | `email` 或提供商 id |
| `email` | 邮箱（邮箱流程） |
| `ticket` / `ts` | 签名凭据与签发时间戳 |
| `username` / `name` / `avatar` | 第三方账号资料（OAuth 流程） |

服务端校验凭据（无需任何密钥）：

```
GET https://<授权中心>/api/check?ticket=<ticket>
→ {"ok":true,"email":"a@b.com","ts":"1712345678901","exp":"1712345738901"}

```

ticket 格式为 `v1.<base64url(email)>.<ts>.<exp>.<签名>`，有效期由「返回凭据有效期」配置（默认 600 秒）。

## API 列表

### 对外开放（无需登录态）

| 方法 | 路径 | 参数 / 说明 |
| --- | --- | --- |
| GET | `/api/auth` | `redirect=<url>/callback[&自定义参数]` → 对外登录页；`back=<url>` → 302 回跳并透传其余参数 |
| GET | `/api/check` | `ticket=` → 校验凭据，返回 `email` / `ts` / `exp`（无 CORS 头，建议服务端调用） |
| GET | `/api/captcha` | 取算术验证题：`{ok,q,exp,sig}` |
| POST | `/api/send` | `{email, intent, captcha{ans,exp,sig} \| turnstile}` → 发送验证码，返回 `{ok,gap,ttl}` |
| POST | `/api/verify` | `{email, code, intent}` → 校验通过返回 `{ok,email,ticket,url}`；`url` 为空表示独立模式 |
| GET | `/api/platform` | UA + Cloudflare 边缘信息解析结果 |
| GET | `/api/providers` | 已配置的登录方式列表 |
| GET | `/api/status` | 配置体检：`SECRET`、KV、各平台 ID / Secret 是否齐全 |
| GET | `/api/logs` | OAuth 登录历史（来自 `LOG_KV`，未绑定时为空） |
| GET | `/me` | 会话信息，未登录返回 401 `{loggedIn:false}` |
| GET | `/logout` | 清除会话 cookie 并 302 到 `/` |
| POST | `/api/gate` | `{key}` 主页隐藏入口校验，命中返回 `{ok,goto}` 或 `{ok,view}` |

> `intent` 由邮箱页面渲染时签发（页面内隐藏域），绑定 redirect 与来源白名单，直接调 API 无法绕过站点授权。`/api/platform`、`/api/providers`、`/api/status`、`/api/logs`、`/me` 带 `Access-Control-Allow-Origin: *`。

### 需管理授权（cookie `mg_view=admin`）

| 方法 | 路径 | 说明 |
| --- | --- | --- |
| GET | `/XXX(<HOME_KEY> X 某个数)` | 管理面板页 |
| POST | `/admin/save` | 保存面板表单中的业务配置（密码 / API Key 等敏感项留空 = 保持不变） |
| POST | `/admin/test` | `{to}` 发送测试邮件 |
| POST | `/admin/clear` | `{target: cache \| logs}` 清理运行时数据 |
| GET | `/api/sites` | 授权站点列表 |
| POST | `/api/sites/add` | `{site}` 添加授权站点 |
| POST | `/api/sites/remove` | `{site}` 移除授权站点 |

### IP 查询

| 路径 | 说明 |
| --- | --- |
| `GET /`、`/index.html` | 显示访问者 IP、设备、系统、浏览器、地区、时区、数据中心 |
| `GET /?raw=1` | 或 `Accept: text/plain` → 纯文本返回 IP |
| 其余路径 | 404 |

## 安全说明

- **站点白名单**：授权列表为空 = 对所有站点开放；设置任一站点后：未授权来源访问任意登录入口都会返回 403 提示页。
- **签名贯穿全链路**：intent、ticket、视图 cookie、OAuth state / 会话均为 HMAC-SHA256 签名，密钥来自 `SECRET` 或配置中的签名密钥。
- **验证码**：只存 SHA-256 哈希、一次性使用、默认最多试 5 次、重发间隔 60 秒、每 IP 每小时 20 封。
- **人机验证**：默认内置算术题；配置 Turnstile Site Key + Secret 后自动切换。
- **代码保护**：混淆开关会将返回页面的内联脚本做 XOR + base64 处理；加密开关会用 AES-GCM 加密 SMTP 密码、API Key、签名密钥等敏感字段（需 `SECRET`）。
- **发信限制**：Cloudflare Workers 禁止 25 端口出站，SMTP 请使用 465（SSL / 隐式 TLS）或 587（STARTTLS）。

## 许可证

AGPL-3.0，详见 [LICENSE](LICENSE)。
