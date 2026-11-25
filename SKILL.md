---
name: ssh-toolkit
description: 测试环境远程管理工具集（SSH 服务器 + telnet/SNMP 网络设备 + macOS→Windows PowerShell Remoting）。当用户要求连接测试服务器、执行远程命令、查看远程日志、检查远程服务状态、传输文件，或登记/查询测试环境里的交换机等网络设备（telnet、SNMP），或在 macOS 上通过 SSH Remoting 连接 Windows PowerShell 时使用此技能。触发词包括：远程、测试环境、服务器、SSH、查日志、看状态、重启服务、传文件、交换机、网络设备、telnet、SNMP、VLAN、端口、PowerShell、pwsh、Remoting、WinRM、WSMan、Homebrew cask、连不上、超时、sshpass。
version: 4
---

# SSH Toolkit - 测试环境远程管理

## 概述

本技能覆盖三类测试环境设备/链路，**命令通道完全不同，切勿混用**：

| 设备类别 | 连接方式 | 工具 | 凭据 | 清单位置 |
|---|---|---|---|---|
| 服务器（Linux / Windows Server） | SSH | 模型侧 `cred_exec`；用户终端 `sshpass` + `ssh` / `scp` / `rsync` | 口令由 dsh-cred-ref 插件按别名解析，**不写在技能里** | `references/servers.md` 的「服务器列表」 |
| 网络设备（Cisco、华为交换机等） | telnet / SNMP（**不是 SSH**） | 模型侧 `cred_exec`(kind=telnet)；SNMP 用 `snmpwalk`/`snmpget` | 登录与 enable 口令由 dsh-cred-ref 解析 | `references/servers.md` 的「网络设备清单（telnet / SNMP，非 SSH）」 |
| macOS 本机 pwsh → Windows | SSH Remoting（OpenSSH 通道） | macOS 上的 `pwsh` + `-HostName` | 见「macOS → Windows PowerShell Remoting」 | 同上「服务器列表」 |

**模型侧命令执行一律走 dsh-cred-ref 插件**（`cred_list` 查别名 → `cred_exec` 执行），口令只在宿主进程内流动，不进对话上下文。未装插件时降级为密钥免密 ssh 与用户终端模板（见执行流程 A.4）。下文 `sshpass` 模板**仅供用户本人终端手动使用**（口令由用户自填），模型不得填写、索取或复述明文口令。网络设备走 telnet 交互或 SNMP，**不要**给它们套用 sshpass 模板。

## 基础命令模式

### 执行远程命令（模型侧首选）

```
cred_list()                                  # 别名 / 主机 / 用户 / 类型（无口令）
cred_exec(alias=<别名>, command="<命令>")     # ssh 与 telnet 均由宿主解析凭据
```

用户终端手动模板（SSH 服务器，sshpass 已存在；`<口令>` 由用户自填）：

```bash
sshpass -p '<口令>' ssh -n -o StrictHostKeyChecking=no -o ConnectTimeout=10 root@<IP或主机名> "<命令>"
```

> 循环遍历多台主机时，**每个 ssh 都要带 `-n`（或显式 `</dev/null`）**，否则 ssh 会吞掉循环/循环体的 stdin，后面的主机被跳过。

### 文件上传 / 下载 / 目录同步

模型侧：密钥认证的 `scp`/`rsync` 可直接执行；小文件可经 `cred_exec` 以 base64 落盘。**口令认证的传输由用户本人终端执行**（模板如下，`<口令>` 用户自填）：

```bash
sshpass -p '<口令>' scp -o StrictHostKeyChecking=no <本地文件> root@<IP>:<远程路径>
sshpass -p '<口令>' scp -o StrictHostKeyChecking=no root@<IP>:<远程文件> <本地路径>
sshpass -p '<口令>' rsync -avz -e "ssh -o StrictHostKeyChecking=no" <本地目录> root@<IP>:<远程路径>
```

### 双层跳板（堡垒机 → 内网主机）

