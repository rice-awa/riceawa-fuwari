---
title: MCBE WebSockets 完整函数描述文档
published: 2024-08-16
tags: [python, websockets, 我的世界, gpt, ai, doc]
category: '文档'
draft: false
description: 详细介绍对应函数
---

# Minecraft基岩版WebSocket服务器 - 完整函数文档

本文档详细介绍了Minecraft基岩版WebSocket服务器项目中的所有函数。

---

## 目录

1. [总述](#总述)
2. [核心功能函数](#核心功能函数)
   - [`main()`](#main)
   - [`periodic_update()`](#periodic_update)
   - [`handle_connection(websocket, path)`](#handle_connectionwebsocket-path)
   - [`handle_event(websocket, data, conversation)`](#handle_eventwebsocket-data-conversation)
3. [消息处理函数](#消息处理函数)
   - [`handle_player_message(websocket, data, conversation)`](#handle_player_messagewebsocket-data-conversation)
   - [`handle_command_response(websocket, data)`](#handle_command_responsewebsocket-data)
   - [`handle_event_message(websocket, data)`](#handle_event_messagewebsocket-data)
4. [GPT相关函数](#gpt相关函数)
   - [`gpt_main(conversation, player_prompt)`](#gpt_mainconversation-player_prompt)
   - [`handle_gpt_chat(websocket, content, conversation)`](#handle_gpt_chatwebsocket-content-conversation)
   - [`handle_gpt_script(websocket, content, conversation)`](#handle_gpt_scriptwebsocket-content-conversation)
   - [`handle_gpt_save(websocket, conversation)`](#handle_gpt_savewebsocket-conversation)
5. [命令执行函数](#命令执行函数)
   - [`run_command(websocket, command)`](#run_commandwebsocket-command)
   - [`handle_run_command(websocket, content)`](#handle_run_commandwebsocket-content)
   - [`handle_script_run_command(websocket, content)`](#handle_script_run_commandwebsocket-content)
   - [`send_script_data(websocket, content, messageid="server:data")`](#send_script_datawebsocket-content-messageidserverdata)
6. [游戏信息函数](#游戏信息函数)
   - [`gpt_game_weather(websocket, dimension)`](#gpt_game_weatherwebsocket-dimension)
   - [`gpt_game_players(websocket)`](#gpt_game_playerswebsocket)
   - [`gpt_get_time(websocket, dimension)`](#gpt_get_timewebsocket-dimension)
   - [`gpt_run_command(websocket, commands)`](#gpt_run_commandwebsocket-commands)
   - [`gpt_world_entity(websocket, entityid)`](#gpt_world_entitywebsocket-entityid)
   - [`gpt_player_inventory(websocket, player_name=None)`](#gpt_player_inventorywebsocket-player_namenone)
   - [`gpt_get_commandlog(websocket)`](#gpt_get_commandlogwebsocket)
7. [实用函数](#实用函数)
   - [`send_data(websocket, message)`](#send_datawebsocket-message)
   - [`send_game_message(websocket, message)`](#send_game_messagewebsocket-message)
   - [`subscribe_events(websocket)`](#subscribe_eventswebsocket)
   - [`parse_message(message)`](#parse_messagemessage)
   - [`handle_data_part(message, connection_uuid, data_type, other_message=None)`](#handle_data_partmessage-connection_uuid-data_type-other_messagenone)
   - [`handle_display_command_log(websocket)`](#handle_display_command_logwebsocket)
   - [`clear_old_data(websocket, connection_uuid)`](#clear_old_datawebsocket-connection_uuid)
   - [`get_game_information(websocket, connection_uuid)`](#get_game_informationwebsocket-connection_uuid)
8. [认证函数](#认证函数)
   - [`auth.verify_password(content)`](#authverify_passwordcontent)
   - [`auth.generate_token()`](#authgenerate_token)
   - [`auth.save_token(connection_uuid, token)`](#authsave_tokenconnection_uuid-token)
   - [`auth.verify_token(token)`](#authverify_tokentoken)
   - [`auth.get_stored_token(connection_uuid)`](#authget_stored_tokenconnection_uuid)
   - [`auth.is_token_valid(connection_uuid)`](#authis_token_validconnection_uuid)

---

## 总述

这个项目实现了一个Minecraft基岩版（MCBE）的WebSocket服务器，旨在通过WebSocket连接与游戏客户端通信，并利用GPT模型提供智能对话和命令执行功能。服务器能够处理来自游戏的各种事件和命令，并返回相应的游戏信息。项目的核心功能包括处理WebSocket连接、订阅游戏事件、执行游戏命令、与GPT模型交互以及管理游戏状态信息。

---

## 核心功能函数

### `main()`

- **描述**：服务器的主入口点，启动WebSocket服务器并设置周期性更新。
- **逻辑**：
  1. 启动WebSocket服务器，监听指定的IP和端口。
  2. 调用`periodic_update()`函数定期更新游戏信息。

### `periodic_update()`

- **描述**：定期更新所有已连接客户端的游戏信息。
- **逻辑**：
  1. 每隔10秒清理旧数据并获取新的游戏信息。
  2. 调用`clear_old_data()`和`get_game_information()`函数。

### `handle_connection(websocket, path)`

- **描述**：处理新的WebSocket连接。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `path`: 连接路径
- **逻辑**：
  1. 为新连接生成UUID并初始化相关信息。
  2. 发送欢迎消息。
  3. 订阅游戏事件。
  4. 处理来自客户端的消息并调用相应的处理函数。

### `handle_event(websocket, data, conversation)`

- **描述**：处理来自游戏的传入事件。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `data`: 事件数据
  - `conversation`: GPT对话对象
- **逻辑**：
  1. 根据事件类型调用相应的处理函数，如`handle_player_message()`、`handle_command_response()`和`handle_event_message()`。

---

## 消息处理函数

### `handle_player_message(websocket, data, conversation)`

- **描述**：处理玩家消息并执行相应的命令。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `data`: 消息数据
  - `conversation`: GPT对话对象
- **逻辑**：
  1. 解析玩家消息以提取命令和内容。
  2. 根据命令调用相应的处理函数，如登录验证、GPT聊天、执行命令等。

### `handle_command_response(websocket, data)`

- **描述**：处理命令响应。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `data`: 响应数据
- **逻辑**：
  1. 解析命令响应并更新游戏状态信息。
  2. 记录命令日志。

### `handle_event_message(websocket, data)`

- **描述**：处理事件消息，如玩家位置更新。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `data`: 事件数据
- **逻辑**：
  1. 解析事件数据并更新玩家位置信息。

---

## GPT相关函数

### `gpt_main(conversation, player_prompt)`

- **描述**：向GPT模型发送提示并返回响应。
- **参数**：
  - `conversation`: GPT对话对象
  - `player_prompt`: 用户的输入提示
- **逻辑**：
  1. 调用GPT模型生成响应。
  2. 返回GPT的响应消息。

### `handle_gpt_chat(websocket, content, conversation)`

- **描述**：处理GPT聊天请求。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `content`: 聊天内容
  - `conversation`: GPT对话对象
- **逻辑**：
  1. 调用`gpt_main()`生成GPT响应。
  2. 发送GPT响应到游戏客户端。

### `handle_gpt_script(websocket, content, conversation)`

- **描述**：处理GPT脚本请求。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `content`: 脚本内容
  - `conversation`: GPT对话对象
- **逻辑**：
  1. 调用`gpt_main()`生成GPT响应。
  2. 发送脚本数据到游戏客户端。

### `handle_gpt_save(websocket, conversation)`

- **描述**：保存GPT对话并重启对话。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `conversation`: GPT对话对象
- **逻辑**：
  1. 保存当前对话状态。
  2. 重启对话并通知客户端。

---

## 命令执行函数

### `run_command(websocket, command)`

- **描述**：执行游戏命令。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `command`: 要执行的命令
- **逻辑**：
  1. 构建命令消息并发送到游戏客户端。
  2. 记录待响应的命令。

### `handle_run_command(websocket, content)`

- **描述**：处理运行命令请求。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `content`: 命令内容
- **逻辑**：
  1. 调用`run_command()`执行命令。

### `handle_script_run_command(websocket, content)`

- **描述**：处理脚本运行命令请求。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `content`: 脚本命令内容
- **逻辑**：
  1. 调用`send_script_data()`发送脚本命令。

### `send_script_data(websocket, content, messageid="server:data")`

- **描述**：向游戏发送脚本数据。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `content`: 脚本内容
  - `messageid`: 消息ID（默认为"server:data"）
- **逻辑**：
  1. 构建脚本消息并发送到游戏客户端。

---

## 游戏信息函数

### `gpt_game_weather(websocket, dimension)`

- **描述**：检索指定维度的当前天气。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `dimension`: 游戏维度
- **逻辑**：
  1. 获取并返回当前维度的天气信息。

### `gpt_game_players(websocket)`

- **描述**：检索游戏中所有玩家的信息。
- **参数**：
  - `websocket`: WebSocket连接对象
- **逻辑**：
  1. 获取并返回所有玩家的信息。

### `gpt_get_time(websocket, dimension)`

- **描述**：检索当前游戏时间和天数。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `dimension`: 游戏维度
- **逻辑**：
  1. 获取并返回当前游戏时间和天数。

### `gpt_run_command(websocket, commands)`

- **描述**：使用gpt智能运行命令。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `commands`: 要执行的命令列表
- **逻辑**：
  1. 调用`run_command()`执行命令。

### `gpt_world_entity(websocket, entityid)`

- **描述**：检索特定实体的信息。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `entityid`: 实体ID
- **逻辑**：
  1. 获取并返回特定实体的信息。

### `gpt_player_inventory(websocket, player_name=None)`

- **描述**：检索特定玩家或所有玩家的物品栏。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `player_name`: 玩家名称（可选）
- **逻辑**：
  1. 获取并返回特定玩家或所有玩家的物品栏信息。

### `gpt_get_commandlog(websocket)`

- **描述**：检索命令执行日志。
- **参数**：
  - `websocket`: WebSocket连接对象
- **逻辑**：
  1. 获取并返回命令执行日志。

---

## 实用函数

### `send_data(websocket, message)`

- **描述**：通过WebSocket连接发送数据。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `message`: 要发送的消息
- **逻辑**：
  1. 将消息转换为JSON格式并发送到WebSocket连接。

### `send_game_message(websocket, message)`

- **描述**：向游戏聊天发送格式化消息。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `message`: 要发送的消息
- **逻辑**：
  1. 格式化消息并发送到游戏聊天。

### `subscribe_events(websocket)`

- **描述**：订阅游戏事件。
- **参数**：
  - `websocket`: WebSocket连接对象
- **逻辑**：
  1. 构建订阅消息并发送到游戏客户端。

### `parse_message(message)`

- **描述**：解析玩家消息以提取命令和内容。
- **参数**：
  - `message`: 来自玩家的原始消息
- **逻辑**：
  1. 检查消息中是否包含预定义命令。
  2. 返回命令和内容。

### `handle_data_part(message, connection_uuid, data_type, other_message=None)`

- **描述**：处理来自游戏的部分数据消息。
- **参数**：
  - `message`: 部分数据消息
  - `connection_uuid`: 连接UUID
  - `data_type`: 正在接收的数据类型
  - `other_message`: 附加消息信息（可选）
- **逻辑**：
  1. 解析数据部分并存储。
  2. 检查是否接收到所有数据部分，若是则组合并解析完整数据。

### `handle_display_command_log(websocket)`

- **描述**：显示命令日志。
- **参数**：
  - `websocket`: WebSocket连接对象
- **逻辑**：
  1. 获取并显示命令执行日志。

### `clear_old_data(websocket, connection_uuid)`

- **描述**：清理旧的游戏信息数据。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `connection_uuid`: 连接UUID
- **逻辑**：
  1. 清理GameInformation对象中的临时数据。

### `get_game_information(websocket, connection_uuid)`

- **描述**：获取游戏信息。
- **参数**：
  - `websocket`: WebSocket连接对象
  - `connection_uuid`: 连接UUID
- **逻辑**：
  1. 发送游戏信息查询命令。
  2. 更新游戏信息。

---

## 认证函数

### `auth.verify_password(content)`

- **描述**：验证登录密码。
- **参数**：
  - `content`: 要验证的密码
- **逻辑**：
  1. 检查密码是否正确。

### `auth.generate_token()`

- **描述**：生成新的认证令牌。
- **逻辑**：
  1. 生成新的随机令牌。

### `auth.save_token(connection_uuid, token)`

- **描述**：为连接保存认证令牌。
- **参数**：
  - `connection_uuid`: 连接UUID
  - `token`: 认证令牌
- **逻辑**：
  1. 将令牌与连接UUID关联并存储。

### `auth.verify_token(token)`

- **描述**：验证认证令牌。
- **参数**：
  - `token`: 要验证的令牌
- **逻辑**：
  1. 检查令牌是否有效。

### `auth.get_stored_token(connection_uuid)`

- **描述**：获取存储的认证令牌。
- **参数**：
  - `connection_uuid`: 连接UUID
- **逻辑**：
  1. 返回存储的令牌。

### `auth.is_token_valid(connection_uuid)`

- **描述**：检查令牌是否有效。
- **参数**：
  - `connection_uuid`: 连接UUID
- **逻辑**：
  1. 检查与连接UUID关联的令牌是否有效。

---
