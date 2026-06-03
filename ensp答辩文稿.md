# 华为eNSP园区网项目答辩文稿

---

## 一、项目定位

### 1.1 组网类型判断

本项目的组网类型为**中小型企业园区公司局域网**。

依据如下：
- 网络规模：2台接入交换机（LSW1、LSW2） + 2台汇聚交换机（LSW3、LSW4，VRRP主备） + 2台路由器（R1出口、R2模拟ISP）
- 终端规模：4台内网PC（PC1~PC4）、1台内网服务器（Server1）、1台外网PC（PC5）
- 采用典型的三层园区网架构：**接入层 → 汇聚层 → 出口路由层**

### 1.2 三层组网架构

| 层级 | 设备 | 功能职责 |
|------|------|----------|
| **接入层** | LSW1、LSW2 | 连接终端PC，划分VLAN，提供端口接入 |
| **汇聚层** | LSW3（主）、LSW4（备） | VLAN间路由（三层VLANIF）、DHCP服务、VRRP网关冗余、STP防环、OSPF路由 |
| **出口路由层** | R1（企业出口路由器） | NAT地址转换、ACL访问控制、OSPF路由互联、连接ISP |
| **ISP/外网** | R2（模拟运营商）、PC5（外网主机） | 模拟公网环境，验证内网访问互联网 |

### 1.3 项目整体业务需求

本项目模拟了一个中小型企业的实际组网需求，要点如下：

1. **部门隔离与互通**：通过VLAN划分不同部门/区域，通过三层VLANIF实现跨VLAN通信
2. **DHCP自动分配IP**：PC1~PC4通过DHCP自动获取IP地址，减少运维工作量
3. **网关高可用（VRRP）**：汇聚层采用双设备VRRP主备模式，单台汇聚交换机故障不影响终端网关
4. **链路冗余与STP防环**：接入层双链路连接汇聚层，STP防止二层环路
5. **内部服务器对外发布**：Server1（192.168.50.100）通过NAT静态映射对外提供服务
6. **内网用户访问互联网**：通过ACL控制允许访问外网的网段，通过NAT实现私网地址转换
7. **全网路由可达**：通过OSPF动态路由协议实现内网所有网段互通

---

## 二、拓扑梳理

### 2.1 完整拓扑链路（层级文字结构）

```
┌─────────────────────────────────────────────────────────────────────────┐
│                             外网 PC5                                    │
│                        IP:200.200.201.100/24                           │
│                        网关:200.200.201.1                              │
└────────────────────────────────┬────────────────────────────────────────┘
                                 │
                            [GE0/0/1]
┌───────────────────────────────┴────────────────────────────────────────┐
│                          R2（模拟ISP/互联网）                           │
│                    GE0/0/0: 200.200.200.100/24                        │
└───────────────────────────────┬────────────────────────────────────────┘
                                │ 200.200.200.0/24
                    ┌───────────┴───────────┐
                    │   GE0/0/2             │
                    │  200.200.200.1/24     │
┌───────────────────┴───────────────────────┴───────────────────────────┐
│                 R1（企业出口路由器 · 核心路由器）                       │
│  GE0/0/0: 192.168.60.100/24        GE0/0/1: 192.168.70.100/24       │
└───────┬──────────────────────────────────────────────┬────────────────┘
        │                                              │
        │ 192.168.60.0/24                              │ 192.168.70.0/24
        │ VLAN 60                                      │ VLAN 70
        │                                              │
┌───────┴──────────────────────┐  ┌────────────────────┴───────────────┐
│ LSW3（汇聚主 · VRRP Master）  │  │ LSW4（汇聚备 · VRRP Backup）       │
│ GE0/0/4→VLAN60(access)      │  │ GE0/0/4→VLAN70(access)             │
│ GE0/0/5→VLAN50(access)      │  │                                     │
│ VLANIF10: 192.168.10.1/24   │  │ VLANIF10: 192.168.10.2/24          │
│ VLANIF20: 192.168.20.1/24   │  │ VLANIF20: 192.168.20.2/24          │
│ VLANIF30: 192.168.30.1/24   │  │ VLANIF30: 192.168.30.2/24          │
│ VLANIF40: 192.168.40.1/24   │  │ VLANIF40: 192.168.40.2/24          │
│ VLANIF50: 192.168.50.1/24   │  │ VLANIF70: 192.168.70.2/24          │
│ VLANIF60: 192.168.60.1/24   │  │                                     │
└────┬──────────────┬─────────┘  └────────┬──────────────┬────────────┘
     │              │                      │              │
     │ Trunk        │ Trunk                │ Trunk        │ Trunk
     │ VLAN ALL     │ VLAN ALL             │ VLAN ALL     │ VLAN ALL
     │              │                      │              │
┌────┴──────┐ ┌────┴──────┐      ┌───────┴────┐  ┌──────┴──────┐
│  LSW1     │ │  LSW2     │      │  Server1   │  │  (预留)      │
│ (接入层)   │ │ (接入层)   │      │(静态IP)    │  │             │
└───┬───┬───┘ └───┬───┬───┘      │192.168.50. │  │             │
    │   │         │   │          │   100      │  │             │
  E0/0/1 E0/0/2  E0/0/1 E0/0/2   └────────────┘  └────────────┘
   VLAN10 VLAN20  VLAN30 VLAN40
   (PC1)  (PC2)   (PC3)  (PC4)
   DHCP   DHCP    DHCP   DHCP
```

