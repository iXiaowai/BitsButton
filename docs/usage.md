# BitsButton 使用说明

## 1. 简介

BitsButton 是一个面向嵌入式系统的按键状态检测组件。

组件将物理按键输入转换为按键状态事件，并通过状态位序列 `key_value` 表示连续的按键动作。

典型处理流程：

```text
┌──────────────┐
│ GPIO / 外部输入 │
└──────┬───────┘
       │
       ▼
read_button_level()
       │
       ▼
┌─────────────────┐
│    BitsButton   │
│                 │
│  状态机          │
│  消抖            │
│  单键检测        │
│  组合键检测      │
│  长按检测        │
│  事件缓冲        │
└──────┬──────────┘
       │
       ├──────────────► callback
       │
       ▼
bits_button_get_key_result()
       │
       ▼
   应用层处理
```

BitsButton 本身不直接操作 MCU 的 GPIO 外设，而是通过用户提供的按键电平读取函数与具体硬件连接。

---

# 2. 文件组成

推荐的工程结构：

```text
BitsButton/
├── include/
│   └── bits_button.h
├── src/
│   └── bits_button.c
├── examples/
│   ├── example_callback.c
│   └── example_poll.c
├── cases/
│   ├── basic/
│   ├── combo/
│   ├── edge/
│   └── performance/
├── docs/
│   ├── porting.md
│   └── usage.md
├── README.md
├── LICENSE
├── .gitignore
└── .gitattributes
```

实际集成时，核心代码只有：

```text
include/bits_button.h
src/bits_button.c
```

然后由用户的工程负责：

```text
GPIO
时间基准
任务 / 定时器
日志输出
```

---

# 3. 基本数据类型

BitsButton 使用以下类型表示按键 ID 和状态位：

```c
typedef uint32_t key_value_type_t;
typedef uint32_t state_bits_type_t;
typedef state_bits_type_t button_mask_type_t;
```

其中：

* `key_value_type_t`：按键状态序列。
* `state_bits_type_t`：状态位序列。
* `button_mask_type_t`：按键组合使用的 bit mask。

---

# 4. 按键状态

组件定义了以下按键状态：

```c
typedef enum {
    BTN_STATE_IDLE,
    BTN_STATE_PRESSED,
    BTN_STATE_LONG_PRESS,
    BTN_STATE_RELEASE,
    BTN_STATE_RELEASE_WINDOW,
    BTN_STATE_FINISH
} bits_btn_state_t;
```

状态的基本含义：

| 状态                         | 含义         |
| -------------------------- | ---------- |
| `BTN_STATE_IDLE`           | 空闲状态       |
| `BTN_STATE_PRESSED`        | 按键已经按下     |
| `BTN_STATE_LONG_PRESS`     | 已进入长按状态    |
| `BTN_STATE_RELEASE`        | 检测到释放      |
| `BTN_STATE_RELEASE_WINDOW` | 释放后的时间窗口   |
| `BTN_STATE_FINISH`         | 当前按键动作处理完成 |

---

# 5. 默认时间参数

头文件中定义了以下默认参数：

| 参数                                     |     默认值 | 含义                         |
| -------------------------------------- | ------: | -------------------------- |
| `BITS_BTN_TICKS_INTERVAL`              |    5 ms | `bits_button_ticks()` 调用周期 |
| `BITS_BTN_DEBOUNCE_TIME_MS`            |   40 ms | 按键消抖时间                     |
| `BITS_BTN_SHORT_TIME_MS`               |  350 ms | 短按时间参数                     |
| `BITS_BTN_LONG_PRESS_START_TIME_MS`    | 1000 ms | 长按开始时间                     |
| `BITS_BTN_LONG_PRESS_PERIOD_TRIGER_MS` | 1000 ms | 长按周期触发时间                   |
| `BITS_BTN_TIME_WINDOW_TIME_MS`         |  300 ms | 时间窗口                       |

其中 `BITS_BTN_TICKS_INTERVAL` 可以通过宏重新定义：

```c
#define BITS_BTN_TICKS_INTERVAL 10
```

但实际系统需要保证 `bits_button_ticks()` 的调用周期与配置一致。

---

# 6. 初始化单个按键

单个按键使用 `button_obj_t` 描述：

```c
typedef struct button_obj_t {
    uint8_t  active_level : 1;
    uint8_t current_state : 3;
    uint8_t last_state : 3;
    uint16_t key_id;
    uint16_t long_press_period_trigger_cnt;
    uint32_t state_entry_time;
    state_bits_type_t state_bits;
    const bits_btn_obj_param_t *param;
} button_obj_t;
```

