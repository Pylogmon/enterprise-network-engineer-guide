# 附录 C：常用交换机命令速查

本附录整理交换机日常配置、验证和排错中最常用的命令。不同厂商命令格式不同，但排查思路相同：

```text
接口状态 -> VLAN 归属 -> Trunk 放行 -> MAC 学习 -> 网关/ARP -> STP/聚合 -> 路由/ACL
```

本附录以华为 VRP 命令为主，同时给出 Cisco/H3C 风格的常见等价命令。真实项目中应以设备型号和软件版本为准。

## C.1 命令视图对照

| 操作目标 | 华为 VRP | Cisco IOS 风格 | H3C Comware 风格 |
| --- | --- | --- | --- |
| 进入配置模式 | `system-view` | `configure terminal` | `system-view` |
| 返回上一层 | `quit` | `exit` | `quit` |
| 保存配置 | `save` | `write memory` 或 `copy running-config startup-config` | `save` |
| 查看运行配置 | `display current-configuration` | `show running-config` | `display current-configuration` |
| 删除配置 | `undo ...` | `no ...` | `undo ...` |

初学者最常见的问题是“命令视图错误”。例如，端口 VLAN 配置必须进入接口视图；VLANIF 地址必须进入 VLANIF/三层接口视图。

## C.2 基础状态检查

| 目标 | 华为 VRP | Cisco IOS 风格 | H3C Comware 风格 |
| --- | --- | --- | --- |
| 查看版本和运行时间 | `display version` | `show version` | `display version` |
| 查看设备时间 | `display clock` | `show clock` | `display clock` |
| 查看当前配置 | `display current-configuration` | `show running-config` | `display current-configuration` |
| 查看已保存配置 | `display saved-configuration` | `show startup-config` | `display saved-configuration` |
| 查看日志 | `display logbuffer` | `show logging` | `display logbuffer` |
| 查看接口摘要 | `display interface brief` | `show ip interface brief` 或 `show interfaces status` | `display interface brief` |

排障时建议先执行：

```text
display interface brief
display vlan
display port vlan
display mac-address
display logbuffer
```

这几条命令能快速判断“链路是否 up、VLAN 是否存在、端口是否归属正确、是否学到 MAC、设备是否记录异常日志”。

## C.3 VLAN 创建与查看

### 华为 VRP

```text
system-view
vlan batch 10 20 30 60 80 250
vlan 10
 description OFFICE
quit
vlan 20
 description RD
quit
```

查看：

```text
display vlan
display vlan 10
```

### Cisco IOS 风格

```text
configure terminal
vlan 10
 name OFFICE
exit
vlan 20
 name RD
exit
```

查看：

```text
show vlan brief
show vlan id 10
```

### 排错关注点

| 现象 | 检查点 |
| --- | --- |
| 端口配置了 VLAN 但业务不通 | VLAN 是否已经创建 |
| 某 VLAN 跨交换机不通 | 两端 Trunk 是否都放行该 VLAN |
| 管理 VLAN 无法访问 | VLAN、VLANIF、Trunk、默认路由是否完整 |

## C.4 Access 端口配置

Access 端口通常连接终端、打印机、摄像头、AP 管理口或服务器单 VLAN 网卡。一个 Access 端口只属于一个业务 VLAN。

### 华为 VRP

```text
interface GigabitEthernet0/0/1
 description OFFICE-PC-001
 port link-type access
 port default vlan 10
 stp edged-port enable
quit
```

### Cisco IOS 风格

```text
interface GigabitEthernet1/0/1
 description OFFICE-PC-001
 switchport mode access
 switchport access vlan 10
 spanning-tree portfast
exit
```

### H3C Comware 风格

```text
interface GigabitEthernet1/0/1
 description OFFICE-PC-001
 port link-mode bridge
 port link-type access
 port access vlan 10
 stp edged-port
quit
```

### 查看端口 VLAN

| 厂商风格 | 命令 |
| --- | --- |
| 华为 VRP | `display port vlan GigabitEthernet0/0/1` |
| Cisco IOS | `show interfaces GigabitEthernet1/0/1 switchport` |
| H3C Comware | `display interface GigabitEthernet1/0/1 brief`、`display vlan 10` |

