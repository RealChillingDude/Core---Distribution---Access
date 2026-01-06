# Core---Distribution---Access
第一部分：项目全貌总结 (Project Summary)
这个网络设计是一个典型的 三层架构 (Core - Distribution - Access)。

拓扑结构：
核心 (GW-BE): 一台路由器，负责连接互联网 (ISP)、DMZ 服务器区域和通过 VPN 连接荷兰总部。它负责 NAT (上网) 和 VPN 隧道。
分布层 (DS-BE): 一台三层交换机，它是内网的大脑。它负责 VLAN 之间的通信 (Inter-VLAN Routing) 和 DHCP 中继。
接入层 (AS-xx-BE): 四台二层交换机，负责连接电脑、打印机和 WiFi AP。

关键技术：
VLAN: 划分了 8 个不同的部门网络 。
OSPF: 路由器和三层交换机之间使用 OSPF 自动学习路由 
DHCP Relay: 核心服务器在 VLAN 70，其他 VLAN 通过 ip helper-address 获取 IP 。
NAT/PAT: 允许内网所有设备通过一个公网 IP 上网 。

第二部分：完整指令大全 (Command Reference)
所有密码统一为：Console/VTY=secret123 (用户 admin), Enable=cisco1234 。


1. 分布层交换机 (DS-BE) —— 内网的核心大脑
这是配置最复杂、最重要的设备。
Current configuration : 3499 bytes
!
! --- 全局配置 ---
hostname DS-BE
enable secret cisco1234                  ! 设置特权模式加密密码 [cite: 114]
service password-encryption              ! 加密显示所有明文密码
ip routing                               ! 【关键】开启三层路由功能。没有它，VLAN之间无法通信！
username admin password secret123        ! 创建管理员账号 [cite: 116]

! --- VLAN 数据库配置 (作为 VTP Server) ---
vtp mode server                          ! 设置为服务器模式，向其他交换机广播 VLAN
vlan 10
 name In/Verkoop
vlan 20
 name Financ
! (以此类推创建所有 VLAN 10,20,30,40,50,60,70,99)

! --- 连接接入层交换机的接口 (Trunk) ---
interface range FastEthernet0/1 - 4
 switchport trunk encapsulation dot1q    ! 【关键】必须指定 802.1Q 封装，否则无法设为 Trunk
 switchport mode trunk                   ! 设置为 Trunk 模式，允许所有 VLAN 通过
 no shutdown

! --- 连接路由器的接口 (三层接口) ---
interface GigabitEthernet0/1
 no switchport                           ! 关闭交换功能，变成路由接口
 ip address 192.168.0.1 255.255.255.0    ! 配置连接 GW-BE 的 IP [cite: 72]
 no shutdown

! --- 配置 VLAN 网关 (SVI) & DHCP 中继 ---
! 以 VLAN 10 为例，其他 VLAN (20-60, 99) 类似
interface Vlan10
 ip address 192.168.10.252 255.255.255.0 ! 网关IP (倒数第三个可用) [cite: 67]
 ip helper-address 192.168.70.254        ! 【关键】DHCP中继，指向 VLAN 70 的服务器 

! --- VLAN 60 (访客WiFi) 的特殊限制 ---
interface Vlan60
 ip address 192.168.60.252 255.255.255.0
 ip helper-address 192.168.70.254
 ip access-group RestrictWlan in         ! 应用 ACL，限制访客访问内网 [cite: 123]

! --- 路由协议 ---
router ospf 1
 network 192.168.0.0 0.0.0.255 area 0    ! 宣告与路由器连接的网段 
 network 192.168.10.0 0.0.0.255 area 0   ! 宣告 VLAN 10
 ! (以此类推宣告所有 VLAN 网段)

! --- 安全 ACL (限制访客) ---
ip access-list extended RestrictWlan     ! 定义规则 [cite: 124]
 permit udp 192.168.60.0 0.0.0.255 host 192.168.70.254 eq domain ! 允许 DNS
 permit tcp 192.168.60.0 0.0.0.255 host 172.16.0.14 eq www       ! 允许访问 DMZ Web
 permit ip any any                       ! 允许上网 (注意：最后通常会有隐式 deny)
2. 边界路由器 (GW-BE) —— 通往世界的桥梁
hostname GW-BE
enable secret cisco1234

! --- 外部接口 (连接 ISP) ---
interface GigabitEthernet0/0
 ip address dhcp                         ! 自动从 ISP 获取公网 IP [cite: 66]
 ip nat outside                          ! 标记为 NAT 外部接口
 no shutdown

! --- DMZ 接口 (服务器区) ---
interface GigabitEthernet0/1
 ip address 172.16.0.12 255.255.255.240  ! 配置 /28 的网关地址 [cite: 67]
 ip nat inside                           ! 标记为 NAT 内部接口 (服务器也要上网)
 no shutdown

