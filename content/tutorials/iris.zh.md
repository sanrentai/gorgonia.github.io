---
title: "鸢尾花数据集的多元线性回归"
date: 2019-10-31T14:53:37+01:00
draft: false
---

## 关于

我们将使用Gorgonia创建一个线性回归模型。

目标是根据鸢尾花的特征预测其物种：

* sepal_length（花萼长度）
* sepal_width（花萼宽度）
* petal_length（花瓣长度）
* petal_width（花瓣宽度）

我们要预测的物种有：

* setosa（山鸢尾）
* virginica（弗吉尼亚鸢尾）
* versicolor（变色鸢尾）

本教程的目标是使用Gorgonia根据鸢尾花数据集找到正确的$\Theta$值，以便编写一个命令行工具，如下所示：

```text
./iris
花萼长度: 5
花萼宽度: 3.5
花瓣长度: 1.4
花瓣宽度: 0.2

这可能是一朵山鸢尾（setosa）
```

{{% notice warning %}}
本教程仅用于学术目的。其目标是描述如何使用Gorgonia实现这一功能；
这不是解决这个特定问题的最先进方法。
{{% /notice %}}

### 数学表示

我们认为鸢尾花的物种是其花萼长度和宽度以及花瓣长度和宽度的函数。

因此，如果我们认为$y$是物种的值，那么我们要解决的方程是：

$$ y = \theta_0 + \theta_1 * sepal\_length + \theta_2 * sepal\_width + \theta_3 * petal\_length + \theta_4 * petal\_width$$

让我们考虑向量$x$和$\Theta$，如下所示：

$$ x =  \begin{bmatrix} sepal\_length & sepal\_width & petal\_length & petal\_width & 1\end{bmatrix}$$

$$
\Theta = \begin{bmatrix}
        \theta_4 \\
        \theta_3 \\
        \theta_2 \\
        \theta_1 \\
        \theta_0 \\
        \end{bmatrix}
$$

我们有

$$y = x\cdot\Theta$$

### 线性回归

为了找到正确的值，我们将使用线性回归。
我们将数据（从不同花朵观察到的真实事实）编码到一个包含5列的矩阵$X$中（花萼长度、花萼宽度、花瓣长度、花瓣宽度和用于偏置的1）。
矩阵的一行代表一朵花。

然后我们将相应的物种编码为一个带有浮点值的列向量$Y$。

* setosa = 1.0
* virginica = 2.0
* versicolor = 3.0

在学习阶段，成本表达式如下：

$cost = \dfrac{1}{m} \sum_{i=1}^m(X^{(i)}\cdot\Theta-Y^{(i)})^2$

我们将使用梯度下降来降低成本并获得$\Theta$的准确值