如果终端拿不到地址，优先检查 Access VLAN 是否正确，其次检查上联 Trunk、DHCP 和网关。

## C.5 Trunk 端口配置

Trunk 端口通常用于交换机之间、交换机到防火墙、交换机到虚拟化服务器、交换机到无线 AP 或无线控制器的多 VLAN 承载。

### 华为 VRP

```text
interface GigabitEthernet0/0/24
 description TO-CORE
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 80 250
quit
```

### Cisco IOS 风格

```text
interface GigabitEthernet1/0/48
 description TO-CORE
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,80,250
exit
```

### H3C Comware 风格

```text
interface GigabitEthernet1/0/48
 description TO-CORE
 port link-type trunk
 port trunk permit vlan 10 20 30 80 250
quit
```

### Trunk 排错表

| 现象 | 常见原因 | 检查命令 |
| --- | --- | --- |
| VLAN 10 通，VLAN 20 不通 | Trunk 只放行了 VLAN 10 | `display port vlan`、`show interfaces trunk` |
| 本交换机同 VLAN 通，跨交换机不通 | 上联 Trunk 未放行或对端配置不一致 | 两端都查 Trunk |
| 管理地址不通 | 管理 VLAN 未放行 | 查管理 VLAN 和上联 Trunk |
| 接 AP 后 SSID 不通 | AP 上联 VLAN/Native/PVID 不匹配 | 查 AP 端口模式和无线 VLAN 映射 |

Trunk 必须两端同时检查。只看本端配置，很容易漏掉对端未放行 VLAN 的问题。

## C.6 VLANIF/SVI 网关配置

三层交换机上常通过 VLANIF/SVI 为 VLAN 提供网关。

### 华为 VRP

```text
interface Vlanif10
 description GW-OFFICE
 ip address 10.28.10.1 255.255.255.0
quit
```

### Cisco IOS 风格

```text
interface Vlan10
 description GW-OFFICE
 ip address 10.28.10.1 255.255.255.0
 no shutdown
exit
```

### H3C Comware 风格

```text
interface Vlan-interface10
 description GW-OFFICE
 ip address 10.28.10.1 255.255.255.0
quit
```

查看：

| 厂商风格 | 命令 |
| --- | --- |
| 华为 VRP | `display ip interface brief`、`display interface Vlanif10` |
| Cisco IOS | `show ip interface brief`、`show interface Vlan10` |
| H3C Comware | `display ip interface brief`、`display interface Vlan-interface10` |

VLANIF/SVI up 通常需要 VLAN 存在，并且该 VLAN 至少有一个活动二层端口或 Trunk 承载路径。不同设备对 up/down 条件可能略有差异。

## C.7 MAC 地址表和 ARP 检查

MAC 地址表说明交换机从哪个端口学到了某个终端的二层地址。ARP 表说明三层网关是否解析到某个 IP 对应的 MAC。

| 目标 | 华为 VRP | Cisco IOS 风格 | H3C Comware 风格 |
| --- | --- | --- | --- |
| 查看全部 MAC | `display mac-address` | `show mac address-table` | `display mac-address` |
| 查看某 VLAN MAC | `display mac-address vlan 10` | `show mac address-table vlan 10` | `display mac-address vlan 10` |
| 查看某端口 MAC | `display mac-address interface GigabitEthernet0/0/1` | `show mac address-table interface Gi1/0/1` | `display mac-address interface GigabitEthernet1/0/1` |
| 查看 ARP | `display arp` | `show ip arp` | `display arp` |
| 查看某 VLAN ARP | `display arp vlan 10` | 视平台而定 | `display arp vlan 10` |

判断方法：

| 现象 | 含义 |
| --- | --- |
| 端口 up 但没有 MAC | 终端未发包、VLAN 错、认证未通过或链路异常 |
| MAC 在两个端口来回变化 | 可能有环路、错误接线或双上联未聚合 |
| 有 MAC 但无 ARP | 二层到达但三层网关未收到/未解析 IP |
| ARP 表项错误 | IP 冲突、欺骗、静态绑定错误 |

## C.8 STP 检查

