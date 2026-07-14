# 每日重构技巧 —— 2026-07-14

## 坏味道：用宏开关伪造"多态"

嵌入式 C 里最常见的反模式之一：用 `#ifdef` 在同一个函数里拼出多种行为。

```c
void motor_control(uint16_t pwm) {
#ifdef CONFIG_MOTOR_TYPE_A
    TIM2->CCR1 = pwm;
#elif defined(CONFIG_MOTOR_TYPE_B)
    PWM_SetDuty(pwm * 80 / 100);
#else
    analog_write(pwm >> 2);
#endif
}
```

看起来省事，但问题很明显：

- **编译单元膨胀**：每次切配置都要全量重编，CI 跑一次二十分钟
- **测试盲区**：`#else` 分支几乎不会被任何配置覆盖到
- **隐藏依赖**：调用方根本不知道传进来的 pwm 经过了怎样的缩放

## 重构：提取为函数指针表

把宏开关变成查表，每个分支独立编译、独立测试。

```c
typedef void (*pwm_apply_fn)(uint16_t raw);

static void apply_a(uint16_t raw) {
    TIM2->CCR1 = raw;
}

static void apply_b(uint16_t raw) {
    PWM_SetDuty(raw * 80 / 100);
}

static void apply_analog(uint16_t raw) {
    analog_write(raw >> 2);
}

static const pwm_apply_fn table[] = {
    [0] = apply_a,
    [1] = apply_b,
    [2] = apply_analog,
};

void motor_control(uint16_t pwm, uint8_t hw_type) {
    if (hw_type >= ARRAY_SIZE(table)) return;
    table[hw_type](pwm);
}
```

效果对比：

| 维度 | 重构前 | 重构后 |
|------|--------|--------|
| 编译时间 | 全量重编 20min | 只重编改动模块 <30s |
| 分支覆盖率 | else 分支 0% | 每支独立单元测试 |
| 新增硬件类型 | 改核心函数 + 全量重编 | 加一个 static 函数 + 表项 |
| 运行时开销 | 零（宏展开） | 一次间接调用（<1 cycle） |

## 实践练习

打开你项目里最大的那个 `.c` 文件，搜 `#ifdef` 或 `#if defined`。如果某个函数里有超过 2 个宏分支，挑一个写成一个独立的 static 函数，放进查表里。

**今晚的思考题**：如果 `hw_type` 本身也是通过 I2C 从 EEPROM 读取的，读取失败时函数应该 panic 还是优雅降级？在 FreeRTOS 任务里和在 ISR 里，这两种选择的代价有什么不同？