! --- 内部接口 (连接 DS-BE) ---
interface GigabitEthernet0/2
 ip address 192.168.0.252 255.255.255.0  ! 连接内部网络的接口
 ip nat inside                           ! 标记为 NAT 内部接口
 no shutdown

! --- VPN 隧道配置 (连接荷兰) ---
interface Tunnel0
 ip address 10.0.0.2 255.255.255.252     ! 隧道 IP [cite: 85]
 tunnel source GigabitEthernet0/0        ! 隧道起点 (自己的外网口) [cite: 87]
 tunnel destination 88.0.0.2             ! 隧道终点 (对方公网 IP)
 tunnel mode gre ip                      ! 设置 GRE 模式 [cite: 86]

! --- 路由配置 ---
router ospf 1
 network 192.168.0.0 0.0.0.255 area 0    ! 告诉 DS-BE 怎么走
 default-information originate           ! 把“默认路由”告诉 DS-BE，让内网能上网
ip route 0.0.0.0 0.0.0.0 GigabitEthernet0/0 ! 默认路由指向 ISP
ip route 192.168.3.0 255.255.255.0 10.0.0.1 ! 去荷兰内网的流量走 VPN 隧道 [cite: 91]

! --- NAT (上网配置) ---
! 1. 允许内网所有流量做 NAT
ip access-list extended NAT
 permit ip any any
! 2. 将匹配列表的流量转换为接口 IP (PAT)
ip nat inside source list NAT interface GigabitEthernet0/0 overload 
! 3. 端口映射 (让外网能访问 DMZ Web Server)
ip nat inside source static tcp 172.16.0.14 80 90.0.0.130 80 [cite: 103]
3. 接入层交换机 (例如 AS-0-BE) —— 连接用户
所有接入交换机配置逻辑相似，区别在于分配的 VLAN 不同。
hostname AS-0-BE
vtp mode client                          ! 设置为客户端，自动从 DS-BE 学习 VLAN

! --- 上联接口 (连接 DS-BE) ---
interface GigabitEthernet0/2
 switchport mode trunk                   ! 必须是 Trunk 才能传多个 VLAN
 no shutdown

! --- 用户接口 (分配 VLAN) ---
interface FastEthernet0/1
 switchport mode access                  ! 接入模式
 switchport access vlan 10               ! 分配给 VLAN 10 (销售部)
 no shutdown
interface FastEthernet0/24
 switchport mode access
 switchport access vlan 50               ! 分配给 VLAN 50 (打印机) [cite: 111]

! --- 管理 IP (为了能 Telnet 它) ---
interface Vlan99
 ip address 192.168.99.1 255.255.255.0   ! 唯一的管理 IP [cite: 64]
 no shutdown
ip default-gateway 192.168.99.252        ! 设置网关，否则跨网段无法管理它

! --- 端口安全 (仅在 AS-3-BE 的 Fa0/24 上) ---
interface FastEthernet0/24
 switchport mode access
 switchport access vlan 99
 switchport port-security                ! 开启端口安全
 switchport port-security mac-address sticky ! 自动记住插上的第一台电脑 MAC [cite: 127]

第三部分：常见问题排查 (Troubleshooting) & 注意点
1. 为什么我的电脑获取不到 IP (169.254.x.x)？
排查点 A (Helper): 检查 DS-BE 的 VLAN 接口下有没有 ip helper-address 192.168.70.254。没有这个，广播包出不去。
排查点 B (DHCP Server): 确保服务器 (DC-BE) 的 IP 是静态的 .254，并且 DHCP 池配置正确。
排查点 C (VLAN): 你的端口真的在正确的 VLAN 吗？输入 show vlan 检查。

2. 为什么 switchport mode trunk 报错？
注意点: 3560/3750 等三层交换机必须先定义“封装格式”。
指令: 先打 switchport trunk encapsulation dot1q，再打 mode trunk。

3. 为什么内网 Ping 不通 DMZ 或 外网？
排查点 A (Routing): 在 DS-BE 上输入 show ip route。如果你看不到 OSPF 路由，检查 ip routing 是否开启。
排查点 B (NAT): 在 GW-BE 上输入 show ip nat translations。如果你 ping 外网时这里是空的，说明 NAT 配置错误（比如 inside/outside 接口设反了）。

4. VTP 的坑
注意点: 如果你手动在 AS 交换机上建 VLAN，可能会导致配置混乱。
最佳实践: 确保 DS-BE 是 Server，AS-xx 是 Client。只要域名 (Domain) 一样，你在 DS-BE 上改一次 VLAN 名，下面全都会自动同步。
这张逻辑图展示了数据流向： PC -> Access Switch (VLAN Tag) -> Trunk -> Distribution Switch (SVI Routing/Helper) -> Core Router (NAT) -> Internet。
