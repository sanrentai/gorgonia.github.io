---
title: "LispMachine"
date: 2019-10-29T19:50:15+01:00
draft: false
---

`LispMachine` 被设计为接受一个图作为输入，并直接在图的节点上执行。
如果图发生变化，只需创建一个新的轻量级 `LispMachine` 来执行它即可。
`LispMachine` 适合用于创建无固定大小的循环神经网络等任务。

权衡是，对于相同的图的静态"图像"，在 `LispMachine` 上执行图通常比在 `TapeMachine` 上慢。