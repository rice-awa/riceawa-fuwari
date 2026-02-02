---
title: '使用PyInstaller打包Python程序为EXE文件'
published: 2025-06-20
tags: [python, exe, pack]
category: '教程'
draft: false
description: 详细教程：如何使用PyInstaller将Python单文件脚本打包成Windows可执行程序
---

# 使用PyInstaller打包Python程序指南

## 准备工作

1. 准备Python脚本文件（如`main.py`）
2. 可选：准备程序图标文件（`.ico`格式）

## 打包步骤

### 1. 打开命令窗口

在资源管理器中导航到脚本所在目录，在地址栏输入`cmd`回车：
![打开命令窗口](assets/images/pyinstaller/2.png)

### 2.安装PyInstaller

### 基础安装命令

```bash
pip install pyinstaller
```

### 使用国内镜像源加速安装

```bash
pip install -i https://pypi.tuna.tsinghua.edu.cn/simple pyinstaller
```

![安装pyinstaller](assets/images/pyinstaller/3.png)

### 3. 基本打包命令

```bash
pyinstaller -F 你的脚本.py
```

![基本打包](assets/images/pyinstaller/4.png)

### 4. 带图标的打包命令

```bash
pyinstaller -F -i 图标.ico 你的脚本.py
```

### 5. 高级选项

| 参数          | 说明                          |
| ------------- | ----------------------------- |
| `-w`          | 隐藏控制台窗口（GUI程序适用） |
| `--clean`     | 清理临时文件                  |
| `--onefile`   | 单文件打包（与`-F`相同）      |
| `--noconsole` | 不显示控制台窗口              |

## 查找生成文件

打包完成后，在`dist`文件夹中可以找到生成的EXE文件：

## 代码示例

```python
import random

while True:
    print("欢迎来到猜数字游戏！")
    ans = random.randint(1, 100)
    #print(ans)
    times = 0
    max_times = 3

    while True:
        try:
            num = int(input("请输入一个1-100之间的正整数："))
        except ValueError:
            print("输入不合法！请输入一个整数！！！")
            continue

        if num < 1 or num > 100:
            print("你输入的数不在1-100之间！请重新输入！！！")
            continue

        times = times + 1

        if num > ans:
            print(f'第{times}次尝试：大了！')
        elif num < ans:
            print(f'第{times}次尝试：小了！')
        else:
            print(f'恭喜你，猜对了！你总共猜了{times}次')
            break

        if times == max_times:
            print(f'你已用完{max_times}次机会，游戏失败！这个数是：{ans}')
            break

    play_again = input("再玩一次吗？(输入y继续，其他任意键退出)：")
    if play_again.lower() != 'y':
        break

```

## exe示例文件下载

- [无图标](https://wwp.lanzoup.com/iZZ1A2z9jmng) 提取码: 3936
- [有图标](https://wwp.lanzoup.com/iCV7U2z9jlwj) 提取码: i8vn

> [!NOTE]
> 可以使用[图标转换工具](https://convertio.co/zh/)将图片转为ICO格
