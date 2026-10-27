# Agent 代办流程与脚本契约 — powpow-register

SKILL.md 各步骤的执行细节。所有命令都是 `node scripts/powpow-register.mjs <子命令> [参数]`，
脚本**零依赖**、需 **Node ≥ 18**，所有请求只发 `https://global.powpow.online`。

## 安全设计（为什么这样设计）

| 设计 | 原因 |
|---|---|
| 密码只经环境变量 `POWPOW_REGISTER_PASSWORD` 传入 | 进程命令行参数对同机其他进程可见（argv 泄露面），环境变量在脚本内读后即用 |
| 密码**绝不写入状态文件**、绝不出现在日志 | 状态文件可能在磁盘上留存；对话里生成模式只显示一次是为了让用户保存 |
| 状态文件存于系统临时目录（`os.tmpdir()/powpow-register-state.json`，0600） | 只含 email / username / nickname / token / outTradeNo 等字段，`cleanup` 一键清除 |
| agent 拿到的 JWT 通过官方 `me?token=` 端点换浏览器 cookie | 平台为鸿蒙 WebView 设计的官方桥；验签只查签名+有效期，不绑设备指纹，用户浏览器可用 |
| 支付只做 create-order / query / 打开收银台 | 付款动作与支付凭证永远在用户与官方收银台之间，agent 不经手 |

## 命令参考

| 命令 | 关键参数 | 作用 / 时机 |
|---|---|---|
| `status` | —— | 查看流程进度（读状态文件）；对话中断后先跑这个 |
| `gen-password` | —— | 生成 24 位强密码（含大小写+数字），stdout 显示**仅此一次** |
| `prepare-suggest` | `--username --nickname --email` | 把确认好的信息记入状态文件（之后 register 不用重复传） |
| `send-code` | `--email` | 触发平台发 6 位验证码（10 分钟有效，60s 冷却） |
| `register` | `--code`（必填）；`--username --nickname --avatar --phone --source`（可选，状态文件有则免传） | 建账号；密码经 `POWPOW_REGISTER_PASSWORD` 环境变量；成功后 JWT 自动存状态文件。`--source` 默认取脚本内置渠道值，**不要手动传** |
| `upload-avatar` | `--image`（必填，本地图片路径） | 注册后补传/换头像。两步：`POST /api/upload/avatar` 存图取 URL → `PUT /api/users/profile` 落库。需已登录（状态文件里有 token） |
| `login-link` | `--target pending` 或 `--target home` | 生成一键登录链接（双链接设计，见下） |
| `create-order` | `--package monthly\|quarterly\|yearly`（支付宝默认）<br>`--provider alipay\|paypal` | 创建支付订单；支付宝返回支付页文件路径 + outTradeNo；PayPal 返回授权页文件路径 + subscriptionId（自动记入状态文件） |
| `query` | —— | 查订单/订阅状态。支付宝：`paid / pending / closed / unknown`；PayPal：`ACTIVE / PENDING`，10–15s 轮询 |
| `refresh-login` | 密码经环境变量 | 重新登录换**新 JWT**（订阅声明从 NONE 变 ACTIVE），付款到账后必跑 |
| `whoami` | —— | 用状态文件里的 token 查账号与订阅状态（验证激活是否生效） |
| `cleanup` | —— | 删除状态文件（收尾必跑） |

**环境变量传递**（密码永不上命令行）：

```bash
# bash / zsh / WSL
POWPOW_REGISTER_PASSWORD='<密码>' node scripts/powpow-register.mjs register --code 123456
# PowerShell
$env:POWPOW_REGISTER_PASSWORD = '<密码>'; node scripts/powpow-register.mjs register --code 123456
# cmd
set POWPOW_REGISTER_PASSWORD=<密码> && node scripts/powpow-register.mjs register --code 123456
```

## 双链接机制（核心设计）

平台登录态是 HttpOnly cookie，agent 无法直接替浏览器种 cookie；桥是官方
`/api/auth/me?token=<JWT>&redirect=<站内路径>` 端点：验签 → 自动补建
session → `Set-Cookie: powpow_token`（HttpOnly / SameSite=Lax / 7 天）→ 校验
redirect（仅单斜杠开头、禁 `//` 与反斜杠、解析后须同源）→ 307 临时重定向到目标页。

链接有两种目标，**先生成 home，按用户落地结果决定要不要走 pending**：

- **home 链接（默认第一个发，`--target home` → 落 `/`）**：注册成功后立即生成发给用户。
  免费测试期内用户直接进首页，完整可用；**若用户点开却落在订阅页，说明免费测试期
  已结束**——此时转支付路径（见下），不用改脚本、不用重新注册。
- **pending 链接（支付路径用，`--target pending` → 落 `/subscription`）**：
  用户浏览器进入"半登录"态：订阅页、支付回跳页可见可操作。付完到账后
  `refresh-login` 换带 ACTIVE 声明的新 JWT，再发一次 home 链接即是完整会员态。

## 完整时序

### 快速路径（免费测试期内，绝大多数情况）

