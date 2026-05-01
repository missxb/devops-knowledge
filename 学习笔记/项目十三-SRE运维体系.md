# 项目十三: SRE 运维体系

## 项目概述
SRE（Site Reliability Engineering）体系化方法论学习，涵盖稳定性衡量标准、指标体系、错误预算、故障处理全流程等核心概念。

## 核心知识点

### 1. 稳定性衡量指标
- **MTBF**（Mean Time Between Failure）：平均故障间隔时间，越大越好
- **MTTR**（Mean Time To Repair）：故障平均修复时间，越小越好
- 提升稳定性的两个方向：提升 MTBF（减少故障次数）、降低 MTTR（提升修复效率）

### 2. 系统可用性计算
- **时间维度**：Availability = Uptime / (Uptime + Downtime)
- **请求维度**：Availability = Successful Request / Total Request
- 衡量标准：非 5xx 占比，持续时间（如 95% 成功率，持续 10 分钟）

### 3. SLI 和 SLO
- **SLI（Service Level Indicator）**：服务等级指标，选择哪些指标衡量稳定性
- **SLO（Service Level Objective）**：服务等级目标，设定稳定性目标
- **VALET 方法**快速识别 SLI：
  - **V**olume（容量）：QPS、TPS、会话数、连接数
  - **A**vailability（可用性）：服务是否正常
  - **L**atency（时延）：响应是否足够快
  - **E**rrors（错误率）：5xx、4xx 错误情况
  - **T**ickets（人工介入）：是否需要人工处理

### 4. 错误预算（Error Budget）
- 通过 SLO 反向推导：100% - SLO = 错误预算
- 作用：提示还有多少犯错机会
- 应用场景：
  - 稳定性燃尽图
  - 故障定级
  - 稳定性共识机制
  - 基于错误预算的告警
- 规则：预算充足时容忍变更；预算即将耗尽时中止变更

### 5. MTTR 细分指标
- **MTTI**：故障平均确认时间（发现到响应）
- **MTTK**：故障平均定位时间（响应到定位）
- **MTTF**：故障平均修复时间（定位到修复）
- **MTTV**：故障平均验证时间（修复到验证）

### 6. On-Call 协作机制
- 确保关键角色在线
- 组织 War Room 应急组织
- 建立合理的呼叫方式
- 确保资源投入的升级机制

### 7. 故障处理原则
- 一切以恢复业务为最高优先级
- 故障复盘：根因分析、改进措施、经验沉淀

## 面试高频题

### Q1: SLI 和 SLO 的关系是什么？如何选择 SLI？
**答：** SLI 是衡量指标，SLO 是目标值。SLI 选择原则：① 能够标识主体稳定性；② 优先选择用户体验强相关的指标（时延、可用性、错误率）。使用 VALET 方法快速识别：Volume（容量）、Availability（可用性）、Latency（时延）、Errors（错误率）、Tickets（人工介入）。

### Q2: 什么是错误预算？如何应用？
**答：** 错误预算 = 1 - SLO（如 SLO=99.9%，错误预算=0.1%）。应用：① 稳定性燃尽图可视化剩余预算；② 故障定级基于消耗的预算比例；③ 预算充足时容忍变更，预算耗尽时冻结变更；④ 指导告警阈值设置。

### Q3: MTBF 和 MTTR 在 SRE 中的作用？
**答：** MTBF（平均故障间隔）衡量系统可靠性，MTTR（平均修复时间）衡量运维效率。SRE 的所有工作都可以归结为：提升 MTBF（减少故障发生）和降低 MTTR（加快故障恢复）。技术手段（监控告警）提升 MTTI，流程手段（On-Call）提升整体 MTTR。

### Q4: SLO 的有效性如何衡量？
**答：** 三个维度：① SLO 达成情况（Met/Missed）；② 人工投入程度（Toil High/Low）；③ 用户满意度（Customer Satisfaction High/Low）。好的 SLO 应该：达成率合理、人工投入低、用户满意度高。

## 学习心得
- SRE 不只是工具和系统，更是一套方法论和思维方式
- SLI/SLO 是 SRE 的基石，选错指标会导致后续所有工作方向偏差
- 错误预算是连接开发和运维的桥梁，用数据驱动变更决策
- 故障处理的 MTTR 细分（MTTI/MTTK/MTTF/MTTV）是优化运维效率的抓手
- On-Call 机制是缩短 MTTI 的关键，技术监控和流程机制缺一不可
