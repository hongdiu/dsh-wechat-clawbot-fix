# dsh-wechat-clawbot-fix

微信 ClawBot ↔ DeepSeek Harness 的桥接插件（修补版）。

> 这是 [`dsh-wechat-clawbot@0.3.1`](https://github.com/zhengjy01/dsh-wechat-clawbot) 的修补分支，**只为让它在 DSH 0.2.x 上正常工作**，不是上游官方版本。上游原文档保留在 [`README.zh.md`](README.zh.md)。

## 为什么有这个包

上游 `0.3.1` 在 `package.json` 里已声明支持 `@deepseek-ai/dsh-settings ^0.2.0-rc.2`，但代码仍在用 0.1.x 的 API。结果是插件在 DSH 0.2.x 上**完全无法激活**：

```
failed to import: SyntaxError: The requested module '@deepseek-ai/dsh-settings'
does not provide an export named 'settingsNamespace'
```

此外还有两个影响日常使用的问题：PC 上发起的任务审批不会发到微信、DSH 重启后审批静默回落到电脑。本分支把它们一起修了。

## 与上游的差异

| # | 文件 | 问题 | 修复 |
|---|---|---|---|
| 1 | `dsh-wechat-bot/index.js` | 导入已被 0.2 删除的 `settingsNamespace` | 改用 `SettingsForms.describe()` 按 entry id 取配置段 |
| 2 | `dsh-wechat-bridge/index.js` | `approval/request` 对非微信会话直接 `return next()` | 非 bridge 会话也走 `askInMessenger`，队列 key 用 `pc:<sessionId>` |
| 3 | `dsh-wechat-bridge/index.js` | `claimApprovalReply` 只按当前会话找队列，PC 任务的 `1` 会被当成聊天 | 按 key 找不到时回退认领最新的 `pc:` 待批问题 |
| 4 | `dsh-wechat-bot/index.js` | `notify` 只认内存里的 `lastRecipient`，重启后为空 → 抛错 → 静默回落 GUI | 从 `~/.dsh-wechat/last-inbound.json` 恢复收件人 |

## 已验证的行为

环境：DSH `0.2.1-rc.2` + Windows + `profiles/desktop`。

1. 在 GUI 会话里请求「在桌面新建 HelloWorld.java」（工作区之外，必然触发审批）
2. 手机微信收到「⚠️ 需要审批 / 工具：fs_write / 回复 1 = 批准一次 / 2 = 拒绝」
3. 回复 `1`
4. 文件写入成功，微信收到「✅ 已批准一次」

旁证：`~/.dsh-wechat/window-state.json` 的 `lastSend.at` 在审批时向前推进，`ret: 0 / send_ok`，说明问句确实经过网关发出。

## 安装

**先卸掉上游同名包**，否则两个实例会抢 `51235` 端口：

```
dsh plugin --profile desktop remove dsh-wechat-clawbot
```

从 git 安装：

```
dsh plugin --profile desktop add github:<owner>/<repo>
```

或从本地目录安装：

```
dsh plugin --profile desktop add "file:D:\path\to\this\package"
```

装完重启 DSH，右下角出现绿色悬浮球 → 扫码登录。

## 已知限制

- **单实例**：网关固定监听 `127.0.0.1:51235`，同一台机器只能跑一个。
- **审批发给「最近联系人」**：PC 会话没有自己的微信对话，问句发给最后一次发消息进来的那个微信号。
- **180 秒超时**：不回复会自动落回电脑上的 GUI 弹窗（原逻辑，未改）。
- **上下文隔离**：微信消息进独立会话，不会进入你 GUI 里正在用的会话。
- **仅在 DSH 0.2.1-rc.2 验证过**。DSH 升级后如果上游把这些修复合并了，直接换回上游包即可。

## 上游与许可

- 上游：https://github.com/zhengjy01/dsh-wechat-clawbot
- 更早来源：https://github.com/lubaiUwU/DSH-WeChatClawBot
- 微信协议核心来自腾讯官方渠道插件 [`@tencent-weixin/openclaw-weixin`](https://www.npmjs.com/package/@tencent-weixin/openclaw-weixin)（MIT）

MIT。`LICENSE` 原样保留上游版权声明（`Copyright (c) 2026 lubai`），本分支的修改同样以 MIT 发布。

> ⚠️ 个人微信自动化存在账号风控风险，建议用小号，自行评估。
