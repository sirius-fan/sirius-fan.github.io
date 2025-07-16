# 


## 问题
### get_peers 和 announce_peer

这里再解释下 get_peers 和 announce_peer

#### get_peers

`get_peers` 请求中，用作“Key”来查找目标的**是种子 (Torrent) 的 info_hash**。

当一个客户端要下载一个文件,会通过get_peers 利用info_hash查找文件的位置, 文件位置信息存在info_hash 的 '附近' 节点的peer里, get_peer的发起方要找文件. 他有信息需要文件.

解释如下：

1. **`get_peers` 的目的**: 这个 RPC 的目的是找到**正在分享特定 Torrent 文件**的其他 Peer（对等节点）。
2. **Torrent 的唯一标识**: 在 BitTorrent 网络中，一个 Torrent 文件（及其内容）的唯一标识符就是它的 **info_hash** (信息哈希值)。
3. **DHT 的 Kademlia 基础**: BitTorrent DHT 基于 Kademlia。在 Kademlia 中，查找操作（无论是找节点还是找值）都是围绕一个**目标 ID** 来进行的。网络会寻找在 XOR 距离上与这个目标 ID 最接近的节点。
4. **`get_peers` 中的目标 ID**: 对于 `get_peers` 操作，你关心的不是某个特定的 DHT 节点，而是与某个 **Torrent 内容** 相关联的 Peer 列表。因此，这个 **Torrent 的 info_hash** 就充当了 Kademlia 查找过程中的**目标 ID (Key)**。
5. **请求参数**: `get_peers` 请求消息的参数 (`a` 字典) 包含：
    - `id`: 发起请求的节点自身的 Node ID。这用于让接收方更新路由表，但**不是**查找的目标 Key。
    - `info_hash`: **这才是查找的目标 Key**。接收节点会根据这个 `info_hash` 来查找存储的 Peer 信息，或者返回离这个 `info_hash` 更近的其他节点。

**总结:**

当你发起 `get_peers` 请求时，你是在问 DHT 网络：“谁知道关于 `info_hash` 为 X 的这个 Torrent 的 Peer 信息？” 网络会把你的请求路由到那些 Node ID 与这个 `info_hash` 在 XOR 距离上最接近的节点，因为根据 Kademlia 原则，这些节点最有可能存储着你想要的信息（即该 `info_hash` 对应的 Peer 列表）。

**与之对比的是 `find_node` 请求，它的目标 Key 是一个节点的 ID (`target` 参数)，目的是查找离那个特定节点 ID 最近的其他节点。**

####  announce_peer
`announce_peer` RPC 的用处与 `get_peers` 相辅相成，它是 BitTorrent DHT 实现 Peer 发现的关键另一半。简单来说，它的作用是：

**让一个正在下载或上传某个 Torrent 的 Peer（客户端），主动告诉 DHT 网络中负责存储该 Torrent 信息的节点：“我在这里，我正在参与这个 Torrent，这是我的联系方式。”**

具体用途和流程如下：

1. **成为可发现的 (Become Discoverable)**:
    
    - 当你的 BitTorrent 客户端开始下载或做种某个 Torrent（通常在你通过 `get_peers` 或 Tracker 找到最初的几个 Peer 之后）时，为了让**其他**也想下载这个 Torrent 的人能够找到你，你需要宣告你的存在。
    - `announce_peer` 就是你用来进行这种宣告的工具。
2. **通知相关节点**:
    
    - 你不是向 DHT 网络中的所有节点宣告，而是只向那些**“负责”**该 Torrent 信息存储的节点宣告。根据 Kademlia 原则，这些是 Node ID 与该 Torrent 的 **info_hash** 在 XOR 距离上最近的 `k` 个节点。
    - 通常，这些“负责”的节点就是你在之前执行 `get_peers(info_hash)` 时，最终找到并交互过的那批节点。
3. **提供联系信息**:
    
    - 在 `announce_peer` 请求中，你需要提供：
        - 你正在参与的 Torrent 的 `info_hash`。
        - 你用于接收 BitTorrent 连接的**端口号**。
        - 一个重要的**`token`**。
4. **Token 验证 (安全机制)**:JSON
    
    - `token` 是你之前从**同一个目标节点**那里调用 `get_peers` 时收到的。
    - 接收 `announce_peer` 请求的节点会**验证**这个 `token`。这个验证通常与请求来源的 IP 地址和时间有关。
    - **目的**: 验证 `token` 是为了防止任何人随意向 DHT 网络注入关于任意 Torrent 的虚假 Peer 信息（称为 DHT 投毒或污染）。只有能提供有效 `token` 的 Peer（意味着它最近确实从该节点的 IP 地址查询过这个 info_hash）才被认为是合法的，其宣告才会被接受。
5. **信息存储**:
    
    - 如果 `token` 验证通过，接收节点就会将你的联系信息（你的 IP 地址（从 UDP 包来源获取）和你在请求中声明的 `port`）与你提供的 `info_hash` 关联起来，存储在它的 Peer 列表里。
    - 这个存储通常有**时效性**，Peer 信息会在一段时间后过期。
6. **完成 Peer 发现闭环**:
    
    - 当**其他**客户端为同一个 `info_hash` 发起 `get_peers` 请求，并且它们的请求路由到了存储了你信息的这个节点时，该节点就会在其响应中包含你的 IP 地址和端口号。
    - 这样，其他 Peer 就能找到并尝试连接你，实现了通过 DHT 的 Peer 发现。
7. **定期宣告**:
    
    - 由于存储的信息会过期，并且节点的 IP 或端口可能变化，客户端需要对其正在活跃参与的 Torrent **定期**执行 `announce_peer` 操作，以保持自己在 DHT 中的可发现状态。

**总结:**

如果说 `get_peers` 是在 DHT 网络中**“查询”**某个 Torrent 的 Peer 信息，那么 `announce_peer` 就是向 DHT 网络**“注册/宣告”**自己是某个 Torrent 的 Peer。两者结合，使得 BitTorrent 客户端能够在一个去中心化的网络中互相发现，即使没有中心 Tracker 服务器也能有效地进行文件分享。`announce_peer` 通过 `token` 机制增加了安全性，确保了信息的基本可信度。