可以使用：

```c
BITS_BUTTON_INIT(_key_id, _active_level, _param)
```

进行初始化。

例如：

```c
button_obj_t btns[] = {
    BITS_BUTTON_INIT(0, 0, NULL),
    BITS_BUTTON_INIT(1, 0, NULL),
};
```

其中：

| 参数              | 含义      |
| --------------- | ------- |
| `_key_id`       | 当前按键 ID |
| `_active_level` | 按键有效电平  |
| `_param`        | 按键检测参数  |

---

# 7. 自定义按键检测参数

如果需要修改单个按键的时间参数，可以使用：

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
static const bits_btn_obj_param_t button_param = {
    .short_press_time_ms = 350,
    .long_press_start_time_ms = 1000,
    .long_press_period_triger_ms = 1000,
    .time_window_time_ms = 300,
};
```

然后：

```c
button_obj_t btns[] = {
    BITS_BUTTON_INIT(0, 0, &button_param),
};
```

如果 `param == NULL`，当前按键状态机不会继续进行检测。

---

# 8. 按键电平读取接口

BitsButton 不直接读取 GPIO。

需要用户提供：

```c
typedef uint8_t (*bits_btn_read_button_level)(
    struct button_obj_t *btn
);
```

例如：

```c
static uint8_t read_button_level(button_obj_t *btn)
{
    switch (btn->key_id) {
    case 0:
        return gpio_read(KEY0_GPIO);
    case 1:
        return gpio_read(KEY1_GPIO);
    default:
        return 0;
    }
}
```

这里的 `gpio_read()` 只是示例，实际工程中应该替换成 MCU 对应的 GPIO API。

注意：

```c
active_level
```

必须与硬件实际有效电平保持一致。

例如：

```text
GPIO = 0 → 按下
GPIO = 1 → 松开
```

那么：

```c
.active_level = 0
```

---

# 9. 初始化 BitsButton

核心初始化函数：

```c
int32_t bits_button_init(
    button_obj_t* btns,
    uint16_t btns_cnt,
    button_obj_combo_t *btns_combo,
    uint16_t btns_combo_cnt,
    bits_btn_read_button_level read_button_level_func,
    bits_btn_result_callback bits_btn_result_cb,
    bits_btn_debug_printf_func bis_btn_debug_printf
);
```

基本调用方式：

```c
int ret;

ret = bits_button_init(
    btns,
    ARRAY_SIZE(btns),
    btns_combo,
    ARRAY_SIZE(btns_combo),
    read_button_level,
    button_result_callback,
    debug_printf
);
```

初始化返回值：

|  返回值 | 含义             |
| ---: | -------------- |
|  `0` | 初始化成功          |
| `-1` | 组合键中存在无效的单键 ID |
| `-2` | 输入参数无效         |
| `-3` | 组合键数量超过最大限制    |

其中，当没有组合键时，可以传：

```c
NULL
```

并将：

```c
btns_combo_cnt = 0
```

---

# 10. 组合按键

组合按键使用：

```c
button_obj_combo_t
```

描述：

```c
typedef struct button_obj_combo {
    uint8_t suppress;
    uint8_t key_count;
    uint16_t *key_single_ids;
    button_mask_type_t combo_mask;

    button_obj_t btn;
} button_obj_combo_t;
```

可以通过：

```c
BITS_BUTTON_COMBO_INIT(
    _key_id,
    _active_level,
    _param,
    _key_single_ids,
    _key_count,
    _single_key_suppress
)
```

初始化。

例如，一个组合键由按键 `0` 和 `1` 构成：

```c
uint16_t combo_keys[] = {
    0,
    1,
};

button_obj_combo_t btns_combo[] = {
    BITS_BUTTON_COMBO_INIT(
        10,
        0,
        &button_param,
        combo_keys,
        ARRAY_SIZE(combo_keys),
        1
    ),
};
```

初始化过程中，BitsButton 会根据组合键中的单键 ID 生成：

```c
combo_mask
```

如果组合键引用了不存在的单键 ID，则：

```c
bits_button_init()
```

返回：

```text
-1
```

---

# 11. 组合键数量限制

默认：

```c
#define BITS_BTN_MAX_COMBO_BUTTONS 8
```

因此最多支持 8 个组合按键对象。

如果：

```c
btns_combo_cnt > BITS_BTN_MAX_COMBO_BUTTONS
```

初始化会返回：

```text
-3
```

---

# 12. BitsButton 时间驱动

BitsButton 不是自行产生系统时钟，而是需要外部周期调用：

```c
void bits_button_ticks(void);
```

头文件要求该函数由定时器以默认 5 ms 间隔反复调用。

典型方式：

```text
定时器 / SysTick
      │
      │ 5 ms
      ▼
