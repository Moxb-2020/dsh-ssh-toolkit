# dsh-ssh-toolkit

测试环境远程管理技能（Agent Skill）：SSH 服务器 + telnet/SNMP 网络设备 + macOS→Windows PowerShell Remoting。适用于 [DeepSeek Harness](https://github.com/deepseek-ai/deepseek-harness) 及任何识别 `SKILL.md`（agentskills 格式）的 harness（Claude Code、OpenClaw 等）。

## 它解决什么

- 三类设备命令通道不同（SSH / telnet / SNMP / SSH Remoting），混用必失败——技能给出分流与模板
- 连通性闸口：私钥/known_hosts/端口前置检查、超时三步甄别（沙箱断网 vs 主机离线）、判离线即停手不重试
- **口令零上下文**：模型侧经 [dsh-cred-ref](https://github.com/Moxb-2020/dsh-cred-ref) 插件按别名执行（`cred_list` / `cred_exec`），明文口令只存本地 chmod 600 凭据库；技能与 references 内不含任何口令
- 未装插件时降级：密钥免密 ssh + 用户本人终端的 sshpass 模板

## 安装

```bash
# DeepSeek Harness
git clone https://github.com/Moxb-2020/dsh-ssh-toolkit.git ~/.dsh/skills/ssh-toolkit

# Claude Code
git clone https://github.com/Moxb-2020/dsh-ssh-toolkit.git ~/.claude/skills/ssh-toolkit
```

或把本目录复制进对应 skills 目录。推荐同时安装 [dsh-cred-ref](https://github.com/Moxb-2020/dsh-cred-ref) 获得口令零上下文与配置页凭据管理；不装也能用（密钥/用户终端路径）。

## 使用

1. 把 `references/servers.md` 的示例表替换为你的环境（示例 IP 为 RFC 5737 文档专用段）
2. 口令登记进 dsh-cred-ref（插件配置页或本地凭据库），**不要**写进技能文件
3. 对话里说「连 server-a 查日志」即可；技能会引导先做连通性闸口再执行

## 安全模型

- 技能、references、对话、长期记忆：一律不出现明文口令
- 模型禁止读取/复述凭据库、禁止请求用户在对话粘贴口令
- 口令认证的文件传输等插件覆盖不了的操作，引导用户本人终端执行

## License

MIT
