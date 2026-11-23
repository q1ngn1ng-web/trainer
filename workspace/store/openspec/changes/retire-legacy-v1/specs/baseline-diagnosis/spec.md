# baseline-diagnosis Specification

## REMOVED Requirements

### Requirement: 系统为新训练生成 3 道基线诊断题

**Reason**: 一代基线诊断流程已被「理解边缘探测」取代，相关代码（`baseline_service`）在现役产品中不可达。

**Migration**: 摸底能力由 `understanding-edge-assessment` 规格下的边缘探测承担；存量数据表与列保持不动。

### Requirement: 系统对用户作答评分并判定基线档位

**Reason**: 一代评分服务（`scoring_service`）在现役产品中不可达；边缘探测的判定由 `edge_probe_grade` 承担。

**Migration**: 无需迁移；`trainings.baseline_score`、`baseline_level` 与 `baseline_history` 表保持原样。

### Requirement: 基线「低」时生成预训练清单

**Reason**: 预训练清单（`pretrain_checklist`）只存在于一代流程，现役产品不可达。

**Migration**: 前置知识的补齐由路径骨架中的 `prerequisite` 类训练项承担。

### Requirement: 基线诊断完整结果写入训练文件

**Reason**: 一代的「训练文件落盘」机制（`file_writer`、`renderer`）已随十要素文件生成一并下线。

**Migration**: 无需迁移；`trainings.directory` 列保持原样。