bits_button_ticks()
      │
      ▼
更新所有按键状态
```

例如：

```c
void timer_callback(void)
{
    bits_button_ticks();
}
```

实际项目中应根据 MCU 和 RTOS 的定时机制进行适配。

---

# 13. 事件结果

BitsButton 使用：

```c
bits_btn_result_t
```

表示检测结果：

```c
typedef struct bits_btn_result {
    uint8_t event;
    uint16_t key_id;
    uint16_t long_press_period_trigger_cnt;
    state_bits_type_t key_value;
} bits_btn_result_t;
```

字段含义：

| 字段                              | 含义         |
| ------------------------------- | ---------- |
| `event`                         | 当前事件状态     |
| `key_id`                        | 产生事件的按键 ID |
| `long_press_period_trigger_cnt` | 长按周期触发计数   |
| `key_value`                     | 当前按键状态位序列  |

---

# 14. 按键状态位序列

BitsButton 使用 bit sequence 表示按键动作。

例如：

```text
0b010
```

表示：

```c
#define BITS_BTN_SINGLE_CLICK_KV 0b010
```

双击：

```text
0b01010
```

对应：

```c
#define BITS_BTN_DOUBLE_CLICK_KV 0b01010
```

其它预定义序列：

```text
0b01011   单击后长按
0b0101011 双击后长按

0b011     长按开始
0b0111    长按保持
0b01110   长按结束
```

对应宏：

```c
BITS_BTN_SINGLE_CLICK_KV
BITS_BTN_DOUBLE_CLICK_KV
BITS_BTN_SINGLE_CLICK_THEN_LONG_PRESS_KV
BITS_BTN_DOUBLE_CLICK_THEN_LONG_PRESS_KV
BITS_BTN_LONG_PRESEE_START_KV
BITS_BTN_LONG_PRESEE_HOLD_KV
BITS_BTN_LONG_PRESEE_HOLD_END_KV
```

应用层可以根据：

```c
result.key_value
```

判断完整按键动作。

---

# 15. Callback 模式

初始化时可以注册：

```c
bits_btn_result_callback
```

其定义为：

```c
typedef void (*bits_btn_result_callback)(
    struct button_obj_t *btn,
    struct bits_btn_result button_result
);
```

例如：

```c
static void button_result_callback(
    button_obj_t *btn,
    bits_btn_result_t result)
{
    printf(
        "key=%d event=%d value=0x%lx\n",
        result.key_id,
        result.event,
        (unsigned long)result.key_value
    );
}
```

然后：

```c
bits_button_init(
    btns,
    ARRAY_SIZE(btns),
    btns_combo,
    ARRAY_SIZE(btns_combo),
    read_button_level,
    button_result_callback,
    NULL
);
```

这种方式适合需要在按键事件产生时立即进行处理的场景。

---

# 16. Poll 模式

除了 callback，BitsButton 还提供事件缓冲区。

应用层可以主动读取：

```c
bool bits_button_get_key_result(bits_btn_result_t *result);
```

例如：

```c
bits_btn_result_t result;

while (bits_button_get_key_result(&result)) {
    printf(
        "key=%d event=%d value=0x%lx\n",
        result.key_id,
        result.event,
        (unsigned long)result.key_value
    );
}
```

因此可以采用：

```text
按键状态机
     │
     ▼
环形缓冲区
     │
     ▼
应用任务周期读取
```

这种方式比较适合 RTOS task 或主循环统一处理输入事件的架构。

---

# 17. 环形缓冲区

默认：

```c
#define BITS_BTN_BUFFER_SIZE 10
```

内部使用原子变量维护：

```c
atomic_size_t read_idx;
atomic_size_t write_idx;
```

因此事件不会直接依赖 callback 才能被应用层获取。

可以查询缓冲区状态：

```c
bits_btn_is_buffer_empty();
bits_btn_is_buffer_full();
get_bits_btn_buffer_count();
```

还可以获取缓冲区覆盖次数：

```c
get_bits_btn_overwrite_count();
```

---

# 18. 缓冲区满处理

事件写入使用环形缓冲区。

当使用覆盖方式写入时，如果缓冲区已经满，旧数据会被向前覆盖，并增加：

```c
overwrite_count
```

因此调试过程中如果发现按键事件丢失，可以检查：

```c
get_bits_btn_overwrite_count()
```

以及：

```c
get_bits_btn_buffer_count()
```

---

# 19. 低功耗恢复

系统进入低功耗模式之前，按键状态可能停留在某一个中间状态。

从低功耗恢复后，可以调用：

```c
bits_button_reset_states();
```

该函数会：

1. 将单键状态恢复到 `BTN_STATE_IDLE`；
2. 清除状态 bit sequence；
3. 清除长按周期计数；
4. 将组合键状态恢复；
5. 重新读取当前物理按键状态；
6. 同步 `current_mask` 和 `last_mask`；
7. 清空事件缓冲区。

典型流程：

```text
进入低功耗
    │
    ▼
