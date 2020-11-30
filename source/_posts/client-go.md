---
title: client-go
date: 2020-11-27 11:09:15
tags: [Kubernetes, client-go]
categories: Kubernetes源码解析
---

Kubernetes 系统使用 client-go 作为 Go 语言的官方编程式交互客户端库，提供对 Kubernetes API Server 服务的交互访问。client-go 的源码路径为 vendor/k8s.io/client-go。

## client-go 源码结构

源码目录 | 说明
--- | --- 
discovery | 提供 DiscoveryClient 发现客户端
dynamic | 提供 DynamicClient 动态客户端
informers | 每种 Kubernetes 资源的 Informer 实现
kubernetes | 提供 ClientSet 客户端
listers | 为每一个 Kubernetes 资源提供 Lister 功能，该功能对 Get 和 List 请求提供只读的缓存数据
plugin | 提供 OpenStack 、GCP 和 Azure 等云服务商授权插件
rest |提供 RESTClient 客户端，对 Kubernetes API Server 执行 RESTful 操作
scale | 提供 ScaleClient 客户端，用于扩容或缩容 Deployment,ReplicaSet,Replication Controller 等资源对象
tools | 提供常用工具，例如 SharedInformer,Reflector,DealtFIFO 及 Indexers。提供 Client 查询和缓存机制，以减少向 kube-apiserver 发起的请求数等
transport | 提供安全的 TCP 连接，支持 Http Stream,某些操作需要在客户端和容器之间传输二进制流，例如 exec,attach 等操作，该功能由内部的 spdy 包提供支持
util | 提供常用方法，例如 WorkQueue 工作队列，Certificate 证书管理等

## Client 客户端对象

* RESTClient 是最基础的客户端，RESTClient 对 HTTP Request 进行了封装，实现了 RESTful 风格的 API。ClientSet,DynamicClient 及 DiscoveryClient 客户端都是基于 RESTClient 实现的。

* ClientSet 在 RESTClient 的基础上封装了对 Resource 和 Version 的管理方法，每一个 Resource 可以理解为一个客户端，而 ClientSet 则是多个客户端的集合，每一个 Resource 和 Version 都以函数的方式暴露给开发者。ClientSet 只能够处理 Kubernetes 内置资源，它是通过 client-gen 代码生成器自动生成的。

* DynamicClient 与 ClientSet 最大的不同之处是，ClientSet 仅能访问 Kubernetes 自带的资源，不能直接访问 CRD 自定义资源，DynamicClient 能够处理 Kubernetes 中的所有资源对象，包括 Kubernetes 内置资源与 CRD 自定义资源。

* DiscoveryClient 发现客户端，用于发现 kube-apiserver 所支持的资源组、资源版本、资源信息。

以上 4 中客户端都可以通过 kubeconfig 配置信息连接到指定的 Kubernetes API Server。

### kubeconfig 配置管理

kubeconfig 用于管理访问 kube-apiserver 的配置信息，同时也支持访问多 kube-apiserver 的配置管理，可以再不同的环境下管理不同的 kube-apiserver 集群配置，不同业务线也可以拥有不同的集群，kubernetes 的其他组件都使用 kubeconfig 配置信息来连接 kube-apiserver 组件的。

Kubeconfig 配置信息如下：

```
$ cat ~/.kube/config

apiVersion: v1
kind: Config
preferences: {}

clusters:
- cluster:
    name: dev-cluster

users: 
- name: dev-user

contexts:
- context
  name: dev-context
```

* clusters: 定义 Kubernetes 集群信息，例如 kube-apiserver 的服务地址及集群的证书信息等。
* users: 定义 Kubernetes 集群用户身份验证的客户端凭据，例如 client-certificate、client-key、token及username/password 等。
* contests: 定义 Kubernetes 集群用户信息和命名空间等，用于将请求发送到指定的集群。

client-go 会读取 kubeconfig 配置信息并生成 config 对象，用于与 kube-apiserver 通信，代码示例如下： 

```
func main() {
    config, err := clientcmd.BuildConfigFromFlags("", "~/.kube/config")
    if err != nil {
        panic(err)
    }
    ...
}
```

上述代码中，clientcmd.BuildConfigFromFlags 函数会读取 kubeconfig 配置信息并实例化 rest.Config 对象，其中 kubeconfig 最核心的功能是管理多个访问 kube-apiserver 集群的配置信息，将多个配置信息合并成一份，在合并过程中会解决多个配置文件字段冲突的问题，该过程由 Load 函数完成，可以分为两步：第一步，加载 kubeconfig 配置信息；第二步，合并多个 kubeconfig 配置信息。代码示例如下：

1. 加载 kubeconfig 配置信息

