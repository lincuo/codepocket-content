# 每日架构原则 — 设计原则在嵌入式场景的误用

## 别把 PC 后端的刀法用在嵌入式上

SOLID、DRY、KISS 这些原则没有错，错的是把它们当成万能公式，不分场景地套用到嵌入式项目里。

## 反模式：为了开闭原则（OCP）造出五层抽象

STM32 + FreeRTOS 项目中常见这样的代码：

```c
/* ❌ 过度抽象 —— 只为"以后可能换传感器" */

typedef struct {
    float (*read_temp)(void *ctx);
    float (*read_humid)(void *ctx);
    void  (*init)(void *ctx);
} SensorVTable_t;

typedef struct {
    SensorVTable_t *vtbl;
    void *impl;
} Sensor_t;

/* 每个新传感器都要写一个 VTable 初始化函数 */
static float dht22_read_temp(void *ctx) { ... }
static float dht22_read_humid(void *ctx) { ... }

static const SensorVTable_t dht22_vtbl = {
    .read_temp = dht22_read_temp,
    .read_humid = dht22_read_humid,
};

Sensor_t sensor_create_dht22(void) {
    Sensor_t s;
    s.vtbl = &dht22_vtbl;
    s.impl = malloc(sizeof(DHT22_Context_t));
    return s;
}

/* 读一次温度：函数指针间接调用 + 两次解引用 */
float read_temp(Sensor_t *s) {
    return s->vtbl->read_temp(s->impl);
}
```

**问题在哪？**

1. **性能代价**：每次 `read_temp()` 是两次函数指针间接调用，ARM Cortex-M 上约 4-6 个周期。如果这个调用在 1ms 定时器中断里——它不该在——开销翻倍。
2. **内存浪费**：`DHT22_Context_t` 可能只有 8 字节，但 `malloc` 返回的指针至少占用 4 字节的 heap 元数据，外加 VTable 指针 4 字节。一个传感器对象占用了 20+ 字节——对于一个 256KB RAM 的 MCU，这不是小数。
3. **维护成本**：新增一个传感器要写 VTable、创建函数、实现结构体。三个传感器就是 150 行样板代码。
4. **Rule of Three 被违反**：你只有一种传感器，就建了完整的面向对象继承体系。抽象应该在第三次重复出现时才引入。

## 正确的做法：按需抽象

```c
/* ✅ 先写具体实现，抽象留给真正需要的时候 */

typedef struct {
    uint8_t i2c_addr;
    uint8_t last_error;
} DHT22_Context_t;

bool dht22_init(DHT22_Context_t *ctx, uint8_t addr);
bool dht22_read(DHT22_Context_t *ctx, float *temp, float *humid);

/* 如果后来真的需要替换传感器，再引入不透明指针和统一接口 */
```

## 嵌入式场景下设计原则的正确姿势

| 原则 | PC 后端理解 | 嵌入式正确姿势 |
|------|------------|----------------|
| **SRP** | 一个类做一件事 | 一个函数做一件事，一个文件一个模块 |
| **OCP** | 用接口扩展行为 | 用配置/参数扩展，抽象等第三次重复 |
| **LSP** | 子类可替换父类 | C 语言没有继承，用组合替代 |
| **ISP** | 接口小而专 | 驱动 API 按功能分组，不要一个巨型 struct |
| **DIP** | 依赖抽象而非具体 | 依赖函数指针或静态链接的模块边界 |
| **KISS** | 简单直接 | 永远适用，嵌入式尤其如此 |
| **DRY** | 消除重复 | 只在确定性重复时抽取，不确定性时复制粘贴 |

## 实践建议

给抽象设一个门槛：**同一个模式出现三次以上才抽取**。前两次允许复制粘贴，第三次再重构。嵌入式项目的生命周期往往比 PC 软件短得多——产品迭代两代，硬件 BOM 就变了，之前为"通用性"写的适配层直接废弃。把精力花在真正影响性能、内存和安全的地方。

## 思考题

在你的 STM32 项目中，找一个你写了 VTable 或回调函数注册的地方，问自己：这里是真的需要多态，还是只是"怕以后改"？如果现在让你删掉所有抽象层，直接用具体实现，代码会不会更清晰？动手试一次。

> "过早的优化是万恶之源"——但过早的抽象，是嵌入式项目最大的技术债务。
> —— 一位经历过三次固件重写的架构师
