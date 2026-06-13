# SSH 与 CPA 公网入口稳定性修复学习笔记

## 先说结论

这次问题不是单纯的“SSH 密码错了”或“端口没开”，而是服务器在某些时刻进入了这样一种状态：

- TCP 端口能连上。
- 但 SSH 还没发出 banner，就直接关闭连接。
- Web 端口也可能 TCP 通，但 HTTP 请求收不到响应。
- 本地 CPA 能用，但公网 CPA 依赖 SSH 反向隧道，所以公网入口会跟着不稳定。

最后采用的修复思路是：

1. 保留 OpenSSH 主入口 `2222`。
2. 新增更轻量的 Dropbear 备用 SSH 入口 `2022`。
3. CPA 公网反向隧道优先走 `2022`，失败再走 `2222`。
4. SSHD 配置保留足够连接容量，但设置上限，避免无限连接把内存打满。
5. 本地 watchdog 保持单实例、单隧道、退避重试，避免重复启动后堆进程。

现在的常用入口是：

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_154" -p 2022 xuan@154.219.106.185
```

主入口仍保留：

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_154" -p 2222 xuan@154.219.106.185
```

公网 API 入口是：

```text
http://154.219.106.185:8318/v1
model: gpt-5.5
```

本文不记录密码、API key、私钥内容等敏感信息。

## 1. 当时具体出了什么问题

最典型的报错是：

```text
kex_exchange_identification: Connection closed by remote host
Connection closed by 154.219.106.185 port 2222
```

这类错误有一个关键点：它发生在 SSH 认证之前。

正常 SSH 连接大致是：

1. 客户端 TCP 连接服务器端口。
2. 服务端返回 SSH banner，例如 `SSH-2.0-OpenSSH...`。
3. 双方协商算法。
4. 客户端提交密钥认证。
5. 登录成功后执行命令或进入 shell。

这次失败发生在第 1 步之后、第 2 步之前：

```text
TCP connected
服务端没有发 SSH banner
服务端直接断开
```

所以它不是私钥问题，也不是用户名问题。私钥错了通常会走到认证阶段，然后报 `Permission denied`。这次连认证阶段都没到。

## 2. 为什么“端口通”不等于“服务正常”

很多排查会先看端口：

```powershell
Test-NetConnection 154.219.106.185 -Port 2222
```

如果结果是 `True`，只能说明：

```text
网络能到这台机器
这个端口有东西在监听或接受连接
```

但它不能说明：

```text
sshd 一定能完成握手
系统一定有足够资源创建会话
应用层一定会返回正确响应
```

这次就出现了“端口通，但服务层不工作”的情况：

- `2222` TCP 通，但 SSH banner 前关闭。
- `8318` TCP 通，但 HTTP 请求有时无响应。
- `24045`、`6099`、`3211` 也曾出现 TCP 通但 HTTP 空响应或超时。

所以判断服务是否正常，要看应用层：

```powershell
ssh -p 2022 xuan@154.219.106.185 "echo OK"
Invoke-RestMethod http://154.219.106.185:8318/v1/models
```

## 3. CPA 公网入口的整体运行逻辑

这套 CPA 公网入口不是服务器自己直接跑模型服务，而是通过本机服务加反向 SSH 隧道暴露出去。

逻辑链路是：

```text
外部客户端
  |
  | HTTP: http://154.219.106.185:8318/v1
  v
服务器 154.219.106.185:8318
  |
  | local-cpa-forward.py
  v
服务器本机 127.0.0.1:18318
  |
  | SSH 反向隧道 -R
  v
本机 127.0.0.1:8317
  |
  | CLIProxyAPI
  v
模型接口 gpt-5.5
```

也就是说：

- 本机 CPA 服务监听 `127.0.0.1:8317`。
- 本机通过 SSH 连到服务器。
- SSH 命令创建反向端口：

```text
服务器 127.0.0.1:18318 -> 本机 127.0.0.1:8317
```

- 服务器上的 `local-cpa-forward.py` 再把公网 `8318` 转到服务器本机 `18318`。

所以公网入口能不能用，取决于几件事同时成立：

1. 本机 CLIProxyAPI 正常。
2. 本机到服务器的 SSH 隧道正常。
3. 服务器 `18318` 反向端口存在。
4. 服务器 `8318` 转发服务正常。
5. 客户端请求的模型名存在。

