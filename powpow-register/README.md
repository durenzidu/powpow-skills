# powpow-register

PowPow 泡泡注册助手（OpenClaw 版）：带一个新用户从「不知道泡泡是什么」走到
「一键登录、开始玩」——产品介绍（含界面截图）、agent 代办注册（含头像上传）、
一键登录、可选的支付宝/PayPal 订阅下单与到账跟踪、报错排查。

**v2.0.0 起为 agent 代办模式，v2.2.0 起适配平台免费测试期，v2.3.4 起头像独立成步骤
并支持注册后补传**：核心脚本
`scripts/powpow-register.mjs` 零依赖（Node ≥ 18），用户全程不脱离对话，只有一次
暂时离开：去邮箱看验证码。当前平台免费测试期内注册后即可使用全部功能，订阅支付
是可选升级；若测试期结束用户落地到订阅页，流程自动转支付路径。没有 Node 或用户想
自己动手时，自动回退纯指引模式。

## 安装

```
openclaw skills install @durenzidu/powpow-register
```

## 功能

1. **产品介绍**：~120 字开场 + 随包内置的界面截图（`assets/screenshots/` 下 6 张：
   地图页、帖子详情页等，按场景选发 1–3 张；无需外部图床、离线可用）。
2. **代办注册**：agent 主动建议用户名/昵称、密码二选一（用户提供或 agent 生成）、
   验证码发送与提交、常见报错自动处理（占用/过期/限流）；注册请求自动带上渠道来源
   （`source=openclaw` / `workbuddy`），供平台统计各渠道注册量。
3. **头像（独立步骤）**：单独一轮主动询问——用户给本地图片就随注册直传（≤5MB 非 SVG），
   没有就主动提议代生成一张；注册时跳过的，之后可用 `upload-avatar` 补传/更换
   （脚本封装了「上传存图 + 写回资料」两步）。
4. **一键登录（双链接）**：注册成功即给直达首页的登录链接（免费测试期内点开即用）；
   若落在订阅页（测试期结束）或用户主动要升级，再走订阅流程，付款到账后换新登录态
   发完整会员链接（走平台官方 `me?token=` 桥，自动种 HttpOnly cookie）。
5. **订阅支付（可选升级）**：支付宝（月 ¥6.99 / 季 ¥19.99 / 年 ¥69.99，一次性买断
   不自动续费）与 PayPal（Premium Monthly 约 $1/月，自动续费可随时取消）均可
   agent 代办下单 + 轮询到账；微信扫码/华为 IAP 指路。
6. **报错排查**：收不到验证码、用户名/邮箱被占用、登录落地页异常、账户锁定等
   常见问题，全部来自平台源码并标注出处。

## 文件结构

```
powpow-register/
├── SKILL.md                            # 主指引（触发条件、10 步流程、语气规范、硬规则）
├── README.md
├── .clawhubignore
├── assets/
│   └── screenshots/                    # 随包产品截图（powpow-01~06.jpg）
├── scripts/
│   └── powpow-register.mjs             # 核心脚本（零依赖 Node≥18，12 个子命令）
└── references/
    ├── agent-flow.md                   # 脚本契约、完整时序、双链接机制、安全设计
    ├── registration-flow.md            # 字段规则、限流、报错排查表（含源码出处）
    ├── payment-and-activation.md       # 支付宝 vibepay 契约、PayPal 通道、激活链路、排查
    └── screenshots.md                  # 随包截图清单 + 场景速查
```

## 安全

- **密码**只经环境变量传入脚本进程，绝不写命令行参数、不落盘、不进状态文件；
  agent 生成的密码只显示一次供用户保存。
- **验证码**只用于官方注册接口，用完即弃；状态文件（邮箱/用户名/token/订单号）
  存系统临时目录，流程结束 `cleanup` 一键清除。
- **支付**：agent 只创建订单、打开收银台、查询到账；绝不接触用户的支付宝/PayPal
  账号或支付凭证，绝不代付。订阅自愿，不催不劝。
- 所有请求只发官方域名 `https://global.powpow.online`；不含任何密钥。

## 许可

MIT-0（ClawHub 统一许可）。

## 链接

- 官网：https://global.powpow.online
- 注册：https://global.powpow.online/register
- 了解泡泡（视频）：https://www.bilibili.com/video/BV1Wu826UEVz/
