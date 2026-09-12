<h1 align="center">BitsButton</h1>

<p align="center">
  面向嵌入式系统的轻量级按键检测组件
</p>
<p align="center">
  项目来源开源，<a href="https://github.com/530china/BitsButton">原项目链接</a>。为了方便后续移植，当前已经过裁剪。
</p>
---

## 一、简介 👋

**BitsButton** 是一款面向嵌入式系统设计的轻量级按键检测组件。

通过二进制位序列记录按键状态，可以直观地描述和识别单击、双击、连击、长按以及复杂按键序列。

同时支持单键和组合按键，并提供 Callback 和 Poll 两种事件获取方式，方便集成到不同的嵌入式软件架构中。

组件本身尽量减少对具体硬件平台和工程框架的依赖，适用于 ARM、RISC-V、nRF 以及其他 MCU / SoC 平台。

---

## 二、工程结构

```text
BitsButton/
├── include/
│   └── bits_button.h          # 核心头文件
├── src/
│   └── bits_button.c          # 核心实现
├── examples/
│   ├── example_callback.c     # Callback 使用示例
│   └── example_poll.c         # Poll 使用示例
├── cases/
│   ├── basic/                 # 基础功能测试
│   ├── combo/                 # 组合按键测试
│   ├── edge/                  # 边界及异常情况测试
│   └── performance/           # 性能相关测试
├── docs/
│   ├── porting.md             # 移植说明
│   └── usage.md               # 使用说明
├── README.md                  # 本文件
├── LICENSE
├── .gitignore
└── .gitattributes
```

### 目录职责

| 目录          | 作用              |
| ----------- | --------------- |
| `include/`  | 对外提供的头文件        |
| `src/`      | BitsButton 核心实现 |
| `examples/` | 基础使用示例          |
| `cases/`    | 功能、边界和性能测试用例    |
| `docs/`     | 使用及移植相关文档       |

BitsButton 核心代码仅包含：

```text
include/bits_button.h
src/bits_button.c
```

因此可以直接加入现有工程中使用。

---

## 三、主要特性 🌱

### 1. 按键事件回溯

BitsButton 使用二进制位序列记录按键状态：

* `1`：按下
* `0`：松开

例如：

| 位序列            | 含义      |
| -------------- | ------- |
| `0b0`          | 未按下     |
| `0b010`        | 单击      |
| `0b01010`      | 双击      |
| `0b01010...n`  | n 连击    |
| `0b011`        | 长按开始    |
| `0b0111`       | 长按保持    |
| `0b01110`      | 长按结束    |
| `0b01011`      | 短按后长按   |
| `0b0101011`    | 双击后长按   |
| `0b01010..n11` | n 连击后长按 |

这种表示方式可以将复杂的按键行为转换为直观的位序列，方便进行状态识别和调试。

---

### 2. 高级按键检测

| 功能   | 描述                      |
| ---- | ----------------------- |
| 单键检测 | 支持按下、抬起、单击、双击、连击、长按     |
| 组合按键 | 支持多个按键组合                |
| 序列识别 | 支持单击→长按、单击→长按→双击等复杂操作序列 |
| 长按检测 | 支持长按开始、保持和结束            |
| 事件回溯 | 保存按键历史状态并生成对应事件         |

---

### 3. 模块化设计

BitsButton 将按键检测逻辑与具体硬件平台分离。

核心组件负责：

```text
按键状态
    ↓
状态转换
    ↓
按键序列
    ↓
事件识别
    ↓
事件输出
```

底层硬件只需要提供按键状态获取接口，即可将核心逻辑集成到不同平台。

因此可以将：

```text
MCU GPIO
按键扫描器
触摸按键
矩阵键盘
其他输入设备
```

与 BitsButton 核心逻辑进行组合。

---

### 4. Callback / Poll 两种事件获取方式

根据应用架构，可以选择：

**Callback**

按键事件产生后主动调用用户注册的回调函数。

适合事件驱动的软件架构。

**Poll**

由应用程序主动获取按键事件。

适合主循环、任务轮询等软件架构。

示例：

```text
examples/
├── example_callback.c
└── example_poll.c
```

---

## 四、核心数据结构

### 单按键对象

```c
typedef struct button_obj_t {
    uint8_t active_level : 1;
    uint8_t current_state : 3;
    uint8_t last_state : 3;
    uint16_t key_id;
    uint16_t long_press_period_trigger_cnt;
    uint32_t state_entry_time;
    state_bits_type_t state_bits;
    const bits_btn_obj_param_t *param;
} button_obj_t;
```

### 组合按键对象

