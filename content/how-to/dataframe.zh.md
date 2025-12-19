---
title: "如何从数据框（gota）创建张量"
date: 2019-10-30T22:57:09+01:00
draft: false
---

本指南将解释如何使用 [gota](https://github.com/go-gota/gota) 库从数据框创建张量。
目标是读取 CSV 文件并创建一个形状为 (2,2) 的 [`*tensor.Dense`](https://godoc.org/gorgonia.org/tensor#Dense)。

## 从 CSV 文件创建数据框

考虑一个包含以下内容的 CSV 文件：

```text
sepal_length,sepal_width,petal_length,petal_width,species
5.1         ,3.5        ,1.4         ,0.2        ,setosa
4.9         ,3.0        ,1.4         ,0.2        ,setosa
4.7         ,3.2        ,1.3         ,0.2        ,setosa
4.6         ,3.1        ,1.5         ,0.2        ,setosa
5.0         ,3.6        ,1.4         ,0.2        ,setosa
...
```

{{% notice info %}}
这是 [鸢尾花数据集](https://en.wikipedia.org/wiki/Iris_flower_data_set) 的一部分。
数据集的副本可以在 [这里](https://gist.github.com/owulveryck/19a5ba9553ff8209b3b4227b5325041b#file-iris-csv) 找到。
{{% /notice %}}

我们想要创建一个包含除 species 列之外所有值的张量。

## 使用 gota 创建数据框

gota 的 dataframe 包有一个 [`ReadCSV`](https://godoc.org/github.com/kniren/gota/dataframe#ReadCSV) 函数，它接受一个 io.Reader 作为参数。

```go
f, err := os.Open("iris.csv")
if err != nil {
    log.Fatal(err)
}
defer f.Close()
df := dataframe.ReadCSV(f)
```

`df` 是一个 [`DataFrame`](https://godoc.org/github.com/kniren/gota/dataframe#DataFrame)，包含文件中的所有数据。

{{% notice info %}}
gota 使用 CSV 的第一行来引用数据框中的列。
{{% /notice %}}

让我们移除 species 列：

```go
xDF := df.Drop("species")
```
## 将数据框转换为矩阵

为了方便起见，我们将数据框转换为 gonum 定义的 `Matrix`（参见 [matrix godoc](https://godoc.org/gonum.org/v1/gonum/mat#Matrix)）。
`matrix` 是一个接口。gota 的 dataframe 不满足 `Matrix` 接口。正如 gota 文档中所描述的，
我们创建一个围绕 DataFrame 的包装器来满足 `Matrix` 接口。

```go
type matrix struct {
    dataframe.DataFrame
}

func (m matrix) At(i, j int) float64 {
    return m.Elem(i, j).Float()
}

func (m matrix) T() mat.Matrix {
    return mat.Transpose{Matrix: m}
}
```
## 创建张量

现在，我们可以通过将数据框包装到 `matrix` 结构中，使用函数 [`tensor.FromMat64`](https://godoc.org/gorgonia.org/tensor#FromMat64) 创建一个 `*Dense` 张量。

```go
xT := tensor.FromMat64(mat.DenseCopyOf(&matrix{xDF}))
```