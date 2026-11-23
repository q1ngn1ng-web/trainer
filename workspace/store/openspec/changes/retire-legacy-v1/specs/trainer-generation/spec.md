# trainer-generation Specification

## REMOVED Requirements

### Requirement: 用户能创建新训练主题

**Reason**: 一代的「主题输入页」属于已下线的 Streamlit 界面；现役产品通过向导创建训练。

**Migration**: 由 `goal-clarification` 规格下的「用户能用自然语言描述学习内容」承担。

### Requirement: 主题必须通过具体性校验

**Reason**: 具体性校验（`topic_validation`）在现役产品中不可达。

**Migration**: 无替代校验。目标是否清晰改由澄清对话与用户确认来保证；若日后需要入口处的具体性校验，另立提案。

### Requirement: 系统自动生成关键词白名单

**Reason**: 关键词白名单只服务一代的基线出题与文件生成，两者均已下线。

**Migration**: 无需迁移；`trainings.keywords`、`must_cover_count`、`forbidden` 列保持原样。

### Requirement: 系统按十要素模板生成 10 份训练文件

**Reason**: 十要素文件生成（`trainer_service`、`renderer`、模板）已被现役的路径骨架与排期取代；模板已在 89c2909 删除。

**Migration**: 无需迁移。

### Requirement: 创建完成后跳转到日常 surface

**Reason**: 该跳转属于已下线的 Streamlit 界面导航。

**Migration**: 现役前端在向导第 4 步进入训练。
