<!-- ---
title: etcd etcdctl
date: 2019-08-04 01:09:49
category: showcode, etcd
--- -->

# etcdctl 入口函数

> etcdctl 是etcd 终端工具。

## main 函数

etcdctl 终端工具基于`github.com/spf13/cobra` 终端库编写。

```go
// go.etcd.io/etcd/etcdctl/main.go
import (
	"go.etcd.io/etcd/etcdctl/ctlv3"
	// ...
)

// 使用v3 版本终端功能
func main() {
	// 查看v3 版本逻辑
	ctlv3.Start()
}
```

## init 初始化

`go.etcd.io/etcd/etcdctl/ctlv3` 包的`init` 初始化函数，在`main.go` 文件中引入包时会执行。

`init` 初始化时解析终端参数，并且注册终端子命令。

```go
// go.etcd.io/etcd/etcdctl/ctlv3/ctl.go

// 创建rootCmd 根命令，这里根命令没有Run 函数
var (
	rootCmd = &cobra.Command{
		Use:        cliName,
		Short:      cliDescription,
		SuggestFor: []string{"etcdctl"},
	}
)

// init 在引入包时初始化执行
func init() {
	// 解析终端参数
	rootCmd.PersistentFlags().StringSliceVar(&globalFlags.Endpoints, "endpoints", []string{"127.0.0.1:2379"}, "gRPC endpoints")
	// ...

	// 注册etcdctl 子命令
	rootCmd.AddCommand(
		command.NewGetCommand(),
		command.NewPutCommand(),
		command.NewAlarmCommand(),
		command.NewWatchCommand(),
		command.NewVersionCommand(),
		command.NewLeaseCommand(),
		command.NewMemberCommand(),
		// ...
	)
}
```

## Start 启动etcdctl

运行etcdctl 终端命令，这里执行`rootCmd.Execute`。因为`rootCmd` 作为根命令，负责从终端参数中解析出子命令名称，再从注册的子命令中找出子命令运行。

```go
// go.etcd.io/etcd/etcdctl/ctlv3/ctl_nocov.go

// 运行终端命令
func Start() {
	// ...
	if err := rootCmd.Execute(); err != nil {
		// 运行抛出err 时，需要打印err
		command.ExitWithError(command.ExitError, err)
	}
}

// Execute 运行命令
func (c *Command) Execute() error {
	_, err := c.ExecuteC()
	return err
}

// ExecuteC 执行命令逻辑
// 这里会根据终端参数中解析出子命令名称，再从注册的子命令中找出子命令运行
func (c *Command) ExecuteC() (cmd *Command, err error) {
	// ...
	args = os.Args[1:]
	// 根据终端参数中解析出子命令
	cmd, flags, err = c.Find(args)

	// 运行子命令
	err = cmd.execute(flags)
	// ...

	return cmd, err
}

// 子命令对象上执行命令逻辑
func (c *Command) execute(a []string) (err error) {
	// 解析子命令注册的终端参数
	err = c.ParseFlags(a)
	// ...

	// 执行子命令主体逻辑
	// c.Run 函数在创建子命令时指定
	c.Run(c, argWoFlags)
	// ...

	return nil
}
```

## 参考资料

- go.etcd.io/etcd > etcdctl/main.go

