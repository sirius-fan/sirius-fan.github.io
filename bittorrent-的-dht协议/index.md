# BitTorrent  的 DHT协议


本文介绍 BitTorrent 使用的 DHT（分布式哈希表）网络，它通常被称为 "Mainline DHT"。

### **背景与目的**

最初，BitTorrent 依赖于 **Tracker 服务器** 来协调 Peer（下载或上传同一个文件的用户）之间的连接。用户客户端连接到 Tracker，报告自己的信息和想要的 Torrent 文件，Tracker 则返回其他正在下载/上传该文件的 Peer 的列表。这种方式的问题在于 Tracker 是一个**中心点**：

1. **单点故障**：如果 Tracker 服务器宕机或关闭，依赖它的 Torrent 就很难找到新的 Peer。
2. **可审查性/易受攻击**：Tracker 容易被监控或攻击。
3. **扩展性限制**：大型 Tracker 需要处理大量连接。

为了解决这些问题，BitTorrent 社区开发并集成了一个基于 Kademlia 算法的 DHT 网络。其**主要目的**是：

- **去中心化 Peer 发现**：即使没有 Tracker 服务器，或者 Tracker 失效，客户端也能通过 DHT 网络找到其他下载相同 Torrent 的 Peer。
- **降低对 Tracker 的依赖**：使得 "Trackerless" Torrent 成为可能，提高了整个 BitTorrent 生态的健壮性。

### **基于 Kademlia 的实现**

BitTorrent 的 Mainline DHT 在很大程度上遵循了 Kademlia 算法的核心原则：

1. **节点 ID**：每个运行 DHT 功能的 BitTorrent 客户端（即 Peer）都会生成一个 160 位的随机 Node ID，代表其在 DHT 网络中的身份和位置。
2. **XOR 距离**：节点间以及节点与数据之间的“距离”同样使用 XOR 度量来计算。
3. **K-桶 (k-buckets)**：每个客户端维护一个路由表，包含若干 K-桶，用于存储它所知道的其他 DHT 节点（Peer）的信息（Node ID, IP 地址, UDP 端口），组织方式与标准 Kademlia 相同（按 XOR 距离范围划分，优先保留稳定节点）。

### **关键区别与 BitTorrent 特定应用**

与通用的 Kademlia 用于存储任意 (Key, Value) 不同，BitTorrent DHT 的核心应用是**存储和查询 Peer 信息**。

- **"Key" 是 Info-Hash**：在 BitTorrent DHT 中，最重要的“Key”是 Torrent 文件的 **Info-Hash**。这是一个根据 Torrent 文件元数据（`info` 字典部分）计算出的 160 位哈希值，唯一标识了一个特定的 Torrent 内容。
- **"Value" 是 Peer 列表**：DHT 网络存储的“Value”不是任意数据，而是与特定 Info-Hash 相关联的 **Peer 联系信息列表**（IP 地址和端口号）。

### **核心工作流程：查找和宣告 Peer**

1. **查找 Peer (`get_peers` RPC)**：
    
    - 当一个 BitTorrent 客户端想要加入某个 Torrent（知道了它的 Info-Hash）时，它会将这个 **Info-Hash** 作为目标 Key，在 DHT 网络中发起查找。
    - 它执行类似 Kademlia 的迭代查找过程：向自己路由表中距离 Info-Hash 最近的节点发送 `get_peers` 请求。
    - 收到 `get_peers(info_hash)` 请求的节点：
        - 如果它**存储**了该 Info-Hash 对应的 Peer 列表，它会返回这个列表（一部分或全部）。
        - 如果它**没有**存储该 Info-Hash 的 Peer 信息，它会像 Kademlia 的 `find_node` 一样，返回自己路由表中距离该 Info-Hash 最近的 `k` 个节点的信息。
    - 发起查找的客户端不断向返回结果中更接近 Info-Hash 的节点发送 `get_peers` 请求，逐步缩小范围，直到找到拥有该 Torrent 的 Peer，或者已经查询了它所知道的最接近 Info-Hash 的 k 个节点。
    - **重要**：当节点回复 `get_peers` 时（无论是返回 Peer 列表还是节点列表），它还会附带一个临时的 `token`。这个 `token` 在稍后的宣告步骤中需要用到。
2. **宣告自己 (`announce_peer` RPC)**：
    
    - 一旦客户端开始下载某个 Torrent，为了让其他 Peer 能找到自己，它需要向 DHT 网络“宣告”自己的存在。
    - 它首先执行 `get_peers` 查找（如上所述），找到网络中距离该 Torrent 的 Info-Hash 最近的 `k` 个节点。
    - 然后，它向这些节点发送 `announce_peer` 请求。这个请求包含：
        - Torrent 的 Info-Hash。
        - 客户端监听 BitTorrent 连接的端口号。
        - 从之前该节点回复 `get_peers` 时收到的那个 `token`。
    - 收到 `announce_peer` 请求的节点会验证 `token` (通常是基于该节点的 IP 地址和时间戳生成，有时效性)，验证通过后，它就会将发送方（宣告者）的 IP 地址和端口号存储起来，与对应的 Info-Hash 关联。这样，当其他 Peer 查询这个 Info-Hash 时，该节点就能提供宣告者的联系信息。
    - **Token 的作用**：防止任何人随意宣告任意 Peer 的信息，确保只有真正参与该 Torrent 下载（并执行过 `get_peers` 查询）的客户端才能成功宣告。


