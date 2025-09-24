# router-switch

---

# VPN 场景下的 NAT 例外配置 (NAT 0)

## 关键需求

*   当网络设备（如路由器或防火墙）**同时**执行 **NAT**（网络地址转换）和 **VPN**（如 IPSec VPN）功能时。
*   必须确保 **VPN 流量**（在总部和分支站点之间传输的流量）**不被 NAT 转换**。
*   如果 VPN 流量被 NAT 转换，会导致 VPN 隧道无法正确建立或流量无法通过隧道传输（因为源/目的 IP 地址被修改，不再匹配 VPN 对等体配置的加密 ACL）。

## 错误配置示例（会导致 VPN 流量被 NAT）

```cisco
! 定义需要走 VPN 的流量 ACL (通常也是 IPSec Crypto ACL)
ip access-list extended VPN-TRAFFIC
 permit ip 192.168.1.0 0.0.0.255 172.16.1.0 0.0.0.255 ! 允许本地 LAN (192.168.1.0/24) 访问远程 LAN (172.16.1.0/24)

! 错误：将此 ACL 直接用于 NAT 转换
ip nat inside source list VPN-TRAFFIC interface GigabitEthernet0/1 overload
```

**问题：** 此配置会将 `192.168.1.0/24` 访问 `172.16.1.0/24` 的流量进行 NAT 转换（源 IP 转换为 `GigabitEthernet0/1` 接口的 IP）。这会导致 VPN 隧道建立失败或流量无法进入隧道（源 IP 改变，不再匹配 VPN 配置）。

## 正确配置：NAT 例外 (NAT 0 / NAT Exemption)

```cisco
! 定义 NAT 例外 ACL (NONAT)
ip access-list extended NONAT
 deny   ip 192.168.1.0 0.0.0.255 172.16.1.0 0.0.0.255 ! 关键：拒绝 VPN 流量被 NAT
 permit ip 192.168.1.0 0.0.0.255 any                ! 允许本地 LAN 访问其他任何地方（如互联网）时进行 NAT

! 将例外 ACL 应用于 NAT
ip nat inside source list NONAT interface GigabitEthernet0/1 overload
```

**关键原理：**

1.  **路由选择先行：** 设备首先根据路由表决定流量去向。去往 `172.16.1.0/24` 的流量，其路由下一跳应指向 VPN 隧道接口（如 `Tunnel0`），而不是出接口 `GigabitEthernet0/1`。
2.  **NAT 处理：** NAT 处理发生在路由之后。对于匹配 `NONAT` ACL 中 `deny` 语句的流量（即 VPN 流量），NAT 规则明确 **拒绝** 对其进行转换。
3.  **NAT 执行：** 对于匹配 `NONAT` ACL 中 `permit` 语句的流量（如本地 LAN 访问互联网），NAT 规则允许其源 IP 被转换为 `GigabitEthernet0/1` 接口的 IP（使用 PAT/Overload）。
4.  **VPN 处理：** 未被 NAT 转换的 VPN 流量（源 IP 保持为 `192.168.1.x`，目的 IP 为 `172.16.1.x`）到达 VPN 处理模块。该流量精确匹配 IPSec VPN 配置的加密 ACL（通常与 `VPN-TRAFFIC` ACL 相同），因此被允许进入 VPN 隧道进行加密传输。

**结果：**
*   `192.168.1.0/24` <-> `172.16.1.0/24` 的流量 **不被 NAT**，保持原始 IP 地址，能正确匹配 IPSec VPN 的 ACL 并通过隧道传输。
*   `192.168.1.0/24` 访问其他目标（如互联网）的流量 **正常进行 NAT** 转换。