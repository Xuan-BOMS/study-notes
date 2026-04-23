# STM32 Cortex-Debug 工程原理详解

这份文档解释的是目录 `D:\task\Cortex-Debug\text` 里的工程。

目标读者是“只有基础计算机知识的人”。

你不需要先懂嵌入式、汇编、链接脚本、OpenOCD 才能看懂。本文会把这些东西放进一个完整故事里，让你知道：

1. 这个工程到底想做什么。
2. 每个文件存在的意义是什么。
3. 代码是怎样一步一步变成芯片里能运行的程序的。
4. VS Code、CMake、编译器、OpenOCD、Cortex-Debug 各自扮演什么角色。
5. 为什么这个模板以后可以迁移到别的 STM32 芯片。

---

## 1. 先用一句话概括整个工程

这个工程是一个“给 STM32F407ZG 做最小可运行、可烧录、可调试程序”的模板。

它做的事情非常克制：

- 写一段最简单的 MCU 程序。
- 用 `arm-none-eabi-gcc` 把这段程序编译成 ARM Cortex-M4 能运行的机器代码。
- 用链接脚本把程序放到 STM32F407ZG 的正确存储区域。
- 用启动文件让芯片上电后知道第一步该做什么。
- 用 VS Code 的 `Cortex-Debug` 插件配合 `OpenOCD`，让你能下载程序、打断点、单步执行、看变量。

所以，这个工程不是“业务程序”，而是“基础设施工程”。

它的重点不是功能，而是把“开发链路”搭通。

---

## 2. 这类工程和普通 PC 程序有什么本质区别

如果你平时接触的是 Windows、浏览器、Python、Java、C++ 桌面程序，那么你习惯的是：

- 操作系统已经存在。
- 程序加载器已经存在。
- 内存分配器已经存在。
- 文件系统已经存在。
- 启动时有人帮你调用 `main()`。

但是 STM32 这种单片机程序不是这样。

对于 STM32 来说：

- 通常没有完整操作系统。
- 芯片上电后只会从固定地址取第一条指令。
- 没有人自动帮你准备运行环境。
- 你必须自己告诉芯片：
  - 栈顶在哪里。
  - 中断向量表在哪里。
  - `.data` 段怎么从 Flash 复制到 RAM。
  - `.bss` 段怎么清零。
  - 然后才能进入 `main()`。

这就是为什么嵌入式工程比普通 C 工程多出：

- 启动文件
- 链接脚本
- 芯片配置
- 下载调试配置

这些文件不是“附属品”，而是程序能跑起来的必要条件。

---

## 3. 你可以把整个工程想象成一条流水线

这条流水线可以概括成：

`源代码 -> 编译 -> 链接 -> 生成 elf/bin/hex -> OpenOCD 下载到芯片 -> GDB 调试`

再展开一点，就是：

1. 你写 `main.c`。
2. 编译器把 `main.c` 和启动汇编翻译成目标文件 `.obj`。
3. 链接器按照链接脚本，把代码和数据放到 STM32 的正确地址。
4. 生成 `.elf`、`.bin`、`.hex`。
5. OpenOCD 通过 ST-Link 和 SWD 接口把程序写进 MCU Flash。
6. Cortex-Debug 启动 `arm-none-eabi-gdb`，连接 OpenOCD。
7. 你在 VS Code 里打断点、单步、看寄存器、看变量。

你可以把这里的角色理解成：

- `CMake`：总调度员，决定怎么构建。
- `Ninja`：执行员，真正去跑编译命令。
- `arm-none-eabi-gcc`：翻译员，把 C/汇编翻译成 ARM 机器码。
- `链接器 ld`：排版师，把代码和数据摆到正确内存地址。
- `objcopy`：格式转换器，把 elf 转成 bin/hex。
- `OpenOCD`：下载和调试服务器。
- `GDB`：调试控制器。
- `Cortex-Debug`：VS Code 里的人机界面。

---

## 4. 工程目录总览

工程的核心目录是：

