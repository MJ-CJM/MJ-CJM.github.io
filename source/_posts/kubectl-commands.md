---
title: kubectl 命令行交互
date: 2020-11-24 10:04:56
tags: [Kubernetes, kubectl 命令行交互]
categories: Kubernetes源码解析
---

从 Kubernetes 架构设计的角度来看，kubectl 工具是 Kubernetes API Server 的客户端。一些命令可以自行查阅。

## Cobra 命令行参数解析

Cobra 是一个创建强大的现代化 CLI 命令行应用程序的 Go 语言库，可以用来生成应用程序的文件。

Cobra Example:

```
func main() {
    var Version bool
    var rootCmd = &cobra.Command{
        Use: "root[sub]",
        Short: "root command",
        Run: func(cmd *cobra.Command, args []string) {
            fmt.Printf("Inside rootCmd Run with args: %v\n", args)
            if Version {
                fmt.Printf("Version: 1.0\n")
            }
        },
    }

    flags := rootCmd.Flags()
    flags.BoolVarP(&Version, "version", "v", false, "Print version information and quit")
    _ = rootCmd.Execute()
}
```

由此可知 Cobra 基本应用步骤分为如下 3 步：

1. 创建 rootCmd 主命令，并定义 Run 执行函数，也可以通过 rootCmd.AddCommand 方法添加子命令。
2. 为命令添加命令行参数。
3. 执行 rootCmd 命令调用的函数，rootCmd.Execute 会在内部回调 Run 执行函数。

Kubernetes 核心组件都通过 Cobra 来管理 CLI 交互方式，
下面以 kubectl 为例：

```
kubectl    get   pod  pod_name  -n kube-system
App Name/Command/Type/  Name  / Flag
```

* Command: 指定命令操作，命令后可以加子命令。
* TYPE: 指定资源类型，资源类型不区分大小写。
* NAME: 指定资源名称，可以指定多个，资源名称需要区分大小写。
* Flag: 指定可选命令行参数

同样是三步骤 1：创建 Command, 2: 为 get 命令添加命令行参数 3：执行命令

### 创建 Command 

实例化 cobra.Command 对象，并通过 cmds.AddCommand 方法添加命令或子命令，每个 cobra.Command 对象都可设置 Run 执行函数，
代码示例如下：

```
func NewKubectlCommand(in io.Reader, out, err io.Writer)  *cobra.Command {
    ...
    groups := templates.CommandGroups{
        ...
        {
            Message: "Basic Commands (Intermediate):",
            Commands: []*cobra.Command{
                explain.NewCmdExplain("kubectl", f, ioStreams),
                get.NewCmdGet("kubectl", f, isStreams),
                edit.NewCmdEdit(f, ioStreams),
                delete.NewCmdDelete(f, ioStreams),
            },
        },
        ...
    }
    groups.Add(cmds)
    ...
    cmds.AddCommand(alpha)
    cmds.AddCommand(cmdconfig.NewCmdConfig(f, clientcmd.NewDefaultPathOptions(), ioStreams))
    cmds.AddCommand(plugin.NewCmdPlugin(f, ioStreams))
    cmds.AddCommand(version.NewCmdVersion(f, ioStreams))
    ...

    return cmds
}
```

NewKubectlCommand 函数实例化了 cobra.Command 对象，templates.CommandGroups 定义了 kubectl 的 8 种命令类别，通过 cmds.AddCommand 添加命令类别。
get 命令的 Command 定义如下：

```
func NewCmdGet (parent string, f cmdutil.Factory, streams genericclioptions.IOStreams) *cobra.Command {
    o := NewGetOptions(parent, streams)

    cmd := &cobra.Command {
        Use: "get ...",
        DisableFlagsInUseLine: true,
        Short: ...
        Long: ...
        Example: getExample,
        Run: func(cmd *cobra.Command, args []string) {
            cmdutil.CheckErr(o.Complete(f, cmd, args))
            cmdutil.CheckErr(o.Validate(cmd))
            cmdutil.CheckErr(o.Run(f, cmd, args))
        },
        SuggestFor: []string{"list", "ps"},
    }
    ...
} 
```

在 cobra.Command 对象中， Use, Short, Long 和 Example 包含描述命令的信息，最重要的是定义 Run 执行函数，
> Cobra 中 Run 函数家族成员有很多，执行顺序有 PersistentPreRun -> PreRun -> Run -> PostRun -> PersistentPostRun。具体参考 cobra.Command 中的结构体定义。

### 为 get 命令添加命令行参数

get 命令行参数比较多，这里以 --all -namespaces 参数为例:

```
func NewCmdGet (parent string, f cmdutil.Factory, streams genericclioptions.IOStreams) *cobra.Command {
    ...
    cmd.Flags().BoolVarP(&o.AllNamespaces // 接受命令行参数的变量, "all-namespaces" // 指定命令行参数的名称, "A" // 指定命令行参数的名称简写, o.AllNamespaces // 设置命令行参数的默认值, "If present, list the requested object(s) across all namespqaces.Namespace in current context is ignored even if specified with --namespace." // 设置命令行参数的提示信息)
    ...
}
```

### 执行命令

```
func main() {
    command := cmd.NewDefaultKubectlCommand()
    ...
    if err := command.Execute(); err != nil {
        fmt.Printf(os.Stderr, "%v\n", err)
        os.Exit()
    }
}
```

kubectl 的 main 函数中定义了执行函数 command.Execute，原理是对命令中的所有参数解析出 Command 和 Flag，把 Flag 作为参数传递给 Command 并执行。

```
cmd, flags, err = c.Find(args)
...
err = cmd.execute(flags)
```

args 数组中包含所有命令行参数，通过 c.Find 解析出 cmd 和 flags，然后通过 cmd.execute 执行命令中定义的 Run 执行函数。

