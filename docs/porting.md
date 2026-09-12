# BitsButton 移植指南

## 1. 移植目标

BitsButton 的核心代码不直接依赖具体 MCU 的 GPIO、Timer 或 RTOS。

移植的主要工作，是把以下几个平台相关部分接入：

```text
┌──────────────────────────┐
│        应用层             │
└────────────┬─────────────┘
             │
┌────────────▼─────────────┐
│       BitsButton         │
│                          │
│ 状态机 / 组合键 / Buffer  │
└─────┬────────┬───────────┘
      │        │
      │        └──────────► Debug printf
      │
      └───────────────► Button Level
                         │
                         ▼
                       GPIO
```

核心原则：

> 不修改 BitsButton 核心状态机，只适配平台接口。

---

# 2. 需要移植的内容

通常只需要处理以下几项：

| 项目        | 是否必须 | 说明                                 |
| --------- | ---- | ---------------------------------- |
| GPIO 按键读取 | 是    | 实现 `bits_btn_read_button_level`    |
| 周期 Tick   | 是    | 周期调用 `bits_button_ticks()`         |
| 时间基准      | 是    | BitsButton 内部使用 tick 时间            |
| Debug 输出  | 否    | 可以传 `NULL`                         |
| RTOS      | 否    | 根据项目任务结构接入                         |
| 低功耗       | 否    | 恢复后调用 `bits_button_reset_states()` |

---

# 3. 添加核心文件

将：

```text
include/bits_button.h
src/bits_button.c
```

加入目标工程。

代码中：

```c
#include "bits_button.h"
```

确保编译器能够找到：

```text
include/
```

目录。

---

# 4. C 标准要求

当前实现使用：

```c
#include <stdatomic.h>
```

并使用：

```c
atomic_size_t
atomic_init()
atomic_load_explicit()
atomic_store_explicit()
atomic_fetch_add_explicit()
```

因此目标编译环境必须能够提供 C11 原子操作支持。

建议确认编译选项至少支持：

```text
C11
```

例如 GCC / Clang 工具链通常可以使用：

```text
-std=c11
```

具体选项根据目标工具链调整。

---

# 5. 标准头文件依赖

当前代码使用：

```c
#include "stdint.h"
#include "stdio.h"
#include "stdbool.h"
#include "string.h"
#include <stdatomic.h>
```

目标平台需要能够提供这些标准 C 头文件。

如果 MCU SDK 已经提供标准 C 运行库，一般不需要额外处理。

---

# 6. GPIO 按键读取

这是移植时最重要的平台接口。

BitsButton 定义：

```c
typedef uint8_t (*bits_btn_read_button_level)(
    struct button_obj_t *btn
);
```

需要根据：

```c
btn->key_id
```

确定实际 GPIO。

例如：

```c
static uint8_t read_button_level(button_obj_t *btn)
{
    switch (btn->key_id) {

    case KEY_POWER:
        return GPIO_ReadPin(KEY_POWER_PORT,
                            KEY_POWER_PIN);

    case KEY_PLAY:
        return GPIO_ReadPin(KEY_PLAY_PORT,
                            KEY_PLAY_PIN);

    default:
        return 0;
    }
}
```

这里的：

```c
GPIO_ReadPin()
```

只是平台 API 示例。

实际移植时替换为目标 MCU 的 GPIO 读取函数。

---

# 7. 有效电平

每一个 `button_obj_t` 都包含：

```c
uint8_t active_level : 1;
```

它表示：

> GPIO 读取值等于 `active_level` 时，认为按键处于有效/按下状态。

例如：

### 低电平有效

```text
未按下：1
按下：  0
```

配置：

```c
BITS_BUTTON_INIT(KEY_POWER, 0, &param)
```

### 高电平有效

```text
未按下：0
按下：  1
```

配置：

```c
BITS_BUTTON_INIT(KEY_POWER, 1, &param)
```

移植时不要默认所有硬件都是低电平有效。

---

# 8. Tick 移植

BitsButton 提供：

```c
void bits_button_ticks(void);
```

头文件要求该函数由定时器周期调用。

默认：

```c
#define BITS_BTN_TICKS_INTERVAL 5
```

即：

```text
5 ms
```

一次。

因此平台需要提供一个稳定的周期调用机制。

---

# 9. 使用硬件 Timer

例如目标 MCU 有一个 5 ms 周期定时器：

```c
void timer_5ms_callback(void)
{
    bits_button_ticks();
}
```

结构：

```text
Hardware Timer
      │
      │ 5 ms
      ▼
timer_5ms_callback()
      │
      ▼
bits_button_ticks()
```

不要在没有周期保证的情况下随意调用。

---

# 10. 使用 SysTick

如果系统已经有 SysTick：

```c
void SysTick_Handler(void)
{
    system_tick();

    bits_button_ticks();
}
```

但如果 SysTick 本身不是 5 ms 周期，就需要根据实际系统 Tick 重新安排调用频率。

