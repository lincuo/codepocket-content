# 每日架构原则｜2026-07-03

## 圈复杂度：在 Flash 里，"简洁"是有物理重量的

科(D)，早上好。

昨天聊了分层，今天聊一个听起来很抽象、实际上直接影响你固件大小的指标：**圈复杂度（Cyclomatic Complexity）**。

圈复杂度是 McCabe 在 1976 年提出的——它衡量的是一个函数中独立路径的数量。`if`、`for`、`while`、`case` 每一条分支都会让复杂度 +1。工具告诉你某个函数复杂度是 12，意思是这个函数有 12 条不同的执行路径，你需要 12 个测试用例才能覆盖全部。

在 PC 后端，复杂度 12 可能只是一个"需要重构"的警告。但在嵌入式里，这个数字背后藏着两样东西：**ROM 占用和调试信心**。

### 为什么圈复杂度在嵌入式里更敏感？

先看一段典型的 FreeRTOS 任务代码：

```c
/* ❌ 一个圈复杂度为 15 的"万能任务" */
void comm_task(void *pvParams) {
    for (;;) {
        /* 协议解析 */
        if (uart_has_data()) {
            uint8_t cmd = uart_read();
            if (cmd == CMD_START) {
                if (state == IDLE) {
                    state = RUNNING;
                    timer_start();
                } else if (state == STOPPED) {
                    state = RUNNING;
                    timer_restart();
                } else {
                    /* 已经在运行，忽略 */
                }
            } else if (cmd == CMD_STOP) {
                if (state == RUNNING) {
                    state = STOPPED;
                    timer_stop();
                    motor_brake();
                }
                /* 已经在 STOPPED 状态，忽略 */
            } else if (cmd == CMD_STATUS) {
                if (state == RUNNING) {
                    send_status(PWM_ACTIVE);
                } else if (state == STOPPED) {
                    send_status(MOTOR_BRAKE);
                } else {
                    send_status(IDLE_STATE);
                }
            } else if (cmd == CMD_CALIBRATE) {
                if (can_calibrate()) {
                    run_calibration();
                }
            }
            /* ... 还有 8 种命令 */
        }
        /* 超时处理 */
        if (timer_expired() && state == RUNNING) {
            emergency_stop();
        }
        /* 看门狗喂狗 */
        watchdog_feed();
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

这个函数的圈复杂度大约是 **15**——光 `if-else` 嵌套就有 12 层分支。问题在哪里？

**1. ROM 膨胀。** 编译器为每个 `if` 分支生成跳转指令。Cortex-M 的分支预测器对短距离跳转很友好，但 15 条路径意味着编译器要生成至少 15 个跳转目标标签。如果你的 Flash 只有 256KB，一个函数吃掉 2-3KB 的代码空间并不夸张。

**2. 测试盲区。** 你说"这个任务我只测了 CMD_START 和 CMD_STOP"。15 条路径，你只覆盖了 3 条。剩下的 12 条路径里，有多少是你从未在示波器上见过的？在嵌入式里，**没被测试覆盖的路径不一定是死代码——它可能是某个极端时序下的唯一逃生通道。**

**3. 调试噩梦。** 当客户报告"设备在运行 3 小时后突然停机"，你能在 15 条路径里快速定位是哪一条吗？

### 重构：让复杂度回归可管理的范围

```c
/* ✅ 拆分为职责单一的函数 */

/* 命令分发：每个函数复杂度 < 4 */
static void handle_start_cmd(void) {
    if (state == IDLE) {
        state = RUNNING;
        timer_start();
    } else if (state == STOPPED) {
        state = RUNNING;
        timer_restart();
    }
    /* RUNNING 状态下忽略，无需 else */
}

static void handle_stop_cmd(void) {
    if (state == RUNNING) {
        state = STOPPED;
        timer_stop();
        motor_brake();
    }
}

static void handle_status_cmd(void) {
    switch (state) {
        case RUNNING:  send_status(PWM_ACTIVE);    break;
        case STOPPED:  send_status(MOTOR_BRAKE);   break;
        default:       send_status(IDLE_STATE);    break;
    }
}

/* 任务主循环：复杂度降至 5 */
void comm_task(void *pvParams) {
    for (;;) {
        if (uart_has_data()) {
            uint8_t cmd = uart_read();
            switch (cmd) {
                case CMD_START:   handle_start_cmd();   break;
                case CMD_STOP:    handle_stop_cmd();    break;
                case CMD_STATUS:  handle_status_cmd();  break;
                case CMD_CALIBRATE: handle_calibrate(); break;
                default:          handle_unknown(cmd);    break;
            }
        }
        if (timer_expired() && state == RUNNING) {
            emergency_stop();
        }
        watchdog_feed();
        vTaskDelay(pdMS_TO_TICKS(10));
    }
}
```

拆分之后，`comm_task` 的圈复杂度从 15 降到了约 **5**，每个 handler 函数的复杂度都 ≤ 3。**更重要的是：新增一个命令时，你只需要写一个新的 handler 函数，不需要去改一个已经 200 行的怪物。**

### 可操作的实践建议

**给你的 FreeRTOS 任务设一个"复杂度预算"。** 不是每个函数都要做到 1，那会变成函数指针满天飞的过度设计。但你可以定一个规则：

> 每个 FreeRTOS 任务的入口函数（那个 `for(;;)` 循环）圈复杂度不超过 **8**，每个命令/事件处理函数不超过 **4**。

用 `cppcheck` 或 `clang-tidy` 跑一下：

```bash
cppcheck --enable=complexity --inline-suppr src/
```

把超标的函数逐个拆分。**拆分不是目的，可测试才是。** 当你把一个 15 复杂度的函数拆成三个 3 复杂度的函数时，你会发现每个都可以被单元测试覆盖——这在嵌入式里曾经是奢望，但现在用 CMock + Unity 完全可以做到。

### 一句话总结

**圈复杂度不是一个"代码洁癖"指标——在嵌入式里，它是 Flash 预算、测试覆盖率和调试信心的综合反映。控制复杂度，就是在控制你的固件风险。**

> 《代码大全》（Steve McConnell, 第 2 版, §19.4）：" McCabe 圈复杂度告诉我们，程序的逻辑复杂性是可以测量的，而测量的目的不是追求完美的数字，而是识别那些超出团队控制能力的模块。"

**今日思考题**：打开你项目里那个最大的 FreeRTOS 任务函数，用 cppcheck 或手动数一下它的分支数量。如果复杂度超过 10——你能在不改变功能的前提下，把它拆成几个职责单一的函数吗？试着画出拆分后的调用关系图，看看有没有哪个 handler 函数本身就值得继续拆分。
