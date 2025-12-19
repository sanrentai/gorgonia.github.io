+++
title = "快速开始"
description = "Gorgonia快速入门"
date = 2019-10-29T17:42:44+01:00
weight = -10
chapter = true
+++

## 获取Gorgonia

Gorgonia支持go get和go modules。要获取库及其依赖，只需运行

```bash
$ go get gorgonia.org/gorgonia
```

## 第一个简单计算程序

创建一个简单的程序来检查环境是否正常：

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

	// 创建一个VM来运行程序
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

运行程序应该输出结果：`4.5`

如果你看到以下错误消息：

```
panic: Something in this program imports go4.org/unsafe/assume-no-moving-gc to declare that it assumes a non-moving garbage collector, but your version of go4.org/unsafe/assume-no-moving-gc hasn't been updated to assert that it's safe against the go1.19 runtime. If you want to risk it, run with environment variable ASSUME_NO_MOVING_GC_UNSAFE_RISK_IT_WITH=go1.19 set. Notably, if go1.19 adds a moving garbage collector, this program is unsafe to use.
```

那么请执行

`export ASSUME_NO_MOVING_GC_UNSAFE_RISK_IT_WITH=go1.XX ` 

其中`XX`是你的`go.mod`文件中定义的次要版本。

然后再次运行程序。

有关进一步的解释，请参阅[Hello World教程](/tutorials/hello-world)。