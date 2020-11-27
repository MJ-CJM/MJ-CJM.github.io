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

## 创建资源对象的过程

内部运行原理是，客户端和服务端进行一次 HTTP 请求的交互。创建资源对象的流程可分为：
1. 实例化 Factory 接口，通过 Builder 和 Visitor 将资源对象描述文件（xxx.yaml）文本格式转换成资源对象。
2. 将资源对象以 HTTP 请求的方式发送给 kube-apiserver，并得到响应结果。
3. 最终根据 Visitor 匿名函数集的 errors 判断是否成功创建了资源对象。

### 编写资源对象描述文件

Kubernetes 系统的资源对象可以使用 JSON 或 YAML 文件来描述,一般使用 YAML 文件居多。

```
apiVersion: v1       #必选，版本号，例如v1
kind: Pod       #必选，Pod
metadata:       #必选，元数据
  name: string       #必选，Pod名称
  namespace: string    #必选，Pod所属的命名空间
  labels:      #自定义标签
    - name: string     #自定义标签名字
  annotations:       #自定义注释列表
    - name: string
spec:         #必选，Pod中容器的详细定义
  containers:      #必选，Pod中容器列表
  - name: string     #必选，容器名称
    image: string    #必选，容器的镜像名称
    imagePullPolicy: [Always | Never | IfNotPresent] #获取镜像的策略 Alawys表示下载镜像 IfnotPresent表示优先使用本地镜像，否则下载镜像，Nerver表示仅使用本地镜像
    command: [string]    #容器的启动命令列表，如不指定，使用打包时使用的启动命令
    args: [string]     #容器的启动命令参数列表
    workingDir: string     #容器的工作目录
    volumeMounts:    #挂载到容器内部的存储卷配置
    - name: string     #引用pod定义的共享存储卷的名称，需用volumes[]部分定义的的卷名
      mountPath: string    #存储卷在容器内mount的绝对路径，应少于512字符
      readOnly: boolean    #是否为只读模式
    ports:       #需要暴露的端口库号列表
    - name: string     #端口号名称
      containerPort: int   #容器需要监听的端口号
      hostPort: int    #容器所在主机需要监听的端口号，默认与Container相同
      protocol: string     #端口协议，支持TCP和UDP，默认TCP
    env:       #容器运行前需设置的环境变量列表
    - name: string     #环境变量名称
      value: string    #环境变量的值
    resources:       #资源限制和请求的设置
      limits:      #资源限制的设置
        cpu: string    #Cpu的限制，单位为core数，将用于docker run --cpu-shares参数
        memory: string     #内存限制，单位可以为Mib/Gib，将用于docker run --memory参数
      requests:      #资源请求的设置
        cpu: string    #Cpu请求，容器启动的初始可用数量
        memory: string     #内存清楚，容器启动的初始可用数量
    livenessProbe:     #对Pod内个容器健康检查的设置，当探测无响应几次后将自动重启该容器，检查方法有exec、httpGet和tcpSocket，对一个容器只需设置其中一种方法即可
      exec:      #对Pod容器内检查方式设置为exec方式
        command: [string]  #exec方式需要制定的命令或脚本
      httpGet:       #对Pod内个容器健康检查方法设置为HttpGet，需要制定Path、port
        path: string
        port: number
        host: string
        scheme: string
        HttpHeaders:
        - name: string
          value: string
      tcpSocket:     #对Pod内个容器健康检查方式设置为tcpSocket方式
         port: number
       initialDelaySeconds: 0  #容器启动完成后首次探测的时间，单位为秒
       timeoutSeconds: 0   #对容器健康检查探测等待响应的超时时间，单位秒，默认1秒
       periodSeconds: 0    #对容器监控检查的定期探测时间设置，单位秒，默认10秒一次
       successThreshold: 0
       failureThreshold: 0
       securityContext:
         privileged:false
    restartPolicy: [Always | Never | OnFailure]#Pod的重启策略，Always表示一旦不管以何种方式终止运行，kubelet都将重启，OnFailure表示只有Pod以非0退出码退出才重启，Nerver表示不再重启该Pod
    nodeSelector: obeject  #设置NodeSelector表示将该Pod调度到包含这个label的node上，以key：value的格式指定
    imagePullSecrets:    #Pull镜像时使用的secret名称，以key：secretkey格式指定
    - name: string
    hostNetwork:false      #是否使用主机网络模式，默认为false，如果设置为true，表示使用宿主机网络
    volumes:       #在该pod上定义共享存储卷列表
    - name: string     #共享存储卷名称 （volumes类型有很多种）
      emptyDir: {}     #类型为emtyDir的存储卷，与Pod同生命周期的一个临时目录。为空值
      hostPath: string     #类型为hostPath的存储卷，表示挂载Pod所在宿主机的目录
        path: string     #Pod所在宿主机的目录，将被用于同期中mount的目录
      secret:      #类型为secret的存储卷，挂载集群与定义的secre对象到容器内部
        scretname: string  
        items:     
        - key: string
          path: string
      configMap:     #类型为configMap的存储卷，挂载预定义的configMap对象到容器内部
        name: string
        items:
        - key: string
```

通过 kubectl create 命令与 kube-apiserver 交互并创建资源对象。（kubectl create -f XXX.yaml）

### 实例化 Factory 接口

