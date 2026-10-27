# 注册流程详解 — powpow-register

供排查问题时查阅。所有规则均来自平台源码（Next.js 前后端一致性校验），标注了出处。
平台更新时以线上实际行为为准。

## 入口

- 注册页：https://global.powpow.online/register
- 登录页：https://global.powpow.online/login
- 订阅页：https://global.powpow.online/subscription
- 若从分享链接等带 `?next=...` 进入，注册+支付完成后会自动跳回原页面。

## 表单字段与规则

| 字段 | 必填 | 规则 | 源码出处 |
|---|---|---|---|
| 头像 | 可选 | 图片 ≤5MB，不支持 SVG；服务端自动压缩（>100KB 时压到 400×400 / 质量 90） | `src/app/api/auth/register/route.ts` |
| 用户名 | 必填 | 2~50 字符，trim 后转**小写**存储（Tom 与 tom 视为同一用户名，注册端与登录端均大小写不敏感） | 同上（zod schema） |
| 密码 | 必填 | 8~100 位，必须同时包含大写字母、小写字母、数字 | 同上 |
| 确认密码 | 必填 | 与密码一致（仅前端校验） | `src/app/(public)/register/page.tsx` |
| 昵称 | 可选 | ≤100 字符；不填默认取用户名 | register route |
| 邮箱 | **必填** | 合法邮箱格式，≤255 字符；一个邮箱只能注册一个账号 | register route |
| 邮箱验证码 | **必填** | 6 位数字；**10 分钟有效**；验证成功即作废；重发会把旧码标记为已使用 | `src/lib/email-verification.ts` |
| 手机号 | 可选 | 大陆手机号 `1[3-9]` 开头共 11 位；一个手机号只能绑一个账号 | register route |
| 来源 source | **脚本自动带** | 只接受 `openclaw` / `workbuddy` 两个值，写入 `users.oauth_provider` 供渠道统计；其余值一律静默丢弃（不报错、不阻断注册）；不传则留空 | register route（`ALLOWED_SOURCES` / `normalizeSource`） |

页面另提供三个第三方登录按钮：**Google、Watcha、华为账号**（`GoogleSignInButton` /
`WatchaLoginButton` / `HuaweiLoginButton`）。第三方账号没有本地密码，之后用密码登录会
得到「该账号未设置密码，请使用其他方式登录」。

## 头像上传的三条路径（别用错）

平台有三个跟头像相关的接口，可用性差别很大，踩过一次就记住了：

| 接口 | 表单字段 | 鉴权 | 写库 | 可用性 |
|---|---|---|---|---|
| `POST /api/auth/register` | `avatar` | 公开（`/api/auth/` 在 proxy 白名单里） | ✅ 直接写入 `users.avatarUrl` | ✅ **注册时直传，唯一对新用户可用的入口** |
| `POST /api/upload/avatar` | `file` | 需 Bearer JWT；付费墙恢复后还需有效订阅 | ❌ **只存图返回 URL，不写库** | ✅ 注册后补传/换头像，须再 `PUT /api/users/profile` |
| `POST /api/upload/temp-avatar` | `file` | 路由本身无鉴权 | ❌ 不写库 | ❌ **已失效**：被 `src/proxy.ts` 付费墙统一拦成 401 |

要点：

- `/api/upload/avatar` 拿到 URL **不等于**头像生效，必须再调 `PUT /api/users/profile`
  提交 `{nickname, avatarUrl}`（昵称规则 1–20 字；省略该字段则保持原值）。
  脚本的 `upload-avatar` 子命令已经把这两步封好了。
- `temp-avatar` 是早期为「注册页匿名上传」留的临时接口，后来注册改成随表单直传，
  这个接口就被付费墙挡死了——不要再拿它做匿名上传。
- 昵称超过 20 字时脚本会**只提交 avatarUrl**，避免把用户昵称改坏。

## 限流与防爆破参数

| 行为 | 限制 | 出处 |
|---|---|---|
| 注册提交 | **3 次/分钟/IP**，超限 429 | `src/lib/rate-limit.ts` `REGISTER_RATE_LIMIT` |
| 发送验证码 | 同 IP 1 次/分钟 **且** 同邮箱 1 次/分钟，超限 429；前端按钮 60 秒倒计时 | `src/app/api/auth/register/send-code/route.ts` |
| 验证码校验 | 每邮箱 10 分钟内最多尝试 **5 次**，达到上限即使码对也拒绝，需重新发码 | `src/lib/email-verification.ts` |
| 忘记密码发码 | 1 次/分钟/IP | `src/app/api/auth/forgot-password/route.ts` |
| 登录 | 5 次/分钟/IP；同一账号连续失败 **5 次锁定 30 分钟**（423），期间剩余次数递减提示 | `src/app/api/auth/login/route.ts`、`rate-limit.ts` |

