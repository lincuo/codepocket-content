# ADR-DAILY-001：KISS > SOLID，嵌入式里尤甚

SOLID 搬进嵌入式，很容易变成"S-O-L-I-D——五个字压垮 Flash"。

反模式见过太多：一个 LED 驱动类，为符合 "单一职责" 硬拆出 LED_interface、LED_hal_v1、LED_hal_v2、LED_simulator 四个子类，代码行数翻倍，维护者却在想——今天到底改哪一个？

```c
/* ❌ 过度抽象——为了抽象而抽象 */
typedef struct {
    void (*init)(void);
    void (*on)(void);
    void (*off)(void);
} led_vtbl_t;

typedef struct {
    const led_vtbl_t *vtab;
    uint32_t timestamp;
    uint8_t blink_interval;
} abstract_led_t;

/* ✅ 嵌入式式KISS：直接操作寄存器（或LL层接口） */
static inline void led_on(GPIO_TypeDef *port, uint16_t pin) {
    port->BSRR = (uint32_t)pin << 16;
}
static inline void led_off(GPIO_TypeDef *port, uint16_t pin) {
    port->BSRR = (uint32_t)pin;
}
```

FreeRTOS 场景同理：为一个简单的传感器采样任务，先建事件队列、状态机回调、配置结构体模板、抽象成"可插拔采集单元"——结果任务函数只剩三行注释。**不要在你还没看到第二遍 Bug 之前，就为它设计架构。**

实践建议：**先写能跑的 C，再加注释；需要第二次修改同一块逻辑时，再考虑封装。** 嵌入式里，KISS 不是口号，是 Flash 和 SRAM 投票选出来的原则。

> "Make everything as simple as possible, but not simpler." — Albert Einstein

**今日思考题**：翻出自己最近一段嵌入式代码，标出所有"理论上以后可能需要扩展，但实际从未被用到过"的抽象层。删掉它们，看看测试是否依然通过。