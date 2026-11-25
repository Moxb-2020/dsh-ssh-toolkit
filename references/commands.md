# Linux 常用运维诊断命令速查

以下命令模型侧一律经 `cred_exec(alias=<服务器别名>, command=<命令>)` 执行（dsh-cred-ref 插件解析凭据，口令不进上下文）。用户终端手动使用时拼接到 `sshpass -p '<口令>' ssh -o StrictHostKeyChecking=no root@<IP>` 后执行（`<口令>` 由用户自填，模型不得填写或索取）。

## 系统状态

| 用途 | 命令 |
|------|------|
| 系统版本 | `cat /etc/os-release` |
| 内核版本 | `uname -r` |
| 运行时长/负载 | `uptime` |
| CPU/内存概况 | `top -bn1 \| head -20` |
| 内存使用 | `free -h` |
| 磁盘使用 | `df -h` |
| 磁盘 IO | `iostat -x 1 3` |

## 服务管理（systemd）

| 用途 | 命令 |
|------|------|
| 查看服务状态 | `systemctl status <服务名>` |
| 启动服务 | `systemctl start <服务名>` |
| 停止服务 | `systemctl stop <服务名>` |
| 重启服务 | `systemctl restart <服务名>` |
| 开机自启 | `systemctl enable <服务名>` |
| 列出运行中的服务 | `systemctl list-units --type=service --state=running` |

## 日志查看

| 用途 | 命令 |
|------|------|
| 系统日志最近100行 | `journalctl -n 100 --no-pager` |
| 指定服务日志 | `journalctl -u <服务名> -n 100 --no-pager` |
| 实时跟踪日志 | `journalctl -u <服务名> -f` (非交互场景慎用) |
| 查找日志文件 | `find /var/log /opt /home -name "*.log" -mtime -7 2>/dev/null` |
| tail 大文件 | `tail -n 200 <日志路径>` |
| grep 关键字 | `grep -i "<关键字>" <日志路径> \| tail -50` |

## 进程与端口

| 用途 | 命令 |
|------|------|
| 查看进程 | `ps aux \| grep <关键字>` |
| 占用端口的进程 | `ss -tlnp \| grep <端口>` |
| 所有监听端口 | `ss -tlnp` |
| 杀进程 | `kill -9 <PID>` |

## 网络诊断

| 用途 | 命令 |
|------|------|
| IP 地址 | `ip addr` |
| 路由表 | `ip route` |
| 连通性 | `ping -c 4 <目标IP>` |
| 端口连通 | `timeout 3 bash -c '</dev/tcp/<目标IP>/<端口>' && echo OPEN \|\| echo CLOSED` |
| 防火墙规则 | `firewall-cmd --list-all` 或 `iptables -L -n` |

## 文件操作

| 用途 | 命令 |
|------|------|
| 查找大文件 | `find / -type f -size +100M 2>/dev/null \| head -20` |
| 目录大小 | `du -sh <目录>` |
| 文件传输到本机 | 用 scp（见 SKILL.md 基础命令模式） |

## 数据库（如装有 MySQL/MariaDB）

| 用途 | 命令 |
|------|------|
| 登录 MySQL | `mysql -uroot -p<密码>` |
| 查看数据库 | `mysql -uroot -e "show databases;"` |
| 查看进程列表 | `mysql -uroot -e "show processlist;"` |
