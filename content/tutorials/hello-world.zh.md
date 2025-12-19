---
title: "Hello World"
date: 2019-10-29T17:54:31+01:00
draft: false
weight: -100
---

这是一个使用Gorgonia进行非常简单计算的分步教程。

我们的目标是使用Gorgonia的所有机制来执行一个简单的操作：

$ f(x,y) = x + y $

其中 `x = 2` 和 `y = 5`

## 工作原理

方程 `x + y = z` 可以表示为一个图：

{{<mermaid align="left">}}
graph LR;
    z[z] --> add(Round edge)
    add[+] --> x
    add[+] --> y
{{< /mermaid >}}

要计算结果，我们使用4个步骤：

* 使用Gorgonia创建一个类似的[图](/reference/exprgraph)
* 在[节点](/reference/node) `x` 和 `y` 上设置一些[值](/reference/value)
* 在[gorgonia虚拟机](/reference/vm)上实例化一个图
* 从节点 `z` 提取[值](/reference/value)
    *

### 创建一个图

使用以下方法创建一个空的[表达式图](/reference/exprgraph)：

```go
g := gorgonia.NewGraph()
```

### 创建节点

我们将创建一些[节点](/reference/node)并将它们与ExprGraph关联起来。

```go
var x, y, z *gorgonia.Node
```

#### 创建占位符

`x` 和 `y` 是标量变量，我们可以使用以下代码创建相应的节点：

```go
x = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("x"))
y = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("y"))
```

{{% notice note %}}
这些函数将exprgraph作为参数；生成的节点会自动与图关联。
{{% /notice %}}

现在创建加法运算符；这个运算符接受两个[节点](/reference/node)并返回一个新节点z：

```go
if z, err = gorgonia.Add(x, y); err != nil {
        log.Fatal(err)
}
```

{{% notice info %}}
返回的节点 `z` 会被添加到图中，即使没有将 `g` 传递给 `z` 或 `Add` 函数。
{{% /notice %}}

### 设置值

我们有一个表示方程 `z = x + y` 的ExprGraph。现在是时候为 `x` 和 `y` 分配一些值了。

我们使用 [`Let`](https://godoc.org/gorgonia.org/gorgonia#Let) 函数：

```go
gorgonia.Let(x, 2.0)
gorgonia.Let(y, 2.5)
```

### 运行图

要运行图并计算结果，我们需要实例化一个[虚拟机](/reference/vm)。
让我们使用[TapeMachine](/reference/vm/tapemachine)：

```go
machine := gorgonia.NewTapeMachine(g)
defer machine.Close()
```

然后运行图：

```go
if err = machine.RunAll(); err != nil {
        log.Fatal(err)
}
```

{{% notice warning %}}
如果需要第二次运行，必须调用 `vm` 对象的 `Reset()` 方法：
` machine.Reset() `
{{% /notice %}}

### 获取结果

现在节点 `z` 包含结果。
我们可以通过调用 `Value()` 方法来提取其[值](/reference/value)：

```go
fmt.Printf("%v", z.Value())
```

{{% notice note %}}
我们也可以通过调用 `z.Value().Data()` 访问底层的"Go"值，在我们的例子中，这将返回一个包含 `float64` 的 `interface{}`
{{% /notice %}}

# 最终结果

```go
package main

import (
        "fmt"
        "log"

        "gorgonia.org/gorgonia"
)

func main() {
        g := gorgonia.NewGraph()

        var x, y, z *gorgonia.Node
        var err error

        // 定义表达式
        x = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("x"))
        y = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("y"))
        if z, err = gorgonia.Add(x, y); err != nil {
                log.Fatal(err)
        }

        // 创建一个虚拟机来运行程序
        machine := gorgonia.NewTapeMachine(g)
        defer machine.Close()

        // 设置初始值然后运行
        gorgonia.Let(x, 2.0)
        gorgonia.Let(y, 2.5)
        if err = machine.RunAll(); err != nil {
                log.Fatal(err)
        }

        fmt.Printf("%v", z.Value())
}
```

```shell
$ go run main.go
4.5
```