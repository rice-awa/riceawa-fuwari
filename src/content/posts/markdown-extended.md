---
title: Markdown 扩展功能
published: 2024-05-01
updated: 2024-11-29
description: '了解更多关于 Fuwari 中的 Markdown 功能'
image: ''
author: saicaca
tags: [演示, 示例, Markdown, Fuwari]
category: '示例'
draft: false
---

## GitHub 仓库卡片
你可以添加链接到 GitHub 仓库的动态卡片,在页面加载时,仓库信息会从 GitHub API 获取。

::github{repo="Fabrizz/MMM-OnSpotify"}

使用代码 `::github{repo="<owner>/<repo>"}` 创建 GitHub 仓库卡片。

```markdown
::github{repo="saicaca/fuwari"}
```

## 提示框

支持以下类型的提示框: `note` `tip` `important` `warning` `caution`

:::note
强调用户应该注意的信息,即使是在浏览时。
:::

:::tip
可选信息,帮助用户更成功。
:::

:::important
用户成功所必需的关键信息。
:::

:::warning
由于潜在风险而需要用户立即关注的关键内容。
:::

:::caution
某个操作的负面潜在后果。
:::

### 基本语法

```markdown
:::note
强调用户应该注意的信息,即使是在浏览时。
:::

:::tip
可选信息,帮助用户更成功。
:::
```

### 自定义标题

提示框的标题可以自定义。

:::note[我的自定义标题]
这是一个带有自定义标题的提示框。
:::

```markdown
:::note[我的自定义标题]
这是一个带有自定义标题的提示框。
:::
```

### GitHub 语法

> [!TIP]
> 也支持 [GitHub 语法](https://github.com/orgs/community/discussions/16925)。

```
> [!NOTE]
> 也支持 GitHub 语法。

> [!TIP]
> 也支持 GitHub 语法。
```

### 剧透

你可以为文本添加剧透。文本还支持 **Markdown** 语法。

内容 :spoiler[被隐藏了 **哈哈**]!

```markdown
内容 :spoiler[被隐藏了 **哈哈**]!

```
