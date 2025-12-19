---
title: "使用 Graphviz (dot) 绘制 ExprGraph"
date: 2019-12-01T10:14:55+01:00
draft: false
---

Gorgonia 的 [`encoding`](https://godoc.org/gorgonia.org/gorgonia/encoding/dot) 包包含一个函数，可以将 [`ExprGraph`](/reference/exprgraph) 转换为 [dot 语言](https://www.graphviz.org/doc/info/lang.html)。

这使得可以使用 [graphviz](https://www.graphviz.org/) 程序生成图形的 png 或 svg 版本。

一个简单的实现方法：

```go
package main

import (
        "fmt"
        "log"

        "gorgonia.org/gorgonia"
        "gorgonia.org/gorgonia/encoding/dot"
)

func main() {
        g := gorgonia.NewGraph()

        var x, y *gorgonia.Node

        // 定义表达式
        x = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("x"))
        y = gorgonia.NewScalar(g, gorgonia.Float64, gorgonia.WithName("y"))
        gorgonia.Add(x, y)
        b, err := dot.Marshal(g)
        if err != nil {
                log.Fatal(err)
        }
        fmt.Println(string(b))
}
```

运行这个程序并将其输出发送到 dot 进程，可以生成一张图片。

例如：

```shell
$ go run main.go | dot -Tsvg > dot-example.svg
```

生成的图形：

![graph](/images/dot-example.svg)