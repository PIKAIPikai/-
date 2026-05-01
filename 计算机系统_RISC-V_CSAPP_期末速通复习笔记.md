# 计算机系统（CSAPP 思路 + RISC-V）期末速通复习笔记

> 适用范围：根据你上传的课程 PDF（Overview、Bits/Ints、Float、RISC-V Machine-Level Programming、ISA、Datapath、Control、Pipeline、Memory Hierarchy、Cache、Virtual Memory）整理。目标是用最短时间抓住期末高频题型：**概念题会说、计算题会算、代码题会翻译、图题会分析**。

---

## 0. 期末满分复习总路线

这门课不要按“章节死背”，要按一条机器执行链来理解：

```text
C 程序
→ 数据如何表示：bits / integers / floating point
→ C 变量如何进入寄存器和内存：RISC-V 汇编
→ 汇编如何变成机器码：R/I/S/B/U/J 指令格式
→ 机器码如何在 CPU 中执行：datapath + control
→ 如何提高吞吐：pipeline
→ 为什么程序有快有慢：memory hierarchy + cache
→ 为什么每个进程看起来有独立内存：virtual memory
```

一句话：

> **Everything is bits；同一串 bits 的意义由“解释方式”和“当前指令”决定。**

例如 `0xFFFFFFFF`：

| 解释方式 | 含义 |
|---|---|
| unsigned int | 4294967295 |
| int 补码 | -1 |
| 地址 | 一个内存地址 |
| 机器码 | 一条或一部分指令 |
| float | 可能是 NaN 或其他特殊编码 |

期末复习优先级：

1. 补码、unsigned/signed 转换、溢出
2. 位运算、移位、符号扩展、零扩展、截断
3. IEEE 浮点数编码、舍入、NaN/Inf
4. RISC-V 基本指令：`add/addi/lw/sw/lb/lbu/beq/bne/blt/bge/jal/jalr`
5. 函数调用：`ra/sp/a0-a7/s0-s11/t0-t6`，栈帧，caller-saved/callee-saved
6. R/I/S/B/U/J 指令格式与立即数
7. 单周期 datapath：IF/ID/EX/MEM/WB 和控制信号
8. pipeline hazard：structural/data/control，stall/forward/flush
9. cache：S/E/B、tag/index/offset、hit/miss、局部性、blocking
10. virtual memory：VA/PA、VPN/VPO、PTE、TLB、page fault、Linux 地址空间

---

# Part 1：Bits、Bytes、Integers

## 1.1 二进制、十六进制、字节

### 核心概念

- bit：0 或 1。
- byte：8 bits。
- 十六进制 1 位 = 二进制 4 位。
- RV32 中一个 word = 32 bits = 4 bytes。

### 十六进制速算表

| Hex | Dec | Bin |
|---|---:|---|
| 0 | 0 | 0000 |
| 1 | 1 | 0001 |
| 2 | 2 | 0010 |
| 3 | 3 | 0011 |
| 4 | 4 | 0100 |
| 5 | 5 | 0101 |
| 6 | 6 | 0110 |
| 7 | 7 | 0111 |
| 8 | 8 | 1000 |
| 9 | 9 | 1001 |
| A | 10 | 1010 |
| B | 11 | 1011 |
| C | 12 | 1100 |
| D | 13 | 1101 |
| E | 14 | 1110 |
| F | 15 | 1111 |

### 例题 1：把 `0x3B6D` 转成二进制和十进制

```text
0x3B6D = 0011 1011 0110 1101₂
       = 3×16³ + 11×16² + 6×16 + 13
       = 15213
```

### 做题技巧

看到十六进制先按 4-bit 分组：

```text
0xFA = 1111 1010
0x0F = 0000 1111
0x80 = 1000 0000
0xFF = 1111 1111
```

---

## 1.2 C 数据类型大小与 RV32

常考表：

| C 类型 | RV32 典型大小 |
|---|---:|
| char | 1 byte |
| short | 2 bytes |
| int | 4 bytes |
| long | 4 bytes |
| float | 4 bytes |
| double | 8 bytes |
| pointer | 4 bytes |

易错点：

- RV32 中 `long` 和 pointer 都是 4 bytes。
- x86-64 常见 `long`/pointer 是 8 bytes，不要把 x86-64 习惯带到 RV32。

---

## 1.3 布尔运算、位运算、逻辑运算

### 位运算

| 运算 | C | RISC-V | 含义 |
|---|---|---|---|
| AND | `&` | `and/andi` | 每一位都为 1 才为 1 |
| OR | `|` | `or/ori` | 任一位为 1 就为 1 |
| XOR | `^` | `xor/xori` | 不同为 1，相同为 0 |
| NOT | `~` | `not` 伪指令 | 按位取反 |

例：

```text
  01101001
& 01010101
= 01000001

  01101001
| 01010101
= 01111101

  01101001
^ 01010101
= 00111100
```

### 逻辑运算

| 运算 | C | 结果 |
|---|---|---|
| 逻辑与 | `&&` | 0 或 1 |
| 逻辑或 | `||` | 0 或 1 |
| 逻辑非 | `!` | 0 或 1 |

### 高频易错

```c
!0x41   // 0，因为非零为 true，!true = false
!!0x41  // 1
0x69 && 0x55 // 1，不是 0x41
0x69 & 0x55  // 0x41
```

`&&` 和 `&` 完全不是一回事。

---

## 1.4 移位运算

### 左移

```c
x << k
```

- 左边丢弃，右边补 0。
- 在不溢出的直觉下，相当于乘以 `2^k`。

例：

```text
0000 0001 << 3 = 0000 1000 = 8
```

### 右移

分两种：

| 类型 | C/RISC-V | 高位补什么 | 用途 |
|---|---|---|---|
| 逻辑右移 | `srl/srli` | 补 0 | unsigned |
| 算术右移 | `sra/srai` | 补符号位 | signed |

例：8-bit 下：

```text
1010 0010 >> 2
逻辑右移：0010 1000
算术右移：1110 1000
```

### 高频易错

- `unsigned` 右移一般是逻辑右移。
- signed 负数右移通常是算术右移。
- C 中移位数量 `< 0` 或 `>= word size` 是未定义行为。

---

## 1.5 无符号数 unsigned

w 位无符号数：

```text
B2U(X) = Σ x_i · 2^i, i = 0 ... w-1
范围：0 ~ 2^w - 1
```

例如 4 位：

```text
0000 = 0
0001 = 1
0111 = 7
1111 = 15
```

---

## 1.6 补码 two's complement

w 位补码：

```text
B2T(X) = -x_{w-1}·2^{w-1} + Σ x_i·2^i, i = 0 ... w-2
范围：-2^{w-1} ~ 2^{w-1}-1
```

4 位补码范围：

```text
1000 = -8 = TMin
1001 = -7
...
1111 = -1
0000 = 0
0001 = 1
...
0111 = 7 = TMax
```

### 必背结论

w 位补码：

```text
TMin = -2^{w-1}
TMax =  2^{w-1} - 1
UMax =  2^w - 1
```

