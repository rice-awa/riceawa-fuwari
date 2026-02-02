---
title: '使用自定义组件构建'
published: 2024-08-14
tags: [minecraft, custom-components, MCBE]
category: '翻译'
draft: false
author: rice_awa
description: '通过本教程开始使用自定义组件构建。'
---

> [!NOTE]
> 原文：[微软文档](https://learn.microsoft.com/en-us/minecraft/creator/documents/customcomponentstutorial?view=minecraft-bedrock-stable)
>
> 日期：2024 年 8 月 14 日
>
> 翻译：[rice_awa](https://github.com/rice-awa)
>
> [源项目许可证](https://github.com/MicrosoftDocs/minecraft-creator?tab=MIT-2-ov-file)

# 使用自定义组件进行构建

## 创建一个带有自定义组件的附加包

> [!NOTE]
> 自定义组件功能目前处于预览阶段，可通过测试版API使用，但计划在未来的Minecraft版本中包含在“稳定API”中——可能是1.21.10版本。我们建议您在尝试此示例时使用最新版本的Minecraft Preview。

自定义方块和物品使用各种组件，在其定义中声明，以增强方块或物品的行为。

到目前为止，所有组件都是内置于Minecraft中的，通过各种参数来控制组件的行为。使用[自定义组件](./CustomComponents.md)，现在处于预览阶段，并将在即将发布的Minecraft版本中更广泛地提供，您可以结合脚本为方块和物品定义自己的行为！在本教程中，我们将通过添加新的草莓作物和草莓物品，制作一个小的方块和物品示例。草莓植物将具有这样的行为：如果在成熟时没有被采摘，它们会变坏。您必须把握好时机才能获得新鲜的草莓，在这个示例中，您将看到管理作物生长速度的方块组件以及自定义物品效果。

![草莓农场和一些腐烂的草莓](assets/images/strawberryfarm.png)

您可以在[github.com/microsoft/minecraft-scripting-samples](https://github.com/microsoft/minecraft-scripting-samples/tree/main/custom-components)上找到此示例的源代码。

### 前提条件

在开始之前，您应该已经完成了附加包入门教程、行为包介绍教程和脚本介绍教程。

- [附加包开发入门](./GettingStarted.md)
- [行为包介绍](./BehaviorPack.md)
- [脚本介绍](./ScriptingIntroduction.md)

您需要熟悉附加包文件夹的结构、行为包应包含的必需文件以及如何在行为包中使用脚本。

一些其他有用的资源是如何在您的附加包中制作方块和物品：

- [创建自定义染料方块](./AddCustomDieBlock.md)

## 物品自定义组件

草莓物品将利用自定义组件为食用它的玩家提供夜视效果。首先让我们制作草莓物品本身。

```json
{
  "format_version": "1.21.10",
  "minecraft:item": {
    "description": {
      "menu_category": {
        "group": "itemGroup.name.crop",
        "category": "nature"
      },
      "identifier": "example:strawberry"
    },
    "components": {
      "minecraft:icon": "strawberry",
      "minecraft:max_stack_size": 64,
      "minecraft:use_modifiers": {
        "use_duration": 1.6,
        "movement_modifier": 0.35
      },
      "minecraft:food": {
        "can_always_eat": true,
        "nutrition": 1,
        "saturation_modifier": 0.5
      },
      "minecraft:use_animation": "eat"
    }
  }
}
```

### 向物品添加自定义组件

要向物品添加自定义组件，请向物品添加一个`minecraft:custom_components`组件，该组件包含一个字符串数组。

在这个数组中，字符串是您要添加的组件的名称。当事件发生时，组件将运行一些脚本，并且这些脚本将按照您在JSON中定义的顺序运行。这意味着一个物品可以以不同于另一个物品的顺序运行相同的组件。对于我们的草莓，我们只需要一个组件，但对于您自己的物品，您可以向数组中添加更多自定义组件。

```json
"minecraft:custom_components": [
    "example:add_night_vision_on_consume"
]
```

您为自定义组件选择的名称需要一个命名空间。在上面的代码中，命名空间是`example`，组件名称是`add_night_vision_on_consume`。

### 在脚本中注册物品自定义组件

现在我们的物品已经配置为具有自定义组件，我们需要在脚本中注册该组件的行为。我们想要的行为是，当您食用草莓时，玩家会获得一段时间的夜视效果。首先，我们使用与JSON组件中定义的相同名称将我们的组件注册到`ItemComponentRegistry`，并提供该组件正在监听的事件列表。在这种情况下，我们将监听物品自定义组件的`onConsume`事件，当玩家食用物品时将运行我们的代码。然后我们可以将夜视效果添加到玩家身上，我们的自定义组件现在可以使用了。

```typescript
import { ItemComponentConsumeEvent, world } from '@minecraft/server'

world.beforeEvents.worldInitialize.subscribe((initEvent) => {
  initEvent.itemComponentRegistry.registerCustomComponent('example:add_night_vision_on_consume', {
    onConsume(arg: ItemComponentConsumeEvent) {
      arg.source.addEffect('minecraft:night_vision', 600)
    },
  })
})
```

注意，组件代码和名称并没有引用草莓物品本身。您可以在具有类似行为的多个物品上重复使用组件。

> [!NOTE]
> 截至撰写本文时，自定义组件仍处于预览阶段。因此，您需要在行为包的manifest.json的依赖项部分使用-beta脚本模块。

```JSON
  "dependencies": [
    {
      "module_name": "@minecraft/server",
      "version": "1.13.0-beta"
    }
  ]
```

## 方块自定义组件

方块自定义组件的工作方式与物品自定义组件非常相似。首先我们需要我们的方块定义：

```json
{
  "format_version": "1.21.10",
  "minecraft:block": {
    "description": {
      "identifier": "example:strawberry_crop",
      "states": {
        "starter:crop_age": [0, 1, 2, 3, 4]
      }
    },
    "permutations": [
      {
        "condition": "query.block_state('starter:crop_age') == 4",
        "components": {
          "minecraft:loot": "loot_tables/strawberry_grown_crop.json"
        }
      }
    ],
    "components": {
      "minecraft:geometry": "geometry.starter_crop_geo",
      "minecraft:loot": "loot_tables/strawberry_seed.json",
      "minecraft:collision_box": false,
      "minecraft:placement_filter": {
        "conditions": [
          {
            "allowed_faces": ["up"],
            "block_filter": ["minecraft:farmland"]
          }
        ]
      },
      "tag:minecraft:crop": {}
    }
  }
}
```

### 向方块添加自定义组件

与物品自定义组件类似，我们在方块JSON中使用`minecraft:custom_components`组件来定义此方块具有哪些自定义组件。在这种情况下，我们将添加两个不同的组件定义：一个在方块的基础部分，另一个在排列中。当方块排列和基础方块都使用`minecraft:custom_components`组件时，只有排列中列出的组件将运行其脚本代码。这允许您重新排序、删除或向方块的特定排列添加新的自定义组件。由于草莓作物的前四个排列（年龄0-3）都使用相同的自定义组件，我们可以将其放在基础方块的组件列表中。最后一个排列（年龄4）将具有一些额外的功能，因此我们可以收获完全成熟的草莓，因此它需要自己的`minecraft:custom_components`来覆盖基础方块中的组件。

```json
{
  "format_version": "1.21.10",
  "minecraft:block": {
    "description": {
      "identifier": "example:strawberry_crop",
      "states": {
        "starter:crop_age": [0, 1, 2, 3, 4]
      }
    },
    "permutations": [
      {
        "condition": "query.block_state('example:crop_age') == 4",
        "components": {
          "minecraft:loot": "loot_tables/strawberry_grown_crop.json"
        },
        "minecraft:custom_components": ["example:crop_harvest"]
      }
    ],
    "components": {
      "minecraft:geometry": "geometry.example_crop_geo",
      "minecraft:loot": "loot_tables/strawberry_seed.json",
      "minecraft:collision_box": false,
      "minecraft:placement_filter": {
        "conditions": [
          {
            "allowed_faces": ["up"],
            "block_filter": ["minecraft:farmland"]
          }
        ]
      },
      "tag:minecraft:crop": {},
      "minecraft:custom_components": ["example:crop_grow"]
    }
  }
}
```

### 在脚本中注册方块自定义组件

与物品类似，我们在脚本中注册组件，并提供组件正在监听的事件列表以及事件触发时应运行的行为。对于物品，我们使用`ItemComponentRegistry`来完成此操作；对于方块，我们使用`BlockComponentRegistry`（或在Minecraft 1.21.0或更早版本中使用`BlockTypeRegistry`）。在这种情况下，我们有两个组件需要填写。上面的物品示例展示了如何通过在注册语句中放置行为来完成此操作。这两个组件将展示两种不同的组织代码的方式。

### "example:crop_grow" 组件

此组件旨在通过监听方块自定义组件的`onRandomTick`事件来生长作物，并将方块的排列更改为下一个年龄。此组件通过为事件提供一个运行函数来注册，而之前的物品自定义组件示例在其代码中具有其行为。

```typescript
import { BlockComponentRandomTickEvent, world } from '@minecraft/server'

function cropGrowRandomTick(event: BlockComponentRandomTickEvent) {
  const age = event.block.permutation.getState('example:crop_age')
  if (age === undefined || typeof age !== 'number') {
    return
  } else if (age === 4) {
    return // fully grown
  }

  event.block.setPermutation(event.block.permutation.withState('example:crop_age', age + 1))
}

world.beforeEvents.worldInitialize.subscribe((initEvent) => {
  initEvent.blockTypeRegistry.registerCustomComponent('example:crop_grow', {
    onRandomTick: cropGrowRandomTick,
  })
})
```

### "example:crop_harvest" 组件

作物生长组件仅存在于准备收获的成品作物上。此组件将允许玩家与方块互动以收获草莓，而无需破坏方块。然后它通过将方块排列更改回第一个生长阶段来“重新种植”草莓。此组件通过创建一个实现BlockCustomComponent对象的新类来注册，为您提供第三种注册组件的方式。物品也可以使用相同的方法和ItemCustomComponent对象。

```typescript
import { BlockCustomComponent, BlockComponentPlayerInteractEvent, world } from '@minecraft/server'

class BlockCropHarvestComponent implements BlockCustomComponent {
  onPlayerInteract(event: BlockComponentPlayerInteractEvent) {
    if (event.player === undefined) {
      return
    }

    const blockPos = event.block.location
    event.dimension.runCommand(
      'loot spawn ' +
        blockPos.x +
        ' ' +
        blockPos.y +
        ' ' +
        blockPos.z +
        ' loot strawberry_grown_crop'
    )
    event.block.setPermutation(event.block.permutation.withState('example:crop_age', 0))
  }
}

world.beforeEvents.worldInitialize.subscribe((initEvent) => {
  initEvent.blockTypeRegistry.registerCustomComponent(
    'example:crop_harvest',
    new BlockCropHarvestComponent()
  )
})
```

恭喜！您现在有一个使用自定义组件的方块和物品。请查看我们的[自定义组件示例在Minecraft脚本示例库](https://github.com/microsoft/minecraft-scripting-samples/tree/main/custom-components)。
