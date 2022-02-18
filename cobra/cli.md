<!-- ---
title: golang cli
date: 2018-10-29 20:11:40
category: src, cli
--- -->

golang cli

cli 应用开发规范

https://clig.dev/#guidelines

目录结构

```
command.go //主代码文件
args.go //参数处理
cobra.go
```

## 架构



## command.go

```go
// 主结构体
type Command struct {
	// Use is the one-line usage message.
	Use string

	// Expected arguments
	Args PositionalArgs

	// Run: Typically the actual work function. Most commands will only implement this.
	Run func(cmd *Command, args []string)

	// commands is the list of commands supported by this program.
	commands []*Command
	// parent is a parent command for this command.
	parent *Command
	
	// args is actual args parsed from flags.
	args []string
	// flags is full set of flags.
	flags *flag.FlagSet
	
	// helpFunc is help func defined by user.
	helpFunc func(*Command, []string)
}
```


入口程序

1. root cmd 执行Execute
2. Execute 中根据命令行参数确定子命令
3. 执行子命令 execute
4. 判断子命令
5. 运行命令的Run


## 参考资料

- [https://github.com/spf13/cobra](https://github.com/spf13/cobra)