例如系统 Tick 为：

```text
1 ms
```

则可以：

```c
static uint8_t cnt;

void SysTick_Handler(void)
{
    system_tick();

    if (++cnt >= 5) {
        cnt = 0;
        bits_button_ticks();
    }
}
```

这样得到：

```text
1 ms × 5 = 5 ms
```

调用周期。

---

# 11. 使用 RTOS

如果项目使用 RTOS，可以通过周期任务调用：

```c
void button_task(void *arg)
{
    while (1) {

        bits_button_ticks();

        osDelay(5);
    }
}
```

具体 API 根据 RTOS 修改。

需要注意：

> `bits_button_ticks()` 的实际调用周期应该与 `BITS_BTN_TICKS_INTERVAL` 的设计保持一致。

---

# 12. 时间参数

默认参数：

```c
#define BITS_BTN_DEBOUNCE_TIME_MS            (40)
#define BITS_BTN_SHORT_TIME_MS               (350)
#define BITS_BTN_LONG_PRESS_START_TIME_MS    (1000)
#define BITS_BTN_LONG_PRESS_PERIOD_TRIGER_MS (1000)
#define BITS_BTN_TIME_WINDOW_TIME_MS         (300)
```

单个按键还可以使用：

```c
bits_btn_obj_param_t
```

进行参数配置：

```c
typedef struct bits_btn_obj_param {
    uint16_t short_press_time_ms;
    uint16_t long_press_start_time_ms;
    uint16_t long_press_period_triger_ms;
    uint16_t time_window_time_ms;
} bits_btn_obj_param_t;
```

例如：

```c
static const bits_btn_obj_param_t param = {
    .short_press_time_ms = 300,
    .long_press_start_time_ms = 800,
    .long_press_period_triger_ms = 500,
    .time_window_time_ms = 300,
};
```

---

# 13. Tick 时间来源

当前核心代码内部维护：

```c
uint32_t btn_tick;
```

并通过：

```c
get_button_tick()
```

获取当前时间。

因此平台移植时，重点不是向 BitsButton 增加 MCU 时间 API，而是确保：

```text
bits_button_ticks()
       │
       ▼
内部 btn_tick
       │
       ▼
状态机时间判断
```

能够稳定运行。

---

# 14. Callback 移植

事件 callback 定义：

```c
typedef void (*bits_btn_result_callback)(
    struct button_obj_t *btn,
    struct bits_btn_result button_result
);
```

平台可以直接连接到应用层：

```c
static void button_result_callback(
    button_obj_t *btn,
    bits_btn_result_t result)
{
    app_button_event(
        result.key_id,
        result.event,
        result.key_value
    );
}
```

如果不需要 callback：

```c
bits_button_init(
    ...,
    NULL,
    ...
);
```

仍然可以通过事件缓冲区：

```c
bits_button_get_key_result()
```

获取结果。

---

# 15. Debug 输出移植

Debug 接口：

```c
typedef int (*bits_btn_debug_printf_func)(const char*, ...);
```

如果 MCU 工程已经实现：

```c
printf()
```

可以直接使用。

如果使用 UART：

```c
uart_printf()
```

可以：

```c
bits_button_init(
    ...,
    uart_printf
);
```

也可以使用：

```text
RTT
SWO
USB CDC
自定义日志系统
```

如果不需要日志：

```c
NULL
```

---

# 16. 环形缓冲区与并发

当前实现使用 C11 原子操作维护事件缓冲区：

```c
atomic_size_t read_idx;
atomic_size_t write_idx;
```

并使用：

```c
memory_order_relaxed
memory_order_consume
memory_order_acquire
memory_order_release
```

因此移植到 RTOS 时，需要明确：

```text
谁调用 bits_button_ticks()
谁读取 bits_button_get_key_result()
```

推荐结构：

```text
Timer / Button Task
       │
       ▼
bits_button_ticks()
       │
       ▼
Ring Buffer
       │
       ▼
Application Task
       │
       ▼
bits_button_get_key_result()
```

尽量保持：

```text
一个地方负责驱动状态机
一个地方负责消费事件
```

避免多个任务同时修改 BitsButton 内部状态。

---

# 17. 缓冲区大小

默认：

```c
#define BITS_BTN_BUFFER_SIZE 10
```

可以在编译前重新定义。

例如：

```c
#define BITS_BTN_BUFFER_SIZE 16
```

具体大小应该根据：

```text
事件产生速度
应用层读取速度
允许的事件积压数量
RAM
```

进行选择。

---

# 18. 低功耗系统移植

对于支持低功耗的 MCU，需要特别处理恢复过程。

系统进入低功耗前：

```text
停止 BitsButton 周期驱动
        │
        ▼
进入低功耗
```

恢复后：

```text
退出低功耗
    │
    ▼
bits_button_reset_states()
    │
    ▼
恢复 BitsButton Tick
```

调用：

