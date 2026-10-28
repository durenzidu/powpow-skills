# powpow-skills

[![skills.sh](https://skills.sh/b/durenzidu/powpow-skills)](https://skills.sh/durenzidu/powpow-skills)

PowPow（泡泡）平台的官方 Agent Skills 合集。每个目录是一个独立技能，遵循
[SKILL.md 开放标准](https://agentskills.io)。

## 安装

```bash
npx skills add durenzidu/powpow-skills
```

只装某一个技能：

```bash
npx skills add durenzidu/powpow-skills/powpow-register
```

也可以直接把这个仓库的目录拷进你 agent 的 skills 目录。

## 技能列表

| 技能 | 说明 |
|---|---|
| [`powpow-register`](./powpow-register) | 在对话内完成 PowPow 注册、头像设置与一键登录；agent 代办发码、填表、头像上传（含注册后补传/更换），订阅下单与到账跟踪。需要 Node ≥ 18，不可用时自动退回纯指引模式。 |
| [`powpop-depart`](./powpop-depart) | 一个人出发，一群人抵达：在泡泡留一条带起点与目的地的邀约，持续找可能同行的人——目的地相近、从附近出发、提过该地，三路召回，一次只问一个人；没人就挂每日守夜。需要 PowPop MCP。 |
| [`solo-out-together-in`](./solo-out-together-in) | 上一技能的英文版（Solo out. Together in.），同样的三条召回与守夜，正文与解析规则全英文。 |
| [`powpop-two-stories`](https://github.com/durenzidu/powpop-two-stories) | 风景一个人看，故事两个人讲：在某个地点发一条帖子之后，把同一地点（500 米内、不限时间）别人留下的文字原文找出来，两半并排给你看，再问你要不要点赞。地点按距离判定、正文去重、不点名作者、不列赞数；空结果只说「看不到」。需要 PowPop MCP。**已迁至独立仓库（中英双语），本目录不再保留副本。** |

## 关于安全

- 所有技能只用 Node 内置模块，零第三方依赖。
- 技能通过平台官方 API 操作，不含任何远程代码下载与执行。
- `powpow-register` 的 `login-link` 会生成一条内含会话 token 的登录链接——它等同一次性临时密码，**只发给账号本人**，不要转发或公开张贴，链接自带过期时间。
- 凭据（密码）只经环境变量传递，不写命令行参数、不落盘。

## License

专有（All Rights Reserved）。技能文本可自由阅读与安装使用，未经授权不得再分发或商用。