在执行每一个 kubectl 命令之前，都需要实例化 cmdutil Factory 接口对象的操作，Factory 是一个通用对象，它提供了与 kube-apiserver 的交互方式，以及验证资源对象等方法。cmdutil Factory 接口代码示例如下：

```
f := cmdutil.NewFactory(matchVersionKubeConfigFlags)

type Factory interface {
    DynamicClient()
    // 动态客户端
    KubernetesClientSet()
    // ClientSet客户端
    RESTClient()
    // RESTClient客户端
    NewBuilder()
    // 实例化 Builder,Builder 用于将命令行获取的参数转换成资源对象
    Validator(...)
    // 验证资源对象
    ...
}
```

### Builder 构建资源对象

Builder 用于将命令行获取的参数转换成资源对象，它实现了一种通用的资源对象转换功能。
Builder 结构体保存了命令行获取的各种参数，并通过不同函数处理不同参数，将其转换成资源对象。

```
r := f.NewBuilder().
    Unstructured().
    Schema(schema).
    ContinueOnError().
    NamespaceParam(cmdNamespace).DefaultNamespace().
    FilenameParam(enforceNamespace, &o.FilenameOptions).
    LabelSelectorParam(o.Selector).
    Flatten().
    Do()
err = r.Err()
if err != nil {
    return err
}
```

首先通过 f.NewBuilder 实例化 Builder 对象，通过函数 Unstructured 等对参数赋值和初始化，将参数保存到 Builder 对象中，最后通过 Do 函数完成对资源的创建。

其中，FilenameParam 函数用于识别 kubectl create 命令行参数是通过哪种方式传入资源对象描述文件：
1. 标准输入 stdin
2. 本地文件
3. 网络文件

Do 函数返回 **Result 对象**，Result 对象的 **info 字段**保存了 RESTClient 与 kube-apiserver 交互产生的结果，可以通过 Result 对象的 infos 或 Object 方法来获取执行结果，而 Result 对象中的结果，是由 **Visitor 执行产生**。

### Visitor 多层匿名函数嵌套

Result 对象中的结果，是由 **Visitor** 执行并产生，Visitor 接口定义如下：

```
type Visitor interface {
    Visit(VisitorFunc) error
}
type VisitorFunc func(*info, error) error
// 该匿名函数则生成或处理 Info 结构
```

在 Kubernetes 源码中，Visitors 被设计为可以多层嵌套（即多层匿名函数嵌套，使用一个 Visitor 嵌套另一个 Visitor）。

Visitor Example 代码示例如下：

```
type Visitor interface {
    Visit(VisitorFunc) error
}

type VisitorFunc func() error

type VisitorList []Visitor

func (l VisitorList) Visit(fn VisitorFunc) error {
    for i := range l {
        if err := l[i].Visit(func() error {
            fmt.Println("In VisitorList before fn")
            fn()
            fmt.Println("In VisitorList after fn")
            return nil
        }); err != nil {
            return err
        }
    }
    return nil
}

type Visitor1 struct {
}

func (v Visitor1) Visit(fn VisitorFunc) error {
    fmt.Println("In Visitor1 before fn")
    fn()
    fmt.Println("In Visitor1 after fn")
    return nil
}

type Visitor2 struct {
    visitor Visitor
}

func (v Visitor2) Visit(fn VisitorFunc) error {
    v.visitor.Visit(func() error {
        fmt.Println("In Visitor2 before fn")
        fn()
        fmt.Println("In Visitor2 after fn")
        return nil
    })
    return nil
}

type Visitor3 struct {
    visitor Visitor
}

func (v Visitor3) Visit(fn VisitorFunc) error {
    v.visitor.Visit(func() error {
        fmt.Println("In Visitor3 before fn")
        fn()
        fmt.Println("In Visitor3 after fn")
        return nil
    })
    return nil
}

func main() {
    var visitor Visitor
    var visitors []Visitor

    visitor = Visitor1{}
    visitors = append(visitors, visitor)
    visitor = Visitor2{VisitorList(visitors)}
    visitor = Visitor3{visitor}
    visitor.Visit(func() error {
        fmt.Println("In visitFunc")
        return nil
    })
}
```

* 其中定义了 Visitor 接口，增加了 VisitorList 对象，该对象相当于多个 Visitor 匿名函数的集合，另外增加了 3 个 Visitor 的类，分别实现 Visit 方法，该方法的 VisitorFunc 函数在执行之前和执行之后分贝输出 print 信息。
* 在 main 函数中，首先将 Visitor1 嵌入 VisitorList 中，VisitorList 是 Visitor 的集合，可存放多个 Visitor。然后将 VisitorList 嵌入 Visitor2 中，接着将 Visitor2 嵌入 Visitor3 中，最终形成 Visitor3{Visitor2{VisitorList{Visitor1}}} 的嵌套关系。

Kubernetes 源码中的 Visitor，代码示例如下：

```
type EagerVisitorList []Visitor
// 当遍历执行 Visitor 时，如果遇到错误，则保留错误信息，继续遍历执行下一个 Visitor，最后一起返回所有错误。
type VisitorList []Visitor
// 当遍历执行 Visitor 时，如果遇到错误，则立刻返回。
```

Kubernetes Visitor 中存在多种实现方法，不同实现方法的作用不同，最终通过 Visitor 的 error 信息为空判断创建资源请求执行成功。