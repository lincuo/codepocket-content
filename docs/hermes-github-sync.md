# Hermes AI 自动更新说明

Hermes AI 在 NAS 上每天执行：

```text
1. 读取当前日期 YYYY-MM-DD
2. 选择今日主题
3. 调用 AI 生成 daily/YYYY-MM-DD.json
4. 校验 JSON 字段
5. 更新 manifest.json
6. git add / commit / push
```

## 建议主题池

```text
UART
I2C
SPI
GPIO
ADC
DMA
Timer
FreeRTOS
状态机
驱动分层
模块解耦
错误处理
低功耗
调试技巧
项目重构
软件架构
```

## Git 提交流程

```bash
git add manifest.json daily/YYYY-MM-DD.json
git commit -m "chore: update daily content YYYY-MM-DD"
git push
```

## NAS cron 示例

```cron
0 6 * * * cd /path/to/codepocket-content && /path/to/hermes-update.sh >> logs/daily.log 2>&1
```
