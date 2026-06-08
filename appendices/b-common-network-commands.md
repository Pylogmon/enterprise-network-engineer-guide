# 附录 B：常用网络命令速查

本附录整理终端、服务器和网络运维中最常用的网络检查命令。命令速查的重点不是“会输入”，而是知道每条命令回答什么问题。

排查网络故障时建议按下面顺序推进：

```text
本机地址 -> 网关连通 -> DNS 解析 -> 路由路径 -> 业务端口 -> 抓包确认
```

如果跳过基础检查，直接改交换机、防火墙或路由配置，容易扩大故障范围。

## B.1 常用排错目标与命令

| 目标 | Windows | Linux | macOS | 说明 |
| --- | --- | --- | --- | --- |
| 查看 IP 地址 | `ipconfig /all` | `ip addr` | `ifconfig` 或 `ipconfig getifaddr en0` | 检查 IP、掩码、网关、DNS |
| 查看路由表 | `route print` | `ip route` | `netstat -rn` | 检查默认路由和目标网段路由 |
| 查看 ARP/邻居表 | `arp -a` | `ip neigh` | `arp -a` | 检查网关 MAC 是否解析 |
| 测试连通性 | `ping` | `ping` | `ping` | 判断基本 IP 可达性 |
| 路径跟踪 | `tracert` | `traceroute` | `traceroute` | 判断路径在哪一跳中断 |
| DNS 查询 | `nslookup` | `dig` 或 `nslookup` | `dig` 或 `nslookup` | 判断域名解析是否正确 |
| 测试 TCP 端口 | `Test-NetConnection` | `nc -vz` 或 `curl -v` | `nc -vz` 或 `curl -v` | 判断服务端口是否可达 |
| 查看连接 | `netstat -ano` | `ss -tunap` | `netstat -an` | 查看监听和已建立连接 |
| 抓包 | Wireshark | `tcpdump` | `tcpdump` | 确认报文是否发出/返回 |

## B.2 Windows 常用命令

### 查看本机地址

```powershell
ipconfig /all
```

重点看：

| 字段 | 说明 | 异常例子 |
| --- | --- | --- |
| IPv4 Address | 本机 IP 地址 | `169.254.x.x` 表示 DHCP 获取失败 |
| Subnet Mask | 子网掩码 | 掩码错误会导致同网段判断错误 |
| Default Gateway | 默认网关 | 为空时通常不能访问其他网段 |
| DNS Servers | DNS 服务器 | 指错 DNS 会导致域名解析失败 |
| DHCP Enabled | 是否使用 DHCP | 应自动获取却变成手工地址时要核查 |

### 释放和重新获取 DHCP 地址

```powershell
ipconfig /release
ipconfig /renew
ipconfig /flushdns
```

适用场景：

- 终端刚移动到新 VLAN。
- DHCP 地址池调整后需要重新获取地址。
- DNS 解析结果可能被本机缓存影响。

如果 `renew` 长时间失败，应继续检查交换机端口 VLAN、Trunk 放行、DHCP Relay、地址池和服务器状态。

### 测试网关和远端连通性

```powershell
ping 10.28.10.1
ping 10.28.60.20
ping 8.8.8.8
```

建议顺序：

1. 先 ping 本 VLAN 网关。
2. 再 ping 内部服务器。
3. 再 ping 外部地址。

如果网关不通，优先检查本机地址、掩码、网关、交换机端口和 VLAN。如果网关通但远端不通，再检查三层路由、ACL、防火墙策略和回程路由。

### 跟踪路径

```powershell
tracert 10.28.60.20
pathping 10.28.60.20
```

`tracert` 用于观察路径经过哪些三层设备。`pathping` 会做更长时间统计，适合观察丢包位置，但执行时间更久。

注意：很多防火墙或路由器会限制 ICMP 超时报文，导致路径显示 `* * *`。这不一定代表业务不通，需要结合 TCP 端口测试判断。

### DNS 查询

```powershell
nslookup intranet.example.com
nslookup intranet.example.com 10.28.60.10
```

第一条使用系统当前 DNS，第二条指定 DNS 服务器。这样可以区分“本机 DNS 配置错误”和“DNS 服务器记录错误”。

### 测试 TCP 端口

```powershell
Test-NetConnection 10.28.60.20 -Port 443
Test-NetConnection intranet.example.com -Port 443
```

重点字段：