### 2.2 全部VLAN、网段与接口对应表

| VLAN ID | 网段 | 用途 | 网关（虚拟IP） | 设备连接方式 | IP获取方式 |
|---------|------|------|---------------|-------------|-----------|
| **VLAN 10** | 192.168.10.0/24 | 部门A（PC1） | 192.168.10.254 | LSW1 E0/0/1(Access) → PC1 | **DHCP自动获取** |
| **VLAN 20** | 192.168.20.0/24 | 部门B（PC2） | 192.168.20.254 | LSW1 E0/0/2(Access) → PC2 | **DHCP自动获取** |
| **VLAN 30** | 192.168.30.0/24 | 部门C（PC3） | 192.168.30.254 | LSW2 E0/0/1(Access) → PC3 | **DHCP自动获取** |
| **VLAN 40** | 192.168.40.0/24 | 部门D（PC4） | 192.168.40.254 | LSW2 E0/0/2(Access) → PC4 | **DHCP自动获取** |
| **VLAN 50** | 192.168.50.0/24 | 服务器区域（Server1） | 192.168.50.1 | LSW3 GE0/0/5(Access) → Server1 | **静态IP**（192.168.50.100/24, 网关192.168.50.1） |
| **VLAN 60** | 192.168.60.0/24 | LSW3→R1互联 | 无VRRP | LSW3 GE0/0/4(Access) / R1 GE0/0/0 | **静态IP** |
| **VLAN 70** | 192.168.70.0/24 | LSW4→R1互联 | 无VRRP | LSW4 GE0/0/4(Access) / R1 GE0/0/1 | **静态IP** |
| **200.200.200.0/24** | — | R1→R2互联（公网段） | — | R1 GE0/0/2 / R2 GE0/0/0 | **静态IP** |
| **200.200.201.0/24** | — | 外网PC5所在网段 | 200.200.201.1 | R2 GE0/0/1 → PC5 | **静态IP**（PC5: 200.200.201.100/24） |

### 2.3 互联接口详细一览

| 设备A | 接口A | 接口模式 | 设备B | 接口B | 接口模式 | 通过VLAN |
|-------|-------|---------|-------|-------|---------|---------|
| LSW1 | E0/0/1 | Access VLAN 10 | PC1 | — | — | VLAN 10 |
| LSW1 | E0/0/2 | Access VLAN 20 | PC2 | — | — | VLAN 20 |
| LSW2 | E0/0/1 | Access VLAN 30 | PC3 | — | — | VLAN 30 |
| LSW2 | E0/0/2 | Access VLAN 40 | PC4 | — | — | VLAN 40 |
| LSW1 | E0/0/3 | Trunk | LSW3 | GE0/0/1 | Trunk | All |
| LSW1 | E0/0/4 | Trunk | LSW4 | GE0/0/1 | Trunk | All |
| LSW2 | E0/0/3 | Trunk | LSW3 | GE0/0/2 | Trunk | All |
| LSW2 | E0/0/4 | Trunk | LSW4 | GE0/0/2 | Trunk | All |
| LSW3 | GE0/0/3 | Trunk | LSW4 | GE0/0/3 | Trunk | All |
| LSW3 | GE0/0/4 | Access VLAN 60 | R1 | GE0/0/0 | — | VLAN 60 |
| LSW3 | GE0/0/5 | Access VLAN 50 | Server1 | — | — | VLAN 50 |
| LSW4 | GE0/0/4 | Access VLAN 70 | R1 | GE0/0/1 | — | VLAN 70 |
| R1 | GE0/0/2 | — | R2 | GE0/0/0 | — | 公网 |
| R2 | GE0/0/1 | — | PC5 | — | — | 公网 |

---

## 三、命令逐条解析（按设备分类）

### 3.1 LSW1（接入交换机，管理VLAN 10、20）