```text
text/
  CMakeLists.txt
  CMakePresets.json
  cmake/
    arm-none-eabi-toolchain.cmake
  linker/
    STM32F407ZG_FLASH.ld
  openocd/
    stm32f407zg.cfg
  src/
    main.c
    system_stm32f4xx.c
  startup/
    startup_stm32f407zg.s
  build/
    f407-debug/
```

你可以把它分成 5 类文件：

1. 构建控制文件
2. MCU 启动相关文件
3. 业务源码
4. 下载调试配置
5. 构建输出文件

下面逐个讲。

---

## 5. `CMakeLists.txt` 是构建总说明书

文件位置：

`D:\task\Cortex-Debug\text\CMakeLists.txt`

它做的事情本质上是告诉 CMake：

- 这是一个什么工程。
- 要编译哪些源文件。
- 用什么 CPU 架构参数。
- 用什么链接脚本。
- 编译和链接时加哪些选项。
- 编译完成后还要生成哪些附加文件。

### 5.1 `project(stm32_f407zg LANGUAGES C ASM)` 是什么意思

意思是：

- 工程名字叫 `stm32_f407zg`
- 这个工程会编译两种语言：
  - `C`
  - `ASM`（汇编）

为什么需要汇编？

因为启动过程里有一部分工作必须非常底层，通常直接写汇编最清楚。

---

### 5.2 为什么这里要写 CPU/FPU/浮点 ABI

文件里有这些设置：

- `MCU_CPU "cortex-m4"`
- `MCU_FPU "fpv4-sp-d16"`
- `MCU_FLOAT_ABI "hard"`

它们是在告诉编译器：

- 目标芯片是 Cortex-M4 内核。
- 有单精度硬件浮点单元。
- 浮点参数传递按硬件浮点规则来。

如果你写错这些参数，常见后果有：

- 程序直接跑飞。
- 链接失败。
- 浮点相关代码行为异常。
- 调试信息和真实硬件不匹配。

所以它们不是装饰信息，而是“生成正确机器码”的关键条件。

---

### 5.3 `add_executable(...)` 为什么不是桌面程序那种可执行文件

这里虽然写的是 `add_executable`，但它不是 Windows 下的 `.exe` 思路。

在 CMake 语义里，它表示“最终产出一个可执行映像”。

这里被加入的源文件有：

- `src/main.c`
- `src/system_stm32f4xx.c`
- `startup/startup_stm32f407zg.s`

也就是说，最终程序至少由三块组成：

- 你的主逻辑
- 系统初始化逻辑
- 上电后的第一段启动逻辑

---

### 5.4 为什么需要编译选项

这里有一组常见选项：

- `-ffunction-sections`
- `-fdata-sections`
- `-Wall`
- `-Wextra`
- `-Og`
- `-g3`

可以这样理解：

- `-Wall -Wextra -Wpedantic`：尽量早点发现代码问题。
- `-Og`：为调试优化，而不是一味追求体积最小或速度最高。
- `-g3`：生成更完整的调试信息，方便 GDB 识别源码和变量。
- `-ffunction-sections -fdata-sections`：让链接器能把“没用到的函数和数据”剔掉。

这就是为什么后面链接时还能用 `--gc-sections`。

两者配合后，最终固件会更干净。

---

### 5.5 为什么还要写链接选项

最关键的几个链接选项是：

- `-T${LINKER_SCRIPT}`
- `-nostartfiles`
- `-Wl,--gc-sections`
- `-Wl,-Map=...`

含义分别是：

- `-T...`：明确指定链接脚本，告诉链接器内存怎么布局。
- `-nostartfiles`：不要使用宿主平台默认启动文件，因为我们自己提供启动代码。
- `--gc-sections`：删除没用到的代码段和数据段。
- `-Map=...`：生成 `.map` 文件，便于分析程序最终布局。

`.map` 文件很重要，因为它告诉你：

- 每个函数最终落在哪个地址。
- 各段用了多少 Flash 和 RAM。
- 某个大对象为什么占空间。

---

### 5.6 为什么构建结束后还要再生成 `.bin` 和 `.hex`

构建后执行了：

- `objcopy -O binary ... .bin`
- `objcopy -O ihex ... .hex`
- `size ...`

原因是：

