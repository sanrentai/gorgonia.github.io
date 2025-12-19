---
title: "Go Machine"
description: "本页面描述Go Machine内部的工作原理"
date: 2019-10-29T19:50:15+01:00
draft: false
---

本页面解释了GoMachine内部的工作原理。

GoMachine是一个在[`xvm`包](https://github.com/gorgonia/gorgonia/tree/master/x/vm)中的实验性特性。该包的API及其名称可能会更改。

本文档基于[commit 7538ab3](https://github.com/gorgonia/gorgonia/tree/7538ab3b58ceae68f162c17d19052324bf1dc587)

## 节点的状态

其原理依赖于节点的状态。

如[_Lexical Scanning in Go_](https://www.youtube.com/watch?v=HxaD_trXwRE)中所述：

- 状态表示我们所在的位置
- 动作表示我们做什么
- 动作导致新的状态


目前，GoMachine期望节点处于这些可能的状态：

- _等待输入_
- _发射输出_

如果节点携带操作符，可能有一个额外的状态：

- _计算中_

{{% notice info%}}
后来，在实现自动微分时，最终将添加一个新状态：_计算梯度_
{{%/notice%}}

这导致了节点可能状态的状态图：

{{<mermaid align="left">}}
graph TB;
    A(初始阶段) --> BB{输入是操作符}
    BB -->|否| D[发射输出]
    BB -->|是| B[等待输入]
    B --> C{输入数量 == 元数}
    C -->|否| B
    C -->|是| Computing
    Computing --> E{有错误}
    E -->|否| D
    E -->|是| F
    D --> F(结束)
{{< /mermaid >}}

### 实现

`node`是一个私有结构：

```go
type node struct {
    // ...
}
```

我们定义一个类型`stateFn`，它表示在特定`context`中对`*node`执行的操作，并返回一个新状态。此类型是一个`func`：

```go
type stateFn func(context.Context, *node) stateFn
```

_注意_：每个状态函数都有责任处理上下文取消机制。这意味着如果收到取消信号，节点应该返回结束状态。为简单起见：

```go
func mystate(ctx context.Context, *node) stateFn { 
    // ...
    select {
        // ...
        case <- ctx.Done():
            n.err = ctx.Error()
            return nil
    }
}
```

我们定义了四个类型为`stateFn`的函数来实现节点所需的操作：

```go
func defaultState(context.Context, *node) stateFn { ... }

func receiveInput(context.Context, *node) stateFn { ... }

func computeFwd(context.Context, *node) stateFn { ... }

func emitOutput(context.Context, *node) stateFn { ... }
```

_注意_：`end`状态是`nil`（`stateFn`的零值）

### 运行状态机

每个节点都是一个状态机。
要运行它，我们设置一个`run`方法，该方法接受一个上下文作为参数。

```go
func (n *node) Compute(ctx context.Context) error {
    for state := defaultState; state != nil; {
        state = state(ctx, n)
    }
    return n.err
}
```
_注意_：`*node`存储一个错误，该错误应由指示其中断状态机原因的stateFn设置（例如，如果在计算过程中发生错误，此错误包含原因）

然后，机器在其自己的goroutine中触发每个`*node`。

### 事件上的状态修改

我们使用反应式编程的范式来从一个状态切换到另一个状态。

`*node`结构中的更改会触发导致状态更改的操作。

例如，让我们看一个简单的计算器，它计算`a+b`。

- $+$正在等待两个输入值来进行求和$a$和$b$
- $a$正在等待一个值
- $b$正在等待一个值

当我们_发送_一个值给$a$

$+$会收到此事件的通知（$a$拥有一个值）；它会接收并将值存储在内部。

然后我们_发送_一个值给$b$，$+$会收到通知，并_接收_该值。然后其状态变为`compute`。

一旦计算完成，$+$会_发送_结果给任何有兴趣使用它的人。

在Go中，发送和接收值以及事件编程是通过通道实现的。

节点结构拥有两个通道，一个用于接收输入（`inputC`），一个用于发射输出（`outputC`）：

```go
type node struct {
    outputC        chan gorgonia.Value
    inputC         chan ioValue
    err            error
    // ...
}
```

_注意_：`ioValue`结构将在本文档后面解释；现在，请考虑`ioValue` = `gorgonia.Value`

## 通信中心

现在我们让所有节点在goroutine中运行；我们需要将它们连接在一起以实际计算公式。

例如，在：$ a\times x+b$中，我们需要将$a\times x$的结果发送到携带_加法_操作符的节点中。

大致如下：
```go
var aTimesX *node{op: mul}
var aTimesXPlusB *node{op: sum}

var a,b,c gorgonia.Value

aTimesX.inputC <- a
aTimesX.inputC <- x
aTimesXPlusB.inputC <- <- aTimesX.outputC 
aTimesXPlusB.inputC <- <- b
```

问题在于，通道不是"主题"，它本身不处理订阅。第一个消费者获取一个值，并清空通道。

因此，如果我们采用这个等式$(a + b) \times c + (a + b) \times d$，实现将无法工作：

{{< highlight go "linenos=table,hl_lines=9 12" >}}
var aPlusB *node{op: add}
var aPlusBTimesC *node{op: mul}
var aPlusBTimesCPlusAPlusB *node{op: add}

var a,b,c gorgonia.Value

aPlusB.inputC <- a
aPlusB.inputC <- b
aPlusBTimesC.inputC <- <- aPlusB.outputC
aPlusBTimesC.inputC <- c
aPlusBTimesCPlusAPlusB <- <- aPlusBTimesC.outputC
aPlusBTimesCPlusAPlusB <- <- aPlusB.outputC // 死锁
{{< / highlight >}}

这将导致死锁，因为`aPlusB.outputC`在第9行被清空，因此第12行将不再接收值。

解决方案是使用临时通道和广播机制，如文章[Go Concurrency Patterns: Pipelines and cancellation](https://blog.golang.org/pipelines#TOC_4.)中所述。

### 发布/订阅

我们设置两个结构：

```go
type publisher struct {
    id          int64
    publisher   <-chan gorgonia.Value
    subscribers []chan<- gorgonia.Value
}

type subscriber struct {
    id         int64
    publishers []<-chan gorgonia.Value
    subscriber chan<- ioValue
}
```

每个通过`outputC`提供输出的节点都是发布者，图中到达此节点的所有节点都是其订阅者**s**。这定义了一个`publisher`对象。对象的ID是提供输出的节点的ID。

每个通过其`inputC`期望输入的节点都是订阅者。发布者**s**是在`*ExprGraph`中到达此节点的节点


#### 合并和广播

发布者通过调用以下函数将其数据广播到订阅者：

```go
func broadcast(ctx context.Context, globalWG *sync.WaitGroup, ch <-chan gorgonia.Value, cs ...chan<- gorgonia.Value) { ... } 
```

订阅者通过调用以下函数合并来自发布者的结果：

```go
func merge(ctx context.Context, globalWG *sync.WaitGroup, out chan<- ioValue, cs ...<-chan gorgonia.Value) { ... }
```

_注意_：两个函数都处理上下文取消

### pubsub

为了连接所有发布者和订阅者，我们使用一个顶级结构：`pubsub`

```go
type pubsub struct {
    publishers  []*publisher
    subscribers []*subscriber
}
```

`pubsub`负责设置通道网络。

然后，`run(context.Context)`方法触发所有元素的`broadcast`和`merge`：

```go
func (p *pubsub) run(ctx context.Context) (context.CancelFunc, *sync.WaitGroup) { ... }
```

此方法返回一个`context.CancelFunc`和一个`sync.WaitGroup`，当取消后所有pubsub都处理完毕时，该WaitGroup将降至零。

#### 关于`ioValue`

订阅者有一个单一的输入通道；输入值可以按任意顺序发送。
订阅者的合并函数跟踪订阅者的顺序，将值包装到ioValue结构中，并添加发出值的操作符的位置：

```go
type ioValue struct {
    pos int
    v   gorgonia.Value
}
```


## 机器

`Machine`是该包唯一导出的结构。

它是节点和pubsub的支持结构。

```go
type Machine struct {
    nodes  []*node
    pubsub *pubsub
}
```

### 创建机器

通过调用以下函数从`*ExprGraph`创建机器：

```go
func NewMachine(g *gorgonia.ExprGraph) *Machine { ... }
```

在底层，它解析图并为每个`*gorgonia.Node`生成一个`*node`。
如果节点携带Op（=实现`Do(... Value) Value`方法的对象），则指向Op的指针将添加到结构中。

{{%notice info%}}
为了过渡，该包声明了一个`Doer`接口。
此接口由`*gorgonia.Node`结构实现。
{{%/notice%}}

处理两种个别情况：

- `*ExprGraph`的顶级节点的`outputC = nil`
- `*ExprGraph`的底部节点的`inputC = nil`

然后`NewMachine`调用`createNetwork`方法来创建`*pubsub`元素。

### 运行机器

调用Machine的`Run`方法会触发计算。
对该函数的调用是阻塞的。
如果以下情况，它会返回错误并停止进程：
- 所有节点都已达到其最终状态
- 一个节点的执行状态返回错误

在错误情况下，会自动向`*pubsub`基础设施发送取消信号，以避免泄漏。

### 关闭机器

计算后，调用`Close`以避免内存泄漏是安全的。
`Close()`关闭`*node`和`*pubsub`持有的所有通道

## 杂项

重要的是要注意，机器独立于`*ExprGraph`。因此，`*gorgonia.Node`持有的值不会更新。

要访问数据，必须调用机器的`GetResult`方法。此方法接受节点ID作为输入（`*node`和`*gorgonia.Node`具有相同的ID）

例如：

```go
var add, err := gorgonia.Add(a,b)
fmt.Println(machine.GetResult(add.ID()))
```

## 示例

这是一个计算两个float32的简单示例

```go
func main(){
    g := gorgonia.NewGraph()
    forty := gorgonia.F32(40.0)
    two := gorgonia.F32(2.0)
    n1 := gorgonia.NewScalar(g, gorgonia.Float32, gorgonia.WithValue(&forty), gorgonia.WithName("n1"))
    n2 := gorgonia.NewScalar(g, gorgonia.Float32, gorgonia.WithValue(&two), gorgonia.WithName("n2"))

    added, err := gorgonia.Add(n1, n2)
    if err != nil {
        log.Fatal(err)
    }
    machine := NewMachine(g)
    ctx, cancel := context.WithTimeout(context.Background(), 1000*time.Millisecond)
    defer cancel()
    defer machine.Close()
    err = machine.Run(ctx)
    if err != nil {
        log.Fatal(err)
    }
    fmt.Println(machine.GetResult(added.ID()))
}
```

输出

```shell
42
```