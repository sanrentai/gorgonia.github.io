---
title: "保存权重"
date: 2019-10-29T20:07:16+01:00
draft: false
---

## 目标

本指南的目标是描述一种保存节点[值](/reference/value)并恢复它们的方法。

## 实现

目前最好的方法是保存相应节点的值并在需要时恢复它们。

张量实现了 `GobEncode` 和 `GobDecode` 接口，这是最佳选择。您也可以将后端保存为元素切片，但这会稍微复杂一些。

以下是示例代码（未进行任何优化，欢迎修改）：

```go
package main

import (
        "encoding/gob"
        "fmt"
        "log"
        "os"

        "gorgonia.org/gorgonia"
        "gorgonia.org/tensor"
)

var (
        backup = "/tmp/example_gorgonia"
)

func main() {
        g := gorgonia.NewGraph()

        var x, y, z *gorgonia.Node
        var err error

        // 创建计算图
        x = gorgonia.NewTensor(g,
                gorgonia.Float64,
                2,
                gorgonia.WithShape(2, 2),
                gorgonia.WithName("x"))
        y = gorgonia.NewTensor(g,
                gorgonia.Float64,
                2,
                gorgonia.WithShape(2, 2),
                gorgonia.WithName("y"))
        if z, err = gorgonia.Add(x, y); err != nil {
                log.Fatal(err)
        }

        // 初始化变量
        xT, yT, err := readFromBackup()
        if err != nil {
                log.Println("无法读取备份，进行初始化", err)
                xT = tensor.NewDense(gorgonia.Float64, []int{2, 2}, tensor.WithBacking([]float64{0, 1, 2, 3}))
                yT = tensor.NewDense(gorgonia.Float64, []int{2, 2}, tensor.WithBacking([]float64{0, 1, 2, 3}))
        }
        err = gorgonia.Let(x, xT)
        if err != nil {
                log.Fatal(err)
        }
        err = gorgonia.Let(y, yT)
        if err != nil {
                log.Fatal(err)
        }

        // 创建一个虚拟机来运行程序
        machine := gorgonia.NewTapeMachine(g)
        defer machine.Close()

        if err = machine.RunAll(); err != nil {
                log.Fatal(err)
        }

        fmt.Printf("%v", z.Value())
        err = save([]*gorgonia.Node{x, y})
        if err != nil {
                log.Fatal(err)
        }
}

func readFromBackup() (tensor.Tensor, tensor.Tensor, error) {
        f, err := os.Open(backup)
        if err != nil {
                return nil, nil, err
        }
        defer f.Close()
        dec := gob.NewDecoder(f)
        var xT, yT *tensor.Dense
        log.Println("解码 xT")
        err = dec.Decode(&xT)
        if err != nil {
                return nil, nil, err
        }
        log.Println("解码 yT")
        err = dec.Decode(&yT)
        if err != nil {
                return nil, nil, err
        }
        return xT, yT, nil
}

func save(nodes []*gorgonia.Node) error {
        f, err := os.Create(backup)
        if err != nil {
                return err
        }
        defer f.Close()
        enc := gob.NewEncoder(f)
        for _, node := range nodes {
                err := enc.Encode(node.Value())
                if err != nil {
                        return err
                }
        }
        return nil
}
```

运行结果：

```text
$  go run main.go
2019/10/28 08:07:26 无法读取备份，进行初始化 open /tmp/example_gorgonia: no such file or directory
⎡0  2⎤
⎣4  6⎦
$  go run main.go
2019/10/28 08:07:29 解码 xT
2019/10/28 08:07:29 解码 yT
⎡0  2⎤
⎣4  6⎦
```