- `.elf`：适合调试，信息最完整，带符号表和调试信息。
- `.bin`：纯二进制，最直接。
- `.hex`：带地址信息的文本格式，很多烧录工具认识。

一般调试最常用 `.elf`，烧录有时会用 `.bin` 或 `.hex`。

在这个工程里：

- VS Code 调试主要使用 `.elf`
- OpenOCD 烧录任务也直接使用 `.elf`

---

## 6. `CMakePresets.json` 是“预设构建方案”

文件位置：

`D:\task\Cortex-Debug\text\CMakePresets.json`

它的意义是：把一套常用的 CMake 配置保存成名字，避免每次手打长命令。

这里的预设名是：

- `f407-debug`

它定义了：

- 生成器是 `Ninja`
- 构建目录是 `build/f407-debug`
- 构建类型是 `Debug`
- 工具链文件是 `cmake/arm-none-eabi-toolchain.cmake`

所以你可以把 preset 理解成：

“我给这套构建参数起了个别名，以后直接用这个别名就行。”

---

## 7. `arm-none-eabi-toolchain.cmake` 是“告诉 CMake 你不是在给 Windows 编译”

文件位置：

`D:\task\Cortex-Debug\text\cmake\arm-none-eabi-toolchain.cmake`

这个文件非常关键，因为 CMake 默认会认为你在给当前电脑编译程序。

但这里不是。

这里是在 Windows 电脑上，给 STM32 这种 ARM 裸机环境编译程序。

这叫交叉编译。

### 7.1 什么是交叉编译

交叉编译的意思是：

- 编译器运行在 A 平台
- 生成的程序运行在 B 平台

本工程中：

- 编译器运行在 Windows
- 程序运行在 STM32F407ZG

所以必须告诉 CMake：

- 目标系统不是 Windows
- 目标处理器是 ARM
- 使用 `arm-none-eabi-*` 这套工具

这里的 `none-eabi` 可以粗略理解为：

- `none`：没有通用操作系统
- `eabi`：使用 ARM 的嵌入式 ABI 规范

---

### 7.2 为什么要找这么多工具

文件里不仅找了 `gcc`，还找了：

- `g++`
- `ar`
- `objcopy`
- `objdump`
- `ranlib`
- `strip`
- `size`

原因是“构建”不只是编译。

完整流程会用到：

- 编译器
- 汇编器
- 链接器
- 归档工具
- 反汇编工具
- 格式转换工具
- 大小统计工具

即使当前模板很小，有些工具当下没明显出场，它们也是标准工具链的一部分。

---

## 8. `main.c` 是程序的主体逻辑

文件位置：

`D:\task\Cortex-Debug\text\src\main.c`

它的内容非常少：

- 定义了一个全局变量 `g_heartbeat`
- 定义了一个忙等待 `delay()`
- 在 `main()` 里无限循环：
  - `g_heartbeat++`
  - 延时

### 8.1 为什么要写得这么简单

因为这个模板的目的不是展示业务功能，而是验证：

- 程序能否被编译
- 程序能否被下载
- 程序能否在芯片上跑起来
- 调试器能否看到变量变化

`g_heartbeat` 就像一个“心跳信号”。

你在调试器里看这个变量持续增长，就能确认：

- 程序没有死机
- `main()` 已经执行起来
- 断点、单步、变量监视链路是通的

这比一上来接串口、点灯、配时钟更适合做最小验证。

---

## 9. `system_stm32f4xx.c` 负责早期系统初始化

文件位置：

`D:\task\Cortex-Debug\text\src\system_stm32f4xx.c`

这个文件现在只做了一件事：

- 打开 FPU 访问权限

代码里写的是：

- 访问 `CPACR` 寄存器
- 允许协处理器 CP10 和 CP11 全访问

你可以把它理解成：

“告诉 CPU：浮点单元可以用了。”

### 9.1 为什么这一步不能省

因为构建参数里已经告诉编译器：

- 这是带硬件浮点的 Cortex-M4
- 浮点 ABI 用 `hard`

如果后续程序真的用了浮点指令，而 FPU 权限没开，程序可能直接异常。