**隐私保护行为**：发验证码 / 忘记密码时，若邮箱已被注册或不存在，接口都统一返回
成功文案，不泄露邮箱是否已注册。

## agent 代办模式与网页注册的等价性

skill 脚本调用的是**同一批公开 API**（`/api/auth/register`、`send-code`、`login`、
`/api/auth/me`），与注册页表单提交走完全相同的校验、限流与报错路径——本文件的所
有规则对两种模式一视同仁。脚本时序与命令契约见 `references/agent-flow.md`。

## 注册成功后的状态流转

> **免费测试期（2026-10-18 起，以平台实际行为为准）**：新注册账号落地后**可直接使用全部功能**
> （信息流、地图、数字人、打卡等），无需付费；未订阅账号访问内容页不再被挡。本节的
> 「pending 拦截」描述的是测试期结束后的常规状态，届时以线上实际行为为准。

1. 用户以 `accountStatus: 'pending'` 落库，初始 **3 个徽章**（register route）。
2. 服务端同时种下两个 HttpOnly cookie：JWT（可直接访问各页面）与
   pendingUserToken（订阅流程身份凭证）。
3. 常规状态（测试期结束/未开放免费期）下，注册页检测到 `pending_payment` → 跳转 `/subscription`
   （保留 `next` 回跳参数）；agent 代办模式下用 `me?token=…&redirect=/subscription` 达到同样效果。
   **免费测试期内这一步不触发**，注册后直接进首页即可使用。
4. 支付完成 → `accountStatus` 变为 `active`（**支付宝走 notify 服务端自动激活**；
   PayPal 走 Webhook 自动激活；微信扫码人工确认；华为 IAP 走 `activate-huawei`）。
   详见 `payment-and-activation.md`。

常规状态下，未激活（pending）用户调用核心 API（信息流、地图、数字人对话、打卡等）会 403；
分享帖子链接除外（保持公开可访问）。免费测试期内该拦截暂停，全部放开。

## 常见报错排查表

| 现象 | 原因 | 怎么办 |
|---|---|---|
| 「用户名已存在」 | 用户名被占用（不分大小写） | 换一个用户名 |
| 「邮箱已被使用」 | 邮箱已注册过 | 换邮箱；或改走登录 +「忘记密码」找回旧账号 |
| 「邮箱验证码错误或已过期」 | 码输错 / 超 10 分钟 / 试错超 5 次被拒 | 点「重新发送」，用**最新一封**邮件的验证码 |
| 收不到验证码 | 60 秒限流；或进了垃圾箱/广告邮件 | 等倒计时结束再发；翻垃圾邮件文件夹 |
| 「手机号已被使用」 | 手机号已绑其他账号 | 删掉手机号（选填）或换号 |
| 「注册请求过于频繁」/ 429 | 注册 3 次/分钟/IP 触顶 | 等几分钟再试 |
| 409「用户名或邮箱已被注册」 | 提交瞬间与他人撞名（并发兜底） | 更换用户名或邮箱后重试 |
| 「图片大小不能超过5MB」/「不支持 SVG」 | 头像不合规 | 换图或跳过头像 |
| 注册成功但登录 403「请完成订阅以激活账户」 | 账号仍是 pending，且平台不在免费测试期（未付款前的预期状态） | 免费测试期内不会出现；若出现说明测试期已结束——先完成订阅；付款后仍 403：说明用了旧 JWT，脚本模式下先 `refresh-login` 换新登录态 |
| 用户点开登录链接落在订阅页（而非首页） | 免费测试期已结束，平台恢复付费墙 | 按 SKILL.md 第 9 步走订阅支付流程；订完 `refresh-login` 再发首页链接 |
| 注册输出里 `source` 为 `null` | 平台还没部署 source 字段，或渠道值不在白名单 | 注册本身已成功，不影响使用；等平台部署即可 |
| `upload-avatar` 401「未登录」 | 状态文件里 token 失效 | 先跑 `refresh-login`（要密码环境变量） |
| `upload-avatar` 403 | 付费墙恢复后上传接口要求有效订阅 | 引导用户在网页「编辑资料」自己换头像 |
| 登录 423「账户已锁定」 | 密码连错 5 次 | 等 30 分钟 |
| 登录 401「还剩 N 次尝试机会」 | 密码错误，接近锁定 | 想清楚再试，或直接走「忘记密码」 |
| 「该账号未设置密码，请使用其他方式登录」 | 第三方（Google/Watcha/华为）账号 | 用原来的第三方按钮登录 |
| 「该账号已注销」 | 账号已 deactivate | 需要重新注册 |

## 登录小知识（顺带排查用）

登录支持**用户名 / 邮箱 / 手机号**三合一输入（三者任一均可作为账号名）。
忘记密码：登录页 →「忘记密码」→ 输入注册邮箱收 6 位验证码 → 重置。