| 字段 | 含义 |
| --- | --- |
| `RemoteAddress` | 域名最终解析到的 IP |
| `TcpTestSucceeded` | TCP 是否连接成功 |
| `PingSucceeded` | ICMP 是否成功，不等同于业务成功 |

如果 `TcpTestSucceeded` 为 `False`，继续检查服务器监听、本机防火墙、中间防火墙策略、NAT 和路由。

### 查看连接和进程

```powershell
netstat -ano
netstat -ano | findstr :443
tasklist /fi "PID eq 1234"
```

用途：

- 判断本机是否正在监听某个端口。
- 判断客户端是否已经建立到服务器的连接。
- 通过 PID 找到对应进程。

## B.3 Linux 常用命令

### 查看地址和链路状态

```sh
ip addr
ip link
```

常见判断：

| 现象 | 可能原因 |
| --- | --- |
| 接口没有 IP | DHCP 失败或未配置地址 |
| 接口 `DOWN` | 网卡未启用、虚拟网卡未连接 |
| 有 IP 但掩码错误 | 手工配置错误或 DHCP 选项错误 |

### 查看路由

```sh
ip route
ip route get 10.28.60.20
```

`ip route get` 很实用，它会告诉你访问某个目的地址时，本机会选择哪个出接口和源地址。

示例：

```text
10.28.60.20 via 10.28.10.1 dev eth0 src 10.28.10.25
```

含义是：访问 `10.28.60.20` 时，从 `eth0` 出去，源地址使用 `10.28.10.25`，下一跳是 `10.28.10.1`。

### 查看 ARP/邻居表

```sh
ip neigh
```

如果网关没有邻居表项，说明本机还没有解析到网关 MAC，常见原因包括 VLAN 错、网关接口 down、二层链路异常或 ARP 被安全策略限制。

### DNS 查询

```sh
dig intranet.example.com
dig @10.28.60.10 intranet.example.com
cat /etc/resolv.conf
```

`dig` 输出中重点看：

| 字段 | 说明 |
| --- | --- |
| `ANSWER SECTION` | 是否有解析结果 |
| `SERVER` | 实际查询的 DNS 服务器 |
| `Query time` | 查询耗时 |
| `status` | `NOERROR`、`NXDOMAIN`、`SERVFAIL` 等状态 |

### 测试端口

```sh
nc -vz 10.28.60.20 443
curl -vk https://10.28.60.20/
```

`nc -vz` 适合快速判断 TCP 端口是否可达。`curl -v` 能继续观察 TLS 握手、HTTP 状态码和重定向。

### 查看监听和连接

```sh
ss -tunlp
ss -tunap
```

常用参数含义：

| 参数 | 含义 |
| --- | --- |
| `-t` | TCP |
| `-u` | UDP |
| `-n` | 不解析名称，直接显示 IP 和端口 |
| `-l` | 只看监听端口 |
| `-p` | 显示进程 |

### 抓包

```sh
sudo tcpdump -i eth0 host 10.28.60.20
sudo tcpdump -i eth0 tcp port 443
sudo tcpdump -i eth0 -nn "host 10.28.60.20 and port 443"
```

抓包判断思路：

| 看到的报文 | 说明 |
| --- | --- |
| 只有 SYN，没有 SYN-ACK | 回包没回来，检查中间策略、路由、服务器 |
| SYN 后收到 RST | 目标可达但端口未监听或被拒绝 |
| TCP 建立后 TLS 失败 | 检查证书、协议版本、SNI、代理 |
| 请求发出后无应用响应 | 检查服务进程、应用日志、负载均衡 |

## B.4 macOS 常用命令

### 查看地址、网关和 DNS

```sh
ifconfig
netstat -rn
scutil --dns
networksetup -getdnsservers Wi-Fi
```

如果使用有线网卡，服务名可能不是 `Wi-Fi`。可以先查看网络服务：

```sh
networksetup -listallnetworkservices
```

### 测试端口和访问

```sh
nc -vz 10.28.60.20 443
curl -vk https://intranet.example.com/
```

macOS 的 `curl` 和 `nc` 很适合做应用访问排查。若浏览器打不开但 `curl` 可以访问，可能是浏览器代理、证书信任、缓存或安全插件影响。

### 抓包

```sh
sudo tcpdump -i en0 -nn host 10.28.60.20
sudo tcpdump -i en0 -nn tcp port 443
```