所以这类初始化虽然看起来很短，但它和编译选项是配套的。

---

## 10. `startup_stm32f407zg.s` 是上电后的“第一段程序”

文件位置：

`D:\task\Cortex-Debug\text\startup\startup_stm32f407zg.s`

这是整个工程里最底层、也最重要的文件之一。

如果把 `main.c` 看成“应用层”，那启动文件就是“引导层”。

### 10.1 芯片上电后不会直接跳到 `main()`

很多初学者会误以为：

- 按下复位
- CPU 自动执行 `main()`

实际上不是。

对 Cortex-M 来说，复位后 CPU 会先从中断向量表取两个最关键的信息：

1. 初始栈顶地址
2. 复位处理函数地址，也就是 `Reset_Handler`

这就是为什么向量表的前两项是：

- `_estack`
- `Reset_Handler`

---

### 10.2 什么是中断向量表

中断向量表可以理解成一张“紧急联系电话表”。

表里的每一项都对应某一种异常或中断发生时应该跳到哪个函数。

例如：

- 复位时跳到 `Reset_Handler`
- 硬 Fault 时跳到 `HardFault_Handler`
- SysTick 中断时跳到 `SysTick_Handler`

这个模板里为了简化，只显式列了前面一部分系统异常入口，剩余中断都先指向 `Default_Handler`。

这意味着：

- 当前模板的重点不是外设功能
- 而是先把最小启动流程打通

以后你要用串口、定时器、外部中断，再把相应中断处理函数补上即可。

---

### 10.3 `Reset_Handler` 到底做了什么

这是理解裸机程序启动的核心。

`Reset_Handler` 主要做 4 件事：

1. 把 `.data` 段初始值从 Flash 复制到 RAM
2. 把 `.bss` 段清零
3. 调用 `SystemInit()`
4. 调用 `main()`

为什么要复制 `.data`？

因为像这种有初始值的全局变量：

```c
int a = 123;
```

它的初始值通常存放在 Flash 里，但运行时要待在 RAM 中。

所以启动时必须把初值搬过去。

为什么要清零 `.bss`？

因为像这种没显式初始化的全局变量：

```c
int b;
```

按 C 语言规则，它启动后应该是 0。

这不是编译器帮你“神奇完成”的，而是启动代码手动清出来的。

如果你没有这一步，那么很多全局变量初始值都会错。

---

### 10.4 为什么 `main()` 返回后进入死循环

这里在 `main()` 调用后，后面跟的是：

- `hang: b hang`

意思是如果 `main()` 退出了，就一直原地循环。

因为裸机程序通常没有“返回给操作系统”这回事。

在 PC 程序里，`main()` 返回后可以回到操作系统。

但 MCU 裸机没有那个接收者，所以最安全的做法就是停在一个可控状态。

---

### 10.5 为什么很多中断处理函数都弱定义到 `Default_Handler`

文件里有很多：

- `.weak NMI_Handler`
- `.thumb_set NMI_Handler, Default_Handler`

这代表一种常见策略：

- 先给所有中断一个默认实现
- 如果你后来自己写了同名函数，就覆盖默认实现

这让模板非常方便扩展。

你只要在 C 文件中写：

```c
void SysTick_Handler(void)
{
    ...
}
```

就可以替换默认死循环版本。

---

## 11. 链接脚本 `.ld` 决定程序最终在芯片里怎么摆放

文件位置：

`D:\task\Cortex-Debug\text\linker\STM32F407ZG_FLASH.ld`

这个文件通常是初学者最陌生的部分。

但你可以把它理解成：

“给链接器的一张内存排版图。”

### 11.1 为什么需要这张图

因为 STM32 不是一整块统一内存。

它通常有不同用途的区域，例如：

- Flash：掉电不丢，用来放程序
- RAM：掉电丢失，用来放运行时数据
- CCMRAM：特殊用途 RAM

链接器必须知道：

- 代码放哪
- 常量放哪
- 已初始化数据运行时放哪
- 未初始化数据放哪
- 栈从哪开始

如果没有链接脚本，链接器就不知道怎么把程序“装进这颗芯片的地址空间”。

---