```c
bits_button_reset_states();
```

会重新读取当前物理按键状态，并同步内部 mask。

同时清空事件缓冲区。

这样可以避免：

```text
进入低功耗前按下
        ↓
系统暂停
        ↓
恢复
        ↓
错误产生 Release / Press 事件
```

---

# 19. 不建议修改核心代码的部分

移植到新 MCU 时，优先修改：

```text
GPIO 读取
Timer / Tick
Debug 输出
工程编译配置
```

不建议为了适配 MCU 而直接修改：

```text
状态机
组合键算法
Ring Buffer
key_value 编码
```

如果平台差异确实需要修改核心代码，应尽量通过宏或独立平台层解决。

---

# 20. 不同平台的典型适配

## 裸机 MCU

```text
Timer ISR
    │
    ▼
bits_button_ticks()

Main Loop
    │
    ▼
bits_button_get_key_result()
```

---

## RTOS

```text
Button Task
    │
    ▼
bits_button_ticks()

Application Task
    │
    ▼
bits_button_get_key_result()
```

---

## Callback 架构

```text
Timer
 │
 ▼
bits_button_ticks()
 │
 ▼
BitsButton
 │
 ▼
callback
 │
 ▼
Application
```

---

# 21. 移植检查清单

完成移植后建议依次确认：

### 编译

```text
[ ] bits_button.h 可以正常包含
[ ] bits_button.c 可以正常编译
[ ] stdatomic.h 可用
[ ] C11 原子操作可用
[ ] 没有缺失标准头文件
```

### GPIO

```text
[ ] key_id 与实际 GPIO 对应
[ ] active_level 正确
[ ] 按下时 read_button_level() 返回有效电平
[ ] 松开时返回无效电平
```

### Tick

```text
[ ] bits_button_ticks() 被周期调用
[ ] 实际周期与配置一致
[ ] 系统 Tick 不会长时间暂停
```

### 功能

```text
[ ] 单击
[ ] 双击
[ ] 长按
[ ] 长按周期触发
[ ] 组合键
[ ] 异常输入
[ ] 状态复位
```

### Buffer

```text
[ ] bits_button_get_key_result() 能正常读取
[ ] buffer count 正常
[ ] overwrite count 正常
```

### 低功耗

```text
[ ] 进入低功耗前正确停止驱动
[ ] 恢复后调用 bits_button_reset_states()
[ ] 恢复后按键状态正常
```

---

# 22. 推荐移植顺序

不要一次把所有功能都接入。

建议：

```text
① 编译核心代码
       ↓
② 实现 GPIO 读取
       ↓
③ 实现 5 ms Tick
       ↓
④ 验证单键按下/释放
       ↓
⑤ 验证单击
       ↓
⑥ 验证双击
       ↓
⑦ 验证长按
       ↓
⑧ 验证组合键
       ↓
⑨ 验证 Buffer
       ↓
⑩ 验证低功耗恢复
```

如果第 ④ 步就出现问题，不要继续排查组合键。

---

# 23. 移植问题定位顺序

出现按键异常时，推荐按照以下顺序排查：

```text
GPIO 电平
    ↓
active_level
    ↓
bits_button_ticks() 调用周期
    ↓
bits_button_init() 返回值
    ↓
button 参数
    ↓
key_id
    ↓
组合键配置
    ↓
事件 Buffer
    ↓
Application
```

这样可以把问题快速定位到：

```text
硬件层
平台适配层
BitsButton 核心层
应用层
```

---

# 24. 最小移植模型

一个最小平台适配只需要完成：

```c
static uint8_t read_button_level(button_obj_t *btn)
{
    return platform_gpio_read(btn->key_id);
}
```

然后：

```c
bits_button_init(
    btns,
    ARRAY_SIZE(btns),
    NULL,
    0,
    read_button_level,
    NULL,
    NULL
);
```

再提供周期驱动：

```c
void button_tick_5ms(void)
{
    bits_button_ticks();
}
```

这样即可建立最基本的：

```text
GPIO
 ↓
BitsButton
 ↓
Ring Buffer
```

处理链路。

---

# 25. 总结

BitsButton 的移植核心可以归纳为三件事：

```text
① 告诉 BitsButton 怎么读取按键
       ↓
bits_btn_read_button_level

② 定期驱动 BitsButton
       ↓
bits_button_ticks()

③ 从 BitsButton 获取事件
       ↓
callback
或
bits_button_get_key_result()
```

因此，移植到不同 MCU、SDK 或 RTOS 时，尽量保持：

```text
                  ┌──────────────┐
GPIO ────────────►│              │
                  │ BitsButton   │───► Application
Timer ───────────►│              │
                  │              │
Debug ───────────►│              │
                  └──────────────┘
```

核心代码保持不变，平台差异集中在外围适配代码中。

这样可以让 BitsButton 更适合作为一个独立的嵌入式组件，在不同 MCU、RTOS 和项目中重复使用。
