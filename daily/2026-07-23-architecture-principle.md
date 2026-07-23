# ADR：嵌入式 C 的接口设计——opaque pointer 是唯一的墙

## 主题：嵌入式 C 语言中的接口设计与信息隐藏（轮换 #4）

---

科(D)，早。

我们聊一个看起来简单、但几乎每个嵌入式项目都会翻车的东西：**模块之间的接口**。

## 反模式：头文件里裸奔

大多数嵌入式 C 项目的模块接口长这样：

```c
/* sensor.h */
typedef struct {
    float temperature;
    float humidity;
    int   raw_adc[4];
    uint8_t calibration_version;
} SensorReading_t;

SensorReading_t sensor_read(void);
void sensor_calibrate(uint8_t version);
```

一眼看上去很清爽。但问题出在——**任何包含这个头文件的文件，都能直接读写 `SensorReading_t` 的每一个字段**。

调用方可以：
- 直接修改 `temperature` 而不经过校准
- 绕过 `sensor_calibrate()` 把 `calibration_version` 改成任意值
- 在多个任务间传递 `SensorReading_t` 的副本，导致缓存一致性问题

**这不是封装，这是裸奔。**

## 正确姿势：opaque pointer + 访问器

```c
/* sensor.h */
typedef struct SensorCtx SensorCtx_t;

SensorCtx_t*  sensor_init(void);
void          sensor_destroy(SensorCtx_t* ctx);
float         sensor_get_temp(const SensorCtx_t* ctx);
float         sensor_get_hum(const SensorCtx_t* ctx);
void          sensor_calibrate(SensorCtx_t* ctx, uint8_t ver);
```

注意几个关键设计：

1. **`struct SensorCtx` 只有声明没有定义**——编译器知道这是一个不完整的类型（incomplete type），任何外部代码都无法知道它的大小，也就不能栈上分配或按值传递。

2. **所有访问走函数**——即使只是读一个 `float` 字段，也必须通过 `sensor_get_temp()`。这意味着你随时可以在 getter 里加滤波、加日志、加断言，而**不需要改一行调用方代码**。

3. **`const` 限定保护读操作**——`sensor_get_temp(const SensorCtx_t* ctx)` 告诉编译器：这个函数承诺不改数据。如果调用方不小心传了不该传的指针，编译期就能拦截。

## 为什么这在意嵌入式系统里特别重要

在 PC 后端，字段被直接改掉顶多是一个 bug，修一下就行。但在嵌入式系统里：

- **共享状态 = 并发 bug 的温床**。FreeRTOS 下两个任务同时碰同一个结构体，没有 volatile 和原子保护，编译器会做各种"优化"让你的逻辑完全失效。
- **内存布局变化 = 二进制不兼容**。如果你把传感器固件拆成 .so 一样的独立 flash 分区，头文件暴露了内部结构，升级一个模块另一个就挂了。
- **调试成本呈指数增长**。字段被意外修改时，在裸奔模式下你连"谁改的"都不知道——因为任何文件都能改。

## 可操作的实践建议

**给你的每个模块写一个 `*_internal.h`**。这个文件只在模块的 `.c` 文件内 `#include`，对外只暴露 `.h` 里的 opaque 接口。

```c
/* sensor.c */
#include "sensor_internal.h"  /* 内部：完整结构定义 */
#include "sensor.h"           /* 外部：opaque 接口 */

struct SensorCtx {
    float temperature;
    float humidity;
    int   raw_adc[4];
    uint8_t calibration_version;
};
```

配合一个简单的 Makefile 规则检查：

```bash
# 确保没有 .c 文件间接 include 了其他模块的内部头文件
grep -r '#include.*_internal\.h' src/ | grep -v 'sensor\.c' && \
    echo "警告：非本模块文件引用了内部头文件"
```

**一句话总结：接口的职责不是让你现在少写几行 getter/setter，而是保证六个月后当需求变更时，你不需要在十个调用方之间做手术。**

> 《C语言接口与实现》——"好的接口像好的法律：你注意不到它的存在，直到你需要它。"

**今日思考题**：打开你手头的一个模块，数一数它的头文件里有多少个 public 结构体字段可以被外部直接修改。如果这个数字大于 3，试着把其中一个改成 opaque pointer + 访问器的形式，看看需要改多少调用方代码——这个数字就是你现在欠的技术债。