32 位：

```text
TMin = -2147483648 = 0x80000000
TMax =  2147483647 = 0x7FFFFFFF
UMax =  4294967295 = 0xFFFFFFFF
```

### 负数补码求法

求 `-x`：

```text
正数 x 的二进制 → 按位取反 → 加 1
```

例：8 位中 `-10`：

```text
10  = 0000 1010
~10 = 1111 0101
+1  = 1111 0110
所以 -10 = 1111 0110
```

---

## 1.7 signed 与 unsigned 转换

### 核心规则

> **位模式不变，只是解释方式改变。**

32 位例子：

```text
0xFFFFFFFF
作为 unsigned：4294967295
作为 int：-1
```

### C 中的坑

当 signed 和 unsigned 混合运算时，signed 往往会被转换成 unsigned。

```c
int x = -1;
unsigned y = 1;
printf("%d", x < y);  // 结果通常是 0
```

因为 `-1` 被解释为 `4294967295U`。

### 高频判断题

```c
-1 < 0U
```

答案：false。

原因：`-1` 转成 unsigned 后是最大无符号数。

---

## 1.8 扩展与截断

### 零扩展 zero extension

用于 unsigned：高位补 0。

```text
8-bit  1111 0000
16-bit 0000 0000 1111 0000
```

### 符号扩展 sign extension

用于 signed：高位补原最高位。

```text
8-bit  1111 0000
16-bit 1111 1111 1111 0000
```

### 截断 truncation

保留低 k 位，高位丢掉。

```text
0x12345678 截断成 16 位 = 0x5678
```

### 做题模板

1. 先写出原始 bit。
2. 看目标类型是 signed 还是 unsigned。
3. 扩展：signed 补符号位，unsigned 补 0。
4. 截断：直接保留低位。
5. 最后再按目标类型解释。

---

## 1.9 整数加法与溢出

### unsigned 加法

w 位无符号加法：

```text
UAdd_w(u, v) = (u + v) mod 2^w
```

超过范围直接回绕。

例：4 位：

```text
15 + 1 = 0   // 因为 16 mod 16 = 0
14 + 3 = 1   // 17 mod 16 = 1
```

### signed 补码加法

补码加法的 bit-level 行为和 unsigned 一样，都是丢弃最高进位，但解释不同。

w 位补码：

- 正溢出：两个正数相加得到负数。
- 负溢出：两个负数相加得到正数。

4 位例子：

```text
  0111  // 7
+ 0001  // 1
= 1000  // -8，正溢出
```

### 溢出判断口诀

| 情况 | 是否可能溢出 |
|---|---|
| 正 + 正 = 负 | 正溢出 |
| 负 + 负 = 正 | 负溢出 |
| 正 + 负 | 不会溢出 |

---

## 1.10 整数乘法与移位优化

### 乘法

w 位乘法只保留低 w 位：

```text
UMult_w(u, v) = u · v mod 2^w
```

补码乘法底层也只保留低 w 位。

### 常数乘法优化

编译器会把乘以常数变成移位加减。

例：

```c
long mul12(long x) { return x * 12; }
```

因为：

```text
12 = 3 × 4 = (1 + 2) × 4
x*12 = (x + x*2) << 2
```

RISC-V 可能生成：

```asm
slli a5, a0, 1   # a5 = 2x
add  a5, a5, a0  # a5 = 3x
slli a0, a5, 2   # a0 = 12x
```

---

## 1.11 除以 2 的幂

### unsigned

```text
u >> k = floor(u / 2^k)
```

使用逻辑右移。

### signed

算术右移对负数是向下取整，不是向 0 取整。

例：

```text
-9 >> 2 = floor(-9/4) = -3
```

但 C 的整数除法通常向 0 截断：

```text
-9 / 4 = -2
```

### 负数除以 2^k 向 0 取整的技巧

对负数加 bias：

```text
(x + (1<<k) - 1) >> k
```

仅当 x < 0 时加 bias。

---

## 1.12 unsigned 循环陷阱

经典错误：

```c
unsigned i;
for (i = cnt - 2; i >= 0; i--) {
    a[i] += a[i+1];
}
```

问题：`i >= 0` 对 unsigned 永远成立，`i--` 到 0 后再减会回绕成 `UINT_MAX`。

正确写法：

```c
for (i = cnt - 2; i < cnt; i--) {
    ...
}
```

或者直接用 signed，并谨慎处理边界。

---

# Part 2：Floating Point 浮点数

## 2.1 二进制小数

二进制小数点右边表示负幂：

```text
101.11₂ = 1×2² + 0×2¹ + 1×2⁰ + 1×2⁻¹ + 1×2⁻²
        = 4 + 1 + 0.5 + 0.25
        = 5.75
```

常见例子：

```text
0.5  = 0.1₂
0.25 = 0.01₂
0.75 = 0.11₂
```

### 易错点

不是所有十进制小数都能精确表示。

```text
1/3  = 0.010101...₂
1/5  = 0.00110011...₂
1/10 = 0.000110011...₂
```

所以 `0.1 + 0.2` 不能期待严格等于 `0.3`。

---

## 2.2 IEEE 浮点数格式

浮点数统一形式：

```text
V = (-1)^s × M × 2^E
```

| 字段 | 含义 |
|---|---|
| s | sign bit，符号位 |
| exp | exponent field，阶码字段 |
| frac | fraction field，尾数字段 |

### 单精度 float

```text
1 sign bit + 8 exp bits + 23 frac bits = 32 bits
Bias = 127
```

### 双精度 double

```text
1 sign bit + 11 exp bits + 52 frac bits = 64 bits
Bias = 1023
```

---

## 2.3 三类浮点数

### 规格化 normalized

条件：

```text
exp != 000...0 且 exp != 111...1
```

规则：

```text
E = Exp - Bias
M = 1.frac
```

特点：隐藏最高位 1，精度多一位。

### 非规格化 denormalized

条件：

```text
exp = 000...0
```

规则：

```text
E = 1 - Bias
M = 0.frac
```

作用：表示非常接近 0 的数，使 0 附近平滑过渡。

### 特殊值 special

条件：

```text
exp = 111...1
```

| frac | 值 |
|---|---|
| 全 0 | infinity |
| 非 0 | NaN |

常见：

```text
1.0 / 0.0 = +inf
sqrt(-1) = NaN
inf - inf = NaN
inf × 0 = NaN
```

---

## 2.4 浮点数编码题模板

题目：把十进制数转换成 IEEE float。

步骤：

1. 写成二进制。
2. 规格化成 `1.xxx × 2^E`。
3. 求符号位 `s`。
4. 求 `Exp = E + Bias`。
5. `frac` 取小数点后面的部分，不够补 0，过长要舍入。

### 例题：`15213.0` 的 float 编码

```text
15213₁₀ = 11101101101101₂
        = 1.1101101101101₂ × 2¹³
```

所以：

```text
s = 0
E = 13
Bias = 127
Exp = 140 = 10001100₂
frac = 11011011011010000000000
```

