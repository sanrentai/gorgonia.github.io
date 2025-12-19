---
title: "图 / Exprgraph"
date: 2019-10-29T19:49:05+01:00
weight: -100
draft: false
---

关于计算图或表达式图已经有很多说法。但它到底是什么？可以把它看作是你想要的数学表达式的AST（抽象语法树）。这是上面例子的图（但用向量和标量加法代替）：

![graph1](https://raw.githubusercontent.com/gorgonia/gorgonia/master/media/exprGraph_example1.png)

顺便说一下，Gorgonia提供了不错的图打印功能。这是方程$y = x^2$及其导数的图示例：

![graph1](https://raw.githubusercontent.com/gorgonia/gorgonia/master/media/exprGraph_example2.png)

读取图很简单。表达式从下往上构建，而导数从上往下构建。这样每个节点的导数大致在同一水平上。

红色轮廓的节点表示它是根节点。绿色轮廓的节点表示它们是叶节点。黄色背景的节点表示它是输入节点。虚线箭头表示哪个节点是指向节点的梯度节点。

具体来说，它表明`c42011e840`（$\frac{\partial{y}}{\partial{x}}$）是输入`c42011e000`（即$x$）的梯度节点。