# 重构技巧：从全局标志到状态枚举——中断上下文里的"野指针"陷阱

科(D)，午间好。今天聊一个你在裸机项目里几乎一定会踩的坑：**用 `volatile uint8_t` 当布尔标志在中断和任务间通信**。

## 坏味道：一个标志，包打天下

```c
/* ❌ 常见的"省事"写法 */
volatile uint8_t g_sensor_ready = 0;

void EXTI_Handler(void) {
    if (EXTI->PR & (1 << 3)) {
        g_sensor_ready = 1;
        EXTI->PR = (1 << 3);
    }
}

int main(void) {
    while (!g_sensor_ready) { /* spin */ }
    // 现在可以读传感器了
    read_sensor_data();
}
```

看着没问题？问题在三个层面。

**第一，语义丢失。** `g_sensor_ready` 只能表达"是/否"两种状态。但真实场景里，传感器可能有 idle → ready → error → calibrating 四种状态。你最终会再搞个 `g_sensor_error`、`g_sensor_calibrating`……五六个全局标志变量散落在文件里，谁改过谁也记不清。

**第二，竞态窗口。** 即使加了 `volatile`，`while (!g_sensor_ready)` 和 `read_sensor_data()` 之间仍然存在时间差。如果 ISR 在这两行之间被触发并清除标志（某些硬件确实在写 PR 寄存器时清除），主循环可能读到旧数据。

**第三，不可组合。** 当你需要同时等待两个外设就绪时，`while (!g_sensor_ready && !g_flash_ready)` 这种写法很快变得难以维护。

## 重构：状态枚举 + 位域掩码

```c
/* ✅ 用枚举表达状态，掩码表达组合条件 */
typedef enum {
    SENSOR_STATE_IDLE    = 0,
    SENSOR_STATE_READY   = 1,
    SENSOR_STATE_ERROR   = 2,
    SENSOR_STATE_CALIB   = 3,
} sensor_state_t;

static volatile sensor_state_t g_sensor_state = SENSOR_STATE_IDLE;
#define EVENT_SENSOR_READY  (1u << 0)
#define EVENT_SENSOR_ERROR  (1u << 1)

void EXTI_Handler(void) {
    if (EXTI->PR & (1 << 3)) {
        g_sensor_state = SENSOR_STATE_READY;
        EXTI->PR = (1 << 3);
    }
}

int main(void) {
    while (g_sensor_state != SENSOR_STATE_READY) {
        if (g_sensor_state == SENSOR_STATE_ERROR) {
            handle_sensor_fault();
        }
        wfi();  /* 省电等待 */
    }
    read_sensor_data();
}
```

## 效果对比

| 维度 | 重构前 | 重构后 |
|------|--------|--------|
| 状态表达能力 | 2 种（是/否） | N 种（枚举扩展） |
| 中断安全 | 读写原子性无法保证 | 单写入点，状态转换集中 |
| 可测试性 | 需模拟 ISR 时序 | 可直接调用状态转换函数 |
| 代码量 | 少 | 多约 30% |

代码量确实多了。但这是**用空间换确定性**——嵌入式世界里，确定性比省那几行代码值钱得多。

## 今晚练习

找一个你项目里（或者写过的 demo 里）用 `volatile uint8_t` 做事件标志的地方，把它改成**枚举状态 + 显式转换函数**的形式。注意：

1. 状态转换函数必须是 `static inline`，内联消除调用开销
2. 每个 ISR 只负责调用转换函数，不直接写全局变量
3. 编译后看反汇编，确认内联确实生效了

**一句话总结：全局变量不是不能用的，但不能让它既当状态又当事件又当配置——每增加一个职责，你就在埋一个 bug。**

> 《C陷阱与缺陷》："volatile 告诉编译器不要对变量做优化假设，但它不告诉程序员这个变量什么时候会被谁修改。"

**今日思考题**：如果你的传感器 ISR 在 `main()` 检查完 `g_sensor_state == READY` 之后、`read_sensor_data()` 执行之前又被触发了一次（比如硬件产生了重复中断），当前代码会出现什么问题？你会怎么加固？