任何一环断了，公网 `8318/v1` 都可能不可用。

## 4. 为什么反复启动 BAT 会让问题更难判断

桌面入口是：

```text
C:\Users\Xuan\Desktop\Start-CPA-Public.bat
```

它实际调用：

```text
D:\tool\CLIProxyAPI_6.9.24_windows_amd64\Start-CPA-Public.ps1
```

这个 PowerShell 脚本负责：

- 启动本机 `cli-proxy-api.exe`。
- 检查本机 `/v1/models` 是否有 `gpt-5.5`。
- 启动 SSH 反向隧道。
- 检查公网 `/v1/models` 是否有 `gpt-5.5`。
- 失败时退避重试。

如果没有限制，反复双击 BAT 可能造成：

- 多个 watchdog 同时运行。
- 多条 SSH 隧道同时重试。
- 多个 CPA 进程争用端口。
- 服务器 SSH 的预认证连接槽被打满。
- 内存和进程数逐渐失控。

后来脚本加入了互斥锁：

```text
Global\CPA_PUBLIC_WATCHDOG_8318
```

它的作用是保证同一时间只运行一个 watchdog。脚本里也会清理旧的 Paramiko 隧道和重复 OpenSSH 隧道。

当前正常状态应该只有：

```text
1 个 cli-proxy-api.exe
1 个 Start-CPA-Public.ps1 watchdog
1 条 ssh.exe 反向隧道
0 个旧 Python 隧道
```

本次复检结果就是：

```text
WATCHDOGS=1
TUNNELS=1
CPA=1
LEGACY_PY=0
```

## 5. SSHD 配置为什么要“有限度地放宽”

OpenSSH 有一个很重要的配置叫 `MaxStartups`。

它控制的是“尚未完成认证的连接”数量。

这类连接包括：

- 正在建立握手的 SSH。
- 扫描器连接进来但不认证。
- 自动化脚本频繁重试造成的半开连接。
- 反向隧道断线后重连阶段的连接。

如果这个值太小，正常连接也可能被丢掉。  
如果这个值太大，攻击流量或错误脚本可能创建太多连接，拖垮内存。

之前曾经把它放得很大：

```text
MaxStartups 500:100:1000
PerSourceMaxStartups 300
```

这个配置可以让很多连接进来，但代价是风险也很高：如果外部扫描或本机脚本重试变多，服务器会承受大量预认证连接。

后来改成更均衡的配置：

```text
LoginGraceTime 10
MaxStartups 120:30:200
PerSourceMaxStartups 60
PerSourceNetBlockSize 32:128
MaxSessions 80
MaxAuthTries 3
UseDNS no
TCPKeepAlive yes
ClientAliveInterval 30
ClientAliveCountMax 10
AllowTcpForwarding yes
PermitListen any
```

这里的意思是：

- `LoginGraceTime 10`：未认证连接最多等 10 秒，避免半开连接长期占位。
- `MaxStartups 120:30:200`：未认证连接达到一定数量后开始概率丢弃，最高不超过 200。
- `PerSourceMaxStartups 60`：同一来源最多 60 个未认证连接。
- `MaxSessions 80`：一个已登录连接内最多 80 个会话。
- `MaxAuthTries 3`：认证失败次数减少，避免无效尝试拖很久。
- `UseDNS no`：不做反向 DNS 查询，减少登录等待。
- `AllowTcpForwarding yes`：允许 SSH 反向隧道。
- `PermitListen any`：允许远端监听反向隧道端口。

这个配置不是追求“无限连接”，而是追求：

```text
正常使用稳定
小规模并发可用
错误重试不会无限堆积
内存不会被 SSH 连接拖爆
```

## 6. 为什么新增 Dropbear 备用入口

OpenSSH 功能完整，但相对重。  
Dropbear 是更轻量的 SSH 服务，常用于嵌入式设备或救援入口。

这次新增：

```text
Dropbear 端口: 2022
```

配置是：

```text
NO_START=0
DROPBEAR_PORT=2022
DROPBEAR_EXTRA_ARGS="-s -w -g -K 30 -I 300"
DROPBEAR_RECEIVE_WINDOW=65536
```