STP 用于防止二层环路。生产网络中，根桥位置、端口角色和边缘端口配置都需要检查。

| 目标 | 华为 VRP | Cisco IOS 风格 | H3C Comware 风格 |
| --- | --- | --- | --- |
| 查看 STP 摘要 | `display stp brief` | `show spanning-tree summary` | `display stp brief` |
| 查看详细 STP | `display stp` | `show spanning-tree` | `display stp` |
| 配置边缘端口 | `stp edged-port enable` | `spanning-tree portfast` | `stp edged-port` |

常见检查点：

| 检查项 | 正常期望 |
| --- | --- |
| 根桥位置 | 核心或汇聚设备成为根桥 |
| 接入口 | 终端口为边缘端口，不应学习到 BPDU |
| 上联口 | Trunk/聚合口参与 STP，阻塞点符合设计 |
| 日志 | 无频繁拓扑变化、MAC 漂移、环路告警 |

如果出现全网变慢、广播暴增、交换机 CPU 高、MAC 表频繁变化，应立即检查 STP、环路和错误接线。

## C.9 链路聚合检查

链路聚合用于把多条物理链路组成一条逻辑链路，提高带宽并提供冗余。

### 华为 VRP LACP 示例

```text
interface Eth-Trunk1
 description TO-CORE
 mode lacp-static
 port link-type trunk
 port trunk allow-pass vlan 10 20 30 80 250
quit

interface GigabitEthernet0/0/23
 eth-trunk 1
quit
interface GigabitEthernet0/0/24
 eth-trunk 1
quit
```

### Cisco IOS 风格示例

```text
interface range GigabitEthernet1/0/47-48
 channel-group 1 mode active
exit

interface Port-channel1
 description TO-CORE
 switchport mode trunk
 switchport trunk allowed vlan 10,20,30,80,250
exit
```

### 查看命令

| 厂商风格 | 命令 |
| --- | --- |
| 华为 VRP | `display eth-trunk 1`、`display lacp statistics eth-trunk 1` |
| Cisco IOS | `show etherchannel summary`、`show interfaces port-channel 1` |
| H3C Comware | `display link-aggregation verbose`、`display interface Bridge-Aggregation1` |

聚合失败常见原因：

- 两端模式不一致，例如一端静态、一端 LACP。
- 成员口速率、双工、光模块或物理链路异常。
- 成员口已有普通接口配置未清理。
- 两端接线交叉到不同设备，但没有做堆叠、IRF、M-LAG 等多机聚合设计。

## C.10 接口错误计数检查

接口 up 不代表链路质量正常。丢包、CRC、错误帧和速率双工异常都会造成业务卡顿。

| 目标 | 华为 VRP | Cisco IOS 风格 | H3C Comware 风格 |
| --- | --- | --- | --- |
| 查看接口详细信息 | `display interface GigabitEthernet0/0/1` | `show interfaces GigabitEthernet1/0/1` | `display interface GigabitEthernet1/0/1` |
| 清除计数器 | `reset counters interface GigabitEthernet0/0/1` | `clear counters GigabitEthernet1/0/1` | `reset counters interface GigabitEthernet1/0/1` |

常见字段含义：

| 字段 | 可能原因 |
| --- | --- |
| CRC 增加 | 网线、光模块、光纤、速率双工、接口硬件问题 |
| Input errors | 入方向错误帧、物理层质量问题 |
| Output drops | 出方向拥塞、队列丢弃 |
| Broadcast/Multicast 异常高 | 环路、广播风暴、异常应用 |
| Flap 日志 | 线缆松动、模块异常、对端重启、电源问题 |

处理链路质量问题时，建议先记录计数器，再清零观察一段时间，确认错误是否持续增长。

## C.11 DHCP 相关交换机检查

终端 DHCP 获取失败时，交换机侧通常要检查 VLAN、Trunk、网关和 Relay。

| 目标 | 华为 VRP 示例 |
| --- | --- |
| 查看端口 VLAN | `display port vlan GigabitEthernet0/0/1` |
| 查看 VLAN 是否存在 | `display vlan 10` |
| 查看上联 Trunk | `display port vlan GigabitEthernet0/0/24` |
| 查看 VLANIF 状态 | `display interface Vlanif10` |
| 查看 DHCP 地址池 | `display ip pool` |
| 查看已分配地址 | `display ip pool name VLAN10-OFFICE used` |
| 查看 ARP | `display arp vlan 10` |