最终：

```text
0 10001100 11011011011010000000000
```

---

## 2.5 浮点数解码题模板

题目：给 `0xC0A00000`，求 float 值。

步骤：

1. 转二进制。
2. 分成 `s exp frac`。
3. 判断类型。
4. 算 `E` 和 `M`。
5. 代入 `(-1)^s M 2^E`。

例：

```text
0xC0A00000
= 1100 0000 1010 0000 0000 0000 0000 0000
= 1 | 10000001 | 01000000000000000000000
```

```text
s = 1
Exp = 129
E = 129 - 127 = 2
M = 1.01₂ = 1.25
V = -1.25 × 2² = -5
```

---

## 2.6 舍入 rounding

默认模式：round to nearest even，向最近偶数舍入。

### 规则

- 小于一半：舍。
- 大于一半：入。
- 正好一半：使最低保留位为偶数。

二进制中：

```text
halfway = 1000...₂
偶数 = 最低保留位为 0
```

### 高频易错

不是所有 `.5` 都向上。

```text
2.5 → 2   // 因为 2 是偶数
3.5 → 4   // 因为 4 是偶数
```

---

## 2.7 浮点运算性质

浮点加法：

```text
x +f y = Round(x + y)
```

浮点乘法：

```text
x *f y = Round(x * y)
```

### 与整数不同

浮点数不满足结合律：

```c
(1e20 + -1e20) + 3.14 = 3.14
1e20 + (-1e20 + 3.14) = 0.0  // 3.14 被舍掉
```

浮点乘法也不满足结合律：

```c
(1e20 * 1e20) * 1e-20 = inf
1e20 * (1e20 * 1e-20) = 1e20
```

---

## 2.8 C 中 float/double/int 转换

| 转换 | 规则 |
|---|---|
| int → double | 通常精确，只要 int 位数 ≤ 52 |
| int → float | 可能舍入 |
| float → double | 精确 |
| double → float | 可能舍入/溢出 |
| float/double → int | 小数部分截断，向 0；越界或 NaN 未定义/实现相关 |

### 高频判断

```c
x == (int)(double)x
```

如果 `x` 是 32-bit int，通常 true，因为 double 有 52 位 frac，能精确表示所有 32-bit int。

```c
x == (int)(float)x
```

不一定 true，因为 float 有效精度约 24 bits，较大的 int 会丢低位。

---

# Part 3：RISC-V 机器级程序基础

## 3.1 ISA、汇编、机器码

ISA 是硬件和软件之间的接口：

```text
C 程序 → 编译器 → RISC-V 汇编 → 汇编器 → 机器码 → CPU 执行
```

RISC-V 是 RISC 风格：指令少、格式规整、硬件容易流水化。

---

## 3.2 RV32I 寄存器

RV32I：

- 32 个通用寄存器：`x0` ~ `x31`。
- 每个寄存器 32 bits。
- `x0` 永远为 0，写入也无效。

常用 ABI 名称：

| 名称 | 编号 | 用途 |
|---|---|---|
| zero | x0 | 恒为 0 |
| ra | x1 | return address |
| sp | x2 | stack pointer |
| t0-t6 | x5-x7, x28-x31 | 临时寄存器，caller-saved |
| s0-s11 | x8-x9, x18-x27 | 保存寄存器，callee-saved |
| a0-a7 | x10-x17 | 参数；a0/a1 也用于返回值 |

### 满分口诀

```text
a：argument / return
s：saved，callee 要保
t：temporary，caller 自己保
ra：返回地址，嵌套调用必须小心
sp：栈顶/当前栈帧位置
```

---

## 3.3 RISC-V 算术指令

格式：

```asm
op rd, rs1, rs2
op rd, rs1, imm
```

含义：

```asm
add  rd, rs1, rs2   # R[rd] = R[rs1] + R[rs2]
sub  rd, rs1, rs2   # R[rd] = R[rs1] - R[rs2]
addi rd, rs1, imm   # R[rd] = R[rs1] + imm
```

例：

```c
f = g + h;
```

假设：

```text
f → x5, g → x6, h → x7
```

RISC-V：

```asm
add x5, x6, x7
```

---

## 3.4 伪指令 pseudoinstruction

伪指令不是硬件真实支持的指令，汇编器会把它翻译成真实指令。

| 伪指令 | 可能翻译 |
|---|---|
| `li rd, imm` | `addi rd, x0, imm` 或 `lui+addi` |
| `mv rd, rs` | `addi rd, rs, 0` |
| `not rd, rs` | `xori rd, rs, -1` |
| `j Label` | `jal x0, Label` |
| `jr rs` | `jalr x0, rs, 0` |

---

# Part 4：RISC-V 数据传送与内存

## 4.1 内存是 byte-addressable

现代机器按字节寻址。

RV32：

```text
地址是 32 位
一个 word = 4 bytes
```

---

## 4.2 小端序 little endian

RISC-V 是 little endian。

规则：

> 多字节数据中，最低有效字节放在最低地址。

例：把 `0xB0BACAFE` 存到地址 `0x100`：

```text
地址       内容
0x100     FE
0x101     CA
0x102     BA
0x103     B0
```

易错点：

- 人写十六进制从高位到低位。
- 内存小端从低位字节开始存。

---

## 4.3 `lw` 和 `sw`

### load word

```asm
lw rd, imm(rs1)
```

含义：

```text
addr = R[rs1] + imm
R[rd] = M[addr]
```

### store word

```asm
sw rs2, imm(rs1)
```

含义：

```text
addr = R[rs1] + imm
M[addr] = R[rs2]
```

### 高频易错

`sw` 的第一个寄存器是要存入内存的数据，不是目的寄存器。

```asm
sw x10, 0(x5)
```

意思是：把 `x10` 的值存到地址 `R[x5]+0`。

---

## 4.4 `lb`、`lbu`、`sb`

### store byte

```asm
sb rs2, imm(rs1)
```

只存 `rs2` 的最低 8 位。

例：

```text
R[x10] = 0x123456EF
sb x10, 0(x5)
```

内存只写入 `0xEF`。

### load byte signed

```asm
lb rd, imm(rs1)
```

加载 1 byte，然后符号扩展到 32 位。

例：

```text
内存 byte = 0xEF = 1110 1111₂
lb  结果 = 0xFFFFFFEF   // -17
```

### load byte unsigned

```asm
lbu rd, imm(rs1)
```

加载 1 byte，然后零扩展到 32 位。

```text
lbu 结果 = 0x000000EF   // 239
```

### 高频考点

看到 `0x80` ~ `0xFF`：

```text
lb  会变成负数
lbu 会保持正数
```

---

# Part 5：RISC-V 控制流与循环

## 5.1 PC 程序计数器

PC 保存当前指令地址。

默认：

```text
PC = PC + 4
```

因为 RV32I 每条指令 32 bits = 4 bytes。

控制流指令会改变 PC：

```text
branch taken → PC = branch target
jump → PC = jump target
```

---

## 5.2 条件分支

