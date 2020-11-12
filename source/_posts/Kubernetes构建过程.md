---
title: Kubernetes构建过程1.0
date: 2020-11-12 10:34:42
tags: Kubernetes
categories: Kubernetes源码解析
---

## 代码生成器

### 举例5个代码生成器

代码生成器 | 说明 
--- | ---
conversion-gen | 自动生成 Convert 函数的代码生成器，用于资源对象的版本转换函数
deepcopy-gen | 自动生成 DeepCopy 函数的代码生成器，用于资源对象的深复制函数
defaulter-gen | 自动生成 Defaulter 函数的代码生成器，用于资源对象的默认值函数
go-bindata | 是一个第三方工具，它能够将静态资源文件嵌入 Go 语言中
openapi-gen | 自动生成 OpenAPI 定义文件的代码生成器

### Tags 

代码生成器通过 Tags(标签)来识别一个包是否需要生成代码及确定生成代码的方式，Kubernetes 提供的 Tags 可以分为如下两种，Tags 被定义在注释中。

#### 全局 Tags

* 定义在每个包的 doc.go文件中，对整个包中的类型自动生成代码
* 代码示例如下：

```
// +k8s:deepcopy-gen=package
// +groupName=example.com
```

该示例表示为包中的每个类型自动生成 DeepCopy 函数，其中// +groupName定义了资源组名称，资源组名称一般用域名形式表示

#### 局部 Tags

* 定义在 Go 语言的类型声明上方，只对指定的类型自动生成代码
* 代码示例如下：

```
// +genclient
// +k8s:deepcopy-gen:interfaces=k8s.io/apimachinery/pkg/runtime.Object

// Pod...
```

该代码示例局部 Tags 定义在 Pod 资源类型的上方，定义并执行两个代码生成器。
> Kubernetes 的 API 文档生成器会根据类型声明的注释信息生成文档，为了避免 Tags 信息出现在文档中，所以将 Tags 定义在注释的上方并空一行

#### deepcopy-gen 代码生成器

给定一个包的目录路径作为输入源，它可以为其生成 DeepCopy 相关函数，这些函数可以有效地执行每种类型的深复制操作。

有如下几种 Tags 形式：
* 为整个包生成 DeepCopy 相关函数：
```
// +k8s:deepcopy-gen=package
```
* 为单个类型生成 DeepCopy 相关函数：
```
// +k8s:deepcopy-gen=true
```
* 为整个包生成 DeepCopy 相关函数时，可以忽略单个类型：
```
// +k8s:deepcopy-gen=false
```

* deepcopy-gen 会遍历包中所有类型，若类型为 types.Struct,则会为该类型生成深复制函数。

#### defaulter-gen 代码生成器

给定一个包的目录路径作为输入源，它可以为其生成 Defaulter 相关函数，这些函数可以为资源对象生成默认值。

* 为拥有不同属性的类型生成不同的 Defaulter 相关函数，其 Tags 形式如下：

```
// +k8s:defaulter-gen=TypeMeta/ListMeta/ObjectMeta
```

* defaulter-gen-input 说明当前包会依赖于指定的路径包，代码示例如下：

```
// +k8s:defaulter-gen-input=../../../vendor/k8s.io/api/rbac/v1
```

* defaulter-gen 会遍历包中所有类型，若类型属性拥有以上三种特定类型，则为该类型生成 Defaulter 函数，并为其生成 RegisterDefaults 注册函数。