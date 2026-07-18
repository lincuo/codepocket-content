# 每日架构原则 — 模块化与分层架构的权衡

## 分层不是画饼图，是分责任

科(D)，早上好。

今天聊一个看起来简单、做起来全是坑的话题：**嵌入式系统的分层架构**。

在 PC 后端，我们习惯画漂亮的分层图——UI 层、业务层、数据层，一层调一层，互不干扰。但嵌入式系统的资源约束（RAM 几十 KB、Flash 几百 KB）让"完美分层"经常变成"完美浪费"。

### 一个常见的反模式

```c
/* ❌ FreeRTOS 任务直接调用 HAL */
void sensor_task(void *arg) {
    while (1) {
        ADC_Read(&hadc1, &raw_value);  // HAL 调用
        filtered = EMA_Filter(raw_value);
        xQueueSend(queue, &filtered, portMAX_DELAY);
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

问题在哪？看起来没问题，对吧？

**三个隐性耦合**：

1. **HAL 泄漏到业务逻辑**：`ADC_Read` 绑死了 STM32 的具体型号。换一颗 MCU，整个任务要重写。
2. **阻塞 API 污染任务调度**：`portMAX_DELAY` 让队列发送永远阻塞——如果下游任务挂了，传感器采集也停了，看门狗会复位整个系统。
3. **没有抽象边界**：HAL 函数既是硬件访问入口，也是任务间的通信协议。一个接口承担两种职责，违反单一职责。

### 正确的做法：用端口隔离变化

```c
/* ✅ 定义端口，隐藏实现 */
typedef struct {
    float (*read)(void);
    void  (*init)(void);
} SensorPort_t;

static float stm32_adc_read(void) {
    uint16_t raw;
    HAL_ADC_PollForConversion(&hadc1, 10);
    raw = HAL_ADC_GetValue(&hadc1);
    return raw * 3.3f / 4096.0f;
}

static SensorPort_t adc_port = { .read = stm32_adc_read };

void sensor_task(void *arg) {
    SensorPort_t *port = (SensorPort_t *)arg;
    float value;
    while (1) {
        value = port->read();
        float filtered = ema_update(&ema_state, value);
        if (xQueueSend(queue, &filtered, pdMS_TO_TICKS(10)) == pdPASS) {
            /* 发送成功，继续 */
        } else {
            /* 超时丢弃，不阻塞下游故障 */
        }
        vTaskDelay(pdMS_TO_TICKS(100));
    }
}
```

这里的核心思想不是"多写几层抽象"，而是**明确边界**：

- **端口（Port）** 定义了"谁能做什么"，不关心"怎么做"
- **超时而非无限等待** 保证任务不会因为下游故障而卡死
- **函数指针表** 在编译时绑定实现，运行时零开销——嵌入式版的接口隔离

### 权衡：什么时候不该分层？

分层不是免费的。每一层抽象都意味着：
- 额外的函数调用开销（在 Cortex-M 上通常可忽略，但在 8 位 MCU 上可能致命）
- 额外的间接寻址（函数指针 vs 直接调用）
- 调试时的跳转路径变长

**经验法则**：
- 如果你的 MCU 只有 8KB RAM，不要为了"未来可能的移植"做三层抽象——直接写，但用 `static inline` 把硬件访问收拢到一个文件里
- 如果你的团队同时维护两套硬件平台，端口抽象值得做
- 如果只有一个硬件平台且产品生命周期短，**写清楚注释比加抽象层更有价值**

### 一句话总结

> **分层的目的不是画出漂亮的架构图，而是让"变化的部分"被限制在最小的范围内。** 在嵌入式系统里，这个范围越小越好。

> *"Programs must be written for people to read, and only incidentally for machines to execute."*  
> —— Harold Abelson, *Structure and Interpretation of Computer Programs*

### 今日思考题

打开你手头的一个 RTOS 项目，找一个直接调用 HAL/LL 函数的任务。问自己三个问题：

1. 如果明天换一颗不同厂商的 MCU，这个文件需要改几行？
2. 如果下游消费者挂了，这个任务会被阻塞多久？
3. 你能不能在不改动任务逻辑的前提下，用另一个传感器驱动替换它？

如果任何一个答案是"说不清"，那就是该引入端口的信号。