```c
typedef struct {
    uint8_t key_count;
    uint16_t key_single_ids[BITS_BTN_MAX_COMBO_KEYS];
    button_obj_t btn;
    button_mask_type_t combo_mask;
    uint8_t suppress;
} button_obj_combo_t;
```

### 按键事件结果

```c
typedef struct {
    uint16_t key_id;
    btn_state_t event;
    uint16_t long_press_period_trigger_cnt;
    key_value_type_t key_value;
} bits_btn_result_t;
```

其中比较重要的字段：

| 字段                              | 用途         |
| ------------------------------- | ---------- |
| `state_bits`                    | 保存按键历史状态位图 |
| `key_id`                        | 标识具体按键     |
| `event`                         | 按键事件类型     |
| `key_value`                     | 按键序列对应的位图  |
| `long_press_period_trigger_cnt` | 长按周期触发次数   |

---

## 五、快速开始 🚀

### 1. 添加源码

将以下两个文件加入你的嵌入式工程：

```text
BitsButton/include/bits_button.h
BitsButton/src/bits_button.c
```

例如：

```text
your_project/
├── app/
├── driver/
├── include/
├── src/
│   └── bits_button.c
└── ...
```

或者保留 BitsButton 原有目录结构，并将：

```text
BitsButton/include
```

加入编译器的头文件搜索路径。

---

### 2. 包含头文件

```c
#include "bits_button.h"
```

---

### 3. 提供按键状态读取接口

BitsButton 不直接依赖具体 GPIO 驱动。

应用层需要根据实际硬件实现按键状态读取。

例如：

```c
uint8_t read_key_state(uint16_t key_id)
{
    /*
     * 根据实际 MCU / GPIO 驱动
     * 返回对应按键当前状态
     */

    return gpio_read(key_id);
}
```

具体实现方式取决于目标 MCU 和硬件平台。

---

### 4. 初始化 BitsButton

根据实际工程配置按键参数，然后调用初始化接口。

例如：

```c
bits_button_init(
    btns,
    ARRAY_SIZE(btns),
    btns_combo,
    ARRAY_SIZE(btns_combo),
    read_key_state,
    bits_btn_result_cb,
    my_log_printf
);
```

其中：

| 参数                   | 作用       |
| -------------------- | -------- |
| `btns`               | 单按键配置    |
| `btns_combo`         | 组合按键配置   |
| `read_key_state`     | 按键状态读取函数 |
| `bits_btn_result_cb` | 按键事件回调   |
| `my_log_printf`      | 日志输出函数   |

---

### 5. 选择事件处理方式

#### Callback

参考：

```text
examples/example_callback.c
```

适用于事件产生后立即处理的场景。

#### Poll

参考：

```text
examples/example_poll.c
```

适用于主循环或任务中主动查询事件的场景。

---

## 六、编译要求

BitsButton 核心代码使用 C11 标准中的相关语言和库特性。

建议使用支持 **C11** 的编译器。

核心代码涉及：

```c
_Atomic
```

以及：

```c
#include <stdatomic.h>
```

如果目标平台的编译器或 C 库对 C11 原子操作支持有限，需要根据具体平台进行适配。

---

## 七、移植新平台

BitsButton 的核心代码尽量与具体硬件解耦。

移植时主要关注以下几个部分：

```text
              BitsButton
                  │
        ┌─────────┴─────────┐
        │                   │
   核心按键逻辑          平台适配
        │                   │
        │             GPIO / Timer
        │             原子操作支持
        │             编译器支持
        │
        └─────────┬─────────┘
                  │
              应用程序
```

详细说明：

```text
docs/porting.md
```

---

## 八、使用说明

包括：

* 按键参数配置
* 单键使用
* 组合键使用
* Callback 使用方式
* Poll 使用方式
* 长按及连击配置
* 事件处理

详细说明：

```text
docs/usage.md
```

---

## 九、测试用例 🧪

项目保留了具有参考价值的功能测试用例。

测试用例按照功能进行分类：

```text
cases/
├── basic/
│   ├── test_buffer_operations.c
│   ├── test_initialization.c
│   ├── test_single_operations.c
│   └── test_state_reset.c
│
├── combo/
│   ├── test_advanced_combo.c
│   └── test_combo_buttons.c
│
├── edge/
│   ├── test_edge_cases.c
│   ├── test_error_handling.c
│   └── test_state_machine_edge.c
│
└── performance/
    └── test_performance.c
```

这些文件主要用于：

* 理解 BitsButton 的内部行为；
* 验证修改后的代码；
* 移植到新平台时进行功能参考；
* 排查状态机相关问题。

> `cases/` 中的文件是测试用例集合，不依赖原项目的完整测试框架。

---

## 十、License

本项目遵循项目根目录下 `LICENSE` 文件中的许可协议。
