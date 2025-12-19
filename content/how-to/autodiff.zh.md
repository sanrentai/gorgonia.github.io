---
title: "如何计算梯度（微分）"
date: 2019-10-29T20:07:07+01:00
draft: false
---

## 目标
考虑这个简单的方程：

$$ f(x,y,z) = ( x + y ) \times z $$

本文的目标是向您展示 Gorgonia 如何计算梯度 $\nabla f$ 及其偏导数：

$$ \nabla f = [\frac{\partial f}{\partial x}, \frac{\partial f}{\partial y}, \frac{\partial f}{\partial z}] $$

### 解释

使用 [链式法则](https://www.khanacademy.org/math/ap-calculus-ab/ab-differentiation-2-new/ab-3-1a/a/chain-rule-review)，我们可以在每一步计算梯度值，如下图所示：

{{<mermaid align="left">}}
graph LR;
    x -->|$x=-2$<br>$\partial f/\partial x = -4$| add
    y -->|$y=5$<br>$\partial f/\partial y = -4$| add
    add(+) -->|$q=3$<br>$\partial f/\partial q = -4$| mul
    z -->|$z=-4$<br>$\partial f/\partial z = 3$| mul
    mul(*) -->|$f=-12$<br>$1$| f
{{< /mermaid >}}


{{% notice info %}}
如需了解更多关于梯度计算的信息，请阅读斯坦福大学 [cs231n](http://cs231n.github.io/optimization-2/) 的这篇文章。
{{% /notice %}}

我们将把这个方程表示为一个 [exprgraph](/reference/exprgraph)，并展示如何让 Gorgonia 计算梯度。

当计算完成后，每个节点将包含一个 [对偶值](/reference/dualvalue)，其中包含实际值和相对于 x 的导数。

例如，考虑节点 x：

```go
var x *gorgonia.Node
```

一旦 Gorgonia 评估了 exprgraph，就可以通过调用以下方法提取 `x` 的值和梯度 $\frac{\partial f}{\partial x}$ 的值：

```go
xValue := x.Value()    // -2
dfdx, _ := x.Grad()    // -4，请在实际代码中检查错误
```

让我们看看如何做到这一点。

## 创建方程

首先，让我们创建表示方程的 [exprgraph](/reference/exprgraph)。

{{% notice info %}}
如果您想了解更多关于这部分的信息，请阅读 [hello world](/tutorials/hello-world/) 教程。
{{% /notice %}}

```go
g := gorgonia.NewGraph()

var x, y, z *gorgonia.Node
var err error

// 定义表达式
x = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("x"))
y = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("y"))
z = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("z"))
q, err := gorgonia.Add(x, y)
if err != nil {
    log.Fatal(err)
}
result, err := gorgonia.Mul(z, q)
if err != nil {
    log.Fatal(err)
}
```

然后设置一些值：

```go
gorgonia.Let(x, -2.0)
gorgonia.Let(y, 5.0)
gorgonia.Let(z, -4.0)
```

### 获取梯度

有两种获取梯度的方法：

* 使用 [LispMachine](/reference/lispmachine) 的 [自动微分](https://en.wikipedia.org/wiki/Automatic_differentiation) 功能；
* 使用 Gorgonia 提供的 [符号微分](https://en.wikipedia.org/wiki/Computer_algebra) 功能；


#### 自动微分

自动微分只能通过 [LispMachine](/reference/lispmachine) 实现。
默认情况下，LispMachine 执行前向模式和后向模式执行。

因此，调用 RunAll 方法就足够获得结果。
```go
m := gorgonia.NewLispMachine(g)
defer m.Close()
if err = m.RunAll(); err != nil {
    log.Fatal(err)
}
```

现在可以提取值和梯度：

```go
fmt.Printf("x=%v;y=%v;z=%v\n", x.Value(), y.Value(), z.Value())
fmt.Printf("f(x,y,z) = %v\n", result.Value())

if xgrad, err := x.Grad(); err == nil {
    fmt.Printf("df/dx: %v\n", xgrad)
}
if ygrad, err := y.Grad(); err == nil {
    fmt.Printf("df/dy: %v\n", ygrad)
}
if xgrad, err := z.Grad(); err == nil {
    fmt.Printf("df/dx: %v\n", xgrad)
}
```


#### 符号微分

另一种选择是使用符号微分。
符号微分通过向图中添加新节点来工作。这些新节点表示相对于作为参数传递的节点的梯度。

要创建这些新节点，我们使用 [Grad()](https://godoc.org/gorgonia.org/gorgonia#Grad) 函数。

Grad 接受一个标量成本节点和一个关于哪些节点的列表，并返回梯度。

考虑以下代码：
```go
var grads Nodes
if grads, err = Grad(result,z, x, y); err != nil {
    log.Fatal(err)
}
```

这表示计算相对于 `z`、`x` 和 `y` 的偏导数（梯度）。



`grads` 是一个 `[]*gorgonia.Node` 数组，顺序与传递的 WRT 相同：

* `grads[0]` = $\frac{\partial f}{\partial z}$
* `grads[1]` = $\frac{\partial f}{\partial x}$
* `grads[2]` = $\frac{\partial f}{\partial y}$

梯度与 [TapeMachine](/reference/tapemachine) 和 [LispMachine](/reference/lispmachine) 兼容，但 TapeMachine 要快得多。

```go
machine := gorgonia.NewTapeMachine(g)
defer machine.Close()
if err = machine.RunAll(); err != nil {
        log.Fatal(err)
}

fmt.Printf("result: %v\n", result.Value())
if zgrad, err := z.Grad(); err == nil {
        fmt.Printf("dz/dx: %v | %v\n", zgrad, grads[0].Value())
}

if xgrad, err := x.Grad(); err == nil {
        fmt.Printf("dz/dx: %v | %v\n", xgrad, grads[1].Value())
}

if ygrad, err := y.Grad(); err == nil {
        fmt.Printf("dz/dy: %v | %v\n", ygrad, grads[2].Value())
}
```

注意，您可以通过两种方式访问偏导数：

1. 使用 `.Grad()` 方法。例如，对于运行示例中 `x` 的梯度，使用 `x.Grad()`
2. 使用梯度节点的 `.Value()` 方法。例如，对于运行示例中 `x` 的梯度，使用 `grads[1].Value()`。

这两种不同方法的原因归结为适用性。当从梯度节点获取值更有意义时（例如，您可能想要计算二阶导数），则使用梯度节点。但如果您想快速获取梯度值，`.Grad()` 方法可能最合适。最终取决于您的偏好。

## 完整代码（自动微分）

```go
func main() {
    g := gorgonia.NewGraph()

    var x, y, z *gorgonia.Node
    var err error

    // 定义表达式
    x = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("x"))
    y = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("y"))
    z = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("z"))
    q, err := gorgonia.Add(x, y)
    if err != nil {
        log.Fatal(err)
    }
    result, err := gorgonia.Mul(z, q)
    if err != nil {
        log.Fatal(err)
    }

    // 设置初始值然后运行
    gorgonia.Let(x, -2.0)
    gorgonia.Let(y, 5.0)
    gorgonia.Let(z, -4.0)

    // 默认情况下，lispmachine 执行前向模式和后向模式执行
    m := gorgonia.NewLispMachine(g)
    defer m.Close()
    if err = m.RunAll(); err != nil {
        log.Fatal(err)
    }

    fmt.Printf("x=%v;y=%v;z=%v\n", x.Value(), y.Value(), z.Value())
    fmt.Printf("f(x,y,z)=(x+y)*z\n")
    fmt.Printf("f(x,y,z) = %v\n", result.Value())

    if xgrad, err := x.Grad(); err == nil {
        fmt.Printf("df/dx: %v\n", xgrad)
    }

    if ygrad, err := y.Grad(); err == nil {
        fmt.Printf("df/dy: %v\n", ygrad)
    }
    if xgrad, err := z.Grad(); err == nil {
        fmt.Printf("df/dz: %v\n", xgrad)
    }
}
```

输出：

```text
$ go run main.go
x=-2;y=5;z=-4
f(x,y,z)=(x+y)*z
f(x,y,z) = -12
df/dx: -4
df/dy: -4
df/dz: 3
```

## 完整代码（符号微分）


```go
func main() {
    g := gorgonia.NewGraph()

    var x, y, z *gorgonia.Node
    var err error
    var grads Nodes

    // 定义表达式
    x = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("x"))
    y = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("y"))
    z = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("z"))
    q, err := gorgonia.Add(x, y)
    if err != nil {
        log.Fatal(err)
    }
    result, err := gorgonia.Mul(z, q)
    if err != nil {
        log.Fatal(err)
    }

    if grads, err = Grad(result,z, x, y); err != nil {
        log.Fatal(err)
    }

    // 设置初始值然后运行
    gorgonia.Let(x, -2.0)
    gorgonia.Let(y, 5.0)
    gorgonia.Let(z, -4.0)

    machine := gorgonia.NewTapeMachine(g)
    defer machine.Close()
    if err = machine.RunAll(); err != nil {
        log.Fatal(err)
    }

    fmt.Printf("x=%v;y=%v;z=%v\n", x.Value(), y.Value(), z.Value())
    fmt.Printf("f(x,y,z)=(x+y)*z\n")
    fmt.Printf("f(x,y,z) = %v\n", result.Value())

    if zgrad, err := z.Grad(); err == nil {
        fmt.Printf("dz/dx: %v | %v\n", zgrad, grads[0].Value())
    }

    if xgrad, err := x.Grad(); err == nil {
        fmt.Printf("dz/dx: %v | %v\n", xgrad, grads[1].Value())
    }

    if ygrad, err := y.Grad(); err == nil {
        fmt.Printf("dz/dy: %v | %v\n", ygrad, grads[2].Value())
    }
}
```

输出：

```text
$ go run main.go
x=-2;y=5;z=-4
f(x,y,z)=(x+y)*z
f(x,y,z) = -12
df/dx: -4 | -4
df/dy: -4 | -4
df/dz: 3 | 3
```