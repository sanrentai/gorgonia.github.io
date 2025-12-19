---
title: "计算图"
date: 2019-11-10T21:09:19+01:00
description: "图和*节点"
weight: -100
draft: false
---

## Gorgonia是基于图的

_注意_：本文的灵感来自[这篇博客文章](http://gopherdata.io/post/deeplearning_in_go_part_1/)

像大多数深度学习库（如Tensorflow或Theano）一样，Gorgonia依赖于方程可以用图表示的概念。

它将方程图公开为[ExprGraph](/reference/exprgraph)对象，程序员可以操作该对象。

因此，不是编写：

```go
func main() {
	fmt.Printf("%v", 1+1)
}
```

程序员应该编写：

```go
func main() {
	// 创建一个图
	g := gorgonia.NewGraph()

	// 创建一个名为"x"的节点，值为1
	x := gorgonia.NodeFromAny(g, 1, gorgonia.WithName("x"))

	// 创建一个名为"y"的节点，值为1
	y := gorgonia.NodeFromAny(g, 1, gorgonia.WithName("y"))

	// z := x + y
	z := gorgonia.Must(gorgonia.Add(x, y))

	// 创建一个VM来执行图
	vm := gorgonia.NewTapeMachine(g)

	// 运行VM。不检查错误
	vm.RunAll()

	// 打印z的值
	fmt.Printf("%v", z.Value())
}
```

#### 数值稳定性

考虑方程$y = log(1+x)$。
这个方程在数值上不稳定 - 对于非常小的$x$值，答案很可能是错误的。
这是因为float64的设计方式 - float64没有足够的位来区分1和1 + 10e-16。
事实上，在Go中正确的做法是使用内置库函数math.Log1p。
这可以通过这个简单的程序来演示：

```go
func main() {
	fmt.Printf("%v\n", math.Log(1.0+10e-16))
	fmt.Printf("%v\n", math.Log1p(10e-16))
}
```

```text
1.110223024625156e-15 // 错误
9.999999999999995e-16 // 正确
```

Gorgonia通过使用最佳实现来处理这个问题，以确保数值稳定性。

### ExpGraph和*Node

ExprGraph是保存方程的对象。这个图的顶点是组成我们想要实现的方程的值或运算符。
这些顶点由一个称为"Node"的结构表示。图持有指向这个结构的指针。

要创建方程，我们需要创建一个ExprGraph，向其中添加一些Node，并将它们链接在一起。

幸运的是，我们不需要手动管理节点之间的连接。

#### 占位符和运算符

Node可以保存一些值（[Value](/reference)是一个Go接口，代表标量或张量等具体类型）。
但它也可以保存[运算符](/reference/operator)。

在计算时，值将沿着图流动，每个包含运算符的节点将执行相应的代码，并将值设置到相应的节点。

### 梯度计算

除此之外，Gorgonia可以进行符号微分和自动微分。
这个[页面](/about/differentiation)详细解释了它的工作原理。