关键参数含义：

- `-s`：禁止密码登录，只走密钥，更安全。
- `-w`：禁止 root 登录。
- `-g`：禁止 root 密码登录。
- `-K 30`：keepalive 间隔。
- `-I 300`：空闲超时。

新增 Dropbear 后，有了两条 SSH 路：

```text
OpenSSH: 2222
Dropbear: 2022
```

这带来两个好处：

1. 主 OpenSSH 偶发卡住时，还能用 `2022` 进服务器处理。
2. CPA 公网隧道可以优先走更轻量的 `2022`，减少和用户手动 SSH 抢主入口的机会。

当前推荐手动登录使用：

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_154" -p 2022 xuan@154.219.106.185
```

## 7. CPA 脚本改了什么

本地脚本：

```text
D:\tool\CLIProxyAPI_6.9.24_windows_amd64\Start-CPA-Public.ps1
```

核心改动是把 SSH 隧道入口从单一端口变成双入口优先级：

```powershell
$sshPorts = @(2022, 2222)
```

也就是：

1. 先尝试 Dropbear `2022`。
2. 如果失败，再尝试 OpenSSH `2222`。

反向隧道仍然是：

```text
服务器 127.0.0.1:18318 -> 本机 127.0.0.1:8317
```

公网入口仍然是：

```text
http://154.219.106.185:8318/v1
```

脚本还保留了这些保护：

- 单实例互斥锁，避免多个 watchdog。
- 清理旧 Paramiko 隧道。
- 清理重复隧道，只保留一条。
- SSH 连接失败后退避。
- 本地 CPA 和公网 CPA 分别健康检查。

当前正式隧道已经通过 `2022` 建立：

```text
ssh.exe ... -p 2022 -R 127.0.0.1:18318:127.0.0.1:8317
```

## 8. 为什么不能追求“无限快速 SSH 每次都成功”

这点很重要。

你想要的是：

```text
我自己 SSH 能稳定进
CPA 公网入口稳定
不会因为测试或脚本导致内存爆炸
```

这和“无限快速新建 SSH 连接，每一条都必须成功”不是一回事。

如果把限制全部放开，短时间内大量 SSH 连接确实更容易被接受，但风险是：

- 扫描流量也会被接受。
- 错误脚本也会大量连接。
- 半开连接占住内存和进程资源。
- 服务器更容易 OOM。

所以最终方案是：

```text
允许正常连接和小并发
给预认证连接设置上限
使用长连接隧道
失败后退避
增加备用 SSH 服务
```

这才是“稳定”和“不吃满内存”之间比较合理的平衡。

## 9. 最终验证结果

本次最终验证包括这些项目。

本地模型接口：

```text
http://127.0.0.1:8317/v1/models
结果：包含 gpt-5.5
```

公网模型接口：

```text
http://154.219.106.185:8318/v1/models
结果：包含 gpt-5.5
```

本地 chat：

```text
返回：LOCAL_CHAT_OK
```

公网 chat：

```text
返回：PUBLIC_CHAT_OK
```

SSH 长连接测试：

```text
2022 LONG OK
2222 LONG OK
```

混合小并发测试：

```text
2222 + 2022 混合 12 条小并发
结果：12/12 成功
```

进程数量：

```text
WATCHDOGS=1
TUNNELS=1
CPA=1
LEGACY_PY=0
```

服务器内存：

```text
总内存: 3.8GiB
可用: 约 1.9GiB
swap: 2.0GiB
```

服务器监听：

```text
2222  OpenSSH
2022  Dropbear
8318  CPA 公网入口
18318 SSH 反向隧道远端监听
```

## 10. 以后怎么判断是哪一层坏了

### 10.1 先看本机 CPA

如果本机都不通，公网一定不通。

```powershell
$headers = @{ Authorization = "Bearer <你的 API key>" }
Invoke-RestMethod -Uri "http://127.0.0.1:8317/v1/models" -Headers $headers
```

如果这里失败，优先查本机 `cli-proxy-api.exe`。

### 10.2 再看 SSH 隧道

看本机是否有一条正式隧道：

```powershell
Get-CimInstance Win32_Process |
  Where-Object {
    $_.Name -eq "ssh.exe" -and
    $_.CommandLine -like "*154.219.106.185*" -and
    $_.CommandLine -like "*18318*"
  } |
  Select-Object ProcessId,CommandLine
