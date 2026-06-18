---
name: port-in-use
description: 检测端口绑定冲突，识别 Linux/macOS/Windows 上占用目标端口的进程。诊断 Port in use 错误，检查端口可用性，并可选择终止占用进程。
version: 1.0.0
---

# 端口占用

诊断端口绑定冲突。当服务因 "Port in use" 或 "端口已被占用" 而启动失败时，此技能可识别冲突进程并提供可操作的解决方案。

## 工作流程

1) 从用户或错误上下文中获取目标端口号。
2) 运行适合当前平台的检测命令。
3) 报告占用进程的 PID、进程名、用户和命令行。
4) 可选地提供终止进程的选项（需用户确认）。

## 检测命令

### macOS

```bash
# 识别特定端口的进程
lsof -i :<port> -P -n

# 备选方案（无 lsof，使用内置工具）
netstat -anvp tcp | grep "\.<port>" | grep LISTEN

# 显示所有监听端口及 PID
sudo lsof -i -P -n | grep LISTEN
```

### Linux

```bash
# 识别特定端口的进程
ss -tlnp | grep ":<port>"

# 备选方案
fuser <port>/tcp

# 显示 PID 和进程名
lsof -i :<port>
```

### Windows

```powershell
# 识别特定端口的进程（cmd）
netstat -ano | findstr :<port>

# 识别特定端口的进程（PowerShell）
netstat -ano | Select-String ":<port>"

# 将 PID 映射到进程名
tasklist /FI "PID eq <PID>"

# 一行命令：端口 -> 进程名（PowerShell）
$port=8080; $pid=netstat -ano | Select-String ":$port" | %{$_ -split '\s+' | select -Last 1}; tasklist /FI "PID eq $pid" 2>$null
```

## 示例输出

```
COMMAND   PID     USER   FD   TYPE             DEVICE SIZE/OFF NODE NAME
node     12345   zhou   24u  IPv4 0x12345678      0t0  TCP *:3000 (LISTEN)
```

## 解决操作

仅在用户确认后执行：

- macOS：`kill <PID>`
- Linux：`kill <PID>` 或 `fuser -k <port>/tcp`
- Windows：
  - cmd：`taskkill /PID <PID> /F`
  - PowerShell：`Stop-Process -Id <PID> -Force`

## 故障排除

- `lsof` 未找到 → 使用 `brew install lsof`（macOS）或 `apt install lsof`（Debian Linux）安装。
- 权限不足 → 添加 `sudo` 前缀（macOS/Linux）或以管理员身份运行（Windows）。
- IPv6 vs IPv4 → 如适用，同时检查 `*.<port>` 和 `*.<port>`。
- Docker 容器 → 在容器**内部**运行检测命令，或使用 `docker ps` 查找主机端口映射。
- Windows `netstat` 可能需要以管理员权限运行才能获取其他用户进程的 PID。
