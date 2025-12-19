+++
title = "Gorgonia工作原理"
date = 2019-10-28T11:41:02+01:00
description = "解释Gorgonia工作原理的文章。"
weight = -9
chapter = true
+++

# 关于

Gorgonia通过创建计算图然后执行它来工作。可以把它想象成一种编程语言，但仅限于数学函数，没有分支能力（没有if/then或循环）。实际上，这是用户应该习惯思考的主导范式。计算图是一个[AST](http://en.wikipedia.org/wiki/Abstract_syntax_tree)（抽象语法树）。

微软的[CNTK](https://github.com/Microsoft/CNTK)及其BrainScript可能最能体现这样的思想：构建计算图和运行计算图是不同的事情，用户在处理它们时应该处于不同的思维模式。

虽然Gorgonia的实现不像CNTK的BrainScript那样强制分离思维，但语法确实有所帮助。

## 进一步了解

本章包含旨在解释Gorgonia工作原理的文章。

{{% notice info %}}
本部分的文章以理解为导向，提供背景和上下文。
{{% /notice %}}

{{% children description="true" %}}