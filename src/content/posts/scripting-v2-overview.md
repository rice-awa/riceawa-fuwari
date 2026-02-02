---
title: '脚本引擎 V2.0.0 概述'
published: 2025-01-29
tags: [minecraft, Scripting]
category: '翻译'
draft: false
author: rice_awa
description: '描述脚本 API 模块版本控制的基本原理。'
---

> [!NOTE]
> 原文：[微软文档](https://learn.microsoft.com/en-us/minecraft/creator/documents/scriptingv2.0.0overview?view=minecraft-bedrock-stable#custom-components-v2)
>
> 日期：2025 年 1 月 29 日
>
> 翻译：[rice_awa](https://github.com/rice-awa)
>
> [源项目许可证](https://github.com/MicrosoftDocs/minecraft-creator?tab=MIT-2-ov-file)

# 脚本引擎 V2.0.0 概述

我们还提供了脚本API v2.0.0-beta概述的视频版本：

> [!VIDEO https://www.youtube.com/embed/owfBDnOHI_o]

## 脚本API v2.0.0的新特性

通常，当你听说喜爱软件的"第二版"时，总会充满期待。一个全新的...事物！一切都不同了，而且应该会更好！但关于脚本API v2.0.0首先要明白的是：它与Minecraft脚本v1.x.0版本的区别并没有想象中那么大。请不要误解：我们对脚本API v2.0.0的基础架构改进感到兴奋，但也希望您能认同它与之前版本差异并不算太大。

您看，脚本版本[遵循语义化版本规则](https://learn.microsoft.com/en-us/minecraft/creator/documents/scriptversioning?view=minecraft-bedrock-stable)。任何破坏向后兼容性的变更都需要升级主版本号。这就是脚本API v2.0.0的意义所在：它修复了API结构中的一些问题，这些改进虽然有益，但如果将针对v1.0.0编写的代码升级到v2.0.0，可能会导致原有代码失效。我们会周期性地推出修复问题和改进功能的新脚本版本，但这些版本可能不完全兼容当前的API"契约"。因此，您应该预期Minecraft脚本API命名空间会有主版本升级；例如，脚本API 3.0.0版本可能在不远的将来就会到来。

只要您在`manifest.json`中指定使用Minecraft API的1.x.0版本，所有针对v1.0.0编写的现有脚本和体验内容都能继续正常工作。未来多年Minecraft都会继续支持这些版本。它们会在最大限度保持1.0.0语义兼容的环境中运行，您的脚本无需任何修改。也就是说，当2.0.0版本结束测试进入稳定阶段后，建议将新项目或进行中的项目迁移到更现代的脚本2.0.0环境，特别是那些仅面向2.0.0的新API发布时。

> [!IMPORTANT]
> 脚本V2.0.0 API目前处于实验预览阶段，需要使用"Beta API"实验性功能。在稳定并正式发布前，这些API可能还会有所调整。

以下是脚本API 2.0的主要变化：

### 脚本环境更早的执行与加载

当前的脚本1.0.0环境是在Minecraft加载完部分其他基础架构（如实体定义等）后才加载到世界中的。即便如此，在脚本1.0.0中，脚本逻辑的首次执行通常发生在第一个区块加载之前，因此环境仍处于半加载状态。通常需要等待后续周期——比如经过若干游戏刻后，或玩家加入时，或方块可用时——才执行附加包的"真正"初始化逻辑。

![世界加载流程](../../assets/images/worldload.png)

在脚本v2.0.0中，脚本初始化的首次执行时间被提前到世界启动和加载的更早阶段。此时大多数API——甚至是像获取世界游戏模式这样的简单属性查询——都尚未就绪。此举是为了让脚本能在其他内容加载前配置服务器和世界的初始化部分。例如，即将推出的自定义组件v2功能现在会在加载方块JSON文件前通过脚本注册，从而在方块JSON文件出错时提供更好的错误信息。

为了让行为更可预测，我们为API添加了更多保护措施，防止在未加载状态下被调用（早期执行特权）。同时还更新了一系列事件（新增`worldLoad`事件和`startup`事件），以便您可以在完全加载后运行代码。

### Promise解析机制的变更

在脚本V2.0.0中，promise现在可以在游戏刻结束时与after事件和系统任务一起解析。在之前的脚本版本中，promise每个游戏刻只解析一次。这项变更将使promise在其等待的操作完成后更频繁、更即时地得到解析。

#### 脚本V2.0.0的刷新顺序

在V1脚本中，系统会在每个游戏刻结束时持续刷新after事件和系统任务，直到脚本看门狗超时或没有待处理任务。在V2版本中，promise解析改为持续刷新机制。promise将单独解析，同时也会在每个after事件类型和每个系统任务执行后解析。

![脚本刻处理流程](../../assets/images/tickprocessing.png)

总的来说，V1和V2脚本的新刷新顺序如下：

- 在游戏刻结束时解析V1 promise一次
- 持续刷新直到没有待处理任务
  - 解析V2 promise
  - 运行系统任务（V1和V2）
    - 每个任务执行后解析V2 promise
  - 运行after事件（V1和V2）
    - 每个事件类型执行后解析V2 promise

#### 示例

```typescript
new Promise<void>(resolve => {
    resolve();
}).then(_ => {
    console.error('Promise已解析');
});
```

在V1中，上述示例会在下一游戏刻打印日志。而在V2中，promise会在同一游戏刻内刷新，这意味着会在当前游戏刻打印日志。

```typescript
await system.waitTicks(1);
await system.waitTicks(0); // 在v1中不可行
```

V2还新增了用`waitTicks`函数等待0刻的能力。之前在V1中，由于promise要到下一游戏刻才会解析，所以无法等待0刻。现在随着promise的即时刷新，可以等待0刻并在当前刻执行。

### 其他API层面的变更

除了脚本加载和promise解析行为的基础架构变更外，还有几个API发生了变化。查看变化的最佳方式是使用`2.0.0-beta`的TypeScript定义（例如运行`npm i @minecraft/server@2.0.0-beta`），然后在您选择的代码编辑器中查看变化。

- `实体`：
  - `applyKnockback`方法现在接受表示击退水平力的`VectorXZ`参数（包含强度/大小）和垂直强度参数。要从V1转换，您应该规范化之前的方向向量并乘以旧的水平强度值。垂直强度与之前相同。

- `维度`：
  - 移除了`runCommandAsync`，因为大多数命令实际上不是异步运行的。如需异步运行函数，请考虑通过`System.runJob`使用Jobs系统。

- `实体组件`：
  - `getComponents`、`getComponent`和`hasComponent`现在会在实体无效时抛出异常
  - `EntityComponent.getEntity`方法在底层实体无效时会抛出异常（之前是返回undefined）
  - `EntityInventoryComponent.container`属性在底层实体无效时会抛出异常（之前是返回undefined）

- 多个类上的`isValid`方法已改为只读属性

- `效果类型`：
  - getName方法现在总是返回带minecraft:命名空间前缀的名称

- `效果`：
  - typeId属性现在总是返回带minecraft:命名空间前缀的名称
  
- `minecraft:air`物品已被移除（但它仍是有效的方块）

### 自定义组件V2

"自定义组件V2"是一项新实验功能，需要同时启用"Beta API"实验功能才能使用。启用实验后：

- `minecraft:custom_components`已被弃用，改用扁平化自定义组件
- 自定义组件现在支持参数

#### 扁平化

在之前版本中，自定义组件必须列在`minecraft:custom_components`组件内。现在不再需要这样做，`minecraft:custom_components`组件已被弃用。您可以像其他Minecraft组件一样编写自定义组件。例如：

```json
{
    "components": {
        "minecraft:loot": "...",
        "minecraft:collision_box": {
            "enabled": true
        },
        "my_custom_component:name": {},
        "my_custom_component:another_component": {}
    }
}
```

#### 参数

除了在JSON中扁平化自定义组件外，您现在还可以为组件提供参数。自定义组件的脚本绑定已升级，支持第二个参数`CustomComponentParameters`，用于访问组件的JSON参数列表。以下示例展示了如何在脚本中使用自定义组件参数：

```json
{
    "components": {
        "some_component:name": {
            "first": "hello",
            "second": 4,
            "third": [
                "test",
                "example"
            ]
        }
    }
}
```

```typescript
type SomeComponentParams = {
    first?: string;
    second?: number;
    third?: string[];
};

system.beforeEvents.startup.subscribe(init => {
    init.blockComponentRegistry.registerCustomComponent('some_component:name', {
        onStepOn: (e : BlockComponentStepOnEvent, p : CustomComponentParameters) : {
            let params = p.params as SomeComponentParams;
            ...
        }
    });
});
```

## 升级到脚本V2.0.0

#### 启动事件

`world.afterEvents.worldInitialize`的用法应改为`world.afterEvents.worldLoad`，无需额外修改。该事件在脚本v2.0.0中进行了重命名。

`world.beforeEvents.worldInitialize`的用法应改为`system.beforeEvents.startup`。`worldInitialize`前置事件已被移除，取而代之的是新的`startup`事件。但新的`startup`事件同样在早期执行阶段运行。这意味着之前在`worldInitialize`中允许的部分代码在`startup`中将不再有效。诸如获取玩家、访问游戏中的角色和方块等操作，应移至`world.afterEvents.worldLoad`事件中。

#### `world`对象

早期执行带来的最大变化是，如果在脚本环境的早期执行阶段调用`world`对象的大多数方法会导致错误。订阅事件仍然有效，因为这不会访问世界状态，但任何与世界状态、实体、方块、玩家交互的操作都会受到限制，直到第一个游戏刻开始。

#### 脚本启动流程

为了向后兼容，脚本V1.x.x的启动时间和能力没有修改。脚本v1.x.x不使用早期执行，会保持当前的时间点运行。服务器加载脚本的一般流程如下：

- 加载并运行V2脚本（早期执行）
- V2脚本在早期执行阶段接收`system.beforeEvents.startup`事件
- 等待世界完成加载和游戏启动...
- 加载并运行V1脚本
- V1脚本接收`world.beforeEvents.worldInitialize`事件
- 第一个游戏刻开始...
- 在游戏刻结束时，所有脚本接收`world.afterEvents.worldLoad`事件（V1中称为`worldInitialize`）

#### 早期执行阶段可用的API有哪些？

以下是脚本v2.0.0-beta在早期执行模式下最初可用的API：

- `world.beforeEvents.*.subscribe`
- `world.beforeEvents.*.unsubscribe`
- `world.afterEvents.*.subscribe`
- `world.afterEvents.*.unsubscribe`
- `system.afterEvents.*.subscribe`
- `system.afterEvents.*.unsubscribe`
- `system.beforeEvents.*.subscribe`
- `system.beforeEvents.*.unsubscribe`
- `system.clearJob`
- `system.clearRun`
- `system.run`
- `system.runInterval`
- `system.runJob`
- `system.runTimeout`
- `system.waitTicks`
- `BlockComponentRegistry.registerCustomComponent`
- `ItemComponentRegistry.registerCustomComponent`

#### 如何处理脚本根上下文中不支持早期执行的代码？

如果您有代码在脚本文件的根上下文中使用API，需要将其延迟到`world.afterEvents.worldLoad`事件期间或之后执行。有多种代码组织方式可以实现这一点。使用类或函数可以帮助组织各个系统的启动逻辑，这些逻辑可以在事件回调中调用。可以创建惰性getter来调用API并缓存结果，但只在getter被调用后执行。在许多情况下，您只需将"根上下文"脚本包装在`world.afterEvents.worldLoad`调用中即可。

## 总结

就是这样！请继续关注2.0.0-beta API的更多更新；在接下来的几周里，我们还会在一些地方更改某些方法和属性的签名。一如既往，我们非常重视您在2.0.0测试期间的反馈和错误报告。您可以在[bugs.mojang.com](https://bugs.mojang.com/projects/MCPE/summary)提交问题。