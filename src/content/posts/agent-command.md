---
title: 'MCBE Agent Command'
published: 2024-08-28
tags: [minecraft, agent, MCBE, command]
category: '教程'
draft: false
description: 介绍MCBE Agent 命令的语法、参数、用法。
---

# MCBE Agent Command
- 介绍MCBE Agent 命令的语法、参数、用法。

## 语法

agent [command: AgentDirectionCommand](#command-agentdirectioncommand) [direction: AgentDirection](#direction-agentdirection)

agent [command: AgentItemCommand](#command-agentitemcommand) [slotNum: int](#slotnum-int)

agent [command: AgentCommand](#command-agentcommand)

agent collect [item: Item](#item-item)

agent collect all

agent drop [slotNum: int](#slotnum-int) [quantity: int](#quantity-int-和-count-int) [direction: AgentDirection](#direction-agentdirection)

agent place [slotNum: int](#slotnum-int) [direction: AgentDirection](#direction-agentdirection)

agent setitem [slotNum: int](#slotnum-int) [item: Item](#item-item) [count: int](#quantity-int-和-count-int) [aux: int](#aux-int)

agent tp [destination: x y z] [y-rot: float]

agent tp [destination: x y z](#destination-x-y-z) facing [lookAtPosition: x y z](#lookatposition-x-y-z)

agent transfer [slotNum: int](#slotnum-int) [quantity: int](#quantity-int-和-count-int) [dstSlotNum: int](#dstslotnum-int)

agent turn [direction: AgentTurnDirection](#direction-agentturndirection)

## 参数

- ### **command: AgentDirectionCommand**

  - 包括 `move` | `destroy` | `detect` | `attack` | `detectredstone` | `dropall` | `inspect` | `inspectdata` | `interact` | `till`
  - 用于命令：`agent <command: AgentDirectionCommand> <direction: AgentDirection>`

- ### **command: AgentItemCommand**

  - 包括 `getitemcount` | `getitemspace` | `getitemdetail`
  - 用于命令：`agent <command: AgentItemCommand> <slotNum: int>`

- ### **command: AgentCommand**

  - 包括 `create` | `getposition`
  - 用于命令：`agent <command: AgentCommand>`

- ### **direction: AgentDirection**

  - 包括 `up` | `down` | `forward` | `back` | `left` | `right`
  - 用于命令：`agent <command: AgentDirectionCommand> <direction: AgentDirection>`，`agent drop <slotNum: int> <quantity: int> <direction: AgentDirection>`，`agent place <slotNum: int> <direction: AgentDirection>`

- ### **direction: AgentTurnDirection**

  - 包括 `left` | `right`
  - 用于命令：`agent turn <direction: AgentTurnDirection>`

- ### **slotNum: int**

  - 指定要改变的智能体物品栏槽位。有效值为0至26的整数。
  - 用于命令：`agent <command: AgentItemCommand> <slotNum: int>`，`agent drop <slotNum: int> <quantity: int> <direction: AgentDirection>`，`agent place <slotNum: int> <direction: AgentDirection>`，`agent setitem <slotNum: int> <item: Item> <count: int> <aux: int>`，`agent transfer <slotNum: int> <quantity: int> <dstSlotNum: int>`

- ### **item: Item**

  - 指定要给予或收集到物品栏槽位内的物品。必须为物品ID，或具有物品形态的方块的ID。
  - 用于命令：`agent collect <item: Item>`，`agent setitem <slotNum: int> <item: Item> <count: int> <aux: int>`

- ### **quantity: int** 和 **count: int**

  - 指定被丢出、给予或移动的物品数量。
  - 用于命令：`agent drop <slotNum: int> <quantity: int> <direction: AgentDirection>`，`agent setitem <slotNum: int> <item: Item> <count: int> <aux: int>`，`agent transfer <slotNum: int> <quantity: int> <dstSlotNum: int>`

- ### **aux: int**

  - 指定参与命令的物品的数据值。
  - 用于命令：`agent setitem <slotNum: int> <item: Item> <count: int> <aux: int>`

- ### **destination: x y z**

  - 指定要被传送到的坐标。必须为三维坐标，元素为单精度浮点数。允许相对坐标（`~ ~ ~`）或局部坐标（`^ ^ ^`）。
  - 用于命令：`agent tp [destination: x y z] [y-rot: float]`，`agent tp <destination: x y z> facing <lookAtPosition: x y z>`

- ### **y-rot: float**

  - 指定传送后智能体的水平旋转角度。
  - 用于命令：`agent tp [destination: x y z] [y-rot: float]`

- ### **lookAtPosition: x y z**

  - 指定传送后智能体朝向的位置。必须为三维坐标，元素为单精度浮点数。允许相对坐标（`~ ~ ~`）或局部坐标（`^ ^ ^`）。
  - 用于命令：`agent tp <destination: x y z> facing <lookAtPosition: x y z>`

- ### **dstSlotNum: int**
  - 指定目标物品栏槽位。
  - 用于命令：`agent transfer <slotNum: int> <quantity: int> <dstSlotNum: int>`
