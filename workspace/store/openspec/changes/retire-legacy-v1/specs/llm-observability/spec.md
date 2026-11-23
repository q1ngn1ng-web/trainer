# llm-observability Specification

## ADDED Requirements

### Requirement: 提示词注册表只保留有调用方的用途

`PROMPT_REGISTRY` 与 `SCHEMA_REGISTRY` 中的每个用途 SHALL 至少有一个现役调用点
（可从 `src/api/app.py` 或命令行入口到达的模块）。删除某个用途时，其提示词、Schema、
静态降级与专属校验 MUST 一并删除。

#### Scenario: 注册表与调用点一致

- **WHEN** 遍历 `PROMPT_REGISTRY` 的每个用途，在 `src/` 中查找 `complete()` 的调用点
- **THEN** 每个用途至少有一个调用点位于现役可达的模块里

## REMOVED Requirements

### Requirement: 关键词白名单的覆盖率校验独立于 JSON Schema

**Reason**: 关键词白名单只存在于一代的关键词生成与基线出题用途，两者均无现役调用方。

**Migration**: 无需迁移；JSON Schema 校验对全部现役用途继续生效。