```bash
# === 基础配置 ===
sysname LSW1                       # 设置设备名称为LSW1
undo info-center enable             # 关闭信息中心（减少日志输出，简化演示）

# === VLAN配置 ===
vlan batch 10 20                    # 批量创建VLAN 10和VLAN 20
# VLAN 10：PC1所在部门A
# VLAN 20：PC2所在部门B

# === STP生成树 ===
stp mode stp                        # 启用STP生成树协议（IEEE 802.1D）
# 作用：防止二层环路。LSW1双链路上行到LSW3和LSW4，STP自动阻塞冗余端口

# === 接口配置（面向PC——Access端口） ===
interface Ethernet0/0/1
 port link-type access               # 接口模式：Access（接入链路）
 port default vlan 10                # 划分到VLAN 10，连接PC1

interface Ethernet0/0/2
 port link-type access
 port default vlan 20                # 划分到VLAN 20，连接PC2

# === 接口配置（上行汇聚层——Trunk端口） ===
interface Ethernet0/0/3
 port link-type trunk                # 接口模式：Trunk（干道/中继链路）
 port trunk allow-pass vlan 2 to 4094  # 允许所有VLAN通过（透传）
# 连接LSW3 GE0/0/1

interface Ethernet0/0/4
 port link-type trunk
 port trunk allow-pass vlan 2 to 4094  # 连接LSW4 GE0/0/1
# LSW1双链路分别上联LSW3和LSW4，实现链路冗余
```

### 3.2 LSW2（接入交换机，管理VLAN 30、40）

```bash
# === 基础配置 ===
sysname LSW2
undo info-center enable

# === VLAN配置 ===
vlan batch 30 40                    # 批量创建VLAN 30和VLAN 40
# VLAN 30：PC3所在部门C
# VLAN 40：PC4所在部门D

# === STP生成树 ===
stp mode stp                        # 启用STP

# === 接口配置（面向PC——Access端口） ===
interface Ethernet0/0/1
 port link-type access
 port default vlan 30                # 连接PC3

interface Ethernet0/0/2
 port link-type access
 port default vlan 40                # 连接PC4

# === 接口配置（上行汇聚层——Trunk端口） ===
interface Ethernet0/0/3
 port link-type trunk
 port trunk allow-pass vlan 2 to 4094  # 连接LSW3 GE0/0/2

interface Ethernet0/0/4
 port link-type trunk
 port trunk allow-pass vlan 2 to 4094  # 连接LSW4 GE0/0/2
```

### 3.3 LSW3（汇聚交换机主设备——VRRP Master for VLAN 10/20，DHCP服务器，OSPF路由）