外层连堡垒机，内层在堡垒机上再 ssh 内网主机；内层也必须 `-n`，两端都设短 `ConnectTimeout` 以免卡死（用户终端模板）：

```bash
for h in <内网IP1> <内网IP2>; do
  echo "===== $h ====="
  sshpass -p '<口令>' ssh -n -o StrictHostKeyChecking=no -o ConnectTimeout=20 <堡垒机用户>@<堡垒机IP> \
    "ssh -n -o BatchMode=yes -o ConnectTimeout=10 -i <堡垒机上的内网私钥> <内网用户>@$h 'hostname; ps -ef | grep -c <服务关键字>'"
  echo "exit=$?"
done
```

## 凭据

**明文口令不写在技能及其 references 里。** 统一由 dsh-cred-ref 插件管理：

- 查询：`cred_list()`（别名/主机/用户/类型，无口令）
- 执行：`cred_exec(alias, command)`（ssh 与 telnet；口令仅宿主进程内流动）
- 新增/修改：插件配置页 UI，或用户手动编辑本地凭据库（chmod 600）
- 用户名见 `references/servers.md` 两表；口令与 enable 口令一律按别名解析
- 模型**禁止**读取、复述凭据库内容，禁止请求用户在对话中粘贴口令；插件 guard 会拒绝触碰凭据路径的工具调用
- 网络设备口令与服务器口令**不同**，勿混用（插件按别名各自解析）

> 长期记忆中只保留指向本技能与插件的指针，不要把明文口令写进记忆正文。

## 清单

- `references/servers.md`：两个分区——「服务器列表」（别名/主机名、IP、OS、用途、备注）与「网络设备清单（telnet / SNMP，非 SSH）」（别名/IP、型号、连接方式、用途、备注）。**仓库内为示例占位**，使用时替换为你自己的环境；口令不进清单。
- `references/commands.md`：Linux 环境常用诊断命令速查（系统状态、服务管理、日志查看、网络诊断、磁盘/内存）。

若用户提到的设备不在清单中，按设备类别追加到对应分区：

- SSH 服务器登记格式：`别名 | IP | OS | 用途 | 备注`
- 网络设备登记格式：`别名 | IP | 型号 | 连接方式 | 用途 | 备注`（连接方式写 `telnet` 或 `SNMP`）
- 只 telnet 的老交换机/防火墙按网络设备登记，**不要**塞进 SSH 服务器表格。
- 新设备的**口令**登记进 dsh-cred-ref（配置页或用户手动编辑凭据库），不在清单文件里写口令。

## 网络设备（telnet / SNMP）操作

- **不用 sshpass**：`sshpass -p ... ssh root@IP` 对交换机无效（Cisco 默认只开 telnet，华为部分型号同理），会把设备卡在交互 shell 里。模型侧用 `cred_exec`(kind=telnet)。
- **只读优先**：查询端口、VLAN、接口状态、ARP/MAC 表等只读采集，先向用户确认只读意图再记入清单。
- **只读采集思路**（具体命令随型号不同，先只读、不写入任何配置）：
  - SNMP（首选，可脚本化）：`snmpget -v2c -c <community> <IP> <OID>` / `snmpwalk -v2c -c <community> <IP> <OID>`；常见如 `ifDescr`、`ifOperStatus`、`dot1qVlanStaticName`
  - telnet：模型侧 `cred_exec` 已封装交互登录与 enable；用户手动时 Cisco 登录后 `enable` → `show interfaces status` / `show vlan brief` / `show ip interface brief`；华为 `display interface brief` / `display vlan`
  - 用户终端手动 telnet 没有 sshpass 那种单行非交互写法，需要 expect 之类的交互封装；DSH 沙箱内 `expect` 会报 `The system has no more ptys`，此时改用 Python3 裸 socket 直连 TCP 23 自行匹配提示符（口令不得进上下文），或请用户手动采集。
  - 连不上时告知用户检查设备是否开启 telnet 或 SSH。
- **任何涉及写入配置（`configure terminal`、`save`、修改 VLAN）的操作都要先向用户确认。**

## 执行流程

