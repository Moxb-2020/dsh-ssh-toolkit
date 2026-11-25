# 测试环境服务器清单（示例模板）

> 使用时：把下表替换为你自己的环境；根据用户提及的主机名/IP 查找对应条目。
> 新服务器登记格式：别名 | IP | OS | 用途 | 备注
> **口令不写在本文件**：登记进 dsh-cred-ref 插件（配置页或本地凭据库）。
> 示例 IP 使用 RFC 5737 文档专用段（192.0.2.0/24），不代表任何真实网络。

## 服务器列表

| 别名 | IP | OS | 用途 | 备注 |
|------|-----|-----|------|------|
| server-a | 192.0.2.10 | SLES 15 | 示例：数据库服务器 | hostname: db-a |
| server-b | 192.0.2.11 | CentOS 7 / Rocky 9 | 示例：备份服务器 | hostname: backup-b |
| win-a | 192.0.2.20 | Windows Server 2019+ | 示例：Windows 测试机 | OpenSSH 已装，登录 shell 为 PowerShell |

## 网络设备清单（telnet / SNMP，非 SSH）

| 别名 | IP | 型号 | 连接方式 | 用途 | 备注 |
|------|-----|------|---------|------|------|
| switch-a | 192.0.2.2 | Cisco Catalyst（IOS 15.x） | telnet 23（SSH 22 亦可） | 示例：核心接入交换机 | 管理地址在 Vlan1；单机非堆叠 |

> ⚠️ 网络设备**不能用 sshpass 那套非交互模式**（Cisco IOS 走 telnet 交互式会话）。
> 模型侧用 dsh-cred-ref 的 `cred_exec`(kind=telnet)，已封装交互登录与 enable。
> 用户手动采集：Python3 裸 socket 直连 TCP 23 自行匹配提示符。DSH 沙箱内 `expect` 会报
> `The system has no more ptys`，不可用；需 full access 或在沙箱外运行。
> 只读采集命令：`show version` / `show inventory` / `show vlan brief` / `show interfaces status`。

## 凭据

**明文口令不写在本文件。** 统一由 dsh-cred-ref 插件管理（`cred_list` / `cred_exec` / 插件配置页）：

### 服务器（SSH）

- 用户名：按设备登记（常见 `root` / `Administrator`；个别 Windows 为小写 `administrator`）
- 口令：按别名由插件解析（本地凭据库 chmod 600）；模型禁止读取或复述凭据库内容

### 网络设备

- 用户名按设备登记（常见 `admin` / 自定义）；登录与 enable 口令按别名由插件解析
  - Cisco 登录后为用户模式 `>`，需 `enable` 进特权模式 `#`（个别老设备 enable 口令可能为 `enable`，以设备实际提示符为准）
  - 网络设备口令与服务器口令**不同**，勿混用（插件按别名各自解析）

## 登记说明

当用户提供新的服务器信息时，按以下格式追加一行：

```
| 别名 | IP | OS(Linux/Windows/版本) | 用途描述 | 端口(若非22)/备注 |
```

新设备的口令登记进 dsh-cred-ref（插件配置页或用户手动编辑凭据库），**不要**写进本文件。