`en0` 常见于 Wi-Fi 或主网卡，但不同设备可能不同。可用 `ifconfig` 先确认活动接口。

## B.5 常见故障的命令组合

### 场景 1：终端无法上网

```text
ipconfig /all
ping 默认网关
nslookup www.example.com
ping 8.8.8.8
Test-NetConnection www.example.com -Port 443
```

判断逻辑：

| 结果 | 方向 |
| --- | --- |
| 没有正确 IP | 查 DHCP、VLAN、接入口 |
| 网关 ping 不通 | 查本机掩码、网关、交换机端口 |
| 能 ping 公网 IP，域名不解析 | 查 DNS |
| DNS 正常但 TCP 443 不通 | 查出口防火墙、代理、NAT |

### 场景 2：能上网但打不开内部系统

```text
nslookup intranet.example.com
ping 10.28.60.20
Test-NetConnection 10.28.60.20 -Port 443
tracert 10.28.60.20
```

重点区分：

- 域名是否解析到正确内网地址。
- TCP 443 是否可达。
- 路径是否走到了服务器区防火墙或核心交换机。
- 服务器是否只允许某些源地址访问。

### 场景 3：服务器端口是否真的监听

Linux：

```sh
ss -tunlp | grep :443
curl -vk https://127.0.0.1/
```

Windows：

```powershell
netstat -ano | findstr :443
Test-NetConnection 127.0.0.1 -Port 443
```

如果服务器本机都没有监听端口，就不要先查网络设备。应先检查服务进程、应用配置和本机防火墙。

### 场景 4：跨网段访问失败

```text
查看本机 IP/掩码/网关
ping 本机网关
查看路由表
tracert/traceroute 目的地址
测试目标 TCP 端口
```

跨网段问题常见原因：

| 类型 | 示例 |
| --- | --- |
| 本机配置 | 网关错误、掩码错误 |
| 三层网关 | VLANIF down、接口地址错 |
| 路由 | 缺少去程或回程路由 |
| 安全策略 | ACL、防火墙策略拒绝 |
| NAT | 源地址被错误转换 |

## B.6 命令输出判断表

| 输出/现象 | 可能含义 | 下一步 |
| --- | --- | --- |
| `169.254.x.x` | DHCP 获取失败 | 查接入 VLAN、DHCP、Relay |
| `Request timed out` | ICMP 无响应 | 结合 TCP 测试，不只依赖 ping |
| `Destination host unreachable` | 本机或网关无路由/不可达 | 查路由表和网关 |
| `TcpTestSucceeded: False` | TCP 建连失败 | 查服务监听、策略、路由 |
| `Connection refused` | 目标可达但端口拒绝 | 查服务进程或本机防火墙 |
| `No route to host` | 本机路由或邻居解析异常 | 查路由、ARP、接口状态 |
| `NXDOMAIN` | 域名不存在 | 查 DNS 记录或域名拼写 |
| `SERVFAIL` | DNS 服务器处理失败 | 查 DNS 转发、递归、上游 |

## B.7 抓包最小过滤条件

抓包时不要一开始就抓全量流量。先限定主机、端口和方向。

| 目标 | tcpdump 示例 |
| --- | --- |
| 某主机全部流量 | `tcpdump -i eth0 -nn host 10.28.60.20` |
| HTTPS 流量 | `tcpdump -i eth0 -nn tcp port 443` |
| DHCP 流量 | `tcpdump -i eth0 -nn "udp port 67 or udp port 68"` |
| DNS 流量 | `tcpdump -i eth0 -nn port 53` |
| ICMP 流量 | `tcpdump -i eth0 -nn icmp` |
| 某源到某目的 | `tcpdump -i eth0 -nn "src 10.28.10.25 and dst 10.28.60.20"` |

如果在客户端能看到请求发出，在服务器看不到请求，故障在中间网络路径或安全策略。如果服务器能看到请求并回包，但客户端收不到回包，重点查回程路径、防火墙状态表和 NAT。

## B.8 自查题

1. 为什么能 ping 通服务器，不代表业务端口一定正常？
2. 终端拿到 `169.254.x.x` 时，应按什么顺序排查？
3. `nslookup` 指定 DNS 服务器查询有什么价值？
4. `tcpdump` 中只看到 SYN 不见 SYN-ACK，通常说明什么？
5. 跨网段访问失败时，为什么必须检查回程路由？
