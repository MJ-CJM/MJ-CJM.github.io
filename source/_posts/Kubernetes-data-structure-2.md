---
title: Kubernetes核心数据结构2.0
date: 2020-11-21 20:45:48
tags: Kubernetes
categories: Kubernetes源码解析
---

## 数据

### 结构化数据

预先知道数据结构的数据类型是结构化数据。要使用这种数据，需要创建一个 struct 数据结构，并且可以通过 Go 语言的 json 库进行反序列化操作。

### 非结构化数据

无法预知数据结构的数据类型或属性名称不确定的数据类型是非结构化数据，其无法通过构建预定的 struct 数据结构来序列化或反序列化数据。

可以通过如下结构来解决问题：

```
var result map[string]interface{}
```

> interface {}类型对应值，可以是任何类型，使用 interface 字段时，通过 Go 语言断言的方式进行类型转换。

```
if description, ok := result["description"].(string); ok {
    fmt.Println(description)
}
```

## Scheme 资源注册表

Kubernetes 系统拥有众多资源，每一种资源就是一个资源类型，这些资源类型需要有统一的注册、存储、查询、管理等机制。目前 Kubernetes 系统中的所有资源类型都已注册到 Scheme 资源注册表中，其是一个内存型的资源注册表。

Scheme 资源注册表支持如下两种类型：

* UnversionedType: 无版本资源类型，这是早期 Kubernetes 系统中的概念，它主要应用于某些没有版本的资源类型，该类型的资源对象并不需要进行转换。

* KnownType: 目前 Kubernetes 最常用的资源类型，也可称其为“拥有版本的资源类型”。

### Scheme 资源注册表数据结构

Scheme 资源注册表数据结构主要由4个map结构组成，分别是：

```
type Scheme struct {
    gvkToType map[scheme.GroupVersionKind]reflect.Type
    // 存储 GVK 与 Type 的映射关系
    
    typeToGVK map[reflect.Type][]scheme.GroupVersionKind
    // 存储 Type 与 GVK 的映射关系，一个 Type 会对应一个或多个 GVK
    
    unversionedTypes map[reflect.Type]scheme.GroupVersionKind
    // 存储 UnversionedType 与 GVK 的映射关系
    
    unversionedKinds map[string]reflect.Type
    // 存储 Kind 名称与 UnversionedType 的映射关系 
}
```

> 这些映射关系可以实现高效的正向和反向检索。

Scheme 资源注册表在 Kubernetes 系统体系中属于非常核心的数据结构，Scheme Example 代码示例如下：

```
func main() {
    // KnownType external
    coreGV := schema.GroupVersion{Group:"", Version: "v1"}
    extensionsGV := schema.GroupVersion{Group: "extensions", Version: "v1beta1"}
    
    // KnownType internal
    coreInternalGV := schema.GroupVersion{Group: "", Version: runtime.APIVersionInternal}

    // UnversionedType 
    Unversioned := schema.GroupVersion{Group: "", Version: "v1"}

    schema := runtime.NewScheme()
    scheme.AddKnownTypes(coreGV, &corev1.Pod{})
    scheme.AddKnownTypes(extensionsGV, &appsv1.DaemonSet{})
    scheme.AddKnownTypes(coreInternalGV, &corev1.Pod{})
    scheme.AddUnversionedTypes(Unversioned, &metav1.Status{})
    // 注册资源类型到 Scheme 资源注册表有以上两种方式
}
```

* GVK 在 Scheme 资源注册表中以 <group>/<version>,Kind=<kind>的形式存在，其中对于 Kind 字段，在注册时如果不指定该字段的名称，那么默认使用类型的名称，通过 reflect 机制获取资源类型的名称。

* 资源类型在 Scheme 资源注册表中以 Go Type（通过 reflect 机制获取）形式存在。

> 需要注意的是，UnversionecdType 类型的对象在通过 scheme.AddUnversionedTypes 方法注册时，会同时存在4个 map 结构中，代码示例如下：

```
func (s *Scheme) AddUnversionedTypes(version schema.GroupVersion, types ...Object) {
    ...
    s.AddKnownTypes(version, types...)
    // 1
    for _, obj := range types {
        t := reflect.TypeOf(obj).Elem()
        gvk := version.WithKind(t.Name())
        // 2
        s.unversionedTypes[t] = gvk
        // 3
        ...
        s.unversionedKinds[gvk.Kind] = t
        // 4
    }
}
```

### 资源注册表注册方法

在 Scheme 资源注册表中，不同的资源类型使用的注册方法不同，分别如下：

* scheme.AddUnversionedTypes: 注册 UnversionedType 资源类型
* scheme.AddKnownTypes: 注册 KnownType 资源类型
* scheme.AddKnownTypesWithName: 注册 KnownType 资源类型，须指定资源的 Kind 资源种类名称

举例 scheme.AddKnownTypes 如下:

```
func (s *Scheme) AddKnownTypes(gv schema.GroupVersion,types ...object){
    s.addObservedVersion(gv)
    for _, obj := range types {
        t := reflect.Typeof(obj)
        // 通过 reflect 机制获取资源类型的名称作为资源种类名称
        if t.Kind() != reflect.Ptr {
            panic("All types must be pointers to structs.")
        }
        t = t.Elem()
        s.AddKnownTypeWithName(gv.WithKind(t.Name()), obj)
        // 调用这种注册方法
    }
}
```

## Codec 编解码器

