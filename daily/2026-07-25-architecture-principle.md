# 每日架构原则｜2026-07-25

## 分层不是目的，解耦才是——嵌入式系统的"够好"抽象

PC 后端写惯了的人，看到"分层架构"四个字就条件反射地画四层：UI → Service → Domain → Infrastructure。然后每个层之间加接口、加依赖倒置、加工厂模式。在 Java 企业应用里这叫优雅，在嵌入式 C 里这叫自杀。

问题的核心不在于"要不要分层"，而在于**每一层抽象的代价由谁承担**。

### 嵌入式 vs PC 后端的成本结构差异

| 维度 | PC 后端 | 嵌入式（Cortex-M） |
|------|---------|-------------------|
| 对象创建 | 几乎免费 | 栈空间紧张，大 struct 直接压栈会炸 |
| 函数调用开销 | 可忽略 | 每次间接调用消耗指令周期 |
| 内存布局 | GC 管理 | 静态分配，每字节都有归属 |
| 调试方式 | 日志 + 断点 | 逻辑分析仪 + 寄存器dump |

这意味着：**嵌入式系统的抽象必须能精确计算其成本**。一个你看不到的间接调用，在实时系统中可能就是 missed deadline 的根因。

### 反模式：为分层而分层的 FreeRTOS 任务架构

```c
/* ❌ 过度分层的典型 */
typedef struct {
    void (*init)(struct Context*);
    void (*read)(struct Context*, SensorData*);
    void (*process)(struct Context*, SensorData*, ProcessedData*);
    void (*send)(struct Context*, ProcessedData*);
} SensorDriver;

static SensorDriver i2c_temp_driver = {
    .init = i2c_temp_init,
    .read = i2c_temp_read,
    .process = temp_to_celsius,
    .send = mqtt_publish,
};

/* 每个操作都要通过函数指针间接调用，
 * 中间还要经过至少两层任务队列传递 */
```

这段代码看起来"干净"——驱动与应用解耦，可以热替换传感器类型。但实际代价是：

1. **四次函数指针间接调用**，每次 3-5 条 Thumb-2 指令
2. **SensorData 结构体跨任务队列拷贝**，在 64KB RAM 的设备上可能占 200 字节
3. **四个阶段分散在不同 tick**，从读到发延迟不可控

### 正确的做法：按需抽象

```c
/* ✅ 只在真正需要多传感器时才抽象 */
/* 单一传感器类型时，直接调用即可 */
i2c_temp_read(&hi2c1, &raw_data);
float celsius = raw_to_celsius(raw_data);
mqtt_send(topic, serialize(celsius));

/* 当需要支持三种温度传感器时，再引入表驱动： */
typedef struct {
    uint8_t addr;
    float (*raw_to_c)(uint16_t);
} TempSensorDef;

static const TempSensorDef sensors[] = {
    [SENSOR_DS18B20] = { 0x28, ds18b20_raw_to_c },
    [SENSOR_TMP117]  = { 0x48, tmp117_raw_to_c },
};
```

关键区别在于：**抽象是在第三次出现重复时引入，而不是在第一次写代码时就预设**。Rule of Three 在嵌入式里不是建议，是纪律。

### 分层决策框架

遇到"要不要加一层抽象"时，问自己三个问题：

1. **这个组件会有第二个实现吗？** 如果只会有一种传感器/一种通信协议，直接调用比封装省代码也省性能。
2. **这个抽象的成本能精确计算吗？** 函数指针间接调用多少 cycle？结构体拷贝占用多少栈？如果不能回答，说明抽象太早了。
3. **拆开后，故障边界更清晰了吗？** 如果分层只是让代码多了两个文件但调试路径没缩短，那这层白分了。

**一句话总结：嵌入式系统的分层不是为了好看，是为了把复杂度从"运行时"搬到"编译时"或"设计时"。如果一个抽象不能在三者之一节省资源，它就是在浪费资源。**

> 《UNIX编程艺术》中 Eric S. Raymond 写道："规则一：使用谦逊，仅当你相信他人比你更懂时才可打破规则。" 在嵌入式领域，这条规则的变体是：**"规则一：不做抽象，除非你能证明它的成本低于不做的成本。"**

**今日思考题**：审视你当前项目中一个"为了可扩展性"而设计的接口层——如果这个模块这辈子只会有一个实现，去掉抽象层后你能省下多少 Flash 和 RAM？把这个数字写下来，它会改变你对" clean code "的理解。
