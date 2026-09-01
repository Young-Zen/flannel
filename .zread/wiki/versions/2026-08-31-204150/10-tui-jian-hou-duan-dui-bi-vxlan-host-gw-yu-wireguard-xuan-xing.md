flannel 的"后端"（backend）决定了 Pod 跨主机通信时，数据包究竟以什么方式到达远端节点。官方明确表示 **VXLAN 是默认推荐**，host-gw 适合有经验且基础设施支持直连的用户，UDP 只用于调试——三者之外，**WireGuard 是需要加密流量时的事实选择**。本页将从数据路径、实现机制和配置参数三个维度对比这三个推荐后端，帮助你做出选型决策。注意：后端一旦确定就**不应在运行时更换**。

Sources: [backends.md](Documentation/backends.md#L3-L5)

## 为什么"推荐名单"里只有这三个

所有后端都通过同一个注册表接入 flannel：每个后端在自己的 `init()` 函数中调用 `backend.Register("类型名", 构造函数)`，把构造器存入全局 `constructors` 映射；运行时管理器按配置中的 `Type` 字段查找并实例化。VXLAN、host-gw、WireGuard 与其他后端的唯一区别，就是注册的字符串和 `New` 函数不同——这种插件注册模式的完整原理在[后端插件注册模式：基于 init() 与构造器映射的扩展点设计](8-hou-duan-cha-jian-zhu-ce-mo-shi-ji-yu-init-yu-gou-zao-qi-ying-she-de-kuo-zhan-dian-she-ji)中有详细解析。

三个推荐后端分别代表了三种典型的数据面思路：**纯路由**（host-gw）、**内核封装**（VXLAN）、**内核加密封装**（WireGuard）。剩下的 UDP/IPIP/IPsec 等后端要么只适合调试，要么仍属实验性，将在[加密与实验性后端：IPsec、IPIP、UDP 与腾讯云 VPC](11-jia-mi-yu-shi-yan-xing-hou-duan-ipsec-ipip-udp-yu-teng-xun-yun-vpc)中介绍。

Sources: [manager.go](pkg/backend/manager.go#L26-L33), [common.go](pkg/backend/common.go#L39-L50)

## 一张图看懂三种数据路径

假设 Pod A（在节点 1 上）要访问 Pod B（在节点 2 上），三种后端的旅程如下。先补充几个新手概念：**封装**指把原始 IP 包再套一层新的 IP 头后发送；**ARP 表**记录"IP → MAC 地址"映射；**FDB 表**（转发数据库）记录"MAC → 隧道对端公网 IP"映射，是 VXLAN 设备专用表。

```mermaid
flowchart TB
    subgraph S1["host-gw：不加封装，路由直发"]
        A1[Pod A 数据包] --> R1[内核路由表<br/>目的子网 → 节点2公网IP网关] --> H1[物理网络直连转发<br/>无额外开销 无加密]
    end
    subgraph S2["VXLAN：内核 UDP 封装"]
        A2[Pod A 数据包] --> R2[路由指向 flannel.1 设备] --> E2[内核 VXLAN 封装<br/>+50 字节 · 外层 UDP 8472] --> H2[物理网络转发<br/>性能略降 无加密]
    end
    subgraph S3["WireGuard：内核加密封装"]
        A3[Pod A 数据包] --> R3[路由指向 flannel-wg 设备] --> E3[内核 WireGuard 加密<br/>+80 字节 · 外层 UDP 51820] --> H3[物理网络转发<br/>性能略降 全程加密]
    end
```

三条路径在代码中对应完全不同的实现结构：host-gw 复用通用的 `RouteNetwork`（只维护路由表）；VXLAN 和 WireGuard 各自拥有专用网络类型，独立处理子网事件。

Sources: [route_network.go](pkg/backend/route_network.go#L37-L47), [vxlan_network.go](pkg/backend/vxlan_network.go#L38-L44), [wireguard_network.go](pkg/backend/wireguard_network.go#L46-L54)

## host-gw：最简单也最快，但要求"同一层二网络"

host-gw 的原理朴素到极致：看到一条远端子网租约，就往系统路由表里加一条 `目的子网 → 远端主机公网 IP（作为网关）` 的路由。数据包由内核直接三层转发到对端主机，**没有任何封装、没有额外 MTU 开销、没有加密**，性能最接近裸网络。

它的核心是通用的 `RouteNetwork` 结构：注册网络时提供一个 `GetRoute` 函数，把租约转换成 `netlink.Route`（目的地址为远端子网、网关为远端 `PublicIP`、出接口为外部网卡）；IPv6 场景则通过 `GetV6Route` 提供对应逻辑。事件循环订阅租约变更：`EventAdded` 时添加路由，`EventRemoved` 时删除路由，并有一个定期巡检协程自动修复被外部改动或丢失的路由。添加路由前还会先查询现有路由——如果目的子网相同但网关不同就先删旧再加新，保证收敛到正确状态。

Sources: [hostgw.go](pkg/backend/hostgw.go#L53-L88), [route_network.go](pkg/backend/route_network.go#L53-L81), [route_network.go](pkg/backend/route_network.go#L142-L175)

host-gw 在构造阶段就有一条硬性前置检查：**节点的公网 IP 必须等于所选网卡的 IP**，否则直接报错拒绝启动——因为它假设对端路由直接指向本机 IP，如果中间隔了一层 NAT，路由根本不可达。这正是"要求直连二层网络"的代码化体现，也是它通常无法用于公有云 VPC 环境（如 AWS/GCE，其虚拟网络会丢弃非本机 MAC/路由的流量）的根本原因。它的 MTU 直接继承外部网卡，无任何折损。

Sources: [hostgw.go](pkg/backend/hostgw.go#L41-L51), [backends.md](Documentation/backends.md#L36-L43)

| 维度 | 表现 |
| --- | --- |
| 性能 | 最优（无封装开销，MTU 零折损） |
| 前置条件 | 所有节点必须在同一二层网络（不能跨路由器/NAT/公有云 VPC） |
| 加密 | 无 |
| 配置复杂度 | 只需 `Type: host-gw` 一项 |
| 双栈支持 | IPv4/IPv6 均支持 |

Sources: [backends.md](Documentation/backends.md#L36-L43), [hostgw.go](pkg/backend/hostgw.go#L68-L88)

## VXLAN：默认且最通用的选择

打开官方部署清单 `kube-flannel.yml` 就能看到，开箱即用的配置就是 `"Type": "vxlan"`。VXLAN 之所以是"最通用"的选择，是因为它只要求节点之间能通 UDP 就行，**不要求二层直连**，因此公有云、跨网段环境都能跑。

Sources: [kube-flannel.yml](Documentation/kube-flannel.yml#L91-L98)

### 每主机三条静态表项的演进结果

flannel 的 VXLAN 实现经历过一次重要重构（详见[VXLAN 数据路径演进：从 L2/L3 miss 回调到纯静态表项](9-vxlan-shu-ju-lu-jing-yan-jin-cong-l2-l3-miss-hui-diao-dao-chun-jing-tai-biao-xiang)），现在的版本完全不依赖内核回调：每当发现一个新远端主机，flannel 就主动写入**每主机恰好三条表项**——1 条指向远端子网、下一跳为远端容器子网 IP 的路由（走 `flannel.1` 设备），1 条静态 ARP 表项（远端子网 IP → 对端 VTEP 的 MAC），1 条 FDB 表项（VTEP MAC → 对端公网 IP）。表项规模随主机数线性增长，且 flannel 崩溃重启也不会出现回调超时问题。VTEP（VXLAN Tunnel Endpoint）的 MAC 地址会以 JSON 形式写进租约的 `BackendData`，供对端读取；重启时还会从节点注解里恢复原 MAC，保证设备重建后表项不失效。

Sources: [vxlan.go](pkg/backend/vxlan/vxlan.go#L44-L54), [vxlan.go](pkg/backend/vxlan/vxlan.go#L92-L120), [vxlan.go](pkg/backend/vxlan/vxlan.go#L190-L200)

### DirectRouting：VXLAN 与 host-gw 的混合模式

VXLAN 有一个值得了解的参数 `DirectRouting`（默认 `false`）：开启后，每收到一个新租约，flannel 会调用内核的 `RouteGet` 查询到对端公网 IP 的路由——如果只有一条且无网关（即两台主机在**同一子网直连**），就直接装 host-gw 式的直连路由，只有跨子网的流量才走 VXLAN 封装。事件处理时两类路由互斥：直连可行则 `RouteReplace` 直连路由，否则按"加 ARP → 加 FDB → 加路由"的顺序写入三条表项（顺序有讲究：先填好 ARP 再加路由，避免内核对网关 IP 发起多余的 ARP 查询；每一步失败都会回滚清理已写入的表项）。

Sources: [iface.go](pkg/ip/iface.go#L232-L242), [vxlan_network.go](pkg/backend/vxlan/vxlan_network.go#L289-L309), [vxlan_network.go](pkg/backend/vxlan/vxlan_network.go#L343-L396)

### 参数与自愈能力

VXLAN 设备名为 `flannel.<VNI>`（默认 VNI 为 1，即常见的 `flannel.1`；Windows 上 VNI 必须 ≥ 4096 且端口固定 4789）。封装会吃掉 50 字节 MTU，flannel 会自动按 `外部网卡 MTU − 50` 设置设备 MTU。此外运行期间有一个设备监视协程订阅 netlink 事件，一旦检测到 `flannel.1` 被删除（`RTM_DELLINK`），就会以指数退避（1 秒起步、上限 30 秒）自动重建设备——这对新手排障很有意义：如果你手动 `ip link delete flannel.1`，flannel 会自己把它建回来。

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `VNI` | 1 | VXLAN 标识符，Linux 上一般不用改 |
| `Port` | 8472 | 外层 UDP 端口；**Windows 必须 4789** |
| `GBP` | false | 组策略标签，Windows 不支持 |
| `DirectRouting` | false | 同子网主机走 host-gw 式直连，Windows 不支持 |
| `Learning` | false | 内核 MAC 学习开关，默认关闭（静态表项已足够） |
| `MTU` | 外部网卡 MTU | 设备实际按此值减 50 生效 |

Sources: [backends.md](Documentation/backends.md#L20-L29), [vxlan.go](pkg/backend/vxlan/vxlan.go#L74-L76), [vxlan.go](pkg/backend/vxlan/vxlan.go#L159-L180), [device.go](pkg/backend/vxlan/device.go#L59-L71), [vxlan_network.go](pkg/backend/vxlan/vxlan_network.go#L46-L48), [vxlan_network.go](pkg/backend/vxlan/vxlan_network.go#L97-L152), [vxlan_network.go](pkg/backend/vxlan/vxlan_network.go#L243-L258)

## WireGuard：唯一"推荐"的加密后端

当集群跨公网（例如混合云、边缘节点）传输流量时，加密就不再是可选项。WireGuard 后端使用**内核态 WireGuard** 完成封装 + 加密，是推荐后端中唯一提供机密性保证的。内核版本低于 5.6 时需要额外安装 WireGuard 包。

Sources: [backends.md](Documentation/backends.md#L45-L68)

### 从"路由 + ARP + FDB"到"对端（Peer）"模型

WireGuard 没有路由表项和 ARP/FDB 表项的细分，取而代之的是 **Peer 概念**：每个远端主机是一个 Peer，由三要素描述——对端**公钥**（来自对端租约 `BackendData` 中的 `PublicKey`）、对端**UDP 端点**（`公网IP:端口`）、以及 `AllowedIPs`（对端子网，等价于"这个网段的流量都发给这个 Peer"）。收到 `EventAdded` 事件时，flannel 通过 wgctrl 库把新 Peer 追加到本地设备，同时给设备加一条指向整个 flannel 大网段的路由（`SCOPE_LINK`，作用域仅限本设备）——**只加一次全网络的大路由**，而不是 host-gw/VXLAN 那种每主机一条子网路由，路由的分发工作交给了 WireGuard 自己的 AllowedIPs 匹配机制。

Sources: [wireguard_network.go](pkg/backend/wireguard/wireguard_network.go#L181-L198), [device.go](pkg/backend/wireguard/device.go#L224-L250), [device.go](pkg/backend/wireguard/device.go#L280-L323)

### 密钥、接口与默认参数

flannel 为每个节点自动生成 WireGuard 私钥，写入 `/run/flannel/wgkey`（可用环境变量 `WIREGUARD_KEY_FILE` 改路径，文件权限收紧为 `0400`）；重启时若该文件存在则复用原私钥，保证公钥不变、对端无需更新。可选的 `PSK` 预共享密钥通过 `wg genpsk` 生成，叠加在对端公钥认证之上提供后量子安全加固。双栈默认采用 `separate` 模式：IPv4 用设备 `flannel-wg`（UDP 51820），IPv6 用独立设备 `flannel-wg-v6`（UDP 51821）；也可用 `auto/ipv4/ipv6` 模式让双栈流量复用单条隧道，`auto` 会按"优先 IPv4、对端与本端均有公网地址时才选 IPv6"的启发式选择隧道地址族。退出时 flannel 会自动删除 `flannel-wg` 设备，不留残余。

| 参数 | 默认值 | 说明 |
| --- | --- | --- |
| `PSK` | 无 | 预共享密钥，用 `wg genpsk` 生成 |
| `ListenPort` | 51820 | IPv4 隧道监听端口 |
| `ListenPortV6` | 51821 | IPv6 隧道监听端口 |
| `MTU` | 外部网卡 MTU | 实际设备按此值减 80 生效 |
| `Mode` | separate | separate / auto / ipv4 / ipv6 四种双栈策略 |
| `PersistentKeepaliveInterval` | 0（禁用） | NAT 保活间隔，穿透 NAT 时需设置 |

Sources: [backends.md](Documentation/backends.md#L49-L66), [device.go](pkg/backend/wireguard/device.go#L52-L127), [device.go](pkg/backend/wireguard/device.go#L129-L184), [wireguard.go](pkg/backend/wireguard/wireguard.go#L113-L128), [wireguard.go](pkg/backend/wireguard/wireguard.go#L141-L165), [wireguard_network.go](pkg/backend/wireguard/wireguard_network.go#L107-L129)

一个值得一提的细节：WireGuard 封装固定开销为 80 字节（IPv4 头 20 / IPv6 头 40 + UDP 头 8 + 类型 4 + 密钥索引 4 + 随机数 8 + 认证标签 16），这个常量在源码注释里有逐项拆解；而 VXLAN 的开销是 50 字节，host-gw 为 0。三者调用的都是同一套 `Backend`/`Network` 接口（`RegisterNetwork` + `MTU` + `Run`），所以切换后端对 flannel 其余部分完全透明。

Sources: [wireguard_network.go](pkg/backend/wireguard/wireguard_network.go#L33-L44), [wireguard_network.go](pkg/backend/wireguard/wireguard_network.go#L74-L76), [vxlan_network.go](pkg/backend/vxlan/vxlan_network.go#L256-L258), [common.go](pkg/backend/common.go#L39-L50)

## 三后端横向对照与选型决策

| 维度 | host-gw | VXLAN | WireGuard |
| --- | --- | --- | --- |
| 数据面机制 | 纯三层路由直发 | 内核 VXLAN UDP 封装 | 内核 WireGuard 加密封装 |
| 加密 | ✗ | ✗ | ✓ |
| MTU 开销 | 0 | 50 字节 | 80 字节 |
| 网络要求 | 同一二层网络，禁止 NAT | 仅需 UDP 可达（跨路由/公有云均可） | 仅需 UDP 可达 |
| 需开放端口 | 无额外端口 | UDP 8472（Windows 4789） | UDP 51820 / 51821 |
| 本机设备 | 无（只写路由表） | `flannel.1`（VNI 命名） | `flannel-wg`、`flannel-wg-v6` |
| 每主机状态 | 1 条子网路由 | 1 路由 + 1 ARP + 1 FDB | 1 个 Peer + 1 条全网大路由 |
| 性能 | 最优 | 中 | 中（加密有 CPU 成本） |
| 调试工具 | `ip route` | `bridge fdb show`、`ip neigh` | `wg show` |

Sources: [backends.md](Documentation/backends.md#L7-L10), [backends.md](Documentation/backends.md#L20-L66), [vxlan.go](pkg/backend/vxlan/vxlan.go#L44-L54), [device.go](pkg/backend/wireguard/device.go#L224-L250), [route_network.go](pkg/backend/route_network.go#L43-L44)

选型建议按场景归纳如下。第一，**默认无脑选 VXLAN**：这是官方清单的出厂配置，兼容性最好，若节点恰好同子网还可叠加 `DirectRouting: true` 兼顾性能。第二，**裸金属同二层网络且追求极致性能选 host-gw**——但务必确认没有 NAT（代码会显式检查并拒绝），且理解流量不加密。第三，**跨公网、多租户或任何合规要求加密的场景选 WireGuard**，代价是加密 CPU 开销和 80 字节 MTU。无论选哪个，若节点启用了 `firewalld`，记得放行对应端口（如 `firewall-cmd --permanent --zone=public --add-port=8472/udp`），这是新手最容易踩的连通性坑。端口与防火墙的系统排障方法可进一步阅读[常见故障排查：连通性、权限、日志与日志级别](25-chang-jian-gu-zhang-pai-cha-lian-tong-xing-quan-xian-ri-zhi-yu-ri-zhi-ji-bie)，MTU 与性能调优的量化分析见[性能调优：MTU 计算、后端选型与防火墙端口](26-xing-neng-diao-you-mtu-ji-suan-hou-duan-xuan-xing-yu-fang-huo-qiang-duan-kou)。

Sources: [backends.md](Documentation/backends.md#L5-L10), [hostgw.go](pkg/backend/hostgw.go#L42-L44)

## 延伸阅读

想继续深入，推荐顺序：了解 VXLAN 表项机制的历史演变请阅读[VXLAN 数据路径演进：从 L2/L3 miss 回调到纯静态表项](9-vxlan-shu-ju-lu-jing-yan-jin-cong-l2-l3-miss-hui-diao-dao-chun-jing-tai-biao-xiang)；其余后端（含 IPsec、IPIP、UDP 与腾讯云 VPC）见[加密与实验性后端：IPsec、IPIP、UDP 与腾讯云 VPC](11-jia-mi-yu-shi-yan-xing-hou-duan-ipsec-ipip-udp-yu-teng-xun-yun-vpc)；`Backend` 段所有配置项的逐项参考见[net-conf.json 网络配置项逐项详解](13-net-conf-json-wang-luo-pei-zhi-xiang-zhu-xiang-xiang-jie)。