# ensp-campus-network
## 项目介绍
华为eNSP中小型企业园区网实训（课程答辩项目）
### 组网架构：接入层+汇聚层+出口路由器三层企业局域网
**设备清单**
接入：LSW1、LSW2
汇聚：LSW3
出口路由：R1、R2
终端：PC1~PC4(DHCP自动获取IP)、Server1(VLAN50静态地址)、外网PC5

### 用到核心技术
VLAN | Trunk/Access | STP生成树 | VRRP冗余网关 | DHCP地址池 | OSPF动态路由 | ACL访问控制 | NAT内网访问互联网

### 仓库文件说明
1. ensp配置汇总.txt：全设备`display current-configuration`原始运行配置
2. ensp答辩文稿.md：答辩全套文稿（项目讲解+逐条命令解析+技术串联+答辩口述稿）

3. ### 在线查看文档
1. [原始设备配置（ensp配置汇总.txt）](https://github.com/tong-0915/ensp-campus-network/blob/main/ensp配置汇总.txt)
2. [完整答辩文稿（ensp答辩文稿.md）](https://github.com/tong-0915/ensp-campus-network/blob/main/ensp答辩文稿.md)