```bash
# === 基础配置 ===
sysname LSW3
undo info-center enable
vlan batch 10 20 30 40 50 60        # 创建所有涉及VLAN
# VLAN 50：服务器Server1
# VLAN 60：上行到R1的互联链路

# === STP生成树 ===
stp mode stp                        # STP模式
stp instance 0 priority 0           # 设置STP优先级为0（最高优先级）
# LSW3作为STP根桥，确保网络拓扑稳定

# === DHCP服务 ===
dhcp enable                         # 全局开启DHCP服务

# === DHCP地址池 ===
ip pool vlan10
 gateway-list 192.168.10.254         # 分配给客户端的网关地址（VRRP虚拟IP）
 network 192.168.10.0 mask 255.255.255.0  # 分配网段
 excluded-ip-address 192.168.10.1 192.168.10.127  # 排除地址段
# LSW3使用192.168.10.1（VLANIF10），排除的地址给LSW4（.2）、静态设备、网管使用
# PC1从128~253范围获取地址，实际PC1分配到.253

ip pool vlan20
 gateway-list 192.168.20.254
 network 192.168.20.0 mask 255.255.255.0
 excluded-ip-address 192.168.20.1 192.168.20.127

ip pool vlan30
 gateway-list 192.168.30.254
 network 192.168.30.0 mask 255.255.255.0
 excluded-ip-address 192.168.30.1 192.168.30.127

ip pool vlan40
 gateway-list 192.168.40.254
 network 192.168.40.0 mask 255.255.255.0
 excluded-ip-address 192.168.40.1 192.168.40.127
# 注意LSW3的排除段是1~127，LSW4的排除段是128~253
# 两设备排除段互补，DHCP分配互不冲突

# === 三层VLANIF接口（核心路由+VRRP+DHCP） ===

# VLAN 10（PC1部门）
interface Vlanif10
 ip address 192.168.10.1 255.255.255.0       # LSW3本机IP（VRRP中Master实际地址）
 vrrp vrid 10 virtual-ip 192.168.10.254      # VRRP虚拟IP（作为PC的网关）
 vrrp vrid 10 priority 120                   # 优先级120（默认100），LSW3为Master
 vrrp vrid 10 track interface GigabitEthernet0/0/4 reduced 30  # 上行链路跟踪
 # 当GE0/0/4（连R1）故障时，优先级减30变为90，低于LSW4的100，自动切换
 dhcp select global                          # 从全局地址池分配IP

# VLAN 20（PC2部门）
interface Vlanif20
 ip address 192.168.20.1 255.255.255.0
 vrrp vrid 20 virtual-ip 192.168.20.254
 vrrp vrid 20 priority 120
 vrrp vrid 20 track interface GigabitEthernet0/0/4 reduced 30
 dhcp select global

# VLAN 30（PC3部门）——LSW3为Backup
interface Vlanif30
 ip address 192.168.30.1 255.255.255.0
 vrrp vrid 30 virtual-ip 192.168.30.254
 # 优先级保持默认100，LSW4优先级120为Master
 dhcp select global

# VLAN 40（PC4部门）——LSW3为Backup
interface Vlanif40
 ip address 192.168.40.1 255.255.255.0
 vrrp vrid 40 virtual-ip 192.168.40.254
 dhcp select global

# VLAN 50（服务器区域）
interface Vlanif50
 ip address 192.168.50.1 255.255.255.0       # Server1的网关
 # Server1为静态IP: 192.168.50.100/24 网关192.168.50.1

# VLAN 60（上行到R1互联）
interface Vlanif60
 ip address 192.168.60.1 255.255.255.0       # LSW3端互联地址
 # R1端为192.168.60.100

# === 二层接口 ===
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 2 to 4094        # 连LSW1 E0/0/3

interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 2 to 4094        # 连LSW2 E0/0/3

interface GigabitEthernet0/0/3
 port link-type trunk
 port trunk allow-pass vlan 2 to 4094        # 连LSW4 GE0/0/3（汇聚间互联）

interface GigabitEthernet0/0/4
 port link-type access
 port default vlan 60                        # 连R1 GE0/0/0，属于VLAN 60

interface GigabitEthernet0/0/5
 port link-type access
 port default vlan 50                        # 连Server1，属于VLAN 50

# === OSPF动态路由 ===
ospf 1                                       # 进程号1
 default-route-advertise                     # 下发默认路由给其他OSPF邻居
 area 0.0.0.0                                # 骨干区域Area 0
  network 192.168.10.0 0.0.0.255
  network 192.168.20.0 0.0.0.255
  network 192.168.30.0 0.0.0.255
  network 192.168.40.0 0.0.0.255
  network 192.168.50.0 0.0.0.255
  network 192.168.60.0 0.0.0.255
# 宣告所有内网网段，全网路由可达

# === 默认路由 ===
ip route-static 0.0.0.0 0.0.0.0 192.168.60.100
# 默认路由指向R1的GE0/0/0接口，内网访问外网的流量转发给R1处理
```

### 3.4 LSW4（汇聚交换机备份——VRRP Master for VLAN 30/40）

```bash
sysname LSW4
undo info-center enable
vlan batch 10 20 30 40 60 70        # 注意LSW4有VLAN 70（连R1），没有VLAN 50

# === STP ===
stp instance 0 priority 4096        # 优先级4096，作为STP备用根桥（LSW3优先级0为主根）

# === DHCP ===
dhcp enable

# === DHCP地址池（排除段与LSW3互补） ===
ip pool vlan10
 gateway-list 192.168.10.254
 network 192.168.10.0 mask 255.255.255.0
 excluded-ip-address 192.168.10.128 192.168.10.253   # 排除128~253

ip pool vlan20
 gateway-list 192.168.20.254
 network 192.168.20.0 mask 255.255.255.0
 excluded-ip-address 192.168.20.128 192.168.20.253

ip pool vlan30
 gateway-list 192.168.30.254
 network 192.168.30.0 mask 255.255.255.0
 excluded-ip-address 192.168.30.128 192.168.30.253

ip pool vlan40
 gateway-list 192.168.40.254
 network 192.168.40.0 mask 255.255.255.0
 excluded-ip-address 192.168.40.128 192.168.40.253
# LSW3排除1~127，LSW4排除128~253，两端互补，互不冲突

# === 三层VLANIF ===

# VLAN 10（LSW4为Backup）
interface Vlanif10
 ip address 192.168.10.2 255.255.255.0
 vrrp vrid 10 virtual-ip 192.168.10.254      # VRRP优先级默认100
 dhcp select global

# VLAN 20（LSW4为Backup）
interface Vlanif20
 ip address 192.168.20.2 255.255.255.0
 vrrp vrid 20 virtual-ip 192.168.20.254
 dhcp select global

# VLAN 30（LSW4为Master，优先级120）
interface Vlanif30
 ip address 192.168.30.2 255.255.255.0
 vrrp vrid 30 virtual-ip 192.168.30.254
 vrrp vrid 30 priority 120
 vrrp vrid 30 track interface GigabitEthernet0/0/4 reduced 30
 dhcp select global

# VLAN 40（LSW4为Master，优先级120）
interface Vlanif40
 ip address 192.168.40.2 255.255.255.0
 vrrp vrid 40 virtual-ip 192.168.40.254
 vrrp vrid 40 priority 120
 vrrp vrid 40 track interface GigabitEthernet0/0/4 reduced 30
 dhcp select global

# VLAN 70（上行到R1互联）
interface Vlanif70
 ip address 192.168.70.2 255.255.255.0       # LSW4端互联地址
 # R1 GE0/0/1为192.168.70.100

# === 二层Trunk接口 ===
interface GigabitEthernet0/0/1
 port link-type trunk
 port trunk allow-pass vlan 2 to 4094        # 连LSW1 E0/0/4

interface GigabitEthernet0/0/2
 port link-type trunk
 port trunk allow-pass vlan 2 to 4094        # 连LSW2 E0/0/4

interface GigabitEthernet0/0/3
 port link-type trunk
 port trunk allow-pass vlan 2 to 4094        # 连LSW3 GE0/0/3

interface GigabitEthernet0/0/4
 port link-type access
 port default vlan 70                        # 连R1 GE0/0/1

# === OSPF（无default-route-advertise，无默认路由） ===
ospf 1
 area 0.0.0.0
  network 192.168.10.0 0.0.0.255
  network 192.168.20.0 0.0.0.255
  network 192.168.30.0 0.0.0.255
  network 192.168.40.0 0.0.0.255
  network 192.168.70.0 0.0.0.255
```

