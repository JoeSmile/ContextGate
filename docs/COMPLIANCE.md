# Compliance — 个保法合规

## Data Flow

```
用户输入 → guardrails_input(PII脱敏) → LLM(无PII) → guardrails_output → 用户
                                              ↓
                                        审计日志(含原始输入)
```

## Data Retention

- 审计日志: 180 天
- 对话记录: 不限制（可配置 TTL）
- 缓存: 5 分钟（精确） / 24 小时（指纹）

## Data Isolation

- 租户行级隔离: 所有查询 WHERE tenant_id=:tid
- 跨租户访问: 仅 super_admin / auditor 角色
- 审计导出: auditor 可导出，不包含对话内容
