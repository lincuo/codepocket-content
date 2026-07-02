# 每日架构原则｜2026-07-02

## 分层不是叠罗汉——嵌入式模块化的第一性原理

科(D)，早上好。

昨天聊了圈复杂度，今天聊一个更根本的问题：**为什么你在嵌入式项目里画的分层架构，最后变成了谁也改不动的大泥球？**

这个问题和 PC 后端项目里的分层不是一回事。在 Java Spring 里，Controller → Service → Repository 三层是标配，因为 JVM 启动慢、磁盘大、内存随便花，多一层间接调用毫不在意。但在嵌入式里，**每一层都不是免费的**。

### 嵌入式分层的隐藏成本

三层架构在嵌入式里意味着什么？

**第一，调用链膨胀。** 每次跨层调用都是函数调用——压栈、保存上下文、跳转、恢复。在 Cortex-M 上虽然只有几个周期，但如果这个调用发生在 ISR 里或者高频定时器回调中，累积效应会显著增加中断延迟。

**第二，数据拷贝。** 层与层之间传递数据，如果不慎用了值拷贝而不是指针，一个 64 字节的传感器数据包可能在调用链上传三次，占用 192 字节的栈空间——在栈只有 2KB 的 FreeRTOS 任务里，这不是小事。

**第三，也是最致命的——边界模糊。** 很多嵌入式项目号称"分层"，但实际上 UI 层直接调了 HAL 层的寄存器配置函数，业务逻辑层又直接操作了底层驱动的结构体。这不是分层，这是**穿了层衣服的单片**。

### 反模式：伪分层——FreeRTOS 任务里的"上帝层"

```c
/* ❌ 看起来分层，实际上每层都在做其他层的事 */

/* 应用层 - 居然直接操作了 HAL 层的数据 */
void app_process_sensor_data(SensorRaw_t *raw) {
    /* 应用逻辑：单位转换 */
    float temp = raw->reg[0] * 0.0625f - 273.15f;
    
    /* 业务逻辑：阈值判断 */
    if (temp > 80.0f) {
        /* 居然在这里直接调了 HAL */
        HAL_GPIO_TogglePin(LED_PORT, LED_PIN);
        /* 还直接操作了底层通信 */
        I2C_Transmit(&hi2c1, &temp, sizeof(temp));
    }
}

/* 驱动层 - 居然包含了应用逻辑 */
void I2C_Transmit(I2C_HandleTypeDef *hi2c, void *data, uint16_t len) {
    /* 这里做了数据校验——本该是应用层的事 */
    if (len > MAX_PAYLOAD) {
        len = MAX_PAYLOAD;  /* 硬编码的魔法数字 */
    }
    HAL_I2C_Master_Transmit(hi2c, ADDR, (uint8_t*)data, len, 100);
}
```

问题很明显：`app_process_sensor_data` 里混了单位转换（应用）、阈值判断（业务）、GPIO 操作（驱动）和 I2C 发送（驱动）。`I2C_Transmit` 里又塞了数据校验（应用/业务）。**每一层都不纯粹，谁都能改谁，最后谁也说不清这段逻辑到底属于哪一层。**

### 正确的分层：严格的信息流

```c
/* ✅ 清晰的分层：每层只和自己的下一层打交道 */

/* 应用层 - 只做业务逻辑，不碰硬件 */
AppResult_t app_handle_temperature(float celsius) {
    if (celsius > TEMP_WARN_THRESHOLD) {
        return APP_ALERT_WARNING;
    }
    if (celsius > TEMP_CRIT_THRESHOLD) {
        return APP_ALERT_CRITICAL;
    }
    return APP_OK;
}

/* 驱动抽象层 - 只负责硬件交互，不含业务逻辑 */
DriverStatus_t driver_i2c_send(const uint8_t *buf, size_t len) {
    if (len == 0 || len > DRIVER_MAX_PAYLOAD) {
        return DRIVER_ERR_INVALID_ARG;
    }
    return HAL_I2C_Master_Transmit(&hi2c1, ADDR, 
                                   (uint8_t*)buf, len, 
                                   TIMEOUT_MS) == HAL_OK 
           ? DRIVER_OK : DRIVER_ERR_TIMEOUT;
}

/* 适配层 - 把应用结果翻译成驱动调用 */
void app_adapter_execute(AppResult_t result) {
    switch (result) {
        case APP_ALERT_WARNING:
            driver_gpio_toggle(LED_PIN);
            driver_i2c_send(warning_buf, sizeof(warning_buf));
            break;
        case APP_ALERT_CRITICAL:
            driver_gpio_toggle(LED_PIN);
            driver_i2c_send(critical_buf, sizeof(critical_buf));
            break;
        default:
            break;
    }
}
```

分层的关键不在于画了几层，而在于**每层的输入和输出是否足够纯粹**。应用层输入是数字（温度值），输出是枚举（AppResult_t）；驱动层输入是字节流，输出是状态码。两者之间有一个薄薄的适配层负责翻译，而不是让应用层直接调用 `HAL_GPIO_TogglePin`。

### 可操作的实践建议

**给你的每个模块写一行"职责声明"。** 不是文档注释，就写在头文件顶部，像这样：

```c
/* temperature_app.h
 * 职责：温度业务逻辑（阈值判断、告警级别判定）
 * 不做什么：不调用 HAL、不操作 GPIO、不直接发送数据
 * 依赖：driver_gpio.h, driver_i2c.h（仅适配层）
 */
```

然后在代码审查时，用这条声明当尺子量：这个函数里有没有出现 `HAL_` 前缀的调用？有没有 `#include <stm32xxx_hal_`？如果有，它就不该在这个模块里。

**一句话总结：嵌入式分层不是为了好看，是为了控制编译依赖和变更影响面。每一层都应该是一个可以单独替换的积木，而不是一坨粘在一起的橡皮泥。**

> 《UNIX 编程艺术》：\"每个程序都应该做一件事，并且把它做好。\"——在嵌入式里，这句话的延伸是：**每一层也应该只做一件事，并且把好这一件事的边界守死。**

**今日思考题**：打开你项目里最大的那个 `.c` 文件，标出其中所有 `HAL_`、`LL_`、`DRV_` 开头的函数调用。如果它们出现在应用逻辑函数里——也就是做了阈值判断、状态机转换、数据聚合的地方——这就是分层模糊的证据。试着把硬件调用抽到一个单独的适配函数里，看看应用函数的可读性有没有改善。