### 3.5 R1（企业出口路由器——NAT转换、ACL控制、OSPF、默认路由）

```bash
sysname AR1
undo info-center enable

# === ACL访问控制列表（用于NAT匹配） ===
acl number 2000                                # 基本ACL 2000（只匹配源IP）
 rule 0 permit source 192.168.10.0 0.0.0.255   # 允许VLAN 10（PC1部门）
 rule 5 permit source 192.168.30.0 0.0.0.255   # 允许VLAN 30（PC3部门）
 rule 10 permit source 192.168.50.0 0.0.0.255  # 允许VLAN 50（服务器区）
 rule 15 permit source 192.168.40.0 0.0.0.255  # 允许VLAN 40（PC4部门）
# 说明：VLAN 20（PC2部门）未被ACL允许，PC2不能访问外网
#       这是模拟企业对某些部门限制互联网访问的典型场景

# === 接口IP配置 ===
interface GigabitEthernet0/0/0
 ip address 192.168.60.100 255.255.255.0      # 连接LSW3 GE0/0/4（VLAN 60）

interface GigabitEthernet0/0/1
 ip address 192.168.70.100 255.255.255.0      # 连接LSW4 GE0/0/4（VLAN 70）

interface GigabitEthernet0/0/2
 ip address 200.200.200.1 255.255.255.0       # 连接R2（ISP），公网地址
 nat static global 200.200.200.10 inside 192.168.50.100 netmask 255.255.255.255
 # 静态NAT一对一映射：外网访问200.200.200.10映射到内网Server1（192.168.50.100）
 # 实现服务器对外发布
 nat outbound 2000
 # 动态NAT（PAT端口地址转换）：匹配ACL 2000的源IP，转换为出接口200.200.200.1的公网IP
 # 实现内网用户访问互联网

# === OSPF动态路由 ===
ospf 1
 area 0.0.0.0
  network 192.168.60.0 0.0.0.255
  network 192.168.70.0 0.0.0.255
  network 200.200.200.0 0.0.0.255
# 注意：R1宣告了公网段200.200.200.0/24到OSPF中，使得LSW3/LSW4知道去往外网的路由

# === 静态路由 ===
ip route-static 0.0.0.0 0.0.0.0 200.200.200.100
# 默认路由指向R2（ISP），所有去往互联网的流量走此路由

ip route-static 192.168.0.0 255.255.0.0 192.168.60.1
# 回程路由：访问内网192.168.0.0/16的流量，下一跳指向LSW3（192.168.60.1）
# 结合OSPF，确保内网回程流量正确到达LSW3
```

### 3.6 R2（模拟ISP路由器）

```bash
sysname AR2
undo info-center enable

# === 接口配置 ===
interface GigabitEthernet0/0/0
 ip address 200.200.200.100 255.255.255.0     # 连接R1 GE0/0/2

interface GigabitEthernet0/0/1
 ip address 200.200.201.1 255.255.255.0       # 连接外网PC5

# === 静态路由 ===
ip route-static 192.168.0.0 255.255.0.0 200.200.200.1
# 回程路由：访问内网192.168.0.0/16的流量通过R1转发
```