系统暂停
    │
    ▼
系统恢复
    │
    ▼
bits_button_reset_states()
    │
    ▼
重新开始 bits_button_ticks()
```

---

# 20. Debug 日志

初始化时可以传入：

```c
bits_btn_debug_printf_func
```

定义：

```c
typedef int (*bits_btn_debug_printf_func)(const char*, ...);
```

例如：

```c
bits_button_init(
    btns,
    ARRAY_SIZE(btns),
    btns_combo,
    ARRAY_SIZE(btns_combo),
    read_button_level,
    button_result_callback,
    printf
);
```

如果不需要 debug 输出，可以传：

```c
NULL
```

---

# 21. 使用示例

完整的应用结构可以概括为：

```c
static uint8_t read_button_level(button_obj_t *btn)
{
    /*
     * 根据 btn->key_id 读取对应 GPIO
     */
}

static void button_result_callback(
    button_obj_t *btn,
    bits_btn_result_t result)
{
    /*
     * 应用层处理按键事件
     */
}

void button_init(void)
{
    bits_button_init(
        btns,
        ARRAY_SIZE(btns),
        btns_combo,
        ARRAY_SIZE(btns_combo),
        read_button_level,
        button_result_callback,
        NULL
    );
}

void button_timer_5ms(void)
{
    bits_button_ticks();
}
```

推荐先从：

```text
单键
 ↓
短按
 ↓
双击
 ↓
长按
 ↓
组合键
 ↓
低功耗恢复
```

逐步验证，而不是一开始就配置复杂组合键。

---

# 22. 测试用例目录

当前仓库保留的测试用例位于：

```text
cases/
├── basic/
├── combo/
├── edge/
└── performance/
```

### basic

基础行为测试：

```text
test_buffer_operations.c
test_initialization.c
test_single_operations.c
test_state_reset.c
```

### combo

组合键测试：

```text
test_advanced_combo.c
test_combo_buttons.c
```

### edge

边界和异常情况：

```text
test_edge_cases.c
test_error_handling.c
test_state_machine_edge.c
```

### performance

性能相关：

```text
test_performance.c
```

这些 `cases` 是验证代码行为的参考，而不是 BitsButton 运行时必须依赖的组件。

---

# 23. 常见问题

## 23.1 按键完全没有反应

优先检查：

```text
bits_button_init()
        ↓
read_button_level()
        ↓
active_level
        ↓
bits_button_ticks()
```

尤其确认 `bits_button_ticks()` 是否按照预期周期持续运行。

---

## 23.2 按下和释放状态反了

检查：

```c
active_level
```

例如硬件：

```text
按下 = 0
释放 = 1
```

则：

```c
active_level = 0
```

---

## 23.3 长按时间不准确

检查：

```c
BITS_BTN_TICKS_INTERVAL
```

以及实际调用：

```c
bits_button_ticks()
```

的周期是否一致。

---

## 23.4 组合键无法触发

检查：

1. `key_single_ids` 中的 ID 是否存在；
2. `key_count` 是否正确；
3. `btns_combo_cnt` 是否超过限制；
4. `active_level` 是否正确；
5. `bits_button_ticks()` 是否持续运行。

如果组合键引用了不存在的单键 ID：

```c
bits_button_init()
```

会返回：

```text
-1
```

---

## 23.5 事件被覆盖

检查：

```c
get_bits_btn_overwrite_count()
```

如果计数持续增加，说明应用层读取事件的速度跟不上事件产生速度。

可以进一步检查：

```c
get_bits_btn_buffer_count()
```

以及：

```c
BITS_BTN_BUFFER_SIZE
```

---

# 24. 推荐集成方式

对于嵌入式工程，推荐：

```text
application/
├── button_app.c
├── button_app.h
│
└── components/
    └── bits_button/
        ├── include/
        │   └── bits_button.h
        └── src/
            └── bits_button.c
```

BitsButton 只负责：

```text
按键状态检测
按键组合识别
按键事件生成
事件缓冲
```

而以下内容由平台层负责：

```text
GPIO
Timer
RTOS
日志
低功耗
```

这样可以保持组件的可移植性。
