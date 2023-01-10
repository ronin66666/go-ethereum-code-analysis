geth是我们的go-ethereum最主要的一个命令行工具。 也是我们的各种网络的接入点(主网络main-net 测试网络test-net 和私有网络)。支持运行在**全节点模式**或者**轻量级节点模式**。 其他程序可以通过它暴露的JSON RPC调用来访问以太坊网络的功能。

如果什么命令都不输入直接运行geth。 就会默认启动一个全节点模式的节点。 连接到主网络。 我们看看启动的主要流程是什么，涉及到了那些组件。

## 使用geth搭建本地私有链

这里搭建环境是在mac上

### geth安装

1. 直接下载

   [https://geth.ethereum.org/downloads/](https://links.jianshu.com/go?to=https%3A%2F%2Fgeth.ethereum.org%2Fdownloads%2F)

2. 源码编译

   以太坊GitHub仓库地址：https://github.com/ethereum/go-ethereum.git

   下载以太坊源码

   `go get -u https://github.com/ethereum/go-ethereum.git`

   编译geth

   ```
   make geth
   ```

   执行后会在项目的`./build/bin/`目录中生成geth客户端，将生生成的geth移动到go环境的bin目录下

   执行`get version`检查是否能打印成功

### 私链搭建

创建文件夹用来存放私链项目

```
./private-ethereum
./private-ethereum/data
```

新建账户用于后面配置创世区块

```bash
cd ./private-ethereum
geth --datadir ./data account new
```

- --datadir：指定账户存放路径

创建配置创世区块的genesis.json文件

`./private-ethereum/genesis.json`

```json
{
    "config": {
        "chainId": 9999,			//链id
        "homesteadBlock": 0,
        "eip150Block": 0,
        "eip155Block": 0,
        "eip158Block": 0,
        "byzantiumBlock": 0,
        "ethash": {}
    },
    "nonce": "0x0",
    "timestamp": "0x0",
    "extraData": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "gasLimit": "0x80000000", //每个区块最大的gas消耗
    "difficulty": "0x01",	//出块难度，数值越大，矿工挖矿出块的难度越大
    "mixHash": "0x0000000000000000000000000000000000000000000000000000000000000000",
    "coinbase": "0x0000000000000000000000000000000000000000",
    "alloc": {//资产配置初始化，提前定义哪些地址所拥有的原生代币的数量，单位是wei
        "0xb321D7d29099c329e7753913Bf3f7a6361f8eE04": {
            "balance": "0xffffffffffffffffffffffffff"
        }
    },
    "number": "0x0",
    "gasUsed": "0x0",
    "parentHash": "0x0000000000000000000000000000000000000000000000000000000000000000"
}

```

 一般来说，在上述文件中需要我们进行配置的是链ID、出块难度和初始资产

文件保存后执行私链配置初始化命令

```
geth init --datadir data genesis.json
```

​    其中--datadir字段用于配置私链数据的存储位置，这里存在data文件夹。输入命令后等待片刻，看到“Successfully wrote genesis state”的日志则表示配置完成。

geth命令启动私链了

```
geth --datadir data --networkid 9999 --http --http.addr 0.0.0.0 --http.port 8545 --http.corsdomain "*" --port 30305 --allow-insecure-unlock console 2>>geth.log
```

- --datadir字段用于指示私链数据的存储位置，即工作目录；
- --networkid字段用于配置私链id，需要与genesis.json文件内预定义的一致；
- --http字段用于启用HTTP-RPC服务，主要应用于与前端页面的交互；
- --http.addr字段表示节点接受的http连接的地址，0.0.0.0表示可以接受所有ip地址的http请求；
- --http.port字段用于指定监听端口，默认是8545；最好不要用这个默认端口号，容易被攻击
- --http.corsdomain字段表示允许跨域请求的域名列表，“*”表示允许所有的跨域请求，不开启的话metamask钱包可能无法连接上搭建的私链；
- --port字段用于指定节点之间通信的端口，默认是30303；
- --allow-insecure-unlock表示允许不安全的账户解锁行为，开启这个选项后通过http连接到私链的钱包才能解锁账户进行转账操作；
- console表示在运行私链节点的同时开启控制台，这样就可以在监控私链节点状态的同时对节点进行操作。因为geth命令运行完后会不断弹出监控日志影响到控制台的使用，因此在命令最后添加2>>geth.log就可以让监控日志输入到当前目录的geth.log文件中而不影响控制台的使用，然后在新开一个终端用tail -f geth.log的命令实时监控节点的日志即可。(如果不愿意这么麻烦可以把2>>geth.log删除)

查看日志：

```
cd ./private-eth
tail -f geth.log
```

geth启动完成后如果需要交易需要在控制台开启挖矿才能交易

```
> miner.start(8) //括号里为线程数
> null
```

`miner.stop`停止挖矿

在新命令行窗口连接私链进入新的控制台：

```
geth attach data/geth.ipc
```

转账

1. 交易前需先解锁账户

   `> personal.unlockAccount(eth.accounts[0],"密码")`

2. 发送转账交易

   ```go
   > eth.sendTransaction({from:eth.accounts[0], to:eth.accounts[1], value:web3.toWei(1000,"ether")})
   ```

geth.ipc: 同一台计算机上的其他进程可以使用该文件与 Geth 通信

## get启动流程

cmd/geth/main.go

看到main函数一上来就直接运行了。  go语言会自动按照一定的顺序先调用所有包的init()函数。然后才会调用main()函数。 

```go
func main() {
	if err := app.Run(os.Args); err != nil {
		fmt.Fprintln(os.Stderr, err)
		os.Exit(1)
	}
}
```

main.go的init函数
app是一个三方包gopkg.in/urfave/cli.v1的实例。 这个三方包的用法大致就是首先构造这个app对象。 通过代码配置app对象的行为，提供一些回调函数。然后运行的时候直接在main函数里面运行 app.Run(os.Args)就行了。

```go
import (
	...
	"gopkg.in/urfave/cli.v1"
)

var (

	app = utils.NewApp(gitCommit, "the go-ethereum command line interface")
	// flags that configure the node
	nodeFlags = []cli.Flag{
		utils.IdentityFlag,
		utils.UnlockedAccountFlag,
		utils.PasswordFileFlag,
		utils.BootnodesFlag,
		...
	}

	rpcFlags = []cli.Flag{
		utils.RPCEnabledFlag,
		utils.RPCListenAddrFlag,
		...
	}

	whisperFlags = []cli.Flag{
		utils.WhisperEnabledFlag,
		...
	}
)
func init() {
	// Initialize the CLI app and start Geth
	// Action字段表示如果用户没有输入其他的子命令的情况下，会调用这个字段指向的函数。
	app.Action = geth
	app.HideVersion = true // we have a command to print the version
	app.Copyright = "Copyright 2013-2017 The go-ethereum Authors"
	// Commands 是所有支持的子命令
	app.Commands = []cli.Command{
		// See chaincmd.go:
		initCommand,
		importCommand,
		exportCommand,
		removedbCommand,
		dumpCommand,
		// See monitorcmd.go:
		monitorCommand,
		// See accountcmd.go:
		accountCommand,
		walletCommand,
		// See consolecmd.go:
		consoleCommand,
		attachCommand,
		javascriptCommand,
		// See misccmd.go:
		makecacheCommand,
		makedagCommand,
		versionCommand,
		bugCommand,
		licenseCommand,
		// See config.go
		dumpConfigCommand,
	}
	sort.Sort(cli.CommandsByName(app.Commands))
	// 所有能够解析的Options
	app.Flags = append(app.Flags, nodeFlags...)
	app.Flags = append(app.Flags, rpcFlags...)
	app.Flags = append(app.Flags, consoleFlags...)
	app.Flags = append(app.Flags, debug.Flags...)
	app.Flags = append(app.Flags, whisperFlags...)

	app.Before = func(ctx *cli.Context) error {
		runtime.GOMAXPROCS(runtime.NumCPU())
		if err := debug.Setup(ctx); err != nil {
			return err
		}
		// Start system runtime metrics collection
		go metrics.CollectProcessMetrics(3 * time.Second)

		utils.SetupNetwork(ctx)
		return nil
	}

	app.After = func(ctx *cli.Context) error {
		debug.Exit()
		console.Stdin.Close() // Resets terminal mode.
		return nil
	}
}
```

如果我们没有输入任何的参数，那么会自动调用geth方法

```go
// geth is the main entry point into the system if no special subcommand is ran.
// It creates a default node based on the command line arguments and runs it in
// blocking mode, waiting for it to be shut down.
// 如果没有指定特殊的子命令，那么geth是系统主要的入口。
// 它会根据提供的参数创建一个默认的节点。并且以阻塞的模式运行这个节点，等待着节点被终止。
func geth(ctx *cli.Context) error {
		if args := ctx.Args().Slice(); len(args) > 0 {
		return fmt.Errorf("invalid command: %q", args[0])
	}
	prepare(ctx)
  
	stack, backend := makeFullNode(ctx) //创建全节点
	defer stack.Close()

	startNode(ctx, stack, backend, false) //开启节点
	stack.Wait() //阻塞模式运行
	return nil
}
```

makeFullNode函数：加载geth配置和创建以太坊后端
```go
//加载geth配置和创建eth后端	
func makeFullNode(ctx *cli.Context) (*node.Node, ethapi.Backend) {
		// 根据命令行参数和一些特殊的配置来创建一个node
	stack, cfg := makeConfigNode(ctx)
  
	// 把eth的服务注册到这个节点上面。 eth服务是以太坊的主要的服务。 是以太坊功能的提供者。
	backend, eth := utils.RegisterEthService(stack, &cfg.Eth)

	// Warn users to migrate if they have a legacy freezer format.
	if eth != nil && !ctx.IsSet(utils.IgnoreLegacyReceiptsFlag.Name) {
		//迁移遗留数据
		...
	}
	
	// 配置日志过滤器 RPC API， 将 eth 日志过滤器添加到节点。
	// 默认最大缓存块数是32
	// 过滤器保活状态是5分钟
	filterSystem := utils.RegisterFilterAPI(stack, backend, &cfg.Eth)

	// Configure GraphQL if requested.
	if ctx.IsSet(utils.GraphQLEnabledFlag.Name) {
		utils.RegisterGraphQLService(stack, backend, filterSystem, &cfg.Node)
	}

	// Add the Ethereum Stats daemon if requested.
	if cfg.Ethstats.URL != "" {
    //注册 以太坊的状态服务，并添加进节点。 默认情况下是没有启动的。
		utils.RegisterEthStatsService(stack, backend, cfg.Ethstats.URL)
	}
	return stack, backend
}
```

makeConfigNode。 这个函数主要是通过配置文件和flag来生成整个系统的运行配置。
```go
func makeConfigNode(ctx *cli.Context) (*node.Node, gethConfig) {
	// Load defaults.
	cfg := gethConfig{
		Eth:     ethconfig.Defaults,
		Node:    defaultNodeConfig(),
		Metrics: metrics.DefaultConfig,
	}

	// Load config file.
	if file := ctx.String(configFileFlag.Name); file != "" {
		if err := loadConfig(file, &cfg); err != nil {
			utils.Fatalf("%v", err)
		}
	}

	// Apply flags.
	utils.SetNodeConfig(ctx, &cfg.Node)
	stack, err := node.New(&cfg.Node)
	if err != nil {
		utils.Fatalf("Failed to create the protocol stack: %v", err)
	}
	// Node doesn't by default populate account manager backends
	if err := setAccountManagerBackends(stack); err != nil {
		utils.Fatalf("Failed to set account manager backends: %v", err)
	}

	utils.SetEthConfig(ctx, stack, &cfg.Eth)
	if ctx.IsSet(utils.EthStatsURLFlag.Name) {
		cfg.Ethstats.URL = ctx.String(utils.EthStatsURLFlag.Name)
	}
	applyMetricConfig(ctx, &cfg)

	return stack, cfg
}
```

RegisterEthService

```go
// RegisterEthService adds an Ethereum client to the stack.
// The second return value is the full node instance, which may be nil if the
// node is running as a light client.
func RegisterEthService(stack *node.Node, cfg *ethconfig.Config) (ethapi.Backend, *eth.Ethereum) {

	// 如果同步模式是轻量级的同步模式。 那么启动轻量级的客户端
	if cfg.SyncMode == downloader.LightSync {
      backend, err := les.New(stack, cfg)
      if err != nil {
        Fatalf("Failed to register the Ethereum service: %v", err)
      }
      stack.RegisterAPIs(tracers.APIs(backend.ApiBackend))
      if err := lescatalyst.Register(stack, backend); err != nil {
        Fatalf("Failed to register the Engine API service: %v", err)
      }
      return backend.ApiBackend, nil
	}
  
  	backend, err := eth.New(stack, cfg)  //创建一个eth对象
		if err != nil {
			Fatalf("Failed to register the Ethereum service: %v", err)
		}
  
  	// 默认LightServ的大小是0 也就是不会启动LesServer
		// LesServer是给轻量级节点提供服务的。
  	if cfg.LightServ > 0 {
			_, err := les.NewLesServer(stack, backend, cfg)
		if err != nil {
			Fatalf("Failed to create the LES server: %v", err)
			}
		}
		if err := ethcatalyst.Register(stack, backend); err != nil {
			Fatalf("Failed to register the Engine API service: %v", err)
		}
		stack.RegisterAPIs(tracers.APIs(backend.APIBackend))
		return backend.APIBackend, backend
}
```


startNode：启动系统节点和所有已注册的协议，之后他解锁任何请求的账户，并启动RPC/IPC接口和矿工

```go
func startNode(ctx *cli.Context, stack *node.Node, backend ethapi.Backend, isConsole bool) {

  debug.Memsize.Add("node", stack)
  // Start up the node itself,isConsole: 是否开启JS控制台
	utils.StartNode(ctx, stack, isConsole)


	// Unlock any account specifically requested
	unlockAccounts(ctx, stack)

  // Register wallet event handlers to open and auto-derive wallets
	events := make(chan accounts.WalletEvent, 16)
	stack.AccountManager().Subscribe(events)
  
	// Create a client to interact with local geth node.
	rpcClient, err := stack.Attach()
	if err != nil {
		utils.Fatalf("Failed to attach to self: %v", err)
	}
	ethClient := ethclient.NewClient(rpcClient)

	go func() {

		// Open any wallets already attached
		for _, wallet := range stack.AccountManager().Wallets() {
			if err := wallet.Open(""); err != nil {
				log.Warn("Failed to open wallet", "url", wallet.URL(), "err", err)
			}
		}
		// Listen for wallet event till termination
		for event := range events {
			switch event.Kind {
			case accounts.WalletArrived:
				if err := event.Wallet.Open(""); err != nil {
					log.Warn("New wallet appeared, failed to open", "url", event.Wallet.URL(), "err", err)
				}
			case accounts.WalletOpened:
				status, _ := event.Wallet.Status()
				log.Info("New wallet appeared", "url", event.Wallet.URL(), "status", status)

				var derivationPaths []accounts.DerivationPath
				if event.Wallet.URL().Scheme == "ledger" {
					derivationPaths = append(derivationPaths, accounts.LegacyLedgerBaseDerivationPath)
				}
				derivationPaths = append(derivationPaths, accounts.DefaultBaseDerivationPath)

				event.Wallet.SelfDerive(derivationPaths, ethClient)

			case accounts.WalletDropped:
				log.Info("Old wallet dropped", "url", event.Wallet.URL())
				event.Wallet.Close()
			}
		}
	}()
  
	// Spawn a standalone goroutine for status synchronization monitoring,
	// close the node when synchronization is complete if user required.
	if ctx.Bool(utils.ExitWhenSyncedFlag.Name) {//根据需求，在同步完成后关闭节点
    //开启一个独立goroutin监控状态同步
		go func() {
			sub := stack.EventMux().Subscribe(downloader.DoneEvent{})
			defer sub.Unsubscribe() //结束后移除监控
			for {
				event := <-sub.Chan()
				if event == nil {
					continue
				}
				done, ok := event.Data.(downloader.DoneEvent)
				if !ok {
					continue
				}
				if timestamp := time.Unix(int64(done.Latest.Time), 0); time.Since(timestamp) < 10*time.Minute {
					log.Info("Synchronisation completed", "latestnum", done.Latest.Number, "latesthash", done.Latest.Hash(),
						"age", common.PrettyAge(timestamp))
          //关闭节点
					stack.Close()
				}
			}
		}()
	}
  
  // Start auxiliary services if enabled
  // 根据需求开启辅助服务
	if ctx.Bool(utils.MiningEnabledFlag.Name) || ctx.Bool(utils.DeveloperFlag.Name) {
		// Mining only makes sense if a full Ethereum node is running
    // 仅当完整的以太坊节点正在运行时，挖矿才是有意义的
		if ctx.String(utils.SyncModeFlag.Name) == "light" {
			utils.Fatalf("Light clients do not support mining")
		}
		ethBackend, ok := backend.(*eth.EthAPIBackend)
		if !ok {
			utils.Fatalf("Ethereum service not running")
		}
		// Set the gas price to the limits from the CLI and start mining
		gasprice := flags.GlobalBig(ctx, utils.MinerGasPriceFlag.Name)
		ethBackend.TxPool().SetGasPrice(gasprice)
		// start mining
		threads := ctx.Int(utils.MinerThreadsFlag.Name)
		if err := ethBackend.StartMining(threads); err != nil {
			utils.Fatalf("Failed to start mining: %v", err)
		}
	}
}
```

总结:

整个启动过程其实就是解析参数。然后创建和启动节点。 然后把服务注入到节点中。 所有跟以太坊相关的功能都是以服务的形式实现的。 


如果除开所有注册进去的服务。 这个时候系统开启的goroutine有哪些。 这里做一个总结。


目前所有的常驻的goroutine有下面一些。  主要是p2p相关的服务。 以及RPC相关的服务。

![image](picture/geth_1.png)