### 11.2 `MEMORY` 段在做什么

这里定义了：

- `FLASH` 起始地址 `0x08000000`，大小 `1024K`
- `RAM` 起始地址 `0x20000000`，大小 `128K`
- `CCMRAM` 起始地址 `0x10000000`，大小 `64K`

这本质上是在给链接器说：

“这颗芯片有这些内存区域，你只能在这些区域里安排内容。”

如果你以后换芯片，这一段通常必须改。

---

### 11.3 `_estack` 是什么

这里有：

`_estack = ORIGIN(RAM) + LENGTH(RAM);`

意思是：

- 栈顶初值设置在 RAM 的最高地址

这是 Cortex-M 裸机里常见做法。

因为栈一般向低地址增长，所以初始值放在 RAM 顶端。

然后启动向量表第一项就会用到它。

---

### 11.4 各个 section 是什么意思

链接脚本里最常见的几个段是：

- `.isr_vector`
- `.text`
- `.data`
- `.bss`

可以这样记：

- `.isr_vector`：中断向量表
- `.text`：代码和只读常量
- `.data`：有初值的全局/静态变量，运行在 RAM，初值来自 Flash
- `.bss`：无初值的全局/静态变量，运行在 RAM，启动时清零

这几个概念是理解 MCU 内存布局的基础。

---

### 11.5 `AT > FLASH` 是什么意思

`.data` 段的写法是：

- 运行位置在 `RAM`
- 加载位置在 `FLASH`

这就是 `> RAM AT > FLASH` 的含义。

它的逻辑是：

- 程序上电前，初值存在 Flash
- 启动后，复制到 RAM
- 真正运行时使用 RAM 里的那份

这正好对应前面启动文件里复制 `.data` 的动作。

所以启动文件和链接脚本其实是在互相配合，不是各写各的。

---

### 11.6 `_sidata`、`_sdata`、`_edata`、`_sbss`、`_ebss` 为什么重要

这些符号不是普通变量，而是“地址标签”。

它们告诉启动代码：

- `.data` 初值从哪开始读
- `.data` 在 RAM 里从哪开始写
- 写到哪结束
- `.bss` 从哪开始清零
- 清到哪结束

所以启动文件里那些名字，看起来像变量，其实更接近“内存边界标记”。

---

## 12. `openocd/stm32f407zg.cfg` 是下载调试连接说明

文件位置：

`D:\task\Cortex-Debug\text\openocd\stm32f407zg.cfg`

OpenOCD 是什么？

你可以把它理解成：

“电脑和调试器之间的翻译中间层。”

它知道：

- 你用什么下载器
- 用什么物理接口
- 对面是什么芯片家族

这里配置的是：

- `interface/stlink.cfg`
- `transport select hla_swd`
- `target/stm32f4x.cfg`

意思基本是：

- 下载器是 ST-Link
- 通信接口用 SWD
- 目标芯片属于 STM32F4 系列

### 12.1 为什么还要单独有 OpenOCD

因为 GDB 本身不直接会操作 ST-Link 硬件。

它更擅长做“调试协议层面”的事：

- 设置断点
- 读取内存
- 单步执行

而 OpenOCD 负责：

- 和 ST-Link 打交道
- 和 MCU 的 SWD/JTAG 打交道
- 把 GDB 的请求翻译成实际硬件操作

所以一个常见关系是：

- VS Code -> Cortex-Debug -> GDB -> OpenOCD -> ST-Link -> STM32

---

## 13. `.vscode/tasks.json` 是“常用命令按钮”

文件位置：

`D:\task\Cortex-Debug\.vscode\tasks.json`

它定义了 VS Code 里的几个任务：

- `CMake Configure`
- `Build Firmware`
- `Clean Firmware`
- `Flash Firmware (OpenOCD)`

### 13.1 为什么要有任务系统

因为如果没有任务系统，你每次都得手动敲一长串命令。

有了任务系统后，你可以：

- 一键配置
- 一键构建
- 一键清理
- 一键烧录

对于初学者尤其重要，因为它减少了“命令拼错”的可能。

---

### 13.2 `Build Firmware` 实际做了什么