```

正常应该看到一条 `ssh.exe`，包含：

```text
-p 2022
-R 127.0.0.1:18318:127.0.0.1:8317
```

### 10.3 再看公网 CPA

```powershell
$headers = @{ Authorization = "Bearer <你的 API key>" }
Invoke-RestMethod -Uri "http://154.219.106.185:8318/v1/models" -Headers $headers
```

如果本机通、公网不通，优先怀疑：

- SSH 隧道没起来。
- 服务器 `local-cpa-forward.service` 异常。
- 服务器 `8318` 或 `18318` 没监听。

### 10.4 再看服务器 SSH

优先用 `2022`：

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_154" -p 2022 xuan@154.219.106.185 "echo OK"
```

再看 `2222`：

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_154" -p 2222 xuan@154.219.106.185 "echo OK"
```

如果 `2222` 偶发失败但 `2022` 正常，优先用 `2022` 进去修 OpenSSH：

```bash
sudo systemctl restart ssh
sudo pkill -TERM -f 'sshd: \[accepted\]' || true
sudo pkill -TERM -f 'sshd: \[net\]' || true
```

### 10.5 看服务器资源

```bash
free -h
ps -eo pid,ppid,user,comm,%cpu,%mem,rss,etime --sort=-rss | head -20
sudo ss -ltnp | grep -E ':(2022|2222|8318|18318)\b' || true
```

如果可用内存很低，先不要压测 SSH。  
这时应该看内存大户，而不是继续疯狂重连。

## 11. 以后使用时的建议

正常使用时：

1. 启动 `C:\Users\Xuan\Desktop\Start-CPA-Public.bat`。
2. 保持窗口打开。
3. 看到公网恢复日志后再使用公网 API。

如果要自己 SSH：

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_154" -p 2022 xuan@154.219.106.185
```

如果 `2022` 不通，再试：

```powershell
ssh -i "$env:USERPROFILE\.ssh\id_ed25519_154" -p 2222 xuan@154.219.106.185
```

不要做这些事：

- 连续疯狂双击 BAT。
- 写脚本无限循环 SSH 重连。
- 在服务器低内存时做高并发连接测试。
- 把 `MaxStartups` 改成无限大。
- 删除 Dropbear 备用入口后不留其他救援通道。

## 12. 这次学到的核心经验

### 端口通不代表服务正常

TCP 通只是第一层。  
SSH、HTTP、模型接口都要分别验证。

### SSH 失败要看失败阶段

如果是：

```text
Permission denied
```

多半是认证问题。

如果是：

```text
kex_exchange_identification
Connection closed
```

而且发生在 banner 前，多半是服务端资源、连接槽、限流、守护进程状态问题。

### 反向隧道适合长连接，不适合频繁重建

CPA 公网入口依赖 SSH 反向隧道。  
这种隧道应该尽量保持一条长连接，而不是频繁断开重连。

### 稳定不是无限放开

真正稳定的设计不是“任何连接都无限接收”，而是：

- 正常流量稳定接收。
- 异常重试受控。
- 有备用入口。
- 有内存保护。
- 有清理重复进程的机制。

### 备用入口很重要

这次新增 `2022` 后，即使 `2222` 偶发 banner 前关闭，也可以通过 Dropbear 进去修。  
这比单纯调大 OpenSSH 参数可靠得多。

## 13. 当前最终状态摘要

```text
服务器 IP: 154.219.106.185
OpenSSH 主入口: 2222
Dropbear 备用入口: 2022
公网 CPA: http://154.219.106.185:8318/v1
本机 CPA: http://127.0.0.1:8317/v1
反向隧道远端: 127.0.0.1:18318
隧道 SSH 优先级: 2022 -> 2222
模型: gpt-5.5
watchdog: 单实例
隧道: 单实例
旧 Python 隧道: 不运行
```

当前这套结构的目标不是“压测永不失败”，而是满足日常使用：

- 本地和公网都能访问模型。
- 用户可以自己稳定 SSH。
- CPA 隧道不抢占大量连接。
- 重试不会堆积进程。
- 服务器内存不会因为连接测试被拖爆。
