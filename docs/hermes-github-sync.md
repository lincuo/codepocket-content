# Hermes AI 更新 CodePocket 内容源执行手册

本文档是给 Hermes AI 助手使用的。你的任务是更新 GitHub 仓库中的每日精选内容，确保 CodePocket 小程序可以读取。

不要修改小程序代码。你只需要维护当前内容仓库。

## 你正在维护的仓库

仓库用途：

```text
CodePocket 小程序每日精选内容源
```

仓库地址：

```text
https://github.com/lincuo/codepocket-content
```

发布后的 manifest 地址：

```text
https://lincuo.github.io/codepocket-content/manifest.json
```

## 你需要更新哪些文件

每次更新只需要改这些文件：

```text
manifest.json
daily/YYYY-MM-DD.json
```

其中 `YYYY-MM-DD` 是当天日期，例如：

```text
daily/2026-05-03.json
```

不要改小程序代码。

不要删除历史内容。

不要删除 `manifest.json` 里已有的历史日期。

## 仓库目录结构

仓库应该保持下面结构：

```text
manifest.json
daily/
  2026-05-03.json
  2026-05-02.json
docs/
  content-format.md
  hermes-github-sync.md
README.md
```

## manifest.json 规则

`manifest.json` 是内容索引。

格式：

```json
{
  "latest": "2026-05-03",
  "files": {
    "2026-05-03": "daily/2026-05-03.json",
    "2026-05-02": "daily/2026-05-02.json"
  }
}
```

更新规则：

1. `latest` 必须设置为最新内容日期。
2. `files` 必须保留所有历史日期。
3. 新增当天内容时，只追加当天日期。
4. 不要覆盖或删除旧日期。
5. 日期格式必须是 `YYYY-MM-DD`。
6. 路径格式必须是：

```text
daily/YYYY-MM-DD.json
```

## 每日内容文件规则

每日内容文件路径：

```text
daily/YYYY-MM-DD.json
```

文件格式：

```json
{
  "date": "2026-05-03",
  "items": []
}
```

`date` 必须等于文件名中的日期。

`items` 是每日精选内容数组。

建议每天生成 3 到 5 条内容。

## 每条 item 必填字段

每条 item 必须包含以下字段：

```text
id
title
topic
mode
language
platform
toolchain
snippetType
difficulty
tags
explanation
content
code
```

字段含义：

| 字段 | 类型 | 说明 |
| --- | --- | --- |
| `id` | string | 全局唯一 ID |
| `title` | string | 中文标题 |
| `topic` | string | 主题 |
| `mode` | string | 学习模式 |
| `language` | string | 编程语言，优先 `C` |
| `platform` | string | 平台，如 `通用嵌入式`、`STM32`、`FreeRTOS` |
| `toolchain` | string | 工具链，如 `GCC`、`Keil MDK` |
| `snippetType` | string | 内容类型 |
| `difficulty` | string | 难度 |
| `tags` | string[] | 标签数组 |
| `explanation` | string | 简短说明 |
| `content` | string | 详细讲解 |
| `code` | string | 代码、接口示例、伪代码或目录结构 |

## id 生成规则

`id` 必须稳定且唯一。

推荐格式：

```text
YYYY-MM-DD-topic-mode-short-name
```

示例：

```text
2026-05-03-uart-architecture-layering
2026-05-03-i2c-refactor-register-map
2026-05-03-freertos-tip-queue-boundary
```

要求：

1. 只使用小写英文、数字和连字符。
2. 不要使用空格。
3. 不要使用中文。
4. 同一天内不能重复。

## mode 可选值

`mode` 只能使用以下值之一：

```text
tip
snippet
architecture
refactor
```

说明：

| mode | 含义 |
| --- | --- |
| `tip` | 技巧、经验、注意事项 |
| `snippet` | 可复用代码片段 |
| `architecture` | 架构设计、模块边界、依赖方向 |
| `refactor` | 重构思路、改进步骤、风险 |

## snippetType 可选值

`snippetType` 只能使用以下值之一：

```text
function
driver
algorithm
config
macro
rtos
protocol
note
```

## difficulty 可选值

`difficulty` 只能使用以下值之一：

```text
beginner
intermediate
advanced
```

## 推荐主题池