它先依赖 `CMake Configure`，然后执行：

- `cmake --build --preset f407-debug`

这相当于：

1. 先确认构建系统文件生成好了
2. 再让 Ninja 按照规则去编译和链接

这说明 VS Code 自己并不懂如何编译 STM32。

它只是负责调用你已经定义好的构建流程。

---

### 13.3 `Flash Firmware (OpenOCD)` 在做什么

它会调用：

- `openocd`
- 读取 `openocd/stm32f407zg.cfg`
- 执行 `program ... verify reset exit`

这条命令的意思大致是：

1. 把程序下载进 Flash
2. 校验写入结果
3. 复位芯片
4. 退出 OpenOCD

所以它是一个“纯烧录任务”，不是调试会话。

---

## 14. `.vscode/launch.json` 是 VS Code 调试入口

文件位置：

`D:\task\Cortex-Debug\.vscode\launch.json`

这是 VS Code 调试系统最关键的配置文件之一。

这里定义了两个调试配置：

- `STM32F407ZG Debug (OpenOCD)`
- `STM32F407ZG Attach (OpenOCD)`

---

### 14.1 `launch` 和 `attach` 的区别

可以粗略理解为：

- `launch`：启动一个新的调试流程，通常会先构建、启动 OpenOCD、加载程序、跑到入口。
- `attach`：程序可能已经在板子上运行了，现在只附加上去调试。

这个工程里，最常用的是 `launch`。

---

### 14.2 关键字段分别是什么意思

几个最关键的字段是：

- `type: "cortex-debug"`
- `servertype: "openocd"`
- `serverpath: "openocd"`
- `configFiles: [...]`
- `device: "STM32F407ZG"`
- `gdbPath: "arm-none-eabi-gdb"`
- `executable: "...stm32_f407zg.elf"`
- `preLaunchTask: "Build Firmware"`

意思分别是：

- 使用 Cortex-Debug 插件提供的调试能力。
- 后端调试服务器选 OpenOCD。
- `openocd` 程序从哪里启动。
- OpenOCD 启动时要读哪个配置文件。
- 告诉调试器目标芯片型号。
- GDB 程序在哪里。
- 要调试的程序文件是哪一个 `.elf`。
- 启动调试前先自动构建。

也就是说，你按一次 F5，背后其实串起了好几个工具。

---

## 15. `build/` 目录里的产物分别是什么

核心输出目录是：

`D:\task\Cortex-Debug\text\build\f407-debug`

你最应该认识的是这些文件：

- `stm32_f407zg.elf`
- `stm32_f407zg.bin`
- `stm32_f407zg.hex`
- `stm32_f407zg.map`
- `compile_commands.json`
- `build.ninja`

### 15.1 `.elf`

最重要。

它包含：

- 机器码
- 符号表
- 调试信息
- 段信息

调试时通常离不开它。

### 15.2 `.bin`

最原始的二进制内容。

没有太多附加信息。

### 15.3 `.hex`

文本形式的固件，带地址。

很多烧录工具和生产流程喜欢它。

### 15.4 `.map`

程序布局说明书。

当你发现：

- 为什么 Flash 突然变大
- 为什么 RAM 不够了
- 某个库拉进来很多东西

就该看 `.map`。

### 15.5 `compile_commands.json`

记录每个源文件真实编译命令。

它常被：

- 代码补全工具
- 静态分析工具
- LSP

使用。

---

## 16. 从按下 F5 到芯片运行起来，完整发生了什么

这是全工程最重要的一段理解。

假设你在 VS Code 里按下 `F5`，选择：

- `STM32F407ZG Debug (OpenOCD)`

那么典型流程是：

