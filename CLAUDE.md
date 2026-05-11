# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## 项目概述

这是一个 Claude Code 技能项目，提供系统化代码重构与技术债评估方法论。技能定义在 `SKILL.md`（YAML 前置元数据 + Markdown 正文），覆盖：调用者计数审计、类型安全边界、单次调用内联、重复分支合并、参数对象重构、派生类型、技术债五维清单、ROI 优先路线图、预防策略与度量指标等。

## 文件说明

- `SKILL.md` — 技能主文件（前置元数据: name=code-refactor）

## 技能工作原理

SKILL.md 的前置元数据决定触发条件：
- **name**: 技能标识符，Claude Code 通过 `/code-refactor` 或技能匹配系统调用
- **description**: 用于自动匹配 —— 当用户请求代码清理、重构审计、内联函数、技术债评估时自动触发

## 修改技能

1. 编辑 `SKILL.md`，保持 YAML 前置元数据格式正确
2. 前置元数据中 `description` 字段控制自动触发匹配 —— 谨慎修改，避免误触发或漏触发
3. 技能正文使用 Markdown，示例代码块指定语言可获得语法高亮
4. 修改后运行 `/check-skills` 验证技能加载正常

## 内容风格

- 技能正文使用英文（Claude Code 技能生态标准）
- 方法论以具体代码示例驱动（Before/After diff 对比）
- 每个原则必须有决策表或正面/反面示例，避免空泛建议