### A. 连通性闸口（每次都做，先于任何远程命令）

1. **解析目标**：根据用户提及的主机名/别名/IP，从 `references/servers.md` 查找对应设备，并判断它是 SSH 服务器还是 telnet/SNMP 网络设备。清单里没有就如实说明，不要凭空编造路径或主机。
2. **确认操作**：对高危操作（重启服务器、删除文件、停止服务、修改设备配置）先向用户确认。
3. **模型侧走 cred_exec**：`cred_list` 核对别名存在后 `cred_exec` 执行；**不要拼 sshpass 口令命令**（明文不得进上下文，guard 亦拒绝凭据路径调用）。sshpass 是否存在只影响用户终端模板与密钥降级路径。
4. **密钥免密 ssh（降级/用户终端/未装插件）**：先 `ls -la ~/.ssh/`、`test -f ~/.ssh/<key> && echo KEY_OK`，再试 `ssh -n -o BatchMode=yes -o ConnectTimeout=10 <user>@<IP> "<命令>"`。仍不可用时**明确提示用户**：`brew install esolitos/ipa/sshpass`（或配密钥免密），口令认证操作请用户本人终端执行。
5. **核对私钥与目标**：用户给的私钥/工作目录先用 `test -f`、`ls -d` 验证存在再拼命令，不存在就不要拼进命令白跑超时。
6. **核对 known_hosts 与端口**：`grep -c "<网段前缀>" ~/.ssh/known_hosts 2>/dev/null` 与 `nc -z -G 5 <host> 22; echo nc-exit=$?`，把结果与私钥检查一起前置报告，不要等 SSH 超时后才补做。
7. **遍历多主机时**：每个 `ssh` 带 `-n`（或 `</dev/null`），每台输出加 `===== <host> =====` 分隔标头，并打印 `exit=$?`；跳板场景见「双层跳板」模板。
8. **超时后的离线甄别（三步法，到判离线就停手）**：
    ```bash
    curl -s -m 8 -o /dev/null -w '%{http_code}\n' https://www.baidu.com   # 200 = 本机外网通
    nc -z -G 5 <host> 22; echo "nc-exit=$?"                                # 1 = 22 端口不通
    ping -c 2 -t 5 <host> 2>&1 | tail -2                                   # 100% packet loss = 主机不响应
    ```
    判定表：
    - 外网非 200（curl 超时/失败）+ nc 不通 → **沙箱断网**，报告网络被限制，不要判定主机故障。
    - 外网 200 + nc 不通 + ping 丢包 → **主机离线/被隔离**，停手。
    - 外网 200 + nc 通 → 主机在线，问题在认证/私钥/防火墙，回到步骤 4–6 修凭据。
9. **判定离线后不要进重试循环**：不再反复 ssh，直接向用户报告判定结论 + 三步证据（URL 状态码、nc-exit、丢包率），然后按「降级核查方案」继续。

### B. 执行