1. VS Code 读取 `launch.json`
2. 发现 `preLaunchTask` 是 `Build Firmware`
3. 执行 `tasks.json` 里的构建任务
4. `cmake --build --preset f407-debug` 被调用
5. CMake 根据 preset 找到 Ninja 和交叉编译器
6. 编译 `main.c`、`system_stm32f4xx.c`、`startup_stm32f407zg.s`
7. 链接器按 `.ld` 脚本生成 `stm32_f407zg.elf`
8. `objcopy` 生成 `.bin` 和 `.hex`
9. Cortex-Debug 启动 `openocd`
10. OpenOCD 连接 ST-Link
11. OpenOCD 通过 SWD 连接 STM32F407ZG
12. Cortex-Debug 启动 `arm-none-eabi-gdb`
13. GDB 连接到 OpenOCD 提供的调试端口
14. GDB 加载 `stm32_f407zg.elf` 的符号信息
15. 程序被下载或重定位到目标板
16. MCU 复位
17. CPU 从向量表读取初始栈和 `Reset_Handler`
18. 启动代码复制 `.data`、清零 `.bss`
19. 调用 `SystemInit()`
20. 进入 `main()`
21. 你可以在 VS Code 中看到变量和断点效果

如果你能把这 21 步串起来，就已经理解了这个工程 80% 的逻辑。

---

## 17. 为什么这个模板“可迁移”

你之前要求的是：

- 先以 F407ZG 为例
- 但结构要能迁移到其他 STM32

这个工程是按这个目标搭的。

### 17.1 什么部分是“和芯片强绑定”的

以下内容通常和具体芯片强绑定：

- 启动文件
- 链接脚本
- OpenOCD target 配置
- CPU/FPU/ABI 选项
- 芯片宏定义
- 调试配置里的 `device`

换芯片时，这些地方常常要改。

---

### 17.2 什么部分是“基本通用”的

以下思路通常不变：

- CMake 管理构建
- 工具链文件指定 `arm-none-eabi-*`
- 用 Ninja 作为构建器
- 用 OpenOCD 作为下载调试后端
- 用 Cortex-Debug 驱动 VS Code 调试
- 用 `.elf` 作为调试输入

也就是说：

你迁移的不是“整套思路”，而是“芯片参数和底层边界条件”。

这就是模板化的价值。

---

## 18. 为什么模板目前没有 HAL、LL、标准库

这个问题很多人会问。

原因是这个工程当前追求的是“最小、透明、可理解”。

一旦你引入 HAL 或 CubeMX，工程会立刻多出：

- 大量头文件
- 大量初始化代码
- 时钟树配置
- 各种外设抽象层

这对做实际产品是有帮助的，但对“理解工程底层原理”反而会形成噪音。

当前模板故意不引入这些东西，是为了让你先看清最底层骨架。

当你已经理解这套骨架，再加 HAL/LL 就不会迷路。

---

## 19. 这个模板现在“故意没做”的事情

为了保持最小化，这个模板目前没有展开：

- 时钟树初始化
- GPIO 点灯
- 串口打印
- SysTick 心跳
- HAL/LL 驱动
- FreeRTOS
- CMSIS 设备头文件体系
- SVD 文件配置

这不是遗漏，而是取舍。

当前模板只负责证明一件事：

“这条构建、烧录、调试链路是通的。”

一旦基础链路通了，后面再叠功能才稳。

---

## 20. 初学者最容易混淆的几个概念

### 20.1 编译和链接不是一回事

编译是：

- 把单个源文件翻译成目标文件

链接是：

- 把所有目标文件和库拼起来
- 决定最终地址布局
- 生成完整可执行映像

---

### 20.2 `.elf` 不是“只是另一种固件格式”

`.elf` 很重要，因为它不只是机器码。

它还带：

- 函数名
- 变量名
- 行号映射
- 段布局

没有这些信息，调试体验会差很多。

---

### 20.3 OpenOCD 不是编译器

OpenOCD 不负责编译代码。

它负责：

- 连调试器
- 连芯片
- 烧录
- 调试转发

---

### 20.4 Cortex-Debug 不是烧录器本身

Cortex-Debug 是 VS Code 插件。

它相当于一个前台界面。

真正干活的是：

- GDB
- OpenOCD
- ST-Link

---

### 20.5 `main()` 不是程序最早执行的地方

在 STM32 裸机中，通常顺序是：

- 向量表
- `Reset_Handler`
- `SystemInit()`
- `main()`

这和 PC 程序的直觉很不一样。

---

## 21. 如果你想真正吃透这个工程，建议按这个顺序观察