```
func (rules *ClientConfigLoadingRules) Load() (*clientcmdapi.Config, error) {
    ...
    kubeConfigFiles := []string{}
    ...
    if len(rules.ExplicitPath) > 0 {
        ...
        kubeConfigFiles = append(kubeConfigFiles, rules.ExplicitPath)
    } else {
        kubeConfigFiles = append(kubeConfigFiles, rules.Precedence...)
    }
    for _, filename := range kubeConfigFiles {
        ...
        config, err := LoadFromFile(filename)
        ...
        kubeconfigs = append(kubeconfigs, config)
    }
    ...
}
```

从上可知，有以上两种方式可以获取 kubeconfig 配置信息路径：第一种，文件路径（即 rules.ExplicitPath）；第二种，环境变量（通过 KUBECONFIG 变量，即 rules.Precedence,可指定多个路径）。最后将配置信息汇总到 kubeConfigFiles 中，这两种方式都通过 LoadFromFile 函数读取数据并把读取到的数据反序列化到 Config 对象中。代码示例如下：

```
func Load(data []byte) (*clientcmdapi.Config, error) {
    config := clientcmdapi.NewConfig()
    ...
    decoded, _, err := clientcmdlatest.Codec.Decode(data, &schema.GroupVersionKind{Version: clientcmdlatest.Version, Kind: "Config"}, config)
    ...
    return decoded.(*clientcmdapi.Config), nil
}
```

2. 合并多个 kubeconfig 配置信息

代码示例如下：

```
config := clientcmdapi.NewConfig()
mergo.MergeWithOverwrite(config, mapConfig)
mergo.MergeWithOverwrite(config, nonMapConfig)
```

mergo.MergeWithOverwrite 函数将 src 字段填充到 dst 结构中，私有字段除外，非空的 dst 字段将被覆盖，另外 dst 和 src 必须拥有有效的相同类型结构。

### RESTClient 客户端

它具有很高的灵活性，数据不依赖于方法和资源，因此 RESTClient 能够处理多种类型的调用，返回不同的数据格式。

RESTClient Example 代码示例如下：

```
func main() {
    config, err := clientcmd.BuildConfigFromFlags("", "/root/.kube/config")
    if err != nil {
        panic(err)
    }
    config.APIPath = "api"
    config.GroupVersion = &corev1.SchemeGroupVersion
    config.NegotiatedSerializer = scheme.Codecs

    restClient, err := rest.RESTClientFor(config)
    // 通过 kubeconfig 配置信息实例化 RESTClient 对象。
    if err != nil {
        panic(err)
    }

    result := &corev1.PodList{}
    err = restClient.Get().Namespace("default").Resource("pods").VersionedParams(&metav1.ListOptions{Limit:500}, scheme.ParameterCodec).Do().Into(result)
    // RESTClient 对象构建 HTTP 请求参数。例如 GET, POST, PUT, DELETE, PATCH 等
    // VersionedParams 函数将一些查询选项添加到请求参数中
    if err != nil {
        panic(err)
    }

    for _, d := range result.Items {
        fmt.Printf("NAMESPACE:%v \t NAME: %v \t STATU: %+v \n", d.Namespace, d.Name, d.Status.Phase)
    }
}
```

以上代码列出 defult 命名空间下的所有 Pod 资源对象的相关信息。首先加载 kubeconfig 配置信息，并设置 config.APIPath 请求的 HTTP 路径，然后设置 config.GroupVersion 请求的资源组/资源版本。最后设置 config.NegotiatedSerializer 数据的编码器。

RESTClient 发送请求的过程对 Go 语言标准库 net/http 进行了封装，由 Do -> request 函数实现，代码示例如下：

```
func (r *Request) Do() Result {
    ...
    var result Result
    err := r.request(func(req *http.Request, resp *http.Response) {
        result = r.transformResponse(resp, req)
    })
}

func (r *Request) request(fn func(*http.Request, *http.Response)) error {
    ...
    for {
        url := r.URL().String()
        // 生成请求的 RESTful URL
        req, err := http.NewRequest(r.verb, url, r.body)
        // 通过 Go 语言标准库 net/http 向 RESTful URL(即 kube-apiserver)发送请求。
        if err != nil {
            return err
        }
        ...
        req.Header = r.headers
        ...
        resp, err := client.Do(req)
        ...
        if err != nil {
            if !net.IsConnectionReset(err) || r.verb != "GET" {
                return err
            }
            resp = &http.Response{
                StatusCode: http.StatusInternalServerError,
                Header: http.Header{"Retry-After": []string{"1"}},
                Body:   iotil.NopCloser(bytes.NewReader([]byte{})),
                // 请求得到的结果存放在 http.Respose 的 Body 中
            }
        }
        ...
        resp.Body.Close()
        // 函数退出时，会通过此命令进行关闭，防止内存溢出
        ...
        fn(req, resp)
        // 将结果转换为资源对象
        ...
    }
}
```

### ClientSet 客户端




