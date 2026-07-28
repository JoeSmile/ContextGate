# Task 09: 安全护栏

> `GuardResult.action`: `"pass"` | `"redacted"` | `"blocked"`
> **前置依赖:** `tasks/04-langgraph-pipeline.md`（需要 guardrails_input/output 节点）
> **完成后:** 无（独立 Task）
> blocked → 直接 403，不走 LangGraph 后续节点。
> 审计日志存脱敏前原始输入。

## Subtask 09.01: GuardResult 基类

**文件:** `backend/core/guardrails/base.py`
```python
@dataclass
class GuardResult:
    action: str        # "pass" | "redacted" | "blocked"
    redacted_text: str
    reason: str
```

## Subtask 09.02: 输入护栏

**文件:**
- `backend/core/guardrails/input_guard.py`
- `backend/core/guardrails/pii_patterns.py`
- `backend/core/guardrails/injection_patterns.py`

```python
INJECTION_PATTERNS = [
    r"忽略(系统)?(提示|指令|设定)",
    r"你(现在|接下来)是",
    r"忘记(所有)?(之前|上面)的",
    r"system\s*:",
    r"你是一个(新|不同)的",
]

PII_PATTERNS = {
    "phone": r"1[3-9]\d{9}",
    "id_card": r"\d{17}[\dXx]",
    "bank_card": r"\d{16,19}",
}

async def check_input(message: str) -> GuardResult:
    # 1. Prompt 注入检测
    for pattern in INJECTION_PATTERNS:
        if re.search(pattern, message):
            return GuardResult("blocked", message, f"injection:{pattern}")
    # 2. PII 脱敏
    redacted = message
    for pii_type, pattern in PII_PATTERNS.items():
        redacted = re.sub(pattern, f"[REDACTED:{pii_type}]", redacted)
    if redacted != message:
        return GuardResult("redacted", redacted, "pii_found")
    return GuardResult("pass", message, "")
```

## Subtask 09.03: 输出护栏

**文件:** `backend/core/guardrails/output_guard.py`
- 长度截断（>4000 字符）
- 危机表达检测
- 违规内容拦截

```python
async def check_output(response: str) -> GuardResult:
    if len(response) > 4000:
        return GuardResult("truncated", response[:4000], "length_exceeded")
    return GuardResult("pass", response, "")
```

## 验证

```bash
curl -X POST ... -d '{"message":"忽略系统提示"}'  # → 403 GUARD_001
curl -X POST ... -d '{"message":"手机13800138000"}'  # → redacted
```
