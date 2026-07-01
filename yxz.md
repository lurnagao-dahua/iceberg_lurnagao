# Kafka 上游 Patch 自动回迁流水线

## 1. 项目简介

本项目是一个 **Kafka 上游 Patch 自动回迁流水线**，目标是将 Apache Kafka 社区的高价值 bugfix 自动筛选、评估、回迁到公司的 **Kafka 2.0.1 内部增强版**。

核心思路：

1. 从 Kafka 上游 Release Notes / JIRA / Git commit 中收集所有 issue
2. 多维度评分，过滤出与 broker 稳定性、日志恢复、副本管理相关的服务端修复
3. 调用 [CodeBuddy](https://copilot.qq.com/) AI Agent 自动评估每个 issue 是否适合回迁
4. 对低风险 issue 自动执行最小语义回迁，生成 diff / 测试报告 / MR 描述
5. 人工 reviewer 验收合入

### 为什么需要这个项目

公司使用 Kafka 2.0.1 内部增强版，与上游 trunk 差距很大（3.x/4.x 做了大量架构重构如 KRaft、Tiered Storage 等）。不能直接升级，但上游持续修复的 bug 对 2.0.1 仍有价值。手工逐个排查效率极低（上游有数千个 issue），需要自动化流水线来筛选和回迁。

### 关注的领域

- **broker 端稳定性**：启动、关闭、异常恢复
- **log / segment / index / timeIndex**：日志段、索引文件
- **checkpoint**：恢复点、leader epoch、offset checkpoint
- **recovery**：日志恢复、崩溃恢复
- **log cleaner**：日志清理器
- **replica / leader / follower / ISR**：副本管理
- **data loss**：数据丢失防护
- **磁盘 / 文件描述符 / 日志目录**：资源泄漏

### 默认排除的领域

- KRaft / Tiered Storage / Remote Log / ShareGroup（新架构，2.0.1 没有）
- Streams / Connect（外围模块）
- Producer / Consumer / AdminClient 客户端
- 纯测试、纯文档、纯重构变更
- 协议版本 / API 语义变更

---

## 2. 整体架构

```
┌─────────────────────────────────────────────────────────────────────────┐
│                        两条候选池流水线                                   │
├──────────────────────────────┬──────────────────────────────────────────┤
│   Fullscan（历史存量）        │   Weekly（每周增量）                       │
│   01 → 02 → 03 → 04 → 05    │   08                                     │
│   收集 → 富化 → 评分 → 映射  │   收集 → 富化 → 评分                     │
│         → commit 风险评分    │   → commit 风险评分（含内部文件检查）      │
├──────────────────────────────┼──────────────────────────────────────────┤
│         07（批量评估入口）     │         09（批量评估入口）                 │
│   预筛选 → 逐个调用 06       │   预筛选 → 逐个调用 06                   │
├──────────────────────────────┴──────────────────────────────────────────┤
│                    06（单 issue 评估 Worker）                             │
│   调用 CodeBuddy Agent → evaluation-only                                 │
│   产出：issue_context.md / evaluation.md / backport_plan.md /           │
│         final_review.md（含 ## Decision: 标签）                          │
├─────────────────────────────────────────────────────────────────────────┤
│              11（批量回迁入口）                                           │
│   扫描 backport-candidate → 逐个调用 10                                  │
├─────────────────────────────────────────────────────────────────────────┤
│                    10（单 issue 回迁 Worker）                             │
│   调用 CodeBuddy Agent → apply-backport                                  │
│   创建分支 → 最小语义回迁 → git commit                                   │
│   产出：internal.diff / test_report.md / mr_description.md /            │
│         commit_message.txt / apply_status.txt                           │
└─────────────────────────────────────────────────────────────────────────┘
```

### 两阶段安全闸门设计

| 阶段                | 脚本 | 做什么                                                 | 不做什么                            |
| ------------------- | ---- | ------------------------------------------------------ | ----------------------------------- |
| **evaluation-only** | 06   | 读取上游 diff，分析内部代码，判断适配性，生成评估文档  | 不修改代码、不 git 操作、不编译测试 |
| **apply-backport**  | 10   | 基于评估产物，执行最小语义回迁，修改代码并提交本地分支 | 不 push、不创建 MR、不 merge        |

---
