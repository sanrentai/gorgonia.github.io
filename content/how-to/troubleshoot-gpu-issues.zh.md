---
title: "GPU问题排查"
date: 2020-07-17T06:24:26+10:00
draft: false
---

本文档是一个正在更新的GPU问题排查指南列表。如果您在使用GPU时遇到问题，本文档应该会有所帮助。

`cu`包附带了一个名为`cudatest`的应用程序，它将有助于排查问题。

要安装`cudatest`，请运行：

```shell
go install gorgonia.org/cu/cmd/cudatest
```

这也假设您已经安装了CUDA和cuDNN。

# 多GPU环境下的初始化错误 #

如果您运行多个GPU，可能会遇到如下消息：

```shell
Error in initialization, please refer to "https://docs.nvidia.com/cuda/cuda-driver-api/group__CUDA__INITIALIZE.html"
```

这通常意味着您的某个GPU不支持CUDA。如果您知道至少有一个GPU支持CUDA，仍然可以使用CUDA运行。

首先，使用`nvidia-smi`查找正在运行的GPU。下面是一个示例：

```shell
Thu Jul 16 17:41:10 2020
+-----------------------------------------------------------------------------+
| NVIDIA-SMI 450.51.05    Driver Version: 450.51.05    CUDA Version: 11.0     |
|-------------------------------+----------------------+----------------------+|
GPU  Name        Persistence-M| Bus-Id        Disp.A | Volatile Uncorr. ECC |
| Fan  Temp  Perf  Pwr:Usage/Cap|         Memory-Usage | GPU-Util  Compute M. |
|                               |                      |               MIG M. |
|===============================+======================+======================||
   0  Tesla K20Xm         On   | 00000000:06:00.0 Off |                    0 ||
N/A   33C    P8    16W / 235W |      0MiB /  5700MiB |      0%      Default ||
                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+|
   1  GeForce GT 1030     On   | 00000000:07:00.0  On |                  N/A ||
35%   33C    P0    N/A /  30W |    656MiB /  1994MiB |     51%      Default ||
                               |                      |                  N/A |
+-------------------------------+----------------------+----------------------+
+-----------------------------------------------------------------------------+
| Processes:                                                                  |
|  GPU   GI   CI        PID   Type   Process name                  GPU Memory |
|        ID   ID                                                   Usage      |
|=============================================================================|
   1   N/A  N/A      XXXX      G   /usr/lib/xorg/Xorg                270MiB |
|    1   N/A  N/A      XXXX      G   /usr/bin/PROGRAMNAME               77MiB |
|    1   N/A  N/A      XXXX      G   /usr/bin/PROGRAMNAME               68MiB |
|    1   N/A  N/A      XXXX      G   ...AAAAAAAAA= --shared-files      221MiB |
+-----------------------------------------------------------------------------+
```

在这里，我们看到有两个GPU：

* GPU ID 0是Tesla K20Xm。
* GPU ID 1是GeForce GT 1030。

GeForce GT 1030不支持CUDA，而Tesla K20Xm支持。要解决此问题，只需添加此环境变量：

```shell
CUDA_VISIBLE_DEVICES=0 cudatest
```

应该会返回类似以下内容：

```shell
$ CUDA_VISIBLE_DEVICES=0 cudatest
CUDA version: 11000
CUDA devices: 1
Device 0
========
Name      :     "Tesla K20Xm"
Clock Rate:     732000 kHz
Memory    :     5977800704 bytes
Compute   :     3.5
```