### 3.7 终端设备汇总

| 设备 | IP地址 | 子网掩码 | 网关 | 获取方式 |
|------|--------|---------|------|---------|
| PC1 | 192.168.10.253 | 255.255.255.0 | 192.168.10.254 | DHCP自动获取 |
| PC2 | 192.168.20.253 | 255.255.255.0 | 192.168.20.254 | DHCP自动获取 |
| PC3 | 192.168.30.127 | 255.255.255.0 | 192.168.30.254 | DHCP自动获取 |
| PC4 | 192.168.40.127 | 255.255.255.0 | 192.168.40.254 | DHCP自动获取 |
| Server1 | 192.168.50.100 | 255.255.255.0 | 192.168.50.1 | 静态配置 |
| PC5（外网） | 200.200.201.100 | 255.255.255.0 | 200.200.201.1 | 静态配置 |

---

## 四、技术串联讲解（全业务流程）

### 4.1 业务流程总览

```
PC1→DHCP获取IP地址 → 通过VLANIF10三层转发 → VRRP网关冗余 → OSPF路由 → ACL访问控制 → NAT地址转换 → 访问互联网PC5
```

### 4.2 第一步：PC通过DHCP获取IP地址

**涉及技术：DHCP**

当PC1接入网络并设置为"自动获取IP"时：
1. PC1发送DHCP Discover广播，寻找DHCP服务器
2. 广播报文到达LSW1的E0/0/1（Access端口，VLAN 10），经Trunk上联到LSW3和LSW4
3. LSW3和LSW4的VLANIF10接口均配置了`dhcp select global`
4. DHCP服务器（LSW3或LSW4）从`ip pool vlan10`地址池中选取可用IP
5. 根据排除规则，PC1分配到192.168.10.253（DHCP地址池范围为128~253）
6. 同时分配给PC1：子网掩码255.255.255.0、网关192.168.10.254（VRRP虚拟IP）、DNS等

> **本项目特点**：LSW3与LSW4的地址池排除段互补，LSW3排除1~127、LSW4排除128~253，即使双服务器同时响应也不会冲突。当VRRP Master切换时，DHCP服务仍能正常提供。

### 4.3 第二步：依托三层VLANIF+VRRP实现跨VLAN互通

**涉及技术：VLAN、三层交换、VRRP**

内网共划分了6个VLAN（10/20/30/40/50/60），各VLAN间通信依赖汇聚交换机的三层VLANIF接口：

1. **VLAN间路由**：LSW3和LSW4各自配置了VLANIF10~VLANIF60接口，作为每个VLAN的网关
   - PC1（VLAN 10）需要访问Server1（VLAN 50）时
   - 数据包目标IP为192.168.50.100，不在同一网段
   - PC1将数据包发送到网关192.168.10.254（VRRP虚拟IP）
   - LSW3的VLANIF10收到数据，根据路由表查询到VLAN 50直连，通过VLANIF50转发
   - 最终Server1收到来自PC1的请求

2. **VRRP网关冗余**：每个用户VLAN（10/20/30/40）都配置了VRRP

   | VLAN | 虚拟IP | Master (优先级) | Backup (优先级) |
   |------|--------|-----------------|-----------------|
   | 10 | 192.168.10.254 | **LSW3** (120) | LSW4 (100) |
   | 20 | 192.168.20.254 | **LSW3** (120) | LSW4 (100) |
   | 30 | 192.168.30.254 | **LSW4** (120) | LSW3 (100) |
   | 40 | 192.168.40.254 | **LSW4** (120) | LSW3 (100) |

   - 采用**负载分担**设计：LSW3作为VLAN 10/20的主网关，LSW4作为VLAN 30/40的主网关
   - 上行链路跟踪机制：当Master的上行接口（GE0/0/4）故障时，优先级自动减30，触发角色切换
   - 切换时间短（约3~4秒），用户几乎无感知

### 4.4 第三步：OSPF保证内网全网段路由可达

**涉及技术：OSPF动态路由协议**

| 设备 | OSPF宣告网段 | 特殊配置 |
|------|-------------|---------|
| LSW3 | 192.168.10.0/24, 20.0/24, 30.0/24, 40.0/24, 50.0/24, 60.0/24 | `default-route-advertise` |
| LSW4 | 192.168.10.0/24, 20.0/24, 30.0/24, 40.0/24, 70.0/24 | 无特殊 |
| R1 | 192.168.60.0/24, 70.0/24, 200.200.200.0/24 | — |

