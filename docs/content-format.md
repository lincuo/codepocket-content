# CodePocket 每日知识库内容规则

## manifest.json

```json
{
  "latest": "2026-05-02",
  "files": {
    "2026-05-02": "daily/2026-05-02.json"
  }
}
```

规则：

- `latest` 是最新可用日期。
- `files` 保存全部历史日期。
- 日期格式必须是 `YYYY-MM-DD`。

## daily/YYYY-MM-DD.json

```json
{
  "date": "2026-05-02",
  "items": []
}
```

每条 item 必须包含：

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

## mode

```text
tip
snippet
architecture
refactor
```

## snippetType

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

## difficulty

```text
beginner
intermediate
advanced
```

## 内容要求

- 面向嵌入式 C / 单片机开发。
- 标题、简介、详细讲解使用中文。
- 架构类内容要讲模块边界、依赖方向、解耦方式和权衡。
- 重构类内容要讲改造前问题、改造步骤、风险和验证方法。
- 代码优先 C99 风格。
- 涉及中断、DMA、RTOS、共享资源时说明注意事项。