第一轮，只看“宏观链路”：

1. 看目录结构。
2. 看 `main.c`。
3. 看 `launch.json`。
4. 理解按 F5 发生什么。

第二轮，理解“程序怎么启动”：

1. 看启动文件里的向量表。
2. 看 `Reset_Handler`。
3. 对照链接脚本里的 `_sidata`、`_sdata`、`_sbss` 等符号。

第三轮，理解“程序怎么被构建”：

1. 看 `CMakeLists.txt`
2. 看 `CMakePresets.json`
3. 看工具链文件
4. 看 `build/` 目录里的 `.elf`、`.map`

第四轮，理解“程序怎么被调试”：

1. 看 `tasks.json`
2. 看 `launch.json`
3. 看 `openocd/stm32f407zg.cfg`
4. 理解 GDB、OpenOCD、ST-Link 的关系

这样学，比一上来追每一行汇编更有效。

---

## 22. 用一句更通俗的话重新总结每个核心文件

- `main.c`
  - 你真正想让芯片做的事。

- `system_stm32f4xx.c`
  - 进入主程序前做一点必要系统准备。

- `startup_stm32f407zg.s`
  - 芯片复位后第一段执行的代码，负责搭好最基本运行环境。

- `STM32F407ZG_FLASH.ld`
  - 告诉链接器程序和数据应该放在芯片哪块内存里。

- `CMakeLists.txt`
  - 告诉构建系统：要编什么、怎么编、怎么链接、生成什么产物。

- `CMakePresets.json`
  - 给常用构建方案起名字。

- `arm-none-eabi-toolchain.cmake`
  - 告诉 CMake 这是 ARM 裸机交叉编译，不是普通 Windows 编译。

- `stm32f407zg.cfg`
  - 告诉 OpenOCD：我用 ST-Link，通过 SWD，连的是 STM32F4。

- `tasks.json`
  - 把配置、构建、烧录这些常用命令做成按钮。

- `launch.json`
  - 把调试流程接进 VS Code。

---

## 23. 你现在应该已经建立的心智模型

如果读到这里，你最好已经形成这样一个整体图景：

- 这不是一个普通 C 程序，而是一个给 STM32 裸机运行的程序。
- 裸机程序必须自己负责启动过程。
- 启动过程依赖启动文件和链接脚本配合。
- 构建过程由 CMake 统一管理。
- 交叉编译器负责生成 ARM 机器码。
- OpenOCD 负责连接下载器和芯片。
- GDB 负责调试控制。
- Cortex-Debug 负责把这一切整合进 VS Code。
- `.elf` 是调试核心文件。
- 当前模板故意保持极简，是为了让底层关系足够清楚。

如果这些点你已经能复述出来，就说明你已经不是“只是会点按钮”，而是开始真正理解这个工程了。

---

## 24. 下一步最合理的学习或扩展方向

当你已经理解当前模板后，建议下一步按这个顺序扩展：

1. 给 F407ZG 加一个 GPIO 点灯。
2. 加 `SysTick` 中断，让节拍来自定时器而不是忙等待。
3. 加串口输出，验证外设初始化链路。
4. 再决定是否引入 CMSIS 设备头文件和 HAL/LL。
5. 最后再考虑更复杂的库，比如 RTOS。

原因很简单：

先把“启动、构建、烧录、调试”理解扎实，再加外设和框架，成本最低。

---

## 25. 最终结论

这个工程的本质不是“写了一个会计数的 STM32 程序”，而是：

它搭起了一条完整的嵌入式开发通路。

这条通路包括：

- 源码组织
- 交叉编译
- 启动初始化
- 内存布局
- 固件生成
- 固件烧录
- 在线调试

而 `STM32F407ZG` 只是这条通路当前选择的第一个目标芯片。

你真正应该学会的，不只是“这个例子怎么跑”，而是：

“为什么一颗裸机 MCU 程序必须依靠这些组件协作，才能从源码变成芯片里真正跑起来的东西。”

一旦这套逻辑你理解了，后面换成别的 STM32，甚至换成别的 ARM Cortex-M 芯片，思路都还是同一套。