### **其他 RPCs**

- `ping`: 与 Kademlia 相同，用于检查节点是否在线。
- `find_node`: 与 Kademlia 相同，用于查找距离目标 ID 最近的节点，主要用于维护路由表。

### **引导 (Bootstrap)**

- 新启动的 BitTorrent 客户端需要知道至少一个已经在 DHT 网络中的节点地址（通常是一些长期运行的、硬编码在客户端或从上次运行保存下来的 “bootstrap nodes”，如 `router.bittorrent.com`）。
- 通过向引导节点发送 `find_node` 请求（查找自己的 Node ID），客户端开始填充自己的路由表并融入 DHT 网络。

### DHT总结

BitTorrent 的 Mainline DHT 是 Kademlia 算法的一个非常成功和广泛的应用实例。它通过去中心化的方式解决了 Peer 发现的关键问题，使得 BitTorrent 协议更加健壮和有弹性，不再完全依赖中心化的 Tracker 服务器。现代几乎所有的 BitTorrent 客户端都支持并默认启用了 DHT 功能。其核心机制就是利用 Kademlia 的路由和查找能力，专门用于在 P2P 网络中映射 Torrent 的 Info-Hash 到下载该 Torrent 的 Peer 列表。





### Node ID 和 info_hash

DHT 通常只有一个核心的路由表（基于 Kademlia 的 K-桶系统），它的组织结构是基于节点 ID 的。


1. **核心路由表 (K-桶):**
    
    - 每个 DHT 节点维护一个路由表，其结构是 Kademlia 的 **K-桶 (k-buckets)**。
    - 这个 K-桶系统是根据**其他节点的 Node ID** 与**当前节点自身的 Node ID** 之间的 **XOR 距离**来组织的。
    - K-桶里存储的是它所知道的其他 DHT 节点的联系信息（Node ID, IP 地址, 端口号）。
    - **这个路由表是用来进行所有查找操作的基础**，无论是查找其他节点 (`find_node`) 还是查找存储特定信息（如 Peer 列表）的节点 (`get_peers`)。
2. **查找过程如何使用这个路由表**:
    
    - **当执行 `find_node(target_node_id)` 时**: 目标 Key 是一个**节点 ID**。节点使用 K-桶查找离 `target_node_id` XOR 距离最近的已知节点，并向它们查询，逐步迭代，最终找到离目标节点 ID 最近的一批节点。
    - **当执行 `get_peers(info_hash)` 时**: 目标 Key 是一个**文件的 info_hash**。虽然 `info_hash` 不是一个节点 ID，但它仍然是 Kademlia ID 空间中的一个点。节点**同样使用基于 Node ID 的 K-桶路由表**，查找其路由表中已知节点中，Node ID 与这个 `info_hash` 的 XOR 距离最近的那些节点。然后向这些节点发送 `get_peers` 请求，逐步迭代，最终找到在 XOR 距离上最接近该 `info_hash` 的一批节点。
3. **Peer 列表的存储 (关键区别)**:
    
    - **路由表 (K-桶)** 的作用是**指路**，告诉你“要找离目标 X 最近的节点，你应该去问谁”。
    - **Peer 列表 (由 `announce_peer` 产生)** 并不是存储在路由表里的。它是**应用层的数据**。
    - 那些**其 Node ID 最接近特定 info_hash** 的节点，除了维护自己的路由表外，还会**额外**维护一个**数据存储区域**（比如一个内存中的哈希表或字典）。这个区域用来存储 `announce_peer` 请求带来的信息，即：`info_hash -> [ (IP1, Port1, timestamp1), (IP2, Port2, timestamp2), ... ]` 以及相关的 `token`。
    - 所以，**存储 Peer 列表**是那些“负责”该 info_hash 的节点的**额外职责**，它们使用一个独立于路由表的机制来存储这些数据。而其他节点只负责通过路由表把 `get_peers` 或 `announce_peer` 请求导向这些负责的节点。

**总结:**

- 只有一个主要的**路由表（K-桶系统）**，它基于 **Node ID** 和 **XOR 距离**来组织，用于**所有类型**的查找（包括基于 info_hash 的查找）。
- `info_hash` 作为 `get_peers` 的**查找目标 Key**，指导查找过程使用路由表向正确的方向进行。
- `info_hash` 到 Peer 列表的映射关系，是作为**数据**被存储在那些 Node ID 最接近该 `info_hash` 的节点上的**独立存储区域**中，而不是存储在路由表本身里。

可以理解为：路由表是地图和导航系统（基于节点地址），而 Peer 列表是特定地点（接近 info_hash 的节点）存放的公告板（上面写着参与者的联系方式）。导航系统（路由表）本身不记录公告板的内容，只负责把你带到正确的地点。
