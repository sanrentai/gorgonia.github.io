---
title: "CUDA支持"
date: 2020-03-24T17:17:17+01:00
draft: false
---

Gorgonia内置了CUDA支持，但使用方式比较特殊。
要使用CUDA，您必须使用构建标签`cuda`来构建应用程序，如下所示：

```shell
go build -tags='cuda' .
```

此外，还有一些额外要求：

- 需要安装[CUDA工具包](https://developer.nvidia.com/cuda-toolkit)。安装后会包含`nvcc`编译器，这是运行CUDA代码所必需的（请确保按照[安装后步骤](http://docs.nvidia.com/cuda/cuda-installation-guide-linux/index.html#post-installation-actions)进行操作）。
- 执行`go install gorgonia.org/gorgonia/cmd/cudagen`安装`cudagen`程序。
- 运行`cudagen`会为Gorgonia生成相关的CUDA代码。注意，您需要在`src\gorgonia.org\gorgonia\cuda modules\target`目录下创建一个文件夹。
- 目前只有某些操作符支持CUDA驱动，它们在单独的[`ops/nn`包](https://godoc.org/github.com/gorgonia/gorgonia/ops/nn)中实现。

{{% notice warning %}}
CUDA需要线程亲和性，因此必须锁定OS线程。在运行VM的主函数中必须调用`runtime.LockOSThread()`。请参考这个[维基](https://github.com/golang/go/wiki/LockOSThread)了解如何在Go程序中正确处理这个问题。
{{% /notice %}}

### 原理

使用CUDA需要如此复杂要求的主要原因很简单：性能。正如Dave Cheney著名的文章所说，[cgo不是Go](https://dave.cheney.net/2016/01/18/cgo-is-not-go)。不幸的是，使用CUDA需要cgo，而使用cgo需要做出很多权衡。

因此，解决方案是将CUDA相关代码放在构建标签`cuda`中。这样默认情况下不使用cgo（嗯，有点例外 - 您仍然可以使用`cblas`或`blase`）。

### 关于`cudagen`

需要[CUDA工具包](https://developer.nvidia.com/cuda-toolkit)和`cudagen`工具的原因是CUDA有许多[计算能力](http://docs.nvidia.com/cuda/cuda-c-programming-guide/index.html#compute-capabilities)，为所有这些能力生成代码会产生一个巨大的二进制文件，没有真正的好理由。相反，鼓励用户为其特定的计算能力进行编译。

{{% notice info %}}
需要明确指定哪些操作符使用CUDA的原因是cgo调用的成本。目前正在进行额外的工作来实现批处理cgo调用，但在此之前，解决方案是对某些操作符进行"升级"。
{{% /notice %}}
最后，需要明确指定哪些操作符使用CUDA的原因是cgo调用的成本。目前正在进行额外的工作来实现批处理cgo调用，但在此之前，解决方案是对某些操作符进行"升级"。

### 支持CUDA的`Op`s

截至目前，只有非常基本的简单操作符支持CUDA：

元素级一元操作：

* `abs`
* `sin`
* `cos`
* `exp`
* `ln`
* `log2`
* `neg`
* `square`
* `sqrt`
* `inv`（数字的倒数）
* `cube`
* `tanh`
* `sigmoid`
* `log1p`
* `expm1`
* `softplus`

元素级二元操作 - 只有算术操作支持CUDA：

* `add`
* `sub`
* `mul`
* `div`
* `pow`

根据作者个人项目的大量分析，真正重要的是`tanh`、`sigmoid`、`expm1`、`exp`和`cube` - 基本上是激活函数。其他操作使用MKL+AVX效果很好，不是神经网络中速度慢的主要原因。

### CUDA改进

在一个简单的基准测试中，谨慎使用CUDA（在这种情况下，用于调用`sigmoid`）显示出比非CUDA代码令人印象深刻的改进（记住CUDA内核非常简单，没有优化）：

```test
BenchmarkOneMilCUDA-8   	     300	   3348711 ns/op
BenchmarkOneMil-8       	      50	  33169036 ns/op
```

## 示例
请查看这个[tutorial](/tutorials/mnist-cuda/)获取完整示例