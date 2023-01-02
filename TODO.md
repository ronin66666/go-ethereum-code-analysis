

1. 以太坊节点同步三种模式的区别 

   ./eth/downloader/modes.go

   ```go
   type SyncMode uint32
   
   const (
   	FullSync  SyncMode = iota // Synchronise the entire blockchain history from full blocks
   	SnapSync                  // Download the chain and the state via compact snapshots
   	LightSync                 // Download only the headers and terminate afterwards
   )
   ```

   - FullSync：全节点同步模式，Geth 按顺序下载并独立验证自创世以来的每个区块，包括重新执行交易以验证状态转换。 虽然 Geth 从创世开始就验证了每个区块，但只有 128 个区块的状态存储在内存中。
   - SnapSync：快照同步模式，Geth目前默认的同步模式，快照同步不是从创世区块开始处理所有曾经发生过的交易（这可能需要数周时间），而是下载块，并且只验证相关的工作量证明（假设状态转换是正确的）。下载所有区块是一个直接而快速的过程，并且会相对快速地重新组装整个链。
   - LightSync：轻节点模式

2. 区块包含哪些信息，如果只下载块，为什么没有任何可用的账户状态（例如余额、随机数、智能合约代码和数据），这些需要单独下载并与最新的块进行交叉检查，什么是状态树

   - 什么是状态数

     在以太坊主网中，已经有大量账户跟踪每个用户/合约的余额、随机数等。 然而，帐户本身不足以运行节点，它们需要以加密方式链接到每个块，以便节点可以实际验证帐户未被篡改。

     这种加密链接是通过创建一个树状数据结构来完成的，其中每个叶子对应一个帐户，每个中间层将它下面的层聚合成一个越来越小的层，直到你到达一个根。 这个包含所有账户和中间加密证明的巨大数据结构称为状态树。

3. 以太坊如果没有配置WS链接则默认使用127.0.0.1作为WS的HOST，代码位置: ./cmd/utils/flags.go `func setWS(ctx *cli.Context, cfg *node.Config)`

4. 以太坊：RPC/IPC

   IPC代表进程间通信。 Geth 在启动时创建一个 geth.ipc 文件，同一台计算机上的其他进程可以使用该文件与 Geth 通信。 RPC 代表远程过程调用。 RPC 是一种可以在不同机器上运行的进程之间的通信模式。 Geth 接受通过 HTTP 或 Websockets 的 RPC 流量。 通过 IPC 或 RPC 向节点发送根据 RPC-API 格式化的请求来调用 Geth 函数。
   
5. 以太坊gas计算公式