| 指令 | 含义 |
|---|---|
| `beq rs1, rs2, L` | 相等则跳转 |
| `bne rs1, rs2, L` | 不等则跳转 |
| `blt rs1, rs2, L` | signed `<` |
| `bge rs1, rs2, L` | signed `>=` |
| `bltu rs1, rs2, L` | unsigned `<` |
| `bgeu rs1, rs2, L` | unsigned `>=` |
| `j L` | 无条件跳转，伪指令 |

### 易错点

- `blt` 是 signed。
- `bltu` 是 unsigned。
- `bgt`、`ble` 通常是伪指令，不是基本 RV32I 指令。

---

## 5.3 if-else 翻译模板

C：

```c
if (i == j) {
    x = y + z;
} else {
    x = y - z;
}
```

寄存器：

```text
x=x10, y=x11, z=x12, i=x13, j=x14
```

推荐翻译：

```asm
bne x13, x14, Else
add x10, x11, x12
j End
Else:
sub x10, x11, x12
End:
```

做题技巧：

> 汇编常常先跳到“不满足条件”的分支，这样主路径顺序执行。

---

## 5.4 循环翻译模板

C：

```c
for (int i = 0; i < 20; i++) {
    sum += arr[i];
}
```

先改成 goto：

```c
int i = 0;
Loop:
if (i >= 20) goto End;
sum += arr[i];
i++;
goto Loop;
End:
```

RISC-V 思路：

```asm
li   t0, 0        # i = 0
li   t1, 20       # limit = 20
li   t2, 0        # sum = 0
Loop:
bge  t0, t1, End  # if i >= 20 goto End
slli t3, t0, 2    # offset = i * 4
add  t4, s0, t3   # address = arr + offset
lw   t5, 0(t4)    # load arr[i]
add  t2, t2, t5   # sum += arr[i]
addi t0, t0, 1    # i++
j    Loop
End:
```

做题技巧：数组 `int arr[i]` 的地址一般是：

```text
base + i * 4
```

所以常见：

```asm
slli offset, i, 2
add  addr, base, offset
lw   value, 0(addr)
```

---

# Part 6：RISC-V 函数调用、栈帧、调用约定

## 6.1 `jal` 与 `jr`

### 调用函数

```asm
jal ra, Function
```

含义：

```text
ra = PC + 4
PC = Function 地址
```

### 返回函数

```asm
jr ra
```

等价于：

```asm
jalr x0, ra, 0
```

含义：

```text
PC = R[ra]
```

---

## 6.2 参数与返回值

| 寄存器 | 用途 |
|---|---|
| a0-a7 | 前 8 个参数 |
| a0-a1 | 返回值 |
| ra | 返回地址 |
| sp | 栈指针 |

例：

```c
int f(int x, int y) { return x + y; }
```

```asm
f:
add a0, a0, a1
jr ra
```

---

## 6.3 栈 stack

RISC-V 栈向低地址增长。

开栈帧：

```asm
addi sp, sp, -16
```

释放栈帧：

```asm
addi sp, sp, 16
```

保存返回地址：

```asm
sw ra, 12(sp)
```

恢复返回地址：

```asm
lw ra, 12(sp)
```

---

## 6.4 caller-saved 与 callee-saved

| 类型 | 寄存器 | 谁负责保存 |
|---|---|---|
| caller-saved | `ra`, `t0-t6`, `a0-a7` | 调用者 caller |
| callee-saved | `sp`, `s0-s11` | 被调用者 callee |

### 高频易错

递归或嵌套函数调用时，`ra` 会被新的 `jal` 覆盖。

所以非叶子函数通常要保存 `ra`：

```asm
addi sp, sp, -16
sw   ra, 12(sp)
...
jal  ra, other_function
...
lw   ra, 12(sp)
addi sp, sp, 16
jr   ra
```

### 判断叶子函数

- 不调用其他函数：leaf function，可能不用保存 `ra`。
- 会调用其他函数：non-leaf function，基本要保存 `ra`，否则回不去。

---

# Part 7：RISC-V 指令格式与机器码

## 7.1 RV32I 六种指令格式

RV32I 指令固定 32 位。六种基本格式：

| 类型 | 用途 |
|---|---|
| R-type | 寄存器-寄存器运算，如 `add/sub/and/or/xor/sll/srl/sra/slt` |
| I-type | 立即数运算、load、jalr，如 `addi/lw/jalr` |
| S-type | store，如 `sw/sb/sh` |
| B-type | branch，如 `beq/bne/blt/bge` |
| U-type | upper immediate，如 `lui/auipc` |
| J-type | jump，如 `jal` |

---

## 7.2 R-type

格式：

```text
31      25 24 20 19 15 14 12 11 7 6    0
funct7     rs2   rs1   funct3 rd   opcode
```

汇编：

```asm
add rd, rs1, rs2
```

例：

```asm
add x18, x19, x10
```

字段：

```text
rd  = x18
rs1 = x19
rs2 = x10
opcode = 0110011
funct3/funct7 决定 add/sub/and/or 等具体操作
```

### 高频易错

- `rd` 是目的寄存器。
- `rs1/rs2` 是源寄存器。
- R-type 的 opcode 相同，靠 `funct3/funct7` 区分操作。

---

## 7.3 I-type

格式：

```text
31      20 19 15 14 12 11 7 6    0
imm[11:0] rs1   funct3 rd   opcode
```

用于：

```asm
addi rd, rs1, imm
lw   rd, imm(rs1)
jalr rd, rs1, imm
```

立即数范围：

```text
12-bit signed immediate: -2048 ~ 2047
```

CPU 使用前会符号扩展到 32 位。

---

## 7.4 S-type

用于 store：

```asm
sw rs2, imm(rs1)
```

格式：

```text
31      25 24 20 19 15 14 12 11      7 6    0
imm[11:5] rs2   rs1   funct3 imm[4:0] opcode
```

### 高频易错

S-type 没有 `rd`，因为 store 不写寄存器。

立即数被拆成两段：

```text
imm[11:5] 和 imm[4:0]
```

---

## 7.5 B-type

用于 branch：

```asm
beq rs1, rs2, Label
```

特点：

- 没有 `rd`。
- 读两个寄存器比较。
- immediate 是 PC-relative offset。
- branch target：

```text
PC = PC + offset
```

### 做题技巧

算 offset：

```text
offset = Label 地址 - 当前 branch 指令地址
```

不是减下一条指令地址，通常是减当前 PC。

---

## 7.6 U-type：`lui` 与 `auipc`

### `lui`

```asm
lui rd, immu
```

含义：

```text
R[rd] = immu << 12
```

### `auipc`

```asm
auipc rd, immu
```

含义：

```text
R[rd] = PC + (immu << 12)
```

---

## 7.7 `lui + addi` 生成 32 位常数

`addi` 只有 12 位立即数，因此大常数需要：

```asm
lui  rd, upper20
addi rd, rd, lower12
```

### 易错点：lower12 若最高位为 1，`addi` 会符号扩展

