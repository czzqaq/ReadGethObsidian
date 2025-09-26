
# urfave 

[官方文档](https://cli.urfave.org/v2/getting-started/)

这里，我结合geth `cmd/` 的文件结构，分析如何使用它。

## app


定义了app：
```go
app.Action = geth
app.Name = "geth"
app.Usage = "The go-ethereum command-line interface"
app.Version = "1.12.0"
app.Commands = []*cli.Command{
    // See chaincmd.go:
    initCommand,
    importCommand,
    //...

}
app.Flags = slices.Concat(
nodeFlags,
rpcFlags,
consoleFlags,
debug.Flags,
metricsFlags,
)

app.Before = func(ctx *cli.Context) error {
    //...
}

app.After = func(ctx *cli.Context) error {
   //...
}

```

这样定义了main 函数：

```go
func main() {
    if err := app.Run(os.Args); err != nil {
        fmt.Fprintln(os.Stderr, err)
        os.Exit(1)
    }
}
```

### Action

定义默认行为。当用户运行主命令（如 `geth`）而不提供任何子命令时，会执行 `Action` 指定的函数。

也就是 
```
geth
```
会运行它。

### version
```
> geth --version

1.12.0
```

### Commands
定义应用的子命令列表。
比如，如果用户运行 `geth init`，CLI 会解析 `Commands`，找到 `initCommand` 并执行其逻辑。

### flags
这些标志适用于主命令和所有子命令，全局标志。子命令可以定义自己的标志，但全局flag 都可用。

和子命令的flags 定义方法一致。

```
geth --datadir /path/to/data init
```


### before after

在执行任何命令或子命令之前之后运行的钩子函数。

before: 设置环境变量、日志配置、检查依赖等
after：关闭日志、释放文件句柄等

## 子命令
```
var (
	initCommand = &cli.Command{
		Action:    initGenesis,
		Name:      "init",
		Usage:     "Bootstrap and initialize a new genesis block",
		ArgsUsage: "<genesisPath>",
		Flags: slices.Concat([]cli.Flag{
			utils.CachePreimagesFlag,
			utils.OverridePrague,
			utils.OverrideVerkle,
		}, utils.DatabaseFlags),
		Description: `
The init command initializes a new genesis block and definition for the network.
This is a destructive action and changes the network in which you will be
participating.

It expects the genesis file as argument.`,
	}
)
```

包括字段：
### Action
这是命令的主要逻辑处理函数。当用户执行 `geth init` 时，会调用 `initGenesis` 函数。
例如，initGenesis 定义为：
```go
func initGenesis(ctx *cli.Context) error
```
这个定义是一定要定义成这样。


### name 
用户在命令行中输入的字符串。
比如，用
```shell
geth init
```
可以触发这个命令

### usage
显示在帮助中

```bash
$ geth help
...
COMMANDS:
   init    Bootstrap and initialize a new genesis block
...
```
### ArgsUsage
显式在帮助中，具体arg 怎么标记

```bash
$ geth init --help
USAGE:
   geth init <genesisPath>
```


### flags

```go
Flags: slices.Concat([]cli.Flag{
	utils.CachePreimagesFlag,
	utils.OverridePrague,
	utils.OverrideVerkle,
}, utils.DatabaseFlags),

// in cmd/utils/flags.go
CachePreimagesFlag = &cli.BoolFlag{
    Name:     "cache.preimages",
    Usage:    "Enable recording the SHA3/keccak preimages of trie keys",
    Category: flags.PerfCategory,
}
```

用户可以通过类似以下命令使用这些标志：
```bash
geth init genesis.json --cache.preimages --override.prague
```

flag 分组的作用是，在help 中，展示起来更方便。

```
$ geth --help

USAGE:
   geth [global options] command [command options] [arguments...]

GLOBAL OPTIONS:
   --help    Show help

CATEGORIES:
   PERFORMANCE TUNING:
      --cache.preimages        Enable recording the SHA3/keccak preimages of trie keys

   NETWORKING:
      --override.prague        Override the Prague network configuration
```


### Description
`geth init --help` 中显示：

```bash
$ geth init --help
NAME:
   geth init - Bootstrap and initialize a new genesis block

USAGE:
   geth init <genesisPath>

DESCRIPTION:
   The init command initializes a new genesis block and definition for the network.
   This is a destructive action and changes the network in which you will be
   participating.

   It expects the genesis file as argument.
```


## Context

### 获取flag
```go
app := &cli.App{
    Flags: []cli.Flag{
        &cli.StringFlag{
            Name:  "datadir",
            Usage: "Directory for storing data",
        },
        &cli.BoolFlag{
            Name:  "verbose",
            Usage: "Enable verbose logging",
        },
    },
    Action: func(ctx *cli.Context) error {
        if not ctx.IsSet("datadir") {
           fmt.Printf("should set datadir")
           os.Exit(1)
        }
        datadir := ctx.String("datadir")
        verbose := ctx.Bool("verbose")
        fmt.Printf("Data directory: %s\n", datadir)
        fmt.Printf("Verbose mode: %t\n", verbose)
        return nil
    },
}
```

### 获取argument

```go
if ctx.Args().Len() != 1 {
    utils.Fatalf("need genesis.json file as the only argument")
}
genesisPath := ctx.Args().First() // 或者 Get(0)
```


# 构建app

1. 在main.go 中，定义app，app 的定义应该放在init 下。
2. app.Commands 散布在不同文件下，这里import 进来。包括具体的Command 定义，以及其action
3. 可能有非常大量的flags，使用`slices.Concat`，按组定义。
4. main 里调用 `app.Run(os.Args)`
5. Action 函数，基本遵循检查入参->解析入参->具体行为的过程。如果`return nil`，代表成功，失败可以是用print+exit(1)，不过最好是：`return fmt.Errorf("invalid command: %q", args[0])`