{{% notice info %}}
可以使用正规方程获得精确的$\theta$值
$$ \theta = \left( X^TX \right)^{-1}X^TY $$
有关使用Gonum的基本实现，请参见此[要点](https://gist.github.com/owulveryck/19a5ba9553ff8209b3b4227b5325041b#file-normal-go)。
{{% /notice %}}


## 使用gota（dataframe）生成训练集

首先，让我们生成训练数据。我们使用dataframe来平滑地完成这个任务。

{{% notice info %}}
有关使用dataframe的更多信息，请参阅此[指南](/how-to/dataframe/)
{{% /notice %}}


```go
func getXYMat() (*mat.Dense, *mat.Dense) {
        f, err := os.Open("iris.csv")
        if err != nil {
                log.Fatal(err)
        }
        defer f.Close()
        df := dataframe.ReadCSV(f)
        xDF := df.Drop("species")

        toValue := func(s series.Series) series.Series {
                records := s.Records()
                floats := make([]float64, len(records))
                for i, r := range records {
                        switch r {
                        case "setosa":
                                floats[i] = 1
                        case "virginica":
                                floats[i] = 2
                        case "versicolor":
                                floats[i] = 3
                        default:
                                log.Fatalf("未知的鸢尾花: %v\n", r)
                        }
                }
                return series.Floats(floats)
        }

        yDF := df.Select("species").Capply(toValue)
        numRows, _ := xDF.Dims()
        xDF = xDF.Mutate(series.New(one(numRows), series.Float, "bias"))
        fmt.Println(xDF.Describe())
        fmt.Println(yDF.Describe())

        return mat.DenseCopyOf(&matrix{xDF}), mat.DenseCopyOf(&matrix{yDF})
}
```

这返回了两个我们可以在Gorgonia中使用的矩阵。

### 创建表达式图

方程$X\cdot\Theta$表示为一个[ExprGraph](/reference/exprgraph)：

```go
func getXY() (*tensor.Dense, *tensor.Dense) {
	x, y := getXYMat()

	xT := tensor.FromMat64(x)
	yT := tensor.FromMat64(y)
	// 去除最后一个维度以创建向量
	s := yT.Shape()
	yT.Reshape(s[0])
	return xT, yT
}

func main() {
	xT, yT := getXY()
	g := gorgonia.NewGraph()
	x := gorgonia.NodeFromAny(g, xT, gorgonia.WithName("x"))
	y := gorgonia.NodeFromAny(g, yT, gorgonia.WithName("y"))
	theta := gorgonia.NewVector(
		g,
		gorgonia.Float64,
		gorgonia.WithName("theta"),
		gorgonia.WithShape(xT.Shape()[1]),
		gorgonia.WithInit(gorgonia.Uniform(0, 1)))

	pred := must(gorgonia.Mul(x, theta))
    // 保存值供以后使用
    var predicted gorgonia.Value
    gorgonia.Read(pred, &predicted)
```

{{% notice info %}}
Gorgonia高度优化；它大量使用指针和内存来获得良好的性能。
因此，在运行时（在执行过程中）调用`*Node`的`Value()`方法可能会导致不正确的结果。
如果我们需要在运行时（例如在学习阶段）访问`*Node`的特定值，我们需要保留对其底层`Value`的引用。这就是我们在这里使用`Read`方法的原因。
`predicted`将随时保存$X\cdot\Theta$的结果值。
{{% /notice %}}

## 准备梯度计算

我们将使用Gorgonia的[符号微分](/how-to/differentiation)功能。

首先，我们将创建成本函数，并使用[求解器](/about/solver)执行梯度下降以降低成本。

### 创建持有成本的节点

我们通过添加成本（$cost = \dfrac{1}{m} \sum_{i=1}^m(X^{(i)}\cdot\Theta-Y^{(i)})^2$）来完成[exprgraph](/reference/exprgraph)：

```go
squaredError := must(gorgonia.Square(must(gorgonia.Sub(pred, y))))
cost := must(gorgonia.Mean(squaredError))
```

我们希望降低这个成本，因此我们评估关于$\Theta$的梯度：

```go
if _, err := gorgonia.Grad(cost, theta); err != nil {
        log.Fatalf("反向传播失败: %v", err)
}
```

### 梯度下降

我们使用梯度下降机制。这意味着我们使用梯度逐步调整参数$\Theta$。

Gorgonia的[Vanilla Solver](https://godoc.org/gorgonia.org/gorgonia#VanillaSolver)实现了基本的梯度下降。
我们将学习率$\gamma$设置为0.001。

```go
solver := gorgonia.NewVanillaSolver(gorgonia.WithLearnRate(0.001))
```

在每一步，我们会要求求解器根据梯度更新$\Theta$参数。
因此，我们设置一个`update`变量，我们将在每次迭代时传递给求解器。

{{% notice info %}}
梯度下降将根据以下方程更新每次传递给`[]gorgonia.ValueGrad`的所有值：
${\displaystyle x^{(k+1)}=x^{(k)}-\gamma \nabla f\left(x^{(k)}\right)}$
重要的是要理解求解器在[`Values`](/reference/value)上工作，而不是在[`Nodes`](/reference/node)上工作。
但为了方便起见，ValueGrad是一个由`*Node`结构实现的接口。
{{% /notice %}}

在我们的例子中，我们想要优化$\Theta$，并要求求解器像这样更新其值：

${\displaystyle \Theta^{(k+1)}=\Theta^{(k)}-\gamma \nabla f\left(\Theta^{(k)}\right)}$

为此，我们需要将$\Theta$传递给`Solver`的`Step`方法：

```go
model := []gorgonia.ValueGrad{theta}
// ...
if err = solver.Step(model); err != nil {
        log.Fatal(err)
}
```

#### 学习迭代

现在我们已经了解了原理，我们需要使用[vm](/reference/vm)运行计算多次，以便梯度下降的魔力发生。

让我们创建一个[vm](/reference/vm)来执行图（并进行梯度计算）：

```go
machine := gorgonia.NewTapeMachine(g, gorgonia.BindDualValues(theta))
defer machine.Close()
```

{{% notice warning %}}
我们将要求求解器更新参数$\Theta$相对于其梯度的值。
因此，我们必须指示TapeMachine存储$\Theta$的值*以及*其导数（其对偶值）。
我们使用[BindDualValues](https://godoc.org/gorgonia.org/gorgonia#BindDualValues)函数来实现这一点。
{{% /notice %}}

现在让我们创建循环并在每一步执行图；机器将学习！

```go
iter := 1000000
var err error
for i := 0; i < iter; i++ {
        if err = machine.RunAll(); err != nil {
                fmt.Printf("迭代过程中出错: %v: %v\n", i, err)
                break
        }

        if err = solver.Step(model); err != nil {
                log.Fatal(err)
        }
        machine.Reset() // 在这样的循环中，Reset是必要的
}
```

#### 获取一些信息

我们可以通过使用这个调用来转储有关学习过程的一些信息：

```go
fmt.Printf("theta: %2.2f  迭代: %v 成本: %2.3f 准确率: %2.2f \r",
        theta.Value(),
        i,
        cost.Value(),
        accuracy(predicted.Data().([]float64), y.Value().Data().([]float64)))
```

其中`accuracy`定义如下：

```go
func accuracy(prediction, y []float64) float64 {
        var ok float64
        for i := 0; i < len(prediction); i++ {
                if math.Round(prediction[i]-y[i]) == 0 {
                        ok += 1.0
                }
        }
        return ok / float64(len(y))
}
```

这将在学习过程中显示如下行：

```text
theta: [ 0.26  -0.41   0.44  -0.62   0.83]  迭代: 26075 成本: 0.339 准确率: 0.61
```

### 保存权重

训练完成后，我们保存$\Theta$的值，以便能够进行预测：

```go
func save(value gorgonia.Value) error {
	f, err := os.Create("theta.bin")
	if err != nil {
		return err
	}
	defer f.Close()
	enc := gob.NewEncoder(f)
	err = enc.Encode(value)
	if err != nil {
		return err
	}
	return nil
}
```

## 创建一个简单的预测CLI

首先，让我们从训练阶段加载参数：

```go
func main() {
        f, err := os.Open("theta.bin")
        if err != nil {
                log.Fatal(err)
        }
        defer f.Close()
        dec := gob.NewDecoder(f)
        var thetaT *tensor.Dense
        err = dec.Decode(&thetaT)
        if err != nil {
                log.Fatal(err)
        }
```

然后，让我们像以前一样创建模型（exprgraph）：

{{% notice info %}}
一个真实的应用程序可能会在一个单独的包中共享模型
{{% /notice %}}

```go
g := gorgonia.NewGraph()
theta := gorgonia.NodeFromAny(g, thetaT, gorgonia.WithName("theta"))
values := make([]float64, 5)
xT := tensor.New(tensor.WithBacking(values))
x := gorgonia.NodeFromAny(g, xT, gorgonia.WithName("x"))
y, err := gorgonia.Mul(x, theta)
```

然后进入一个循环，该循环将从标准输入获取信息，进行计算并显示结果：

```go
machine := gorgonia.NewTapeMachine(g)
values[4] = 1.0
for {
        values[0] = getInput("花萼长度")
        values[1] = getInput("花萼宽度")
        values[2] = getInput("花瓣长度")
        values[3] = getInput("花瓣宽度")

        if err = machine.RunAll(); err != nil {
                log.Fatal(err)
        }
        switch math.Round(y.Value().Data().(float64)) {
        case 1:
                fmt.Println("这可能是一朵山鸢尾（setosa）")
        case 2:
                fmt.Println("这可能是一朵弗吉尼亚鸢尾（virginica）")
        case 3:
                fmt.Println("这可能是一朵变色鸢尾（versicolor）")
        default:
                fmt.Println("未知的鸢尾花")
        }
        machine.Reset()
}
```

这是一个用于获取输入的辅助函数：

```go
func getInput(s string) float64 {
        reader := bufio.NewReader(os.Stdin)
        fmt.Printf("%v: ", s)
        text, _ := reader.ReadString('\n')
        text = strings.Replace(text, "\n", "", -1)

        input, err := strconv.ParseFloat(text, 64)
        if err != nil {
                log.Fatal(err)
        }
        return input
}
```

现在我们可以`go build`或`go run`代码，瞧！
我们有一个完全自主的CLI，可以根据其特征预测鸢尾花的物种：

```text
$ go run main.go
花萼长度: 4.4
花萼宽度: 2.9
花瓣长度: 1.4
花瓣宽度: 0.2
这可能是一朵山鸢尾（setosa）
花萼长度: 5.9
花萼宽度: 3.0
花瓣长度: 5.1
花瓣宽度: 1.8
这可能是一朵弗吉尼亚鸢尾（virginica）
```

# 结论

这是一个循序渐进的示例。
您现在可以使用theta的初始化值进行实验，或者更换求解器来查看Gorgonia中的情况。

完整的代码可以在Gorgonia项目的[示例](https://github.com/gorgonia/gorgonia/tree/master/examples)中找到。

### 奖励：可视化表示

可以使用Gonum绘图库可视化数据集。
以下是如何实现的简单示例：

![iris](/images/iris/iris.png)

```go
import (
    "gonum.org/v1/plot"
    "gonum.org/v1/plot/plotter"
    "gonum.org/v1/plot/plotutil"
    "gonum.org/v1/plot/vg"
    "gonum.org/v1/plot/vg/draw"
)

func plotData(x []float64, a []float64) []byte {
	p, err := plot.New()
	if err != nil {
		log.Fatal(err)
	}

	p.Title.Text = "花萼长度和宽度"
	p.X.Label.Text = "长度"
	p.Y.Label.Text = "宽度"
	p.Add(plotter.NewGrid())

	l := len(x) / len(a)
	for k := 1; k <= 3; k++ {
		data0 := make(plotter.XYs, 0)
		for i := 0; i < len(a); i++ {
			if k != int(a[i]) {
				continue
			}
			x1 := x[i*l+0] // sepal_length
			y1 := x[i*l+1] // sepal_width
			data0 = append(data0, plotter.XY{X: x1, Y: y1})
		}
		data, err := plotter.NewScatter(data0)
		if err != nil {
			log.Fatal(err)
		}
		data.GlyphStyle.Color = plotutil.Color(k - 1)
		data.Shape = &draw.PyramidGlyph{}
		p.Add(data)
		p.Legend.Add(fmt.Sprint(k), data)
	}

	w, err := p.WriterTo(4*vg.Inch, 4*vg.Inch, "png")
	if err != nil {
		panic(err)
	}
	var b bytes.Buffer
	writer := bufio.NewWriter(&b)
	w.WriteTo(writer)
	ioutil.WriteFile("out.png", b.Bytes(), 0644)
	return b.Bytes()
}
```