例如要构造：

```text
0xB0BACAFE
```

低 12 位：

```text
0xAFE
```

`0xAFE` 作为 12-bit signed 是负数。因此需要 upper20 先加 1。

正确思路：

```asm
lui  x10, 0xB0BAD
addi x10, x10, 0xAFE
```

---

# Part 8：单周期 CPU、Datapath 与 Control

## 8.1 CPU = Datapath + Control

| 部分 | 含义 |
|---|---|
| Datapath | 真正搬运和计算数据的硬件，如 PC、RegFile、ALU、Memory、MUX |
| Control | 根据指令产生控制信号，决定 datapath 怎么走 |

类比：

```text
Datapath = 身体/执行部件
Control  = 大脑/指挥部
```

---

## 8.2 状态元件与组合逻辑

### 状态元件

- PC
- Register File
- Memory

特点：在时钟边沿更新。

### 组合逻辑

- ALU
- MUX
- Immediate Generator
- Branch Comparator

特点：输入变化后，经过延迟，输出变化；不存状态。

---

## 8.3 五个阶段 IF/ID/EX/MEM/WB

| 阶段 | 名称 | 做什么 |
|---|---|---|
| IF | Instruction Fetch | 用 PC 从 IMEM 取指令，同时算 PC+4 |
| ID | Decode/Register Read | 解码，读寄存器，生成立即数 |
| EX | Execute | ALU 运算、地址计算、分支目标计算 |
| MEM | Memory Access | load/store 访问 DMEM |
| WB | Write Back | 写回 RegFile |

### 不是所有指令都用满五个阶段

| 指令 | IF | ID | EX | MEM | WB |
|---|---|---|---|---|---|
| add | √ | √ | √ | × | √ |
| addi | √ | √ | √ | × | √ |
| lw | √ | √ | √ | √ | √ |
| sw | √ | √ | √ | √ | × |
| beq | √ | √ | √ | × | × |
| jal | √ | √ | √ | × | √，写 PC+4 到 rd |

---

## 8.4 主要控制信号

| 控制信号 | 作用 |
|---|---|
| `PCSel` | 选择下一 PC：PC+4 或 ALU/branch target |
| `ImmSel` | 立即数类型：I/S/B/U/J |
| `RegWEn` | 是否写寄存器 |
| `BrUn` | 分支比较是否按 unsigned |
| `ASel` | ALU A 输入选 Reg 还是 PC |
| `BSel` | ALU B 输入选 Reg 还是 Imm |
| `ALUSel` | ALU 做 add/sub/and/or/... |
| `MemRW` | Data memory 读还是写 |
| `WBSel` | 写回数据来源：ALU/Mem/PC+4 |

---

## 8.5 各类指令 datapath 速记

### R-type：`add rd, rs1, rs2`

```text
Reg[rs1], Reg[rs2] → ALU → Reg[rd]
PC = PC + 4
```

控制：

```text
RegWEn=1
ASel=Reg
BSel=Reg
ALUSel=Add/Sub/...
MemRW=Read/Don't care
WBSel=ALU
```

### I-type arithmetic：`addi rd, rs1, imm`

```text
Reg[rs1] + imm → Reg[rd]
PC = PC + 4
```

控制：

```text
ImmSel=I
RegWEn=1
ASel=Reg
BSel=Imm
ALUSel=Add
WBSel=ALU
```

### Load：`lw rd, imm(rs1)`

```text
Reg[rs1] + imm → address
DMEM[address] → Reg[rd]
```

控制：

```text
ImmSel=I
RegWEn=1
BSel=Imm
ALUSel=Add
MemRW=Read
WBSel=Mem
```

### Store：`sw rs2, imm(rs1)`

```text
Reg[rs1] + imm → address
Reg[rs2] → DMEM[address]
```

控制：

```text
ImmSel=S
RegWEn=0
BSel=Imm
ALUSel=Add
MemRW=Write
WBSel=Don't care
```

### Branch：`beq rs1, rs2, Label`

```text
比较 Reg[rs1] 和 Reg[rs2]
若 taken：PC = PC + imm
否则：PC = PC + 4
```

控制：

```text
ImmSel=B
RegWEn=0
ASel=PC
BSel=Imm
ALUSel=Add
PCSel 由 branch comparator 结果决定
```

---

## 8.6 Critical Path

critical path：从一个状态元件输出，到下一个状态元件输入之间，延迟最长的路径。

用途：决定时钟周期。

```text
clock period >= max critical path delay
```

常考比较：

- `add`：不访问数据内存，路径较短。
- `lw`：要取指、读寄存器、生成立即数、ALU 算地址、读 DMEM、写回，通常最长。

---

# Part 9：Pipeline 流水线

## 9.1 为什么需要流水线

单周期 CPU：一条指令完整走完后下一条才开始。

流水线 CPU：多条指令同时处于不同阶段。

```text
IF | ID | EX | MEM | WB
```

### 核心结论

> 流水线主要提高 throughput，不一定降低单条指令 latency。

类似洗衣服：洗、烘、叠、收可以流水作业，但一件衣服完整处理时间不变。

---

## 9.2 五级流水线

| 阶段 | 功能 |
|---|---|
| IF | 取指令 |
| ID | 解码/读寄存器 |
| EX | ALU 执行 |
| MEM | 数据内存访问 |
| WB | 写回寄存器 |

每两个阶段之间有 pipeline register，用来保存中间结果。

---

## 9.3 三类 hazard

### 结构冒险 Structural Hazard

硬件资源不够，多条指令同周期争同一个资源。

例：如果指令内存和数据内存共用同一个端口，那么 IF 和 MEM 可能冲突。

解决：

- 增加硬件资源。
- 分离 IMEM 和 DMEM。
- RegFile 支持同周期读写。

### 数据冒险 Data Hazard

后一条指令需要前一条指令还没写回的结果。

例：

```asm
add s0, t0, t1
sub t2, s0, t3
```

`sub` 需要 `add` 的结果。

解决：

- stall：插入 nop。
- forwarding/bypassing：结果一算出来就转发。
- code scheduling：编译器重新排列指令。

### 控制冒险 Control Hazard

分支/跳转导致下一条指令不确定。

解决：

- stall。
- flush 错误指令。
- branch prediction。

---

## 9.4 Forwarding 转发

思想：

> 不等 WB 写回 RegFile，直接把 EX/MEM 或 MEM/WB 阶段的结果送到后面指令的 ALU 输入。

例：

```asm
add s0, t0, t1
sub t2, s0, t3
```

`add` 的 ALU 结果在 EX 结束后已经有了，`sub` 的 EX 阶段可以直接用转发值。

---

## 9.5 Load-use Hazard

经典情况：

```asm
lw  t0, 0(t1)
add t2, t0, t3
```

`lw` 的数据要到 MEM 后才出来，而紧跟的 `add` 在 EX 就需要它。

即使 forwarding，也通常要 stall 1 cycle。

可改为：

```asm
lw  t0, 0(t1)
# 插入一条无关指令
add t2, t0, t3
```

