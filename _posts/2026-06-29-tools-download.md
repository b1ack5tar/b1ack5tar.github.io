---
title: "计算机相关专业常用工具下载"
date: 2026-06-29 18:00:00 +0800
pin: true
categories: [工具]
tags: [工具]
description: 收录计算机相关专业常用工具，方便有需要的读者了解、下载和使用。
image:
  path: /assets/img/posts/tools-download/0.png
---

> 这篇文章把我常用的一些工具整理到一起，方便后续查找、下载和使用。<br>
> 本文会持续更新，后续如果补充新的开发、调试、分析或实验相关工具，也会继续整理到这里。
{: .prompt-info }

## 正点原子串口调试助手

![](/assets/img/posts/tools-download/1.png){: .normal }

点击下载：[正点原子串口调试助手.zip](/assets/files/tools-download/正点原子串口调试助手.zip)

这是一个常见的 Windows 串口调试工具，适合在设备已经枚举成 `COM` 口后，用来查看串口输出、发送测试命令，或者直接收发十六进制数据。

- 支持选择串口号、波特率、数据位、停止位和校验位。
- 支持文本和十六进制模式，适合调试 AT 指令、日志输出或裸字节协议。
- 支持定时发送、时间戳、保存接收窗口等常用功能。

## Zadig

![](/assets/img/posts/tools-download/2.png){: .normal }

点击下载：[zadig-2.9.zip](/assets/files/tools-download/zadig-2.9.zip)

Zadig 用来给 USB 设备安装或替换 `WinUSB`、`libusb-win32`、`libusbK` 等驱动，常见于 `pyusb`、`libusb`、OpenOCD 或其他需要直接访问 USB 设备的场景。

- 当设备没有合适驱动，或者默认驱动不方便用户态程序访问时，可以用它切换到 `WinUSB`。
- 做固件分析、调试开发板、连接某些实验设备时，常常需要先过这一步。
- 更换驱动前最好确认设备名称，避免误选键盘、鼠标、网卡之类的正常外设。
