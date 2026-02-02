---
title: '介绍自定义组件'
published: 2024-08-14
tags: [minecraft, custom-components]
category: '翻译'
draft: false
author: rice_awa
description: '介绍一种新的实验性概念——自定义组件。'
---

> [!NOTE]
> 原文：[微软文档](https://learn.microsoft.com/en-us/minecraft/creator/documents/customcomponents?view=minecraft-bedrock-stable)
>
> 日期：2024 年 8 月 14 日
>
> 翻译：[rice_awa](https://github.com/rice-awa)
>
> [源项目许可证](https://github.com/MicrosoftDocs/minecraft-creator?tab=MIT-2-ov-file)

# 介绍自定义组件

自定义组件是一种将方块和物品的 JSON 配置直接连接到脚本功能的新方式。这种新概念允许在方块和物品之间组合和重用脚本功能，同时确保脚本仅针对特定的方块和物品运行。

这种新模式结合了一种更结构化的方式来监听方块和物品事件，同时获得脚本中的所有功能，并确保这些事件与当前脚本事件在相同的约束下运行。

如果你想开始使用自定义组件构建附加组件，请参阅我们的[自定义组件教程](./CustomComponentsTutorial)。

## 结构

这个新特性被称为*自定义组件*，因为脚本以类似于附加内置 Minecraft 组件的方式连接到给定的方块或物品的 JSON。因此，该特性既有 JSON 方面的内容，也有脚本方面的内容。

## 脚本 API

从脚本 API 开始，我们为方块引入了两个新的主要接口：`BlockTypeRegistry` 和 `BlockCustomComponent`。（注意：在 1.21.0 之后的 Minecraft 预览版本中，BlockTypeRegistry 被称为 BlockComponentRegistry）`BlockTypeRegistry` 包含一个用于按名称注册新自定义组件的方法：

```typescript
/**
 * @beta
 * 提供用于注册方块自定义组件的功能。
 */
export class BlockTypeRegistry {
  /**
   * @remarks
   * 注册一个可以在方块 JSON 配置中使用的方块自定义组件。
   *
   * @param name
   * 表示此自定义组件的 ID。必须有命名空间。此 ID 可以在方块的 JSON 配置中的 'minecraft:custom_components' 方块组件下指定。
   * @param customComponent
   * 事件函数的集合，当使用此自定义组件 ID 的方块上发生事件时将调用这些函数。
   */
  registerCustomComponent(name: string, customComponent: BlockCustomComponent): void
}
```

脚本中的自定义组件是名称/ID 与由事件表示的一组功能的关联。可以监听的事件在第二个接口 `BlockCustomComponent` 中指定。

```typescript
/**
 * @beta
 * 包含为方块引发的一组事件。此对象必须使用 BlockRegistry 绑定。
 */
export interface BlockCustomComponent {
  /**
   * @remarks
   * 当实体踩在绑定此自定义组件的方块上时将调用此函数。
   *
   */
  onStepOn?: (arg: BlockComponentStepOnEvent) => void
}
```

通过脚本，你可以通过监听新的实验性 `worldInitialize` _before_ 事件来访问 `BlockTypeRegistry`。所有自定义组件的注册必须在 worldInitialize 期间进行，因为此功能直接附加到 JSON 中的方块初始化。从此事件中，你可以调用 `registerCustomComponent`，使用唯一的命名空间名称和实现 `BlockCustomComponent` 接口的对象来注册该组件。一旦组件注册成功，任何使用此组件的方块都会在相关事件中调用你的对象的回调。

以下是显示此注册的小代码示例：

```typescript
world.beforeEvents.worldInitialize.subscribe((initEvent) => {
  initEvent.blockTypeRegistry.registerCustomComponent('content:turn_to_air', {
    onStepOn: (e) => {
      e.block.setPermutation(BlockPermutation.resolve(MinecraftBlockTypes.Air))
    },
  })
})
```

在上述示例中，注册了一个名为 `content:turn_to_air` 的自定义组件，该组件监听 `onStepOn` 事件。
也可以将 JavaScript 类注册为自定义组件；唯一重要的是遵循 `BlockCustomComponent` 接口。以下是使用类模式的示例。

```typescript
class TurnToAirComponent implements BlockCustomComponent {
  constructor() {
    this.onStepOn = this.onStepOn.bind(this)
  }

  onStepOn(e: BlockComponentStepOnEvent): void {
    e.block.setPermutation(BlockPermutation.resolve(MinecraftBlockTypes.Air))
  }
}

world.beforeEvents.worldInitialize.subscribe((initEvent) => {
  initEvent.blockTypeRegistry.registerCustomComponent(
    'content:turn_to_air',
    new TurnToAirComponent()
  )
})
```

根据你的附加组件或世界的需求，可以选择你喜欢的模式！它们在功能上是相同的。需要注意的是，所有事件通知都是无状态的，事件参数会告知你它们在世界中涉及哪个具体的方块。

## JSON

注册到特定名称的自定义组件后，就需要将其附加到自定义方块中。
在自定义方块的 JSON 中的 `components` 键下，有一个新的 `minecraft:custom_components` 键，可以在其中注册自定义组件。自定义组件在这里以有序数组的形式列出。此数组允许你控制注册的自定义组件在特定事件中被通知的顺序。以下是注册我们的 `content:turn_to_air` 自定义组件的示例。

```JSON
    "minecraft:block": {
        "description": {
            "identifier": "content:my_block"
        },
        // 基础组件
        "components": {
            "minecraft:custom_components": ["content:turn_to_air"],
            "minecraft:loot": "loot_tables/blocks/my_block.json",
```

在上述示例中，自定义组件与其他组件（如战利品表）一起附加。如果注册了多个自定义组件，则对于每种类型的事件，事件将按此数组中的顺序发送给自定义组件。

也可以将自定义组件添加到组件键下的每个特定排列中。然而，重要的是要注意，排列中的自定义组件数组完全替换了基础组件列表中的数组。
同一个自定义组件也可以在多个文件中使用。例如，这个 `content:turn_to_air` 组件可以在多个自定义方块中重用，从而实现轻松对齐和组合行为。此外，一个行为包中注册的自定义组件可以在另一个行为包中使用。

## 工作流程

在编写自定义组件及其对应的脚本代码时，你可以使用热重载的全部功能，在世界中迭代你的更改。

当对 JSON 和/或新自定义组件的注册进行更改时，需要退出世界并重新进入以查看更改的效果。

一旦进入脚本，你的回调可以像今天的任何其他脚本一样使用脚本 API 中的任何 API，并具有相同的约束。

## 开始构建自定义组件

如果你想开始使用自定义组件构建附加组件，请访问[自定义组件教程](./CustomComponentsTutorial.md)。需要注意的是，在 1.21.0 之后的 Minecraft 预览版本中，BlockTypeRegistry 被称为 BlockComponentRegistry。