如果 DHCP 服务器不在本 VLAN，需要在三层网关配置 DHCP Relay。否则客户端广播无法跨三层转发到 DHCP 服务器。

## C.12 ACL 和端口安全检查

有时链路、VLAN 和路由都正常，但业务仍不通，原因可能是 ACL、端口安全、802.1X、MAC 绑定或 DHCP Snooping 等安全功能。

| 目标 | 华为 VRP | Cisco IOS 风格 | H3C Comware 风格 |
| --- | --- | --- | --- |
| 查看 ACL | `display acl all` | `show access-lists` | `display acl all` |
| 查看接口配置 | `display current-configuration interface GigabitEthernet0/0/1` | `show running-config interface Gi1/0/1` | `display current-configuration interface GigabitEthernet1/0/1` |
| 查看端口安全 | 视型号而定 | `show port-security interface Gi1/0/1` | 视型号而定 |
| 查看 802.1X | 视型号而定 | `show authentication sessions` | 视型号而定 |

排查建议：

1. 先确认故障是否只影响某个用户、某个 VLAN、某个方向或某类业务。
2. 再查接口上是否应用 ACL、认证、安全绑定或限速策略。
3. 查看命中计数或日志，确认是否真的被策略匹配。
4. 不要直接删除安全策略，应先确认业务需求和变更窗口。

## C.13 常见交换机故障快速定位

| 现象 | 第一批检查命令 | 重点判断 |
| --- | --- | --- |
| 单台终端不通 | `display interface brief`、`display port vlan`、`display mac-address interface ...` | 端口 up、VLAN、MAC 学习 |
| 整个 VLAN 不通 | `display vlan`、`display interface VlanifX`、`display arp vlan X` | VLANIF、网关、Trunk |
| 跨交换机同 VLAN 不通 | 两端 `display port vlan`、`display mac-address vlan X` | Trunk 放行和 MAC 学习方向 |
| 终端拿不到 DHCP | `display port vlan`、`display ip pool`、`display interface VlanifX` | VLAN、地址池、Relay |
| 网络时断时续 | `display logbuffer`、`display stp brief`、接口详细信息 | 端口 flap、STP 变化、错误计数 |
| 全网变慢 | `display stp brief`、`display mac-address`、接口流量 | 环路、广播风暴、拥塞 |
| 聚合口只有一条链路工作 | `display eth-trunk`、接口详细信息 | LACP、成员口、速率双工 |
| 管理地址无法登录 | `display ip interface brief`、`display current-configuration`、路由表 | 管理 VLAN、SSH、ACL、默认路由 |

## C.14 上线变更前检查清单

| 检查项 | 要求 |
| --- | --- |
| 端口描述 | 上联、服务器、AP、重要终端端口应写清楚 |
| VLAN 创建 | 所有业务 VLAN、管理 VLAN 已创建 |
| Access 端口 | VLAN 归属和边缘端口符合规划 |
| Trunk 端口 | 两端允许 VLAN 一致，管理 VLAN 未遗漏 |
| VLANIF 网关 | IP 地址、掩码、up/down 状态正确 |
| 路由 | 默认路由、回程路由或动态路由符合设计 |
| STP | 根桥位置、边缘端口、阻塞点符合设计 |
| 聚合 | 成员链路、LACP 状态、Trunk 配置正常 |
| 安全策略 | ACL、端口安全、认证策略符合变更需求 |
| 保存配置 | 业务验证正常后再保存 |
| 回退方案 | 已记录变更前配置和回退步骤 |

## C.15 自查题

1. 为什么排查 VLAN 不通时必须同时检查 Access 端口和 Trunk 端口？
2. 端口 up 但学不到 MAC，可能有哪些原因？
3. VLANIF 配置了 IP 地址但状态 down，应检查哪些内容？
4. 聚合口中只有一条成员链路正常，应从哪些维度排查？
5. 为什么不能在故障排查时随意删除 ACL 或安全策略？