- 所有设备运行在OSPF Area 0（骨干区域）
- LSW3作为默认路由的"通告者"：`default-route-advertise`将默认路由（指向R1）通告给LSW4和R1
- R1将公网网段200.200.200.0/24宣告到OSPF中，使得LSW3/LSW4知道去往外网的路径
- LSW3上有默认路由`0.0.0.0 0 → 192.168.60.100`（指向R1）
- 回程路由：R1上静态`192.168.0.0/16 → 192.168.60.1`（指向LSW3），R2上`192.168.0.0/16 → 200.200.200.1`（指向R1）

**路由流向示例**：
- PC1 (VLAN 10) ping PC5 (200.200.201.100)：
  PC1 → 网关192.168.10.254(VLANIF10 on LSW3) → 查路由 → 匹配默认路由 → 下一跳192.168.60.100(R1) → 查路由 → 下一跳200.200.200.100(R2) → 到达PC5

### 4.5 第四步：依靠ACL+NAT实现内网访问互联网

**涉及技术：ACL访问控制列表、NAT网络地址转换**

#### ACL — 访问控制

R1上配置了基本ACL 2000，用于**控制哪些内网网段允许访问外网**：

```
acl number 2000
 rule 0 permit source 192.168.10.0 0.0.0.255    # VLAN 10 ✓
 rule 5 permit source 192.168.30.0 0.0.0.255    # VLAN 30 ✓
 rule 10 permit source 192.168.50.0 0.0.0.255   # VLAN 50 ✓
 rule 15 permit source 192.168.40.0 0.0.0.255   # VLAN 40 ✓
```

**注意**：VLAN 20（192.168.20.0/24）**不在ACL允许范围内**！这意味着PC2无法访问互联网。这是模拟企业"某些部门限制互联网访问"的安全策略场景。

#### NAT — 地址转换

在R1的GE0/0/2接口（连接ISP）上配置了两种NAT：

1. **动态NAT（PAT端口地址转换）**
   ```
   interface GigabitEthernet0/0/2
    nat outbound 2000
   ```
   - 匹配ACL 2000的源地址（VLAN 10/30/40/50）
   - 转换为出接口GE0/0/2的IP 200.200.200.1
   - 使用端口多路复用（PAT），一个公网IP支撑多台内网主机同时上网
   - 解决私网地址无法在公网路由的问题

2. **静态NAT（一对一映射）**
   ```
   nat static global 200.200.200.10 inside 192.168.50.100 netmask 255.255.255.255
   ```
   - 公网IP 200.200.200.10 固定映射到内网服务器192.168.50.100
   - 互联网用户可以通过访问200.200.200.10来访问公司内部的Server1
   - 用于对外发布企业服务（如Web服务器、FTP服务器等）

### 4.6 五大技术在本项目中的实际用途总结

| 技术 | 本项目实际用途 |
|------|--------------|
| **VLAN** | 将内网划分为7个逻辑网段（VLAN 10~70），实现广播隔离、部门隔离，同时通过Trunk实现跨交换机的VLAN透传 |
| **DHCP** | PC1~PC4自动获取IP地址，无需手动配置，降低运维成本。双DHCP服务器冗余备份，地址池互不冲突 |
| **STP** | LSW1/LSW2双链路上行到LSW3/LSW4构成物理环路，STP自动阻塞冗余端口，防止广播风暴 |
| **VRRP** | LSW3和LSW4互为备份，每台设备承担一部分VLAN的Master角色，实现网关冗余+负载分担。上行链路检测确保故障自动切换 |
| **OSPF** | 在LSW3、LSW4、R1之间动态交换路由信息，实现全网段互通，LSW3下发默认路由使所有设备都知道如何访问外网 |
| **ACL** | 在R1上控制哪些内网网段可以访问互联网（VLAN 20被限制），实现企业安全策略 |
| **NAT** | 动态NAT使内网用户分享一个公网IP上网；静态NAT使内网Server1对外提供可访问的服务 |

---

## 五、答辩总结 & 易错点

### 5.1 本项目设计亮点

1. **链路冗余设计**
   - 接入交换机（LSW1/LSW2）双链路分别上联两台汇聚交换机（LSW3/LSW4）
   - 任意一条上行链路故障，STP自动切换，不影响终端通信

2. **网关备份（VRRP）**
   - 汇聚层部署VRRP虚拟网关，每台汇聚设备既是部分VLAN的Master、又是另一部分的Backup
   - 实现网关冗余的同时做到负载分担，设备利用率最大化
   - 上行链路跟踪（Track）确保故障检测并自动切换

3. **DHCP自动分配IP**
   - 终端PC全部采用DHCP自动获取，集中管理
   - 双DHCP服务器（LSW3和LSW4）冗余部署
   - 地址池排除段互补设计，避免地址冲突