每天从下面主题中选择 3 到 5 个：

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
Bootloader
通信协议
看门狗
Flash 存储
传感器驱动
```

## 每日内容组合建议

每天建议生成：

```text
1 条外设/驱动技巧
1 条架构设计或模块解耦
1 条重构思路
1 条可复用代码片段
可选 1 条 RTOS、调试或低功耗经验
```

不要每天都只生成代码。

CodePocket 的每日精选更重视：

```text
工程经验
架构思维
模块边界
解耦方法
权衡利弊
可落地的代码片段
```

## 内容质量要求

生成内容必须满足：

1. 面向嵌入式 C / 单片机开发。
2. 标题、简介、详细讲解必须使用中文。
3. `explanation` 建议 60 到 160 字。
4. `content` 要讲清背景、问题、设计思路、权衡和落地方式。
5. `code` 可以是 C 代码、接口示例、伪代码、目录结构或配置片段。
6. 代码优先 C99 风格。
7. C 代码缩进使用 4 个空格。
8. 必要时添加简洁中文注释。
9. 避免不必要的动态内存分配。
10. 涉及中断、DMA、RTOS、共享变量时必须说明风险。
11. 架构类内容必须说明模块边界和依赖方向。
12. 重构类内容必须说明改造前问题、改造步骤、风险和验证方式。
13. 不要写空泛鸡汤。
14. 不要只罗列概念。
15. 不要输出 Markdown 文件内容到 JSON 字符串之外。

## 生成内容示例

```json
{
  "id": "2026-05-03-uart-architecture-layering",
  "title": "UART 驱动和协议解析层如何解耦",
  "topic": "UART",
  "mode": "architecture",
  "language": "C",
  "platform": "通用嵌入式",
  "toolchain": "GCC",
  "snippetType": "note",
  "difficulty": "intermediate",
  "tags": ["UART", "架构设计", "解耦", "嵌入式C"],
  "explanation": "协议层只依赖通信端口接口，底层 UART、USB CDC 或 SPI 都可以替换。",
  "content": "在嵌入式项目中，协议解析代码如果直接调用 UART HAL，会导致协议层难以复用和测试。更好的做法是定义通信端口抽象，让协议层只依赖 read/write 接口。这样底层可以替换为 UART、USB CDC、SPI 或测试桩。代价是多一层接口，需要团队保持抽象边界清晰。",
  "code": "typedef struct {\\n    int (*read)(unsigned char *buf, unsigned int len);\\n    int (*write)(const unsigned char *buf, unsigned int len);\\n} comm_port_t;"
}
```

## 生成 Prompt 模板

Hermes 可以使用下面 Prompt 生成每日内容：

```text
你是 CodePocket 每日精选知识库维护助手。

请生成一个合法 JSON 文件，用于 daily/{{DATE}}.json。

要求：
1. 顶层字段必须是 date 和 items。
2. date 必须是 {{DATE}}。
3. items 生成 3 到 5 条。
4. 每条 item 必须包含：
   id、title、topic、mode、language、platform、toolchain、snippetType、difficulty、tags、explanation、content、code。
5. 内容面向嵌入式 C / 单片机开发。
6. 标题、简介、详细讲解使用中文。
7. 不要只生成代码，也要包含架构、重构、解耦、权衡、调试方法等知识。
8. 架构类内容要讲清模块边界、依赖方向、解耦方式和权衡利弊。
9. 重构类内容要说明改造前问题、改造步骤、风险和验证方法。
10. 代码优先 C99 风格，缩进 4 个空格，必要时添加中文注释。
11. 避免不必要的动态内存分配。
12. 涉及中断、DMA、RTOS、共享资源时必须说明注意事项。
13. 返回纯 JSON，不要 Markdown，不要代码围栏。

今日日期：{{DATE}}
今日主题候选：{{TOPICS}}
```

## 更新步骤

Hermes 每天执行：

1. 获取当天日期 `YYYY-MM-DD`。
2. 检查 `daily/YYYY-MM-DD.json` 是否已经存在。
3. 如果已经存在，默认不要重复生成，除非用户要求强制更新。
4. 读取 `manifest.json`。
5. 生成当天 JSON 内容。
6. 校验当天 JSON。
7. 写入 `daily/YYYY-MM-DD.json`。
8. 更新 `manifest.json.latest` 为当天日期。
9. 在 `manifest.json.files` 中追加：

```json
"YYYY-MM-DD": "daily/YYYY-MM-DD.json"
```

10. 保留 `files` 中所有旧日期。
11. 提交并推送。

## 校验清单

写文件前必须校验：

```text
daily/YYYY-MM-DD.json 是合法 JSON
date 等于 YYYY-MM-DD
items 是数组
items.length >= 3
items.length <= 5
每条 item.id 唯一
每条 item.title 非空
每条 item.explanation 非空
每条 item.content 非空
mode 在允许范围内
snippetType 在允许范围内
difficulty 在允许范围内
tags 是数组
manifest.json 是合法 JSON
manifest.files 保留历史日期
```

## Git 提交规则

提交信息格式：

```text
chore: update daily content YYYY-MM-DD
```

示例：

```bash
git add manifest.json daily/2026-05-03.json
git commit -m "chore: update daily content 2026-05-03"
git push
```

## 严禁事项

不要做这些事：

1. 不要修改小程序代码。
2. 不要删除历史 daily 文件。
3. 不要清空 `manifest.json.files`。
4. 不要把 API Key 写入仓库。
5. 不要生成 Markdown 包裹的 JSON。
6. 不要生成无效 JSON。
7. 不要让 `items` 为空。
8. 不要只生成泛泛的软件工程鸡汤。

## 小程序如何读取

小程序会读取：

```text
https://lincuo.github.io/codepocket-content/manifest.json
```

然后根据 `manifest.files` 读取：

```text
daily/YYYY-MM-DD.json
```

因此，只要 `manifest.json` 和 `daily/YYYY-MM-DD.json` 正确，小程序就能显示每日精选。