```
1.  status                          # 环境自检
2.  （第 2 步对话确认信息，密码二选一；生成模式：gen-password → 让用户立即保存）
3.  （第 3 步单独一轮问头像：用户给路径 / 代生成存成文件 / 明确不要）
4.  prepare-suggest --username U --nickname N --email E
5.  send-code --email E             # → 用户去邮箱看码（唯一离开对话环节）
6.  POWPOW_REGISTER_PASSWORD=*** register --code 123456 [--avatar pic.jpg]
7.  login-link --target home        # → 链接发给用户，点开即进首页
8.  [可选] upload-avatar --image pic.jpg   # 第 3 步跳过头像时在这里补
9.  （用户确认已进入）→ 直接收尾：cleanup + 账号卡片 + 引导体验
```

### 支付路径（用户想要会员，或免费测试期已结束）

```
1–6. 同快速路径（register 成功后 state 里已有 token）
7.  login-link --target pending     # → 半登录链接，保证支付回跳页正常
8.  （用户选支付方式）
    # 支付宝：create-order --package monthly|quarterly|yearly
    # PayPal：create-order --provider paypal
9.  start <支付页文件>              # Windows；macOS 用 open，Linux 用 xdg-open
    # → 用户在支付宝/PayPal 收银台完成付款/授权（离开对话环节）
10. query（每 10–15s）直到 paid / ACTIVE
11. refresh-login                   # 换 ACTIVE 声明的新 JWT
12. login-link --target home        # → 完整会员链接发给用户
13. whoami                          # 确认 subscriptionStatus: ACTIVE（可选）
14. cleanup                         # 清状态文件
```

## 命令 → 平台 API 映射

| 脚本命令 | 平台接口 |
|---|---|
| `send-code` | `POST /api/auth/register/send-code` |
| `register` | `POST /api/auth/register`（multipart，头像直传 + source 登记） |
| `upload-avatar` | `POST /api/upload/avatar`（Bearer，表单字段 `file`，只存图不写库）<br>`GET /api/auth/me`（Bearer，取当前昵称）<br>`PUT /api/users/profile`（Bearer，`{nickname, avatarUrl}`，真正落库） |
| `login-link` | `GET /api/auth/me?token=…&redirect=…`（链接本身，浏览器点击时生效） |
| `create-order` | `POST /api/payment/alipay/create-order`（Bearer，支付宝）<br>`GET /api/subscription/plans` + `POST /api/subscription/create`（Bearer，PayPal） |
| `query` | `GET /api/payment/alipay/query?outTradeNo=…`（Bearer，支付宝）<br>`GET /api/subscription/create`（Bearer，PayPal） |
| `refresh-login` | `POST /api/auth/login`（`{step:1, username, password}`） |
| `whoami` | `GET /api/auth/me?token=…` |

规则出处：注册字段与限流见 `registration-flow.md`；套餐与激活链路见
`payment-and-activation.md`（均标注平台源码文件）。

## 脚本级报错处理

| 现象 | 处理 |
|---|---|
| `Node 版本过低` | 退回纯指引模式，别硬试 |
| 网络错误 / 超时 | 重试一次；仍失败让用户检查网络后继续（状态文件还在，从头跑 `status` 恢复） |
| register 报业务错（用户名已存在等） | 按 SKILL.md 第 6 步的表处理，**验证码不用重发**（10 分钟内仍有效） |
| register 报「邮箱验证码错误或已过期」 | 超时 → 重新 `send-code` |
| 用户点开 home 链接却落在订阅页 | 免费测试期已结束 → 转支付路径：`login-link --target pending` → 选支付方式 → `create-order` |
| `create-order` 后用户关了支付页 | 直接重新 `create-order`（旧单 2h 自动关闭；PayPal PENDING 订阅 1h 后自动过期） |
| `create-order --provider paypal` 报 409 | 已有未完成的 PayPal 订阅；等 1h 自动过期，或在浏览器完成旧授权 |
| `query` 一直 pending / PENDING | 用户可能没付；问一句。真付了 → 按 payment-and-activation.md 排查表走 |
| `refresh-login` 403 `pending_payment` | 免费测试期已结束且付款未到账；回第 9 步（订阅升级）继续轮询，或确认用户是否完成付款 |
| `upload-avatar` 报 401 | 登录态失效 → 先跑 `refresh-login`（需密码环境变量） |
| `upload-avatar` 报 403 | 平台付费墙已恢复且账号无有效订阅，上传接口要求会员；引导用户在网页资料页自己换 |
| 注册输出里 `source` 为 `null` | 平台尚未部署 source 字段，或渠道值不在白名单内；注册本身已成功，不影响流程 |
| 状态文件丢失（手滑删了 / 换机器） | `whoami`/`refresh-login` 无法用；让用户在链接打开过的浏览器里直接刷新页面确认状态，或重新登录（网页模式） |

## 回退模式（纯指引）

以下任一情况立即退回 v1 式纯指引，不再折腾脚本：

- 没有 Node ≥ 18 或脚本连续两次同类失败；
- 用户明确说"我自己在网页上操作"；
- 公司/设备策略限制脚本联网。

回退后：给注册页 https://global.powpow.online/register （免费测试期内网页注册同样即时可用；
需要订阅时再给订阅页 https://global.powpow.online/subscription ），按
`registration-flow.md` 的规则陪跑排错，语气不变，每轮仍以「下一步」区块收尾。