---

## 9.6 Branch penalty

简单 RV32I pipeline 中，如果 branch taken，可能需要 flush 后面已经取到的错误指令。

课件中简单流水线的典型结论：

```text
taken branch 代价可能是 3 cycles
```

因为分支结果较晚才确定。

---

# Part 10：Memory Hierarchy 存储层次

## 10.1 为什么要有存储层次

不同存储器速度、容量、价格不同：

```text
寄存器：最快、最小、最贵
Cache：很快、较小、贵
DRAM 主存：较慢、较大、便宜
SSD/HDD：更慢、更大、更便宜
远程存储：最慢
```

### SRAM vs DRAM

| 类型 | 特点 | 用途 |
|---|---|---|
| SRAM | 快、不需要刷新、贵、面积大 | Cache |
| DRAM | 慢、需要刷新、便宜、密度高 | Main memory |

---

## 10.2 局部性 locality

### 时间局部性 temporal locality

刚访问过的数据，很快可能再次访问。

例：循环中的变量 `sum`。

### 空间局部性 spatial locality

访问某个地址后，附近地址很可能被访问。

例：顺序遍历数组。

```c
for (i = 0; i < n; i++) sum += a[i];  // 空间局部性好
```

差的例子：

```c
for (i = 0; i < n; i += 1024) sum += a[i];  // stride 大，空间局部性差
```

---

## 10.3 磁盘访问时间

磁盘访问时间：

```text
Taccess = Tavg seek + Tavg rotation + Tavg transfer
```

| 部分 | 含义 |
|---|---|
| seek time | 磁头移动到目标磁道 |
| rotational latency | 等目标扇区转到磁头下 |
| transfer time | 数据真正传输 |

考试一般考概念和公式，不会太深。

---

# Part 11：Cache Memories

## 11.1 Cache 基本思想

Cache 是小而快的 SRAM，自动缓存主存中的部分 block。

CPU 访问顺序：

```text
先查 cache
hit：直接返回
miss：从下一层存储取 block，放入 cache
```

---

## 11.2 Cache 参数 S、E、B、C

| 参数 | 含义 |
|---|---|
| S = 2^s | set 数 |
| E = 2^e | 每个 set 中 line 数，即 associativity |
| B = 2^b | 每个 block 的字节数 |
| C | cache 数据容量 |

公式：

```text
C = S × E × B
```

注意：C 一般只算数据字节，不算 valid/tag/dirty 等元数据。

---

## 11.3 地址划分

m 位地址被划分为：

```text
| tag t bits | set index s bits | block offset b bits |
```

其中：

```text
b = log2(B)
s = log2(S)
t = m - s - b
```

### Cache 读步骤

1. 用 set index 找到 set。
2. 在 set 内比较所有 line 的 tag。
3. 同时检查 valid bit。
4. tag 匹配且 valid=1 → hit。
5. 用 block offset 取出目标字节/字。

---

## 11.4 直接映射 cache

Direct mapped：

```text
E = 1
每个 set 只有 1 条 line
```

优点：简单快。

缺点：冲突不命中多。

---

## 11.5 组相联 cache

E-way set associative：

```text
每个 set 有 E 条 line
```

查找时：在同一个 set 内比较 E 个 tag。

不命中时：选择一条 line 替换，策略可能是：

- random
- LRU

---

## 11.6 Cache miss 分类

| 类型 | 原因 | 解决思路 |
|---|---|---|
| compulsory/cold miss | 第一次访问，cache 里必然没有 | 无法完全避免 |
| conflict miss | 多个 block 映射到同一 set 互相踢出 | 增大 E，改变访问模式 |
| capacity miss | 工作集太大，cache 放不下 | 增大 cache，blocking |

---

## 11.7 写策略

### Write hit

| 策略 | 含义 |
|---|---|
| write-through | 同时写 cache 和下一层内存 |
| write-back | 先只写 cache，替换时再写回；需要 dirty bit |

### Write miss

| 策略 | 含义 |
|---|---|
| write-allocate | 先把 block 加载进 cache，再写 |
| no-write-allocate | 直接写下一层，不放进 cache |

常见组合：

```text
write-through + no-write-allocate
write-back + write-allocate
```

---

## 11.8 Cache 性能指标

```text
miss rate = misses / accesses
hit rate = 1 - miss rate
```

平均访问时间：

```text
AMAT = hit time + miss rate × miss penalty
```

例题：

```text
hit time = 1 cycle
miss penalty = 100 cycles
```

97% hit：

```text
AMAT = 1 + 0.03×100 = 4 cycles
```

99% hit：

```text
AMAT = 1 + 0.01×100 = 2 cycles
```

结论：99% hit 比 97% hit 快一倍，所以关注 miss rate 更直观。

---

## 11.9 Cache 地址题模板

题目给：

```text
m-bit 地址，S sets，E lines/set，B bytes/block
某地址 A
```

步骤：

1. `b = log2(B)`。
2. `s = log2(S)`。
3. `t = m - s - b`。
4. 把地址 A 写成 m-bit 二进制。
5. 从右往左切：offset b 位，index s 位，tag 剩下。
6. 根据 index 找 set，根据 tag/valid 判断 hit。

### 例题

```text
M = 16 bytes → 地址 4 bits
B = 2 bytes → b = 1
S = 4 sets  → s = 2
E = 1
所以 t = 4 - 2 - 1 = 1
```

地址 7：

```text
7 = 0111₂
tag = 0
index = 11₂ = 3
offset = 1
```

---

## 11.10 Cache-friendly code

### 原则

1. 关注核心函数的内层循环。
2. 尽量 stride-1 顺序访问。
3. 数据一旦加载，尽量多次使用。
4. 矩阵乘法用 blocking。

### 数组访问

C 数组是 row-major。

好：

```c
for (i = 0; i < N; i++)
    for (j = 0; j < N; j++)
        sum += a[i][j];
```

坏：

```c
for (j = 0; j < N; j++)
    for (i = 0; i < N; i++)
        sum += a[i][j];
```

原因：第一种连续访问内存，空间局部性好。

### Blocking 思想

矩阵乘法中，把大矩阵切成小块，让一个小块尽量留在 cache 里反复使用。

限制条件常见：

```text
3B² < C
```

表示 A、B、C 三个 block 能同时放入 cache。

---

# Part 12：Virtual Memory 虚拟内存

## 12.1 为什么需要虚拟内存

虚拟内存 VM 的三个作用：

1. 高效使用主存：把 DRAM 当作虚拟地址空间的 cache。
2. 简化内存管理：每个进程看到统一的线性地址空间。
3. 内存保护：进程之间隔离，用户程序不能随便访问内核。

---

## 12.2 地址空间

| 名称 | 含义 |
|---|---|
| VA | virtual address，CPU 产生 |
| PA | physical address，真正访问内存的地址 |
| MMU | Memory Management Unit，把 VA 翻译成 PA |

地址翻译：

```text
CPU 产生 VA → MMU 翻译 → PA → cache/DRAM
```

---