4. **内网访问外网**
   - OSPF + 默认路由实现全网路由可达
   - ACL控制指定网段允许访问互联网（符合企业安全策略）
   - NAT实现私网→公网地址转换，节省公网IP资源
   - 静态NAT实现内部服务器对外发布

5. **服务器对外发布**
   - 通过NAT静态映射，将内网Server1（192.168.50.100）映射为公网IP 200.200.200.10
   - 外网用户通过公网IP即可访问企业服务器

### 5.2 实操高频配置故障

| 故障现象 | 可能原因 | 解决方案 |
|---------|---------|---------|
| PC获取不到IP地址 | DHCP地址池配置错误、VLANIF未启用dhcp select global、交换机间Trunk未放通VLAN | 检查地址池范围和排除段、检查dhcp enable全局开关、检查Trunk allow-pass vlan |
| PC能获取IP但ping不通网关 | VRRP虚拟IP网关地址配置与PC分配到的网关不一致 | 检查`ip pool`中的gateway-list是否等于VRRP virtual-ip |
| 跨VLAN通信不通 | VLANIF接口未配置IP、OSPF未宣告或宣告错误 | 检查VLANIF状态+IP、检查OSPF network宣告 |
| VRRP主备不切换 | Track配置错误、优先级设置不当 | 检查Track接口是否正确、priority是否合理 |
| 内网无法访问互联网 | ACL未放通、NAT未配置、OSPF未宣告公网段、默认路由缺失 | 检查ACL规则、检查nat outbound、检查OSPF宣告200.200.200.0、检查默认路由 |
| 外网无法访问内网服务器 | 静态NAT配置错误、回程路由缺失 | 检查nat static、检查R2回程路由、检查R1回程路由 |
| STP导致部分链路不通（端口阻塞） | 正常现象，STP阻塞冗余端口防止环路 | 确认阻塞端口为冗余链路即可，非故障 |
| DHCP分配地址超出预期范围 | 排除段配置遗漏或错误 | 检查excluded-ip-address范围 |
| Trunk两端的PVID不一致 | 一端Access一端Trunk或Trunk允许VLAN列表不一致 | 双方接口模式需对应，allow-pass vlan列表应一致 |

### 5.3 专用重点排错命令

```bash
display vlan                        # 查看VLAN信息和端口划分
display stp                         # 查看STP状态，确认根桥和端口角色
display vrrp                        # 查看VRRP状态，确认Master/Backup
display ip pool                     # 查看DHCP地址池使用情况
display ip routing-table            # 查看路由表，检查OSPF是否学习到路由
display ospf peer                   # 查看OSPF邻居状态
display acl all                     # 查看ACL规则
display nat session all             # 查看NAT转换表项
display current-configuration       # 查看当前运行配置
```

### 5.4 简短答辩口述稿（建议2~3分钟）

> **各位老师好，本次我答辩的项目是华为eNSP园区网络搭建。**
>
> **首先，这是一个典型的中小型企业园区局域网项目**，采用接入层+汇聚层+出口路由层的三层架构。接入层使用LSW1和LSW2两台交换机连接终端PC，汇聚层使用LSW3和LSW4双设备主备冗余，出口路由器R1连接互联网，R2模拟ISP运营商。
>
> **在业务需求方面**，我们需要实现VLAN隔离、DHCP自动分配、跨VLAN互通、网关冗余备份、内部服务器对外发布、以及内网用户通过NAT访问互联网。
>
> **下面我从数据流角度串联一下完整业务流程**。PC开机后通过DHCP自动获取IP地址，DHCP服务器由汇聚交换机LSW3和LSW4共同提供。PC获取IP后，通过VRRP虚拟网关实现三层转发，各VLAN间通过VLANIF接口互通。OSPF动态路由协议在整个内网运行，确保所有网段路由可达。当内网PC访问互联网时，流量经默认路由到达R1，R1通过ACL 2000控制允许哪些网段上网，再通过NAT地址转换将私网地址转换为公网地址，最终通过R2到达外网主机。
>
> **本项目亮点主要体现在四个方面**：一是链路冗余，接入层双链路上联汇聚层；二是VRRP网关冗余，同时实现负载分担；三是DHCP双服务器冗余部署，地址池互不冲突；四是ACL+NAT配合实现精细化互联网访问控制和地址转换。
>
> **最后说几个易错点**：DHCP排除段配置不正确会导致地址冲突；VRRP优先级和Track配置错误会导致切换不生效；OSPF宣告漏掉网段会导致路由不可达；ACL规则遗漏会导致某些网段无法上网。
>
> **我的汇报完毕，谢谢各位老师。**

---

> 文档版本：v1.0 | 适用场景：华为eNSP课程答辩 | 技术栈：华为VRP系列（S5700交换机、AR2200路由器）