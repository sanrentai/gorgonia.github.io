---
title: "使用CUDA的卷积神经网络"
date: 2020-02-16T22:08:40+01:00
draft: false
---

本教程介绍如何在GPU上运行简单的卷积神经网络。

本教程中使用的示例基于MNIST数据集。您的开发环境应该已经按照["简单卷积神经网络（MNIST）"](/tutorials/mnist/)教程中的描述准备就绪。

## 准备CUDA绑定

CUDA绑定依赖于CGO和官方的CUDA工具包。您可以手动安装它，或者如果您使用AWS，可以依赖预安装了所有必要组件的AMI。

### 手动安装CUDA工具包

CUDA工具包的安装超出了本教程的范围。但您必须确保：

1. 已安装[CUDA工具包](https://developer.nvidia.com/CUDA-toolkit)（已成功测试版本10）。安装后会包含`nvcc`编译器，这是使用CUDA运行代码所必需的。
2. 您已运行[安装后步骤](http://docs.nvidia.com/CUDA/CUDA-installation-guide-linux/index.html#post-installation-actions)。

### 使用AWS EC2

AWS提供了预安装了CUDA工具包的AMI。

您可以通过以下命令获取这些AMI的列表：

```shell
~ aws ec2 describe-images --owners amazon --filters 'Name=state,Values=available' 'Name=name,Values=Deep Learning AMI (Ubuntu)*' --query 'sort_by(Images, &CreationDate)[].Name'
```

这些AMI已在`g3s.xlarge`实例上针对Gorgonia 0.9.8版本进行了成功测试。

{{% notice info %}}
为了方便起见，您可以在[这里](https://github.com/gorgonia/dev/tree/master/infrastructure/aws/gpu)找到一个[terraform](terraform.io)文件，帮助您在AWS EC2上快速启动VM。
{{% /notice %}}

## 准备代码

硬件种类繁多。为了应对不同硬件的特性，Gorgonia提供了一个命令，可以为您的硬件生成特定的绑定。这个功能由一个名为`CUDAgen`的特定工具提供。

{{% notice warning %}}
`CUDAgen`与go modules不兼容，您需要关闭它们。
{{% /notice %}}

这些命令安装`cudagen`工具并生成CUDA绑定：

```shell
~ export GO111MODULE=off
~ go get gorgonia.org/gorgonia
~ export CGO_CFLAGS="-I/usr/local/cuda-10.0/include/"
~ export PATH=$PATH:/usr/local/cuda/bin/
~ go get gorgonia.org/cu
~ go install gorgonia.org/gorgonia/cmd/cudagen
~ $GOPATH/bin/cudagen
```

## 运行示例

Gorgonia的示例目录包含一个[`convenet_CUDA`](https://github.com/gorgonia/gorgonia/tree/master/examples/convnet_cuda)示例。
这个示例在MNIST数据库上运行卷积神经网络。

{{% notice info %}}
该代码与`convnet`示例相似；唯一的区别在于操作符的导入；
这个版本使用来自[`nnops`](https://github.com/gorgonia/gorgonia/tree/master/ops/nn)的操作符。这个包包含一些主要用于神经网络的操作符定义（`Conv2D`、`Maxpool`等）；这些定义具有与CUDA中的对应操作符兼容的签名。`nnops`包确保在不使用CUDA时与CPU版本的操作符兼容。
{{% /notice %}}

假设测试文件已放置在`../testdata`目录中（如果没有，请参考["简单卷积神经网络（MNIST）"](/tutorials/mnist/)教程），您可以通过以下命令启动支持CUDA的训练阶段：

```text
time go run -tags='CUDA'  main.go -epochs 1 2> /dev/null
Epoch 0 599 / 600 [====================================================]  99.83%
```

您还可以在单独的窗口中运行`nvidia-smi`命令来"监控"CUDA的使用情况。
这应该会显示类似以下的内容：

```text
~  nvidia-smi
Sun Feb 16 22:05:15 2020
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 418.87.00    Driver Version: 418.87.00    CUDA Version: 10.1     |
|-------------------------------+----------------------+----------------------+|
 GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|===============================+======================+======================| |   0  Tesla M60           On   | 00000000:00:1E.0 Off |                    0 |
| N/A   54C    P0    73W / 150W |    841MiB /  7618MiB |     76%      Default |
+-------------------------------+----------------------+----------------------+

+-----------------------------------------------------------------------------+
| Processes:                                                       GPU Memory |
|  GPU       PID   Type   Process name                             Usage      |
|=============================================================================|
|    0     18614      C   /tmp/go-build629284435/b001/exe/main         372MiB |
+-----------------------------------------------------------------------------+
```