## 12.3 页 page

虚拟内存和物理内存都按页划分。

```text
page size = P = 2^p bytes
```

VA 划分：

```text
| VPN | VPO |
```

PA 划分：

```text
| PPN | PPO |
```

关键：

```text
VPO = PPO
```

因为页内偏移不变。

---

## 12.4 页表 page table

页表是 PTE 的数组。

PTE 包含：

- valid bit / present bit
- PPN 或磁盘地址
- permission bits：R/W/X、User/Supervisor 等
- dirty/reference bits 等

### Page hit

```text
PTE valid = 1
VP 已在物理内存中
```

### Page fault

```text
PTE valid = 0
VP 不在物理内存中，触发异常
```

Page fault 处理：

1. 操作系统接管。
2. 找空闲物理页或选择 victim。
3. 必要时写回 victim。
4. 从磁盘读入目标页。
5. 更新 PTE。
6. 重新执行导致 fault 的指令。

这叫 demand paging：需要时才调入。

---

## 12.5 Working set 与 thrashing

Working set：程序当前活跃访问的虚拟页集合。

如果：

```text
working set size < physical memory size
```

性能较好。

如果：

```text
sum of working sets > physical memory size
```

会 thrashing：页面频繁换入换出，性能崩溃。

---

## 12.6 TLB

TLB：Translation Lookaside Buffer，页表项缓存。

作用：缓存 VPN → PPN 映射，避免每次访问内存都查页表。

访问流程：

```text
VA → 拆成 VPN/VPO
VPN 查 TLB
TLB hit：得到 PPN
TLB miss：查 page table，可能 page hit 或 page fault
PPN + VPO/PPO → PA
```

---

## 12.7 地址翻译题模板

题目给：

```text
n-bit VA
m-bit PA
page size P=2^p
TLB 参数
cache 参数
VA = 某地址
```

步骤：

1. `p = log2(P)`。
2. VA 低 p 位是 VPO，高位是 VPN。
3. 用 VPN 切出 TLBT/TLBI。
4. 查 TLB：匹配 tag 且 valid=1 → TLB hit。
5. TLB miss 则查页表。
6. 如果 PTE valid=0 → page fault。
7. 若得到 PPN，则：

```text
PA = PPN || VPO
```

8. 再把 PA 按 cache 规则切成 CT/CI/CO 判断 cache hit。

---

## 12.8 Linux 进程虚拟地址空间

典型从低到高：

```text
.text     程序代码，只读
.rodata   只读数据
.data     已初始化全局变量
.bss      未初始化全局变量
heap      malloc/free，向高地址增长
mmap area 共享库、内存映射文件
stack     函数调用栈，向低地址增长
kernel    内核虚拟地址区域，用户不可访问
```

### `brk`

heap 顶部位置，`malloc` 可能调整它。

### `%rsp` / `sp`

栈指针，指向当前栈位置。

---

## 12.9 C 内存错误高频

| 错误 | 例子 | 后果 |
|---|---|---|
| bad pointer | `scanf("%d", val);` 应该传 `&val` | 写入随机地址 |
| uninitialized memory | 读未初始化局部变量 | 值不可预测 |
| buffer overflow | 数组越界写 | 破坏其他数据/控制信息 |
| use after free | `free(p); *p=1;` | 未定义行为 |
| double free | `free(p); free(p);` | 堆结构损坏 |
| memory leak | malloc 后不 free | 内存泄漏 |
| invalid free | free 非 malloc 指针 | 崩溃/未定义行为 |

---

# Part 13：综合题型与做题技巧

## 13.1 位级题

### 常见题型

- 给 bit pattern，分别按 signed/unsigned 解释。
- 判断表达式是否总为真。
- 计算溢出结果。
- 扩展/截断。

### 做题套路

1. 明确 word size。
2. 写 bit pattern。
3. 判断 signed/unsigned。
4. 运算时记住有限位宽会截断。
5. 最后再解释结果。

---

## 13.2 RISC-V 翻译题

### C → 汇编套路

1. 变量分配寄存器。
2. 表达式用 `add/sub/and/or/slli/...`。
3. 数组访问：`base + index * element_size`。
4. if：用反条件跳过 then。
5. loop：先写 goto 版，再翻译。
6. 函数调用：参数放 `a0-a7`，返回值在 `a0`。
7. 非叶子函数保存 `ra`。

---

## 13.3 指令格式题

### 汇编 → 机器码字段

1. 判断指令类型。
2. 写字段模板。
3. 查 opcode/funct3/funct7。
4. 寄存器转编号，编号转 5-bit。
5. 立即数转补码。
6. 拼接。

### 易错点

- `addi` 的立即数是 signed 12-bit。
- `sw` 的立即数拆开。
- branch/jump offset 是 PC-relative。
- `rd/rs1/rs2` 位置不能乱。

---

## 13.4 Datapath 图题

看到一条指令，问 datapath/control：

1. 是否读 RegFile？读哪些寄存器？
2. ALU 输入来自 Reg 还是 PC/Imm？
3. ALU 做什么？
4. 是否访问 DMEM？读还是写？
5. 是否写 RegFile？写回来源是 ALU/Mem/PC+4？
6. PC 是 PC+4 还是 branch/jump target？

---

## 13.5 Pipeline 题

判断 hazard：

1. 看后一条是否读前一条写的寄存器。
2. 看需要值的阶段：通常 ALU 在 EX 需要，store 数据在 MEM 需要。
3. 看前一条什么时候产生值：
   - ALU 结果 EX 结束可转发。
   - load 数据 MEM 结束才有。
4. load-use 通常 stall 1 cycle。
5. branch taken 要 flush 错误指令。

---

## 13.6 Cache 题

### 地址划分口诀

```text
先 B 后 S：
b = log2(B)
s = log2(S)
t = m - s - b
从右往左切：offset → index → tag
```

### hit/miss 判断

```text
valid = 1 且 tag 匹配 → hit
否则 miss
```

### 局部性判断

- 顺序访问数组：空间局部性好。
- 重复使用同一变量/小数组：时间局部性好。
- 按列访问 C 二维数组：通常空间局部性差。
- blocking 可以提升时间局部性。

---

## 13.7 Virtual Memory 题

### 地址翻译口诀

```text
VA 切 VPN/VPO
VPN 查 TLB/PTE
PTE 给 PPN
PA = PPN + VPO
再用 PA 查 cache
```

### 判断异常

| 情况 | 结果 |
|---|---|
| 地址不属于任何 VM area | segmentation fault |
| 权限不允许，如写只读页 | protection exception，Linux 也可能显示 segfault |
| PTE valid=0 但地址合法 | normal page fault，OS 调页 |

---

# Part 14：期末高频易错清单

## 14.1 Bits/Ints 易错

- `0xFFFFFFFF` 不一定是 -1，要看类型。
- signed 和 unsigned 比较时，signed 可能转 unsigned。
- `i >= 0` 对 unsigned 永远 true。
- `~x` 是按位取反，`!x` 是逻辑取反。
- `&&`/`||` 结果是 0 或 1。
- signed 负数右移通常是算术右移。
- 截断只保留低位。
- 符号扩展补最高位，不是补 1。
- `TMin` 没有对应的正数，`-TMin == TMin` 在补码中会溢出。

