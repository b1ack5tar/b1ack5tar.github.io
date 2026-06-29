---
title: "清华校赛 STM32 IoT CTF"
date: 2025-08-28 12:00:00 +0800
categories: [硬件安全, 固件安全]
tags: [STM32, IoT CTF, 固件分析]
description: 以 STM32F103 固件为例，通过 GPIO、UART、USB 与 SWD 完成多项挑战。
---

## 前言

这是我第一次尝试 IoT CTF。题目使用一套自制的 STM32 固件，通过 GPIO、UART、USB 和 SWD 等接口设置了多项挑战。整个过程踩了不少坑，但对于刚接触二进制与固件安全的我来说很有价值，因此记录如下。

题目信息、附件和芯片手册可参考：[STM32 IoT CTF 清华校赛版](https://github.com/xuanxuanblingbling/stm32ctf_thu/tree/master/attachment)。

## 信息收集与环境准备

### 已有资料

- 芯片手册与接线图
- 固件伪代码
- STM32F103C6T6A 及相关硬件

### 启动逻辑

![](/assets/img/posts/thu-stm32-iot-ctf/1.png){: .normal }

根据芯片手册和固件逻辑，可以得到以下信息：

- 固件存储于内部 Flash，BOOT0 接地时，芯片复位后从主 Flash 启动。
- 该固件依赖外部晶振，实验过程中需要保持晶振连接。
- 主函数包含持续运行的循环，各项挑战逻辑会被反复执行。
- `printf` 等函数的输出可以通过 UART 串口接收。

STM32F103C6T6A 型号中各字段的含义如下。

![](/assets/img/posts/thu-stm32-iot-ctf/2.png){: .normal }

## GPIO：读取摩斯电码

固件通过 GPIO 控制 LED 亮灭，并使用闪烁节奏表示 flag 各字符对应的摩斯电码。记录 LED 的亮灭时长，再按照摩斯编码表解码即可得到 flag。

## UART：接收串口输出

最初调试时串口始终无法正常通信。排查后发现，STM32 的各组电源和接地引脚都需要正确连接，并确保目标板、串口模块共地。

完成供电、TX、RX 和 GND 接线后，打开串口工具即可接收固件输出。

![](/assets/img/posts/thu-stm32-iot-ctf/3.png){: .normal }

## USB：分析复合设备

### USB_1：模拟键盘

第一个 USB 挑战将设备模拟为 HID 键盘。相关伪代码如下：

```c
void start_usb() {
    if (!digitalRead(PB6) && !digitalRead(PB7)) {
        while (!USBComposite);
        Keyboard.println("THUCTF{FLAG_USB1}");
        delay(5000);
    }
}
```

将 PB6 和 PB7 接地，等待 USB 设备初始化完成，再把光标放入任意可输入文本的位置，设备便会以键盘输入的形式输出 flag。

### USB_2：寻找另一个 flag

USB 接口中还隐藏了另一个 flag。可以从 USB 设备信息、描述符和相关字符串入手分析。

![](/assets/img/posts/thu-stm32-iot-ctf/4.png){: .normal }

## SWD：动态调试固件

SWD 部分需要结合静态分析和动态调试。首先通过调试接口提取 32 KB 固件，再使用 IDA 分析。目标函数的伪代码如下：

```c
void start_swd() {
    char a[200] = "THUCTF{FLAG_SWD2}";
    delay((int)a % 1000);

    int i;
    int j;
    if ((int)&i == (int)&j + i) {
        Serial1.println("THUCTF{FLAG_SWD1}");
    }
}
```

该函数既在栈上保存了字符串，又包含串口输出，因此调试时需要同时使用 ST-Link 和 UART。由 ST-Link 为目标板供电时，UART 模块只连接 TX、RX 和 GND，避免重复供电。

### 定位目标函数

![](/assets/img/posts/thu-stm32-iot-ctf/5.png){: .normal }

固件函数较多，直接查找目标函数比较困难。伪代码中存在较有辨识度的立即数 `1000`，可以将其转换为十六进制并在 IDA 中搜索交叉引用，从而定位 `start_swd`。

![](/assets/img/posts/thu-stm32-iot-ctf/6.png){: .normal }

定位后即可查看对应的 Thumb 汇编代码。

![](/assets/img/posts/thu-stm32-iot-ctf/7.png){: .normal }

### 查找栈上数组

目标是确定数组 `a` 在 SRAM 中的位置。汇编中存在如下除法指令：

```text
SDIV R0, R5, R3
```

结合前后指令可知，`R3` 保存立即数 `1000`，该计算对应 `delay((int)a % 1000)`。因此可以推断 `R5` 保存数组 `a` 的起始地址，并在调试器中跟随该地址读取 `FLAG_SWD2`。

![](/assets/img/posts/thu-stm32-iot-ctf/8.png){: .normal }

### 绕过条件判断

输出 `FLAG_SWD1` 前存在条件判断，可以通过两种方式绕过：

1. 修改 PC，直接跳转到条件成立后的串口输出分支。
2. 修改参与比较的寄存器，使条件判断结果为真。

关键比较位置如下。

![](/assets/img/posts/thu-stm32-iot-ctf/9.png){: .normal }

![](/assets/img/posts/thu-stm32-iot-ctf/10.png){: .normal }

第二种方法可以在 `0x0800069C` 设置断点，程序暂停后将 `R2` 修改为与 `R3` 相同的值，再继续执行，即可通过串口收到 flag。

## 总结

- 分析 IoT CTF 固件时，应先根据芯片手册确认启动方式、供电要求和外设接口。
- GPIO 和 UART 题目更依赖正确接线与外围设备配置。
- USB 挑战需要理解 HID、描述符和复合设备等基础概念。
- SWD 动态调试需要将伪代码、Thumb 汇编和寄存器状态结合起来分析。
- IDA 反编译结果适合快速梳理逻辑，但定位变量和修改执行流时仍应以汇编为准。