10. **构造命令**：SSH 服务器与 telnet 设备一律 `cred_exec`（别名取自 servers.md 清单）；密钥认证场景按用户终端模板拼 ssh（带 `-n`）；macOS→Windows 的 PowerShell 走「macOS → Windows PowerShell Remoting」一节。
11. **Windows EncodedCommand 生成**：Python 侧**必须**用 heredoc + `r'''...'''` 原始字符串再 `encode('utf-16-le')`，否则 `\7`、`\a`、`\n` 等会被当转义序列改写，命令静默损坏：

    ```bash
    B64=$(python3 <<'EOF'
    import base64
    cmd = r'''
    $PSVersionTable.PSVersion.ToString()
    try { [void][scriptblock]::Create('echo a && echo b'); 'AND-OK' } catch { 'AND-PARSE-FAIL' }
    '''
    print(base64.b64encode(cmd.encode('utf-16-le')).decode())
    EOF
    )
    # 模型侧：cred_exec(alias=<windows别名>, command="powershell -EncodedCommand <B64>")
    # 用户终端：sshpass -p '<口令>' ssh -n -o StrictHostKeyChecking=no <user>@<IP> "powershell -EncodedCommand $B64"
    ```

    **损坏样例**：用普通字符串 `cmd = '''...\\7...'''` 时 `\7` 被解释成 BEL 字节，Base64 看似正常生成，但远端解码后命令已变形。遇到 PowerShell 命令里有 `\` 的反斜杠路径、`$null`、`&&`，一律用 `r'''`。
12. **执行并报告**：运行命令，返回结果摘要；输出过长时自动截取关键部分；多台主机分组呈现。

### C. 降级核查方案（实机不可达时）

13. 实机判定离线后，按以下顺序给出不依赖目标机的替代验证，**并明确标注哪些断言仍是「待验证」**：
    - **本地等价验证**：本机有同类环境时（如已装 `pwsh`），在本地验证可迁移的行为（语法能否解析、版本特性），用 `python3` / `pwsh -NoProfile -Command` 跑等价脚本。
    - **往期会话证据**：`session_search` 查历史结论，命中的旧证据要标明来源与时间。
    - **官方文档**：查 Windows PowerShell / OpenSSH 官方文档确认版本能力边界（如 `&&` 需 PowerShell 7+，5.1 用 `;`）。
    - **明确报告**：给出「已确认 / 待实机验证」两栏清单，并请用户在主机恢复后或提供可达路径时复跑步骤 8 的甄别命令。

### D. 收尾

14. **改动技能文件后**：若你维护多份副本（如不同 harness 的 skills 目录），同步并用 diff 校核一致。

## 工具调用注意

- 每次 bash 调用都必须带 `description` 参数（DSH 缺省会报 `invalid arguments: missing required property "description"` 并浪费一轮）。
- 读取技能 references 文件用普通 `read` 工具。
- 长命令给 `timeoutMs`（或 `timeout` 参数），并给 ssh 设 `ConnectTimeout`，避免单次调用挂死。

## 常见失败样例

| 现象 | 含义 | 处置 |
|---|---|---|
| `bash: sshpass: command not found` (exit 127) | 本机没装 sshpass | 模型侧本就走 cred_exec；用户终端提示安装/配密钥（步骤 3–4） |
| `Warning: Identity file ... not accessible` | 私钥路径不存在 | 先 `test -f` 前置检查，别拼进命令 |
| `ssh: connect to host X port 22: Operation timed out` (exit 255) | 网络不可达或主机离线 | 三步法甄别，判离线就停手（步骤 8–9） |
| `cd: <目录>: No such file or directory` | 工作目录/项目路径不存在 | 先 `pwd; ls -d` 确认，再决定是否继续 |
| 远端 PowerShell 报语法错、编码字段残缺 | `\7`/`\a` 被 Python 普通字符串转义 | 改用 heredoc + `r'''` 重新生成（步骤 11） |
| `invalid arguments: missing required property "description"` | bash 调用缺 description | 补参数重发 |
| cred_exec 报未知别名 | 凭据库未登记该设备 | 请用户在插件配置页添加，或手动编辑凭据库；不要在对话里索取口令 |

## Windows Server 注意事项

- Windows Server 2019+ 需启用 OpenSSH Server 才可使用上述命令
- **Windows 的 OpenSSH 默认登录 shell 是 PowerShell，不是 bash**：
  - 用 `;` 分隔多条命令，**不能用** Linux 的 `&&`、`&`（会报 `AmpersandNotAllowed`）；PowerShell 7 才支持 `&&`
  - 远程命令示例：`"hostname; (Get-CimInstance Win32_OperatingSystem).Caption; whoami"`
- Windows 下命令需通过 `cmd /c` 或 `powershell -Command` 包装：
  ```
  cred_exec(alias=<windows别名>, command="powershell -Command \"Get-Service\"")
  ```
- 若仅有 RDP 而无 SSH，告知用户需先在目标服务器启用 OpenSSH Server

## macOS → Windows PowerShell Remoting（SSH 通道）

macOS 上连接 Windows 的 PowerShell **只有 SSH Remoting 一条路**；WSMan/WinRM 在 macOS 上不可用。

### macOS 端 pwsh 安装与路径

- 安装：`brew install --cask powershell@preview`（Homebrew **没有**稳定的 `powershell` cask；`brew install --cask powershell` 会报 `Cask 'powershell' is unavailable`）。
- 可执行文件路径**以 `which pwsh` 为准**（常见 `/usr/local/microsoft/powershell/7-preview/pwsh` 或 `/opt/homebrew/bin/pwsh`）。
- `brew info`/`brew search` 因缓存权限失败时改查官方 API：
  ```bash
  curl -sL "https://formulae.brew.sh/api/cask.json" | python3 -c "import sys,json; d=json.load(sys.stdin); print('\n'.join(c['token'] for c in d if 'powershell' in c['token'].lower()))"
  ```

### Windows 端前置条件（管理员 PowerShell 执行）

```powershell
Add-WindowsCapability -Online -Name OpenSSH.Server~~~~0.0.1.0
Start-Service sshd; Set-Service sshd -StartupType Automatic
New-ItemProperty -Path "HKLM:\SOFTWARE\OpenSSH" -Name DefaultShell -Value "C:\Program Files\PowerShell\7\pwsh.exe" -PropertyType String -Force
```

- 目标机需同时安装 PowerShell 7（`C:\Program Files\PowerShell\7\pwsh.exe` 不存在则改指向实际的 `pwsh.exe`）。
- `-HostName` 走的是 SSH 传输，所以 Windows 端 **必须** 有可用的 sshd（上面第一步）。

### macOS 端连接方式

交互式会话（在 `pwsh` 里执行）：

```bash
pwsh -NoProfile
```

```powershell
Enter-PSSession -HostName <windows-ip> -UserName <user> -KeyFilePath ~/.ssh/id_rsa
```

非交互式一次执行命令：

```bash
pwsh -NoProfile -Command '
$session = New-PSSession -HostName <windows-ip> -UserName <user> -KeyFilePath ~/.ssh/id_rsa
Invoke-Command -Session $session -ScriptBlock { Get-Process | Select-Object -First 5 }
Remove-PSSession $session
'
```

纯 `ssh` 也能直接进 Windows 的 pwsh（无需 New-PSSession，适合一次性命令）：

```bash
ssh -n -o StrictHostKeyChecking=no <user>@<windows-ip> "hostname; (Get-CimInstance Win32_OperatingSystem).Caption"
```

### 关键限制

- **必须用 `-HostName`**（SSH 通道），`-ComputerName`（WinRM）在 macOS 上不可用
- 依赖 SSH 密钥认证或密码认证，建议配置免密（`~/.ssh/id_rsa` 对应 Windows 端 `authorized_keys`）
- preview 版本可能存在兼容性问题，生产场景谨慎使用

## 安全提醒

- 测试环境口令仅用于内网测试环境；明文只存本地凭据库（chmod 600），由 dsh-cred-ref 插件管理
- 切勿将这些口令用于生产环境或公网服务器
- 技能、references、对话与长期记忆中**一律不出现明文口令**；记忆只留指针
- 不请求用户在对话中粘贴口令；插件覆盖不了的口令认证操作（如口令认证 scp），引导用户本人终端执行

## Verification

1. `command -v sshpass` / `command -v pwsh` 的结果已在命令前置环节输出（缺失时已改道或提示用户）。
2. 私钥 `test -f`、`grep known_hosts`、`nc -z` 三项检查在远程命令之前完成，结果可复述。
3. 超时时若走到三步甄别，报告里有 `http_code`、`nc-exit`、`ping` 丢包率三组证据，且没有进入重试循环。
4. 多主机/跳板场景的输出带 `===== <host> =====` 标头与 `exit=$?`，每台都有结果。
5. 若用了 `-EncodedCommand`，生成命令的 Python 片段为 `python3 <<'EOF'` + `r'''`，并已核对远端回显未变形。
6. 技能目录（含 references）grep 明文口令零命中；校验用的匹配模式不写进技能文件。
