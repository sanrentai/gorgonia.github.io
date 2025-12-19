---
title: "简单卷积神经网络 (MNIST)"
date: 2019-10-29T20:09:05+01:00
draft: false
weight: -98
---

## 简介

这是一个逐步教程，用于在MNIST数据集上构建和训练卷积神经网络。

完整代码可以在Gorgonia主仓库的`examples`目录中找到。本教程的目的是详细解释代码。有关其工作原理的进一步解释，可以在书籍《Go Machine Learning Projects》中找到。

### 数据集

{{% notice info %}}
这部分是关于加载和打印数据集。如果您想直接进入神经网络部分，请随时跳过它并前往[卷积神经网络部分](#卷积神经网络)。
{{% /notice %}}

训练集和测试集可以从[Yann LeCun的MNIST网站](http://yann.lecun.com/exdb/mnist/)下载：

* [train-images-idx3-ubyte.gz](http://yann.lecun.com/exdb/mnist/train-images-idx3-ubyte.gz)：训练集图像 (9912422 字节)
* [train-labels-idx1-ubyte.gz](http://yann.lecun.com/exdb/mnist/train-labels-idx1-ubyte.gz)：训练集标签 (28881 字节)
* [t10k-images-idx3-ubyte.gz](http://yann.lecun.com/exdb/mnist/t10k-images-idx3-ubyte.gz)：测试集图像 (1648877 字节)
* [t10k-labels-idx1-ubyte.gz](http://yann.lecun.com/exdb/mnist/t10k-labels-idx1-ubyte.gz)：测试集标签 (4542 字节)

如网站所述，这些文件包含多个以二进制编码的图像或标签。每个图像/标签都以魔术数字开头。Go标准库的`encoding/binary`包使读取这些文件变得容易。

##### `mnist`包

为了方便起见，Gorgonia在`examples`子目录中创建了一个`mnist`包。它的目标是从数据中提取信息并创建`tensor.Tensor`。

[`readImageFile`](https://github.com/gorgonia/gorgonia/blob/e6bc7dd8951410b733bb85091d0e4506c25e6f70/examples/mnist/io.go#L50:6)函数创建一个表示读取器中包含的所有图像的字节数组。

类似的[`readLabelFile`](https://github.com/gorgonia/gorgonia/blob/e6bc7dd8951410b733bb85091d0e4506c25e6f70/examples/mnist/io.go#L21)函数提取标签。

```go
// Image保存图像的像素强度。
// 255是前景（黑色），0是背景（白色）。
type RawImage []byte

// Label是0到9之间的数字标签
type Label uint8

func readImageFile(r io.Reader, e error) (imgs []RawImage, err error)

func readLabelFile(r io.Reader, e error) (labels []Label, err error)
```

然后两个函数负责将`RawImage`和`Label`转换为`tensor.Tensor`：

* [prepareX](https://github.com/gorgonia/gorgonia/blob/e6bc7dd8951410b733bb85091d0e4506c25e6f70/examples/mnist/mnist.go#L70)
* [prepareY](https://github.com/gorgonia/gorgonia/blob/e6bc7dd8951410b733bb85091d0e4506c25e6f70/examples/mnist/mnist.go#L99)

```go
func prepareX(M []RawImage, dt tensor.Dtype) (retVal tensor.Tensor)

func prepareY(N []Label, dt tensor.Dtype) (retVal tensor.Tensor)
```

该包唯一导出的函数是[`Load`](https://github.com/gorgonia/gorgonia/blob/e6bc7dd8951410b733bb85091d0e4506c25e6f70/examples/mnist/mnist.go#L24)，它从`loc`读取`typ`文件并返回特定类型（float32或float64）的张量：

```go
// Load将mnist数据加载到两个张量中
//
// typ可以是"train"或"test"
//
// loc表示mnist文件所在的位置
func Load(typ, loc string, as tensor.Dtype) (inputs, targets tensor.Tensor, err error)
```

##### 测试包

现在，让我们创建一个简单的主文件来验证数据是否加载成功。

预期的测试目录布局如下：

```shell
$ ls -alhg *
-rw-r--r--  1 staff   375B Nov 11 13:48 main.go

testdata:
total 107344
drwxr-xr-x  6 staff   192B Nov 11 13:48 .
drwxr-xr-x  4 staff   128B Nov 11 13:48 ..
-rw-r--r--  1 staff   7.5M Jul 21  2000 t10k-images.idx3-ubyte
-rw-r--r--  1 staff   9.8K Jul 21  2000 t10k-labels.idx1-ubyte
-rw-r--r--  1 staff    45M Jul 21  2000 train-images.idx3-ubyte
-rw-r--r--  1 staff    59K Jul 21  2000 train-labels.idx1-ubyte
```

现在让我们编写这个简单的Go文件，它将读取测试和训练数据并显示结果张量：

```go
package main

import (
        "fmt"
        "log"

        "gorgonia.org/gorgonia/examples/mnist"
        "gorgonia.org/tensor"
)

func main() {
        for _, typ := range []string{"test", "train"} {
                inputs, targets, err := mnist.Load(typ, "./testdata", tensor.Float64)
                if err != nil {
                        log.Fatal(err)
                }
                fmt.Println(typ+" inputs:", inputs.Shape())
                fmt.Println(typ+" data:", targets.Shape())
        }
}
```

运行文件：

```shell
$ go run main.go
test inputs: (10000, 784)
test data: (10000, 10)
train inputs: (60000, 784)
train data: (60000, 10)
```

我们有60000张$28\times28=784$像素的图片，60000个相应的"one-hot"编码标签，以及测试集中的10000个测试文件。

#### 图像表示

让我们绘制第一个元素的图片：

```go
import (
        //...
        "image"
        "image/png"

        "gorgonia.org/gorgonia/examples/mnist"
        "gorgonia.org/tensor"
        "gorgonia.org/tensor/native"
)

func main() {
        inputs, targets, err := mnist.Load("train", "./testdata", tensor.Float64)
        if err != nil {
                log.Fatal(err)
        }
        cols := inputs.Shape()[1]
        imageBackend := make([]uint8, cols)
        for i := 0; i < cols; i++ {
                v, _ := inputs.At(0, i)
                imageBackend[i] = uint8((v.(float64) - 0.1) * 0.9 * 255)
        }
        img := &image.Gray{
                Pix:    imageBackend,
                Stride: 28,
                Rect:   image.Rect(0, 0, 28, 28),
        }
        w, _ := os.Create("output.png")
        vals, _ := native.MatrixF64(targets.(*tensor.Dense))
        fmt.Println(vals[0])
        err = png.Encode(w, img)
}
```
{{% notice info %}}
我们使用`native`包来轻松访问底层的`[]float64`后端。此操作不会生成新数据。
{{% /notice %}}

这会生成这个png文件：
![5](/images/mnist_5.png?width=10pc)

以及相应的标签向量，表明它是一个`5`：
```shell
$ go run main.go
[0.1 0.1 0.1 0.1 0.1 0.9 0.1 0.1 0.1 0.1]
```

## 卷积神经网络

我们正在构建一个5层卷积网络。$x_0$是输入图像，如前所述。

前三层$i$定义如下：

$ x\_{i+1} = Dropout(Maxpool(ReLU(Convolution(x_i,W_i)))) $

其中$i$的范围是0-2

第四层基本上是一个dropout层，用于随机将一些激活值归零：

$ x\_{4} = Dropout(ReLU(x_3\cdot W_3)) $

最后一层应用简单的乘法和softmax，以获得输出向量（此向量表示预测的标签）：

$ y = softmax(x_4\cdot W_4)$

### 网络变量

可学习参数是$W_0,W_1,W_2,W_3,W_4$。网络的其他变量是dropout概率$d_0,d_1,d_2,d_3$。

让我们创建一个结构来保存模型的变量和输出节点：

```go
type convnet struct {
	g                  *gorgonia.ExprGraph
	w0, w1, w2, w3, w4 *gorgonia.Node // 权重。后面的数字表示它用于哪一层
	d0, d1, d2, d3     float64        // dropout概率

	out *gorgonia.Node
}
```

#### 可学习参数的定义

卷积使用标准的$3\times3$内核和32个过滤器。由于数据集的图像是黑白的，我们只使用一个通道。这导致了权重的以下定义：

* $W_0 \in \mathbb{R}^{32\times 1\times3\times3}$ 用于第一个卷积运算符
* $W_1 \in \mathbb{R}^{64\times 32\times3\times3}$ 用于第二个卷积运算符
* $W_2 \in \mathbb{R}^{128\times 64\times3\times3}$ 用于第三个卷积运算符
* $W_3 \in \mathbb{R}^{128*3*3\times 625}$ 我们正在准备最终的矩阵乘法，因此需要将4D输入重塑为矩阵 (128x3x3)。625是一个任意数字。
* $W_4 \in \mathbb{R}^{625\times 10}$ 将输出大小减少到一个包含10个条目的向量

{{% notice note %}}
在神经网络优化中，众所周知，如果中间层的大小小于输出和输入，您就是在"挤压"无用的信息。
输入是784；下一层应该更小。625是一个好看的数字。
{{% /notice %}}

dropout概率固定为惯用值：

* $d_0=0.2$
* $d_1=0.2$
* $d_2=0.2$
* $d_3=0.55$

现在我们可以创建结构，为可学习参数占位：

```go
// 注意：为了清晰起见，gorgonia在本例中缩写为G
func newConvNet(g *G.ExprGraph) *convnet {
	w0 := G.NewTensor(g, dt, 4, G.WithShape(32, 1, 3, 3), G.WithName("w0"), G.WithInit(G.GlorotN(1.0)))
	w1 := G.NewTensor(g, dt, 4, G.WithShape(64, 32, 3, 3), G.WithName("w1"), G.WithInit(G.GlorotN(1.0)))
	w2 := G.NewTensor(g, dt, 4, G.WithShape(128, 64, 3, 3), G.WithName("w2"), G.WithInit(G.GlorotN(1.0)))
	w3 := G.NewMatrix(g, dt, G.WithShape(128*3*3, 625), G.WithName("w3"), G.WithInit(G.GlorotN(1.0)))
	w4 := G.NewMatrix(g, dt, G.WithShape(625, 10), G.WithName("w4"), G.WithInit(G.GlorotN(1.0)))
	return &convnet{
		g:  g,
		w0: w0,
		w1: w1,
		w2: w2,
		w3: w3,
		w4: w4,

		d0: 0.2,
		d1: 0.2,
		d2: 0.2,
		d3: 0.55,
	}
}
```

{{% notice info %}}
可学习参数使用Glorot等人的算法采样的一些值进行初始化。有关更多信息：Arxiv上的[All you need is a good init](https://arxiv.org/pdf/1511.06422.pdf)。
{{% /notice %}}

### 网络的定义

现在可以通过向convnet结构添加一个方法来定义网络：

_注意_：为了清晰起见，再次省略了错误检查

```go
// 出于教育原因，此函数特别冗长。实际上，您会将层包装在层结构类型中并执行每层激活
func (m *convnet) fwd(x *gorgonia.Node) (err error) {
	var c0, c1, c2, fc *gorgonia.Node
	var a0, a1, a2, a3 *gorgonia.Node
	var p0, p1, p2 *gorgonia.Node
	var l0, l1, l2, l3 *gorgonia.Node

	// 第0层
	// 这里我们使用步长 = (1, 1) 和填充 = (1, 1) 进行卷积，
	// 这是您的标准卷积神经网络卷积
	c0, _ = gorgonia.Conv2d(x, m.w0, tensor.Shape{3, 3}, []int{1, 1}, []int{1, 1}, []int{1, 1})
	a0, _ = gorgonia.Rectify(c0)
	p0, _ = gorgonia.MaxPool2D(a0, tensor.Shape{2, 2}, []int{0, 0}, []int{2, 2})
	l0, _ = gorgonia.Dropout(p0, m.d0)

	// 第1层
	c1, _ = gorgonia.Conv2d(l0, m.w1, tensor.Shape{3, 3}, []int{1, 1}, []int{1, 1}, []int{1, 1})
	a1, _ = gorgonia.Rectify(c1)
	p1, _ = gorgonia.MaxPool2D(a1, tensor.Shape{2, 2}, []int{0, 0}, []int{2, 2})
	l1, _ = gorgonia.Dropout(p1, m.d1)

	// 第2层
	c2, _ = gorgonia.Conv2d(l1, m.w2, tensor.Shape{3, 3}, []int{1, 1}, []int{1, 1}, []int{1, 1})
	a2, _ = gorgonia.Rectify(c2)
	p2, _ = gorgonia.MaxPool2D(a2, tensor.Shape{2, 2}, []int{0, 0}, []int{2, 2})

	var r2 *gorgonia.Node
	b, c, h, w := p2.Shape()[0], p2.Shape()[1], p2.Shape()[2], p2.Shape()[3]
	r2, _ = gorgonia.Reshape(p2, tensor.Shape{b, c * h * w})
	l2, _ = gorgonia.Dropout(r2, m.d2)

	// 第3层
	fc, _ = gorgonia.Mul(l2, m.w3)
	a3, _ = gorgonia.Rectify(fc)
	l3, _ = gorgonia.Dropout(a3, m.d3)

	// 输出解码
	var out *gorgonia.Node
	out, _ = gorgonia.Mul(l3, m.w4)
	m.out, _ = gorgonia.SoftMax(out)
	return
}
```

### 训练神经网络

我们从训练集获得的输入是一个矩阵$numExample \times 784$。卷积运算符需要一个4D张量BCHW。我们需要做的第一件事是重塑输入：

```go
numExamples := inputs.Shape()[0]
inputs.Reshape(numExamples, 1, 28, 28)
```

我们将按批次训练网络。批次大小是一个变量(`bs`)。我们创建两个新张量，它们将保存当前批次的值和标签。然后我们实例化神经网络：

```go
g := gorgonia.NewGraph()
x := gorgonia.NewTensor(g, dt, 4, gorgonia.WithShape(bs, 1, 28, 28), gorgonia.WithName("x"))
y := gorgonia.NewMatrix(g, dt, gorgonia.WithShape(bs, 10), gorgonia.WithName("y"))
m := newConvNet(g)
m.fwd(x)
```
#### 成本函数

我们定义了一个我们想要最小化的成本函数，它基于简单的交叉熵，通过逐元素乘以预期输出来计算，然后对其求平均：

$cost = -\dfrac{1}{bs} \sum_{i=1}^{bs}(pred^{(i)}\cdot y^{(i)})$

```go
losses := gorgonia.Must(gorgonia.HadamardProd(m.out, y))
cost := gorgonia.Must(gorgonia.Mean(losses))
cost = gorgonia.Must(gorgonia.Neg(cost))
```

并为以后保留成本值的指针：

```go
var costVal gorgonia.Value
gorgonia.Read(cost, &costVal)
```

然后我们将执行符号反向传播：

```go
gorgonia.Grad(cost, m.learnables()...)
```

Learnables定义如下：
```go
func (m *convnet) learnables() gorgonia.Nodes {
    return gorgonia.Nodes{m.w0, m.w1, m.w2, m.w3, m.w4}
}
```

##### 训练循环

首先，我们需要一个[vm](/reference/vm)来运行图，以及一个求解器来在每一步调整可学习参数。我们还需要绑定可学习参数的对偶值，以实际存储梯度值，以便求解器工作。

```go
vm := gorgonia.NewTapeMachine(g, gorgonia.BindDualValues(m.learnables()...))
solver := gorgonia.NewRMSPropSolver(gorgonia.WithBatchSize(float64(bs)))
defer vm.Close()
```

我们根据批次大小定义一个时期包含的批次数：

```go
batches := numExamples / bs
```

然后创建训练循环：

```go
for i := 0; i < *epochs; i++ {
    for b := 0; b < batches; b++ {
        // ...
    }
}
```

##### 循环内部：
现在我们需要从输入张量（即$60000 \times 784$）中为每个批次提取值。
每个输入是($bs\times 784$)。第一批将包含从0到bs-1的值，第二批包含bs到2*bs-1的值，依此类推。然后张量被重塑为4D张量：

```go
var xVal, yVal tensor.Tensor
xVal, _ = inputs.Slice(sli{start, end})

yVal, _ = targets.Slice(sli{start, end})

xVal.(*tensor.Dense).Reshape(bs, 1, 28, 28)
```

然后我们将值分配给图：

```go
gorgonia.Let(x, xVal)
gorgonia.Let(y, yVal)
```
并运行VM和求解器来调整权重

```go
vm.RunAll()
solver.Step(gorgonia.NodesToValueGrads(m.learnables()))
vm.Reset()
```

就是这样，您现在有了一个可以学习的神经网络。

### 结论

运行代码相对较慢，因为涉及大量数据，但它确实会学习。
您可以在[Gorgonia的示例目录](https://github.com/gorgonia/gorgonia/blob/e6bc7dd8951410b733bb85091d0e4506c25e6f70/examples/convnet/main.go#L1)中获取完整代码。

要保存权重，用户可以创建`load`和`save`两个方法，如[iris教程](/tutorials/iris)中所述。
然后，读者可以练习编写一个小工具来使用这个神经网络。

玩得开心！