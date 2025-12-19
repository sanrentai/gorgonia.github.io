---
title: "开始贡献文档"
date: 2020-01-31T14:59:03+01:00
draft: false
---

如果你想开始为Gorgonia文档做贡献，本页面及其链接主题可以帮助你入门。你不需要是开发者或技术作家就可以对Gorgonia文档和用户体验产生重大影响！本页面上的主题只需要一个GitHub账户和一个网络浏览器。

如果你正在寻找如何开始为Gorgonia代码仓库做贡献的信息，请参考[贡献指南](https://github.com/gorgonia/gorgonia/blob/master/CONTRIBUTING.md)。

### 文档基础

Gorgonia文档使用Markdown编写，并使用Hugo处理和部署。源代码位于GitHub上：https://github.com/gorgonia/gorgonia.github.io。大多数文档源代码存储在`/content/`目录中。

你可以通过GitHub网站提交问题、编辑内容和查看他人的更改。你还可以使用GitHub的嵌入式历史和搜索工具。

### 文档布局

文档遵循[《关于文档没有人告诉你的事情》](https://www.divio.com/blog/documentation/)一文中描述的布局。

它分为4个部分。每个部分都是仓库`content/`目录中的一个子目录。

#### 教程

教程：

- 以学习为导向
- 允许新手快速入门
- 是一个课程

类比：教小孩子如何做饭

仓库中的内容来源：[`content/tutorials`](https://github.com/gorgonia/gorgonia.github.io/tree/develop/content/tutorials)

#### 使用指南

使用指南：

- 以目标为导向
- 展示如何解决特定问题
- 是一系列步骤

类比：烹饪书中的食谱

仓库中的内容来源：[`content/how-to`](https://github.com/gorgonia/gorgonia.github.io/tree/develop/content/how-to)

#### 解释

解释：

- 以理解为导向
- 进行解释
- 提供背景和上下文

类比：一篇关于烹饪社会历史的文章

仓库中的内容来源：[`content/about`](https://github.com/gorgonia/gorgonia.github.io/tree/develop/content/about)

#### 参考

参考指南：

- 以信息为导向
- 描述内部机制
- 准确完整

类比：百科全书的参考文章

仓库中的内容来源：[`content/reference`](https://github.com/gorgonia/gorgonia.github.io/tree/develop/content/reference)

### 多语言

文档源代码在/content/目录中提供多种语言版本。每个页面可以通过添加由[ISO 639-1标准](https://en.wikipedia.org/wiki/List_of_ISO_639-1_codes)确定的两个字母代码来翻译。
没有任何后缀的文件默认为英语。

例如，页面的法语文档命名为`page.fr.md`。

## 改进文档

### 修复现有内容

你可以通过修复文档中的错误或拼写错误来改进文档。
要改进现有内容，你需要先创建一个_fork_，然后提交一个_pull request (PR)_。这两个术语是[GitHub特有的](https://help.github.com/categories/collaborating-with-issues-and-pull-requests/)。
对于本主题，你不需要了解它们的所有内容，因为你可以使用网络浏览器完成所有操作。

### 创建新内容

{{% notice info %}}
仓库的源代码在`develop`分支中维护。因此，这个分支必须是新分支的基础，PR也应该指向这个分支。
{{% /notice %}}
要创建新内容，请在与文档主题对应的目录中创建一个新页面（请参阅[文档布局](#layout-of-the-documentation)段落）

如果你在本地安装了`hugo`，你可以使用以下命令创建一个新页面：

```shell
hugo new content/about/mypage.md
```

否则，请创建一个新页面，其头部如下：

```yaml
---
title: "页面标题"
date: 2020-01-31T14:59:03+01:00
draft: false
---

你的内容
```
然后按照下面的说明提交一个pull request。

### 提交pull request

按照以下步骤提交pull request来改进Gorgonia文档。

- 在你看到问题的页面上，点击右上角的"编辑此页面"图标。
  一个新的GitHub页面会出现，并带有一些帮助文本。
- 如果你从未创建过Gorgonia文档仓库的fork，系统会提示你这样做。
  在你的GitHub用户名下去创建fork，而不是你可能是成员的其他组织。
  fork通常有一个URL，如`https://github.com/<username>/website`，除非你已经有一个名称冲突的仓库。

  系统提示你创建fork的原因是你没有直接向Gorgonia仓库推送分支的权限。

- GitHub Markdown编辑器会出现，并加载源Markdown文件。
  进行你的更改。在编辑器下方，填写**Propose file change**表单。
  第一个字段是你的提交信息摘要，长度不应超过50个字符。
  第二个字段是可选的，但如果合适的话，可以包含更多细节。
  点击**Propose file change**。更改会作为提交保存在你的fork的新分支中，该分支会自动命名为`patch-1`之类的名称。

{{% notice info %}}
不要在提交信息中包含对其他GitHub问题或pull request的引用。你可以稍后将它们添加到pull request描述中。
{{% /notice %}}

- 下一个屏幕通过将你的新分支（**head fork**和**compare**选择框）与**base fork**和**base**分支（默认情况下是`gorgonia/gorgonia.github.io`仓库上的`develop`）的当前状态进行比较，来总结你所做的更改。你可以更改任何选择框，但现在不要这样做。查看屏幕底部的差异查看器，如果一切看起来都正确，点击**Create pull request**。

{{% notice info %}}
如果你现在不想创建pull request，你可以稍后再创建，方法是浏览到Gorgonia网站仓库或你的fork仓库的主URL。如果GitHub网站检测到你向fork推送了一个新分支，它会提示你创建pull request。
{{% /notice %}}

- **Open a pull request**屏幕出现。pull request的主题与提交摘要相同，但如果需要，你可以更改它。正文由你的扩展提交消息（如果存在）和一些模板文本填充。阅读模板文本并填写它要求的详细信息，然后删除多余的模板文本。如果你在描述中添加`fixes #<000000>`或`closes #<000000>`，其中`#<000000>`是相关问题的编号，GitHub会在PR合并时自动关闭该问题。
  保持**Allow edits from maintainers**复选框被选中。点击**Create pull request**。

  恭喜！你的pull request可以在[Pull requests](https://github.com/gorgonia/gorgonia.github.io/pulls)中找到。

{{% notice info %}}
请限制每个PR只包含一种语言。例如，如果你需要对多种语言的相同代码示例进行相同的更改，请为每种语言打开一个单独的PR。
{{% /notice %}}

- 等待审核。
  如果审核者要求你进行更改，你可以转到**Files changed**标签页，点击pull request更改过的任何文件上的铅笔图标。当你保存更改后的文件时，会在pull request监控的分支中创建一个新的提交。如果你在等待审核者审核更改，请每7天主动联系审核者一次。你也可以加入[gopherslack](https://invite.slack.golangbridge.org/)上的`#gorgonia`频道，这是一个寻求PR审核帮助的好地方。

- 如果你的更改被接受，审核者会合并你的pull request，更改会在几分钟后在Gorgonia网站上生效。

这只是提交pull request的一种方式。如果你已经是Git和GitHub的高级用户，你可以使用本地GUI或命令行Git客户端，而不是使用GitHub UI。