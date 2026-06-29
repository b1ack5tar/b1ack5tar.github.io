---
title: "PANDA 2025 IoT CTF：Badge Pwn 题解"
date: 2025-09-04 12:00:00 +0800
categories: [网络安全, Pwn]
tags: [STM32, IoT CTF, 栈溢出, 固件分析]
description: 利用 STM32 固件中的栈溢出漏洞跳转后门，并通过 SRAM Shellcode 读取 RDP 保护下的 Flash 数据。
---

## 题目简介

本题来自纽创科技举办的 PANDA 2025 IoT CTF，题目介绍与官方 WP 可参考：[Welcome to PANDA 2025](https://mp.weixin.qq.com/s/QwZVovfwI7JmVRLdaDfcRA)。比赛提供了一块支持 SWD 与 UART 复用的开发板，以及固件文件 [badge.elf](https://gitee.com/yichen115/badge)。

固件的串口输出采用 GBK 编码。测试多个串口工具后，正点原子串口调试助手可以正常显示中文。启动信息如下：

```text
★☆★ Welcome to PANDA 2025! ★☆★
 PA6  -> 常亮，这是挑战起点！
 PC4  -> 您可以在 SRAM 找到 flag 点亮
 PC5  -> 做完《拆弹专家》挑战后由工作人员点亮
 PB0  -> 通过侧信道/故障等方式解出 AES KEY 后以此格式提交：flag{AES KEY} KEY 为全小写
 PB1  -> 通过 VUL 功能漏洞调用后门函数点亮该 LED
 PB2  -> 绕过 STM32 读保护，在 Flash 地址 0x08030030 处读取 flag
 PB10 -> 常亮，这里是挑战的终点哦！
 PB11 -> 进行 AES 运算前后会有拉高拉低，可以此作为触发进行侧信道/故障

===========================
 [+]输入 LED 进入点灯环节
 [+]输入 AES 进行 AES 运算
 [+]输入 VUL 调用漏洞函数
 >
```

> 本文只讨论 PB1 和 PB2，即两个与固件漏洞利用相关的目标。
{: .prompt-info }

## PB1：栈溢出跳转后门

### 漏洞定位

从 `main` 函数进入 `vul` 函数。

![](/assets/img/posts/panda-2025-badge-pwn/1.png){: .normal }

反编译结果显示，程序会将输入的 300 字节数据复制到容量不足的栈空间中，导致栈溢出。

![](/assets/img/posts/panda-2025-badge-pwn/2.png){: .normal }

利用目标是覆盖函数返回地址，使程序跳转到点亮 PB1 的后门函数。

![](/assets/img/posts/panda-2025-badge-pwn/3.png){: .normal }

![](/assets/img/posts/panda-2025-badge-pwn/4.png){: .normal }

后门函数的实际入口地址为 `0x0800552C`。由于 Cortex-M 只执行 Thumb 指令，写入 PC 的目标地址最低位必须为 `1`，因此 payload 中应使用 `0x0800552D = 0x0800552C | 1`：

> 地址最低位用于指示 Thumb 状态，并不属于实际取指地址。如果直接跳转到偶地址 `0x0800552C`，处理器会触发异常。
{: .prompt-warning }

### 分析函数栈帧

常见的 x86-64 函数会使用 `rbp` 建立栈帧，并通过 `ret` 从栈中恢复返回地址：

```text
push rbp
mov  rbp, rsp
sub  rsp, 0x20
...
leave
ret
```

但该固件运行于 Cortex-M，不能直接套用 x86 的栈帧结构。观察 `vul` 的汇编代码可以发现，函数没有建立传统的 `rbp` 栈帧，而是通过 `push` 保存通用寄存器和  lr 寄存器。

![](/assets/img/posts/panda-2025-badge-pwn/5.png){: .normal }

结合调用位置继续分析，可以确认反编译器显示的部分变量实际与调用者准备的栈空间有关。

![](/assets/img/posts/panda-2025-badge-pwn/6.png){: .normal }

函数返回时，保存的寄存器依次恢复，原 `lr` 对应的数据最终被弹入 `pc`。因此只需确定输入缓冲区到该保存值之间的偏移，就能控制程序执行流。

### 构造 payload

如果串口工具以字符串形式发送 `VUL`，实际数据可能为 `VUL\r\n`，即 5 字节，不利于后续数据按 4 字节对齐。可以改用十六进制发送：

![](/assets/img/posts/panda-2025-badge-pwn/7.png){: .normal }

根据汇编中的寄存器保存顺序和动态调试结果，需要先覆盖保存的 `r1` 至 `r5`，再将保存的返回地址覆盖为 `0x0800552D`，最后将输入补足至程序要求的 300 字节。

![](/assets/img/posts/panda-2025-badge-pwn/8.png){: .normal }

payload 执行后，程序跳转至后门函数并点亮 PB1。

## PB2：利用 Shellcode 读取受保护 Flash

### 利用思路

目标芯片开启 RDP Level 1 后，调试器无法直接读取 Flash，但芯片内部代码仍可访问相应数据。因此可以继续利用 `vul` 函数：

1. 将自定义 Shellcode 写入 SRAM。
2. 覆盖返回地址，使程序跳转至 SRAM 执行 Shellcode。
3. 调用固件现有的 `memcpy`，将 `0x08030030` 处的数据复制到可查看的 SRAM 区域。
4. 通过题目环境查看 SRAM，获得目标数据。

### 确定地址与参数

![](/assets/img/posts/panda-2025-badge-pwn/9.png){: .normal }

调试结果显示，保存的返回地址位于 `0x2000088C`，因此可以从相邻的 `0x20000890` 开始存放 Shellcode。跳转地址同样需要设置 Thumb 位，所以 payload 中使用 `0x20000891`。

调用 `memcpy` 时，各寄存器参数如下：

| 寄存器 | 数值 | 含义 |
| --- | --- | --- |
| `r0` | `0x20001000` | SRAM 目标地址 |
| `r1` | `0x08030030` | Flash 中的 flag 地址 |
| `r2` | `0x100` | 复制长度 |
| `r3` | `0x08000243` | `memcpy` 的 Thumb 入口地址 |

![](/assets/img/posts/panda-2025-badge-pwn/10.png){: .normal }

Shellcode 的主要逻辑是加载上述参数，通过 `blx r3` 调用 `memcpy`，随后进入原地循环，便于查看 SRAM 中的复制结果。

![](/assets/img/posts/panda-2025-badge-pwn/11.png){: .normal }

![](/assets/img/posts/panda-2025-badge-pwn/12.png){: .normal }

### 完成利用

payload 先覆盖保存的通用寄存器，再将返回地址设置为 `0x20000891`，随后紧接 Shellcode，并使用 `0x43` 补足 300 字节。

执行后，可以在 `0x20001000` 处看到从 Flash 复制到 SRAM 的 flag。

![](/assets/img/posts/panda-2025-badge-pwn/13.png){: .normal }

按照同样的方法扩大读取范围，理论上也可以分段提取更多受保护的固件内容。

## 总结

- 分析 Cortex-M 固件时，不能直接套用 x86 的函数栈帧模型，必须结合调用约定和汇编指令确认覆盖偏移。
- 跳转到 Thumb 代码时，需要将目标地址最低位置 `1`。
- RDP 会阻止外部调试器直接读取 Flash，但无法阻止合法运行的内部代码访问自身数据。
- 反编译结果便于快速理解逻辑，最终的漏洞利用仍应以汇编和动态调试结果为准。