## 14.2 Float 易错

- 浮点数不是实数，很多十进制小数不能精确表示。
- 浮点加法不满足结合律。
- NaN 与任何数比较都很特殊。
- `float → int` 是向 0 截断。
- `int → float` 可能舍入。
- exp 全 0 是 denorm/zero，exp 全 1 是 inf/NaN。

## 14.3 RISC-V 易错

- `x0` 永远为 0。
- `sw rs2, imm(rs1)` 没有 rd。
- `lb` 符号扩展，`lbu` 零扩展。
- `sra` 算术右移，`srl` 逻辑右移。
- `jal ra, f` 会覆盖 `ra`。
- 非叶子函数要保存 `ra`。
- `a0` 既是第一个参数，也是返回值。
- `s0-s11` 是 callee-saved，`t/a/ra` 是 caller-saved。

## 14.4 ISA 格式易错

- RV32I 指令固定 32 位。
- R-type：`funct7 rs2 rs1 funct3 rd opcode`。
- I-type 立即数是 12-bit signed。
- S-type 立即数拆成两段。
- B/J-type offset 是 PC-relative。
- `lui+addi` 构造大数时，低 12 位若为负，上 20 位要加 1。

## 14.5 Datapath/Pipeline 易错

- 单周期 CPU：一条指令一个周期，但周期必须足够长以容纳最慢指令。
- pipeline 提高吞吐，不一定降低单条指令延迟。
- structural hazard 是资源冲突，不是数据依赖。
- load-use hazard 即使 forwarding 也常要 stall。
- branch taken 需要 flush 错误指令。

## 14.6 Cache/VM 易错

- Cache 容量 `C = S × E × B` 不算 tag/valid。
- offset 位数由 block size 决定。
- index 位数由 set 数决定。
- tag 是剩余高位。
- write-back 需要 dirty bit。
- VPO = PPO。
- TLB 缓存的是 VPN→PPN 映射，不是普通数据。
- page fault 不一定是程序错误；可能只是正常缺页。
- segmentation fault 通常是非法地址或权限错误。

---

# Part 15：速通刷题模板

## 模板 1：补码范围题

w 位补码：

```text
TMin = -2^{w-1}
TMax =  2^{w-1}-1
```

w 位无符号：

```text
0 ~ 2^w - 1
```

---

## 模板 2：溢出判断

补码加法：

```text
正 + 正 = 负 → 正溢出
负 + 负 = 正 → 负溢出
其他不溢出
```

unsigned 加法：

```text
结果 < 任一加数 → 溢出
```

---

## 模板 3：浮点编码

```text
十进制 → 二进制 → 1.xxx × 2^E
s = sign
Exp = E + Bias
frac = xxx，舍入/补零
```

---

## 模板 4：数组寻址

```c
int a[N];
a[i] 地址 = base + 4*i
```

RISC-V：

```asm
slli t0, i, 2
add  t1, base, t0
lw   t2, 0(t1)
```

---

## 模板 5：函数序言/尾声

非叶子函数典型：

```asm
func:
addi sp, sp, -16
sw   ra, 12(sp)
sw   s0, 8(sp)

# body

lw   s0, 8(sp)
lw   ra, 12(sp)
addi sp, sp, 16
jr   ra
```

---

## 模板 6：Cache 地址划分

```text
b = log2(B)
s = log2(S)
t = m - s - b
地址 = tag | index | offset
```

---

## 模板 7：虚拟地址翻译

```text
p = log2(page size)
VA = VPN | VPO
查 TLB/PTE 得 PPN
PA = PPN | VPO
```

---

# Part 16：模拟高频例题

## 例题 1：8 位补码中 `0xF6` 是多少？

```text
0xF6 = 1111 0110
最高位 1，负数
取反：0000 1001
加 1：0000 1010 = 10
所以值为 -10
```

答案：`-10`。

---

## 例题 2：8 位 unsigned 中 `250 + 10` 结果是多少？

```text
250 + 10 = 260
260 mod 256 = 4
```

答案：`4`。

---

## 例题 3：`lb` 与 `lbu`

内存地址 `0x100` 存 `0xEF`。

```asm
lb  x10, 0(x5)
lbu x11, 0(x5)
```

结果：

```text
x10 = 0xFFFFFFEF = -17
x11 = 0x000000EF = 239
```

---

## 例题 4：Cache 地址切分

已知：

```text
地址 8 bits
B = 4 bytes
S = 8 sets
E = 2
地址 A = 0x6D = 0110 1101₂
```

计算：

```text
b = log2(4) = 2
s = log2(8) = 3
t = 8 - 3 - 2 = 3
```

切分：

```text
011 011 01
 tag index offset
```

答案：

```text
tag = 011₂ = 3
index = 011₂ = 3
offset = 01₂ = 1
```

---

## 例题 5：虚拟地址翻译

已知：

```text
VA 14 bits
PA 12 bits
page size = 64 bytes = 2^6
VA = 0x03D4
```

解：

```text
p = 6
VPO = VA 低 6 位
VPN = VA 高 8 位
```

`0x03D4 = 0000 1111 0101 00₂` 按 14 位看：

```text
VPN = VA >> 6
VPO = VA & 0x3F
```

考试可直接用位运算：

```text
VPN = 0x03D4 >> 6 = 0x0F
VPO = 0x03D4 & 0x3F = 0x14
```

之后查 TLB/page table 得 PPN，再拼 PA：

```text
PA = (PPN << 6) | VPO
```

---

# Part 17：最后 24 小时复习建议

## 第 1 轮：2 小时建立框架

快速过：

```text
Bits/Ints → Float → RISC-V → ISA → Datapath → Pipeline → Cache → VM
```

目标：每章能说出 5 个关键词。

## 第 2 轮：4 小时刷计算题

重点刷：

- 补码/unsigned
- 溢出
- 浮点编码/解码
- RISC-V C 翻译
- 指令格式字段
- cache tag/index/offset
- VA→PA 地址翻译

## 第 3 轮：2 小时背模板

背这几张：

1. RISC-V 寄存器用途表。
2. R/I/S/B/U/J 格式。
3. IF/ID/EX/MEM/WB。
4. 控制信号含义。
5. Cache 地址切分公式。
6. VM 地址翻译流程。

## 第 4 轮：考前 30 分钟

只看易错清单，不再看新题。

---

# 一句话总复盘

这门课的满分核心不是背很多术语，而是能把一条 C 语句一路追下去：

```text
它的数据如何用 bits 表示？
它会生成哪些 RISC-V 指令？
这些指令属于什么格式？
在 datapath 中走哪些硬件？
pipeline 中有没有 hazard？
访问内存时 cache 是否命中？
地址是否要经过 TLB/page table 翻译？
```

把这条链打通，期末大部分题都能做。
