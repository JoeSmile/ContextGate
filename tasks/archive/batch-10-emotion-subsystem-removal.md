# Batch 10: 情绪子系统全量拆除 — emotion 变量 / 关键字 / API / DB 字段

> **包含:** 情绪业务域残留的全仓清除（89 文件 / ~1377 处）
> **预估:** 6-8 小时（分 9 个 Phase，每 Phase 一个 commit）
> **依赖:** Batch 8 门禁基线 + 本会话已完成的"核心路径拆除"（见 10.0）
> ⚠️ **原则:** 删的是情绪业务域（emotion/情绪/情感/共情/empathy/mood），不是通用安全能力（高风险关键词检测保留）
> ⚠️ **英文协议标识符**（EMOTIONAL_SUPPORT 等常量、emotion_state 字段名）：能删则删，删不动则保留常量但去掉行为分支——**禁止只改中文注释不改逻辑的半吊子**
> **Commit 约定:** 每 Phase 一个 commit，格式 `refactor: remove emotion subsystem from <layer>\n\nSigned-off-by: Joe`

---

## 10.0 基线（✅ 已完成 — 2026-08-01 Hermes 会话实测）

以下核心路径已在本次会话拆除完毕并验证通过，**Cursor 不要重复处理**：

| 范围 | 已删内容 | 验证 |
|---|---|---|
| 管线 | PipelineState 去 emotion/emotion_intensity；analyze_parallel 重写（只留意图+实体）；load_memory/write_memory 去 emotion；model_router 去 emotion 路由 | ruff ✅ / import 101 routes ✅ / pytest 15 passed ✅ |
| 模型 | models.py 去 Message.emotion、ChatSession.emotion_state、ChatResponse/MultimodalResponse 的 emotion 字段、EmotionAnalysis 类、EvaluationRequest/ComparePromptsRequest 的 user_emotion | 同上 |
| Schema | **整个 backend/schemas/ 包删除**（零引用死代码，含 EmotionType/EmotionTrend*） | 同上 |
| DB | legacy.py 删 EmotionAnalysis 整表 + chat_messages/response_evaluations/user_profiles/memory_items 的 emotion 列；pgvector_session 去 emotion/emotion_intensity/emotion_tags 列；vector_ops 去 emotion 参数 | 同上 |
| 核心 | interfaces.py 删 EmotionResult/IEmotionAnalyzer + IMemoryService/IChatEngine emotion 参数；factories 删 EmotionAnalyzerFactory/get_emotion_analyzer；formatters 删 format_emotion_result 等；skills/builtin/emotion_response.py 删除 | 同上 |
| 路由 | memory.py /users/{user_id}/emotion-trend、enhanced_chat.py emotion-insights、personalization.py preview 情绪参数、evaluation.py user_emotion 引用 全删 | 同上 |
| Agent 工具 | agent_tools.py 删 get_user_mood_trend + tool_caller.py 删 get_emotion_log 工具及 emotion_filter 参数 | 同上 |

> 复验命令：
> ```bash
> cd /Users/guowei/Desktop/github/contextgate
> uv run ruff check backend/ scripts/
> uv run python -c "from backend.app import app; print(len(app.routes))"
> LLM_MOCK=true uv run pytest tests/ -q --tb=short
> ```

---

## Phase 10.1: 品牌基类改名 — EmotionalChatException → ContextGateException

> **目的:** 异常基类还叫"EmotionalChat"是品牌残留。全仓仅 5 个文件引用（30 处）。
> **文件:**
> - `backend/core/exceptions.py` — `class EmotionalChatException` → `ContextGateException`；所有子类继承名同步改；`EXCEPTION_HANDLERS` 里的 key 同步
> - `backend/core/__init__.py` — import + `__all__` 同步
> - `backend/middleware/error_handler.py` — `except EmotionalChatException` → `except ContextGateException`
> - `backend/tests/unit/test_core.py` — import + test 名（test_emotional_chat_exception → test_context_gate_exception）
> - `backend/tests/integration/test_basic.py` — import + assert
>
> ⚠️ **Cursor 会在这里搞砸:** 不要用 `sed`/`grep -r 替换` 全局替换——先改 exceptions.py 定义，再改 4 个引用文件，然后 `grep -rn "EmotionalChatException" backend/` 确认零残留。子类（ConfigurationError 等）的继承括号里也要改。
>
> **验证:** `grep -rn "Emotional" backend/ --include="*.py"` 零命中 + `uv run ruff check backend/ scripts/`

---

## Phase 10.2: LLM 层

> **文件与动作:**

### 10.2.1 `backend/modules/llm/core/llm_core.py`（61 处）
- 删 `self.emotion_keywords = {...}` 情感关键词映射
- 删 `analyze_emotion()` / `_analyze_emotion_simple()` 及所有 emotion 分支（`emotion_scores`、`emotion_label`）
- 删 `save_emotion_analysis` 调用（DB 表已删）
- 检查 `ChatEngine.chat()` 签名里 user_emotion 参数（interfaces.IChatEngine 已删该参数）
- ⚠️ 该文件 2026-08-01 曾被 conda 报错污染过头部，**用 read_file 确认头部是 `#!/usr/bin/env python3` 再动手**；编辑用整文件 write_file，不要用 sed 原位替换

### 10.2.2 `backend/modules/llm/core/llm_with_plugins.py`（60 处）
- 删 `_analyze_emotion_simple`、`emotion_data`、`emotion=emotion_data["emotion"]` 传参、`save_emotion_analysis`
- 该文件已是 PluginManager no-op stub，删完 emotion 后确认仍可 import

### 10.2.3 `backend/modules/llm/services/llm_service.py`（16 处）
- 删 `analyze_emotion()` 方法（含 LLM prompt "请分析以下文本的情绪状态"、emotion/intensity/positive_score JSON 输出说明）
- 查调用方（enhanced_chat_service / optimized_chat_service / performance_optimizer 都调它）→ 同步删

### 10.2.4 `backend/services/performance_optimizer.py`（5 处）
- 删 `emotion_analyzer` 的 `asyncio.gather` 分支、`self.cache_key("emotion", ...)`、结果里的 `"emotion": emotion_result`

### 10.2.5 `backend/services/enhanced_chat_service.py`（30 处）
- 删 `emotion_result = self.chat_engine.analyze_emotion(message)` 步骤、`emotion`/`emotion_intensity` 变量与传参
- `get_emotion_insights` 方法（端点已删）整个删除

### 10.2.6 `backend/services/optimized_chat_service.py`（10 处）
- 删 `self.emotion_analyzer`（历史遗留：四个属性从未定义，2026-08-01 已去掉继承，这次把属性名也删干净）、`processing_result.get("emotion")`、`"emotion": {"emotion": "neutral", "intensity": 5.0}` 默认值

> **验证:** ruff + app import（101 routes 不变）+ pytest

---

## Phase 10.3: Intent 层

> **决策:** Intent 系统整体是"情感意图识别"——`IntentType.EMOTION` 枚举、emotion 分支、情感回复生成器全是情绪域。**删 emotion 意图与分支，保留 greeting/advice/default 通用意图。**

### 10.3.1 `backend/modules/intent/models/intent_models.py`（2 处）
- 删 `EMOTION = "emotion"  # 情绪表达（遗留意图）` 枚举值 + `"emotion": 0.65` 示例
- ⚠️ 枚举值删除后，所有 `IntentType.EMOTION` 引用点会断——按下面顺序同步删

### 10.3.2 `backend/modules/intent/core/intent_classifier.py`（2 处）
- 删 `IntentType.EMOTION` 分支（classifier 返回 emotion 的分支逻辑）

### 10.3.3 `backend/modules/intent/core/rule_engine.py`（1 处）
- 删 `IntentType.EMOTION: [...]` 规则

### 10.3.4 `backend/modules/intent/services/intent_service.py`（7 处）
- 删 `IntentType.EMOTION: {...}` 策略分支（"情感验证"/"提供情绪宣泄空间"）、emotion 相关的 prompt_hint
- 同步删 `extract_memories(..., emotion=...)` 传参

### 10.3.5 `backend/modules/intent/routers/intent_router.py`（4 处）
- 删 "情感表达" name、`"emotion": {"primary": "焦虑"}` 示例

### 10.3.6 `backend/modules/intent/core/dynamic_prompt_builder.py`（45 处）
- 删 `EMOTION_DRIVEN_TEMPLATE` 整块（含"状态标签/状态强度/关注程度"模板变量）
- 删 `emotion_state`/`emotion_label`/`emotion_intensity`/`empathy_level` 相关模板与辅助函数
- ⚠️ 本文件 2026-08-01 已做"内容中性化"（人格/文案已改），这次是**结构拆除**——删 EMOTION_DRIVEN_TEMPLATE 与 emotion 分支函数。删前先 `grep -n "emotion"` 列出全部 45 处，逐块删，不要正则一刀切

### 10.3.7 `backend/modules/intent/core/response_generator.py`（62 处）
> **决策:** 该模块功能=情感回复生成（情感匹配决策引擎/一致性校验/共情回复），**整个模块是情绪域本体**。
> **选项 A（推荐）:** 删除整个模块 + 调用方（intent_service 里的引用、routers 里的引用）
> **选项 B:** 保留模块但把所有 emotion 分支改为 default——不推荐（改完就是空壳，比删更难看）
> 选 A 时: `git rm backend/modules/intent/core/response_generator.py`，然后 `grep -rn "response_generator" backend/` 清理所有 import/调用

### 10.3.8 `backend/modules/intent/core/enhanced_input_processor.py`（3 处）
- 删 emo 简写映射（"emo了"→"情绪不好了"等）

### 10.3.9 `backend/modules/intent/__init__.py`（2 处）
- docstring "情感意图识别模块 / Intent Recognition Module for Emotional Chat" → 通用意图识别描述

> **验证:** `grep -rn "EMOTION\|IntentType.EMOTION" backend/modules/intent/` 零命中 + ruff + app import

---

## Phase 10.4: Agent 层

> **决策:** agent 的 emotion 有两条线：**感知线**（agent_core 的 EmotionAnalyzer/emotion_analyzer 属性）和**记忆线**（memory_hub 的情绪记忆 + reflector 的情绪反思 + planner 的情绪路由）。感知线直接删；记忆线删 emotion 字段与分支，保留记忆检索主流程。

### 10.4.1 `backend/agent/agent_core.py` + `backend/modules/agent/core/agent/agent_core.py`（各 44 处）
- 删 `from backend.emotion_analyzer import EmotionAnalyzer`（⚠️ **该模块 Batch 3.1 已删**，import 在 try/except 软降级里——整个 try/except 块一起删）
- 删 `self.emotion_analyzer = EmotionAnalyzer()` / `self.emotion_analyzer = None`、perception 里的 `"emotion"`/`"emotion_data"` 键
- 删 `print(f"内容分析失败: {e!s}")` 相关的 emotion 分析调用链

### 10.4.2 `backend/agent/agent_core_v2.py`（44 处）
- 删 `EmotionSkill.execute(mode="analyze")` 感知链、`emotion_analyzer` 构造参数、`emotion_tag` 事件字段
- ⚠️ `AssistantEvent(text_delta, emotion_tag=...)` — emotion_tag 字段删了要同步所有 emit 调用点

### 10.4.3 `backend/agent/memory_hub.py` + `backend/modules/agent/core/agent/memory_hub.py`（40/54 处）
- 删 emotion 记忆字段（`experience.get("emotion")`、`memory["emotion"]`）、"情绪关联检索（情绪一致性）"分支、`PREFS_EMOTION_PATH`、L3 情绪基线
- ⚠️ 六层记忆架构的 L3/L4 有情绪基线/情绪响应策略路径——删 emotion 相关路径，保留 topic/user preference 路径

### 10.4.4 `backend/agent/reflector.py` + `modules/agent/.../reflector.py`（各 54 处）
- 删 `emotion_state` 读取（`mcp_message.context.emotion_state.get("emotion")`）、"3. 检查情绪异常" 分支、EMOTIONAL_SUPPORT 相关反射逻辑

### 10.4.5 `backend/agent/planner.py` + `modules/agent/.../planner.py`（各 24 处）
- 删 `EMOTIONAL_SUPPORT`/`EMPATHY_FIRST` 常量与路由分支、`perception.get("emotion")`/`emotion_intensity` 读取、`if emotion_intensity >= 7.0` 分支
- ⚠️ 删分支后确认 `_identify_goal` 的 fallback 逻辑完整（不要留下 None 分支死路）

### 10.4.6 `backend/agent/activity_distiller.py`（39 处）
- 删 emotion_baseline / emotion_response pattern 的提取与合并（`merge_emotion_response_patterns` 等）

### 10.4.7 `backend/modules/agent/protocol/mcp.py`（32 处）
- 删 `MessageContext.emotion_state` 字段（`"emotion_state": {}` 默认值、序列化、`emotion`/`emotion_intensity` 参数）
- ⚠️ 这是**协议字段**——删了要同步所有 `emotion_state=` 传参方（reflector/agent_core 等）。先 grep 全仓 `emotion_state` 列出调用方再删

### 10.4.8 `backend/modules/agent/core/agent/tool_caller.py`（28 处）
- 同 10.0 已删的 `backend/agent/tool_caller.py`：删 `get_emotion_log` 工具注册 + `_get_emotion_log` 实现 + emotion_filter 参数

### 10.4.9 `backend/agent/tools/psychology_db.py` + `modules/agent/.../psychology_db.py`（各 15 处）
- 删情绪内容条目（"如何应对焦虑情绪"/"情绪日记"/emotional_awareness 分类）
- ⚠️ 2026-08-01 已中性化热线条目——这次是删情绪知识内容本身，保留工具骨架或整体删（工具已被 agent_tools 引用，删骨架即可）

### 10.4.10 `backend/agent/tools/audio_player.py` + `modules/agent/.../audio_player.py`（各 7 处）
- 删 `emotion` 参数、情绪到主题映射、"根据用户情绪推荐音频"逻辑——保留播放功能，去掉情绪推荐

### 10.4.11 小文件
- `backend/modules/agent/models/agent_models.py` — 删 `EMOTION_ANALYSIS` 常量 + user_emotion 字段
- `backend/services/agent_service.py` + `backend/modules/agent/services/agent_service.py` — 删 perception 里的 `"emotion"` 键
- `backend/agent/memory_store.py` — docstring "适配 emotional_chat 场景" → "适配 ContextGate"

> **验证:** `grep -rn "emotion" backend/agent/ backend/modules/agent/ | grep -v __pycache__` 归零 + ruff + app import（⚠️ AGENT_ENABLED 默认关，import 不炸不代表运行时 OK——改完跑一次 `uv run python -c "from backend.agent.agent_core import AgentCore"` 冒烟）

---

## Phase 10.5: Services 层

### 10.5.1 `backend/services/proactive_recall_system.py`（64 处）
> **决策:** `EmotionTracker` 类 = 该模块全部功能（情感追踪/主动关怀）。**整体删除模块** + 清理引用。
> ```bash
> git rm backend/services/proactive_recall_system.py
> grep -rn "proactive_recall" backend/ --include="*.py" | grep -v __pycache__   # 清引用
> ```

### 10.5.2 `backend/services/user_profile_builder.py`（41 处）
- 删 `_analyze_emotional_trend`、`avg_emotion_intensity` 计算、`emotional_trend` 字段

### 10.5.3 `backend/services/enhanced_memory_manager.py`（29 处）
- 删 `should_extract_memory(..., emotion=...)` / `extract_memories(..., emotion=...)` 参数、`"emotion": emotion or "neutral"` / `"intensity"` 输出字段、importance 的情绪强度计算（改固定 0.5 或按长度）

### 10.5.4 `backend/services/enhanced_context_assembler.py`（14 处）
- 删 `current_emotion` 上下文块 + emotion 参数

### 10.5.5 `backend/services/context_service.py`（4 处）
- 删 emotion/emotion_intensity 参数与输出键

### 10.5.6 `backend/services/context_rot_solver.py`（11 处）
- 删 `_extract_emotion_trend` 与 `"emotional_state"` 输出键

### 10.5.7 `backend/services/prompt_composer.py`（11 处）
- 删 `compose(context, emotion_state=None)` 的 emotion_state 参数 + `_build_emotion_prompt` stub + `emotion_prompt` 变量
- ⚠️ **调用方**：personalization.py 预览端点（已不传）+ personalization_service.py（还传）——同步删

### 10.5.8 `backend/services/personalization_service.py`（7 处）
- 删 `empathy_level` 配置键（`"empathy_level": 0.8` 默认、`config_db.empathy_level` 读取）、`emotion_state` 参数
- ⚠️ `empathy_level` 在 models.py 的 PersonalizationConfig 里也有（见 10.7.3）+ DB user_profiles.empathy_level 列（10.7.4）——三处联动，一次删完

> **验证:** ruff + app import + pytest

---

## Phase 10.6: Runtime 层

### 10.6.1 `backend/runtime/config/toggles.py`（7 处）
- 删 `emotion_skill` toggle（EMOTIONAL_CHAT__MODULES__EMOTION_SKILL__ENABLED 文档、`toggles.set_enabled("emotion_skill", False)` 示例）

### 10.6.2 `backend/runtime/config/guards.py`（3 处）
- 删 `is_module_enabled("emotion_skill")` 分支 + `emotion_skill.execute(context)` 调用

### 10.6.3 `backend/runtime/conversation/_turn.py`（12 处）
- 删 `emotion_skill` Skill 链环节、`context.emotion_data` 写入、`emotion_tag` 读取
- ⚠️ Skill 链是 `emotion → memory → planning → tool → reflect`——删 emotion 环节后链从 memory 开始

### 10.6.4 `backend/runtime/conversation/_lifecycle.py`（5 处）
- 删 `emotion_analyzer=None` 构造参数与透传（⚠️ `_ = emotion_analyzer # 保留参数兼容` 这种 stub 参数直接删，同步所有构造调用方）

### 10.6.5 `backend/runtime/conversation/_helpers.py`（2 处）
- 删 `emotion_tag` 字段与输出

### 10.6.6 `backend/runtime/skills/planning_skill.py`（14 处）
- 删 `EMOTIONAL_SUPPORT`/`EMPATHY_FIRST` 常量、`emotion_data` 读取与 `_identify_goal(user_input, emotion_data)` 分支——改 `_identify_goal(user_input, context)`

### 10.6.7 `backend/runtime/skills/reflect_skill.py`（8 处）
- 删 `emotion_data` 读取、`is_crisis` 分支、情绪回访类型判断

### 10.6.8 `backend/runtime/skills/memory_skill.py`（2 处）
- 删 `context.emotion_data.get("emotion")` 键

### 10.6.9 `backend/runtime/skills/base.py`（4 处）
- 删 `emotion_data`/`emotion_tag` 字段（⚠️ 是所有 Skill 的基类——字段删了要同步所有子类引用）

### 10.6.10 `backend/runtime/policy/policy_engine.py`（12 处）
- 删 `condition="emotion.is_crisis == true"`、`"emotion": {"is_crisis": True}` 测试数据、`high_intensity_emotion` 规则、`emotion.intensity >= 9.0` 条件

### 10.6.11 `backend/runtime/activity/distiller.py`（16 处）
- 删 `_EMOTION_TRENDS_LIMIT`、`_EMOTION_PATH`、`emotions_updated` 输出键、`emotion_tag` 字段

### 10.6.12 `backend/runtime/activity/tracker.py`（1 处）
- 删 `tracker.record_skill("emotion_skill", ...)` 示例

### 10.6.13 `backend/runtime/hooks/base.py`（3 处）
- 删 `EmotionTrackingHook` 注册与 `emotion_tag` 字段（"情感追踪 Hook — 在 post_llm_call 中记录情感变化"注释）

### 10.6.14 `backend/runtime/protocols/llm_client.py`（3 处）
- 删 `emotion_tag` 字段（"empathy" | "encouragement" 注释）、`emotion_analysis` 遗留字段、docstring "Adapted for emotional chat"

### 10.6.15 `backend/runtime/protocols/tool_executor.py`（2 处）
- 删 `tool_category="emotion"` 枚举值 + docstring

### 10.6.16 `backend/runtime/protocols/permission_prompter.py`（1 处）
- docstring "Adapted for emotional chat" → "Adapted for ContextGate"

### 10.6.17 `backend/runtime/workspace/manager.py`（3 处）
- 删 `EMOTIONAL_CHAT_WORKSPACE_BASE` env 读取、`~/.emotional_chat` 默认路径（⚠️ 改路径会影响 workspace 初始化——确认 run_backend.py 没有依赖旧路径）

### 10.6.18 `backend/runtime/task_packet.py`（4 处）
- 删 `"帮助用户缓解焦虑" → emotion_skill + ...` 示例、`emotion_context` 字段、`skills=["emotion_skill", ...]` 默认值

### 10.6.19 `backend/runtime/__init__.py`（1 处）
- docstring "Runtime — Emotional Chat Agent Runtime" → "Runtime — ContextGate Agent Runtime"

> **验证:** ruff + `uv run python -c "from backend.runtime import ..."`（核心类 import 冒烟）

---

## Phase 10.7: 评估系统

> **决策:** evaluation 的评分维度含 empathy（共情程度），是情绪域。改为准确性/完整性/安全性（2026-08-01 已改 prompt 文案，这次删字段）。

### 10.7.1 `backend/evaluation_engine.py`（44 处）
- 删 `user_emotion`/`emotion_intensity` 参数（evaluate/compare_prompts 签名）、`"用户状态：{user_emotion}"` 模板、`empathy_score`/`empathy_reasoning` 输出字段
- ⚠️ 评分字段删了要同步 DB response_evaluations 列（10.7.4）+ routers/evaluation.py（10.7.2）

### 10.7.2 `backend/routers/evaluation.py`（11 处）
- 删 `empathy_score`/`empathy_reasoning` 读写

### 10.7.3 `backend/models.py`（5 处）
- `EvaluationResponse` 删 `empathy_score` 字段；`FeedbackRequest.feedback_type` 注释里的 lack_empathy 值；PersonalizationConfig 的 `empathy_level` 字段（⚠️ 见 10.5.8 联动）

### 10.7.4 `backend/database/legacy.py`（14 处）
- response_evaluations 表删 `empathy_score`/`empathy_reasoning` 列；user_profiles 表删 `empathy_level` 列（2026-08-01 已删 emotional_baseline/avg_emotion_intensity）
- user_feedback 表 feedback_type 注释去 lack_empathy

### 10.7.5 `alembic/versions/002_add_response_evaluation.py`（迁移）
- 追加新迁移删除列（或直接改 002 + 重建 dev DB——**本地 dev 选重建**：`uv run python -m scripts.reset_db` 或 drop schema 后重跑）

> **验证:** ruff + pytest + 本地 PG 上跑一次查询确认列已删

---

## Phase 10.8: RAG 层

### 10.8.1 `backend/modules/rag/services/rag_service.py`（17 处）
- 删 `user_emotion` 参数、"构建情绪上下文"、`emotion_context` 拼接

### 10.8.2 `backend/modules/rag/routers/rag_router.py`（6 处）
- 删 `user_emotion` 请求字段与示例、`"category": "情绪调节"` 条目

### 10.8.3 `backend/modules/rag/models/rag_models.py`（3 处）
- 删 `user_emotion` 字段、`emotion_triggers` 字段

### 10.8.4 `backend/modules/rag/core/knowledge_base.py`（5 处）
- 删情绪调节类文档条目（"思维影响情绪"/"抑郁情绪"/"焦虑是一种常见的情绪体验"）

> **验证:** ruff + app import

---

## Phase 10.9: 脚本 + 测试

### 10.9.1 `scripts/simple_backend.py`（45 处）
- 删 `EmotionAnalysisRequest`/`EmotionAnalysisResponse` 模型、`/api/emotion/analyze` 端点、标题/描述里的"情感聊天机器人"
- ⚠️ 这是独立演示脚本（FastAPI 单文件），删 emotion 端点即可，不影响主 app

### 10.9.2 `scripts/demo_agent.py`（5 处）
- 删"情感支持"场景、`result['emotion']` 打印

### 10.9.3 `scripts/seed_pgvector.py`（5 处）
- 删 emotion/intensity 种子数据（chat["messages"] 里的 emotion 字段）

### 10.9.4 `scripts/quick_start.py`（1 处）
- ⚠️ **品牌词**: "心语情感陪伴机器人 - 快速启动" → "ContextGate - 快速启动"（make verify 会查"心语"）

### 10.9.5 `scripts/test_rag_eval.py`（3 处）
- 删情绪类测试 query（情感隔离/负面情绪/分手哀伤）

### 10.9.6 `backend/tests/__init__.py`（9 处）
- 删 `create_mock_emotion_analysis`、mock 数据里的 emotion 字段

### 10.9.7 `backend/tests/unit/test_core.py`（9 处）
- 删 `extract_emotion_keywords` 导入与测试、EmotionalChatException 测试（10.1 改名后同步）

### 10.9.8 `backend/tests/integration/test_basic.py`（5 处）
- EmotionalChatException 引用同步（10.1）

> **验证:** `make verify`（品牌 grep）+ ruff + pytest

---

## Phase 10.10: 扫尾 + 最终验证

### 10.10.1 剩余小件
- `backend/core/utils/helpers.py` — 删 `extract_emotion_keywords()` 函数与 __main__ 测试行
- `backend/core/utils/__init__.py` — 删 helpers/formatters 的 emotion 导出
- `backend/core/utils/formatters.py` — 确认无残留（10.0 已清）
- `backend/config/performance_config.py` — 删 `CACHE_EMOTION_TTL` + `"emotion_ttl"` 配置键
- `backend/routers/streaming_chat.py` — 删 `"type": "analysis", "emotion": ...` SSE 事件字段
- `backend/routers/personalization.py` — 删 `empathy_level` 默认值（联动 10.5.8/10.7.3）
- `backend/pipeline/nodes/cache_check.py` — docstring 已改（无情绪词表），确认 `_cheap_fingerprint` 无 emotion 分支

### 10.10.2 全仓终检
```bash
cd /Users/guowei/Desktop/github/contextgate
# 1. 情绪域残留（目标：backend/ 核心路径 0 命中；遗留层只允许"已中性化"的协议常量）
grep -rn "emotion\|情绪\|情感\|共情\|empathy\|mood" backend/ --include="*.py" | grep -v __pycache__ | grep -v "emotional_chat 迁移\|遗留"
# 2. 品牌词
grep -rn "心语\|XINYU\|情感陪伴" backend/ scripts/ --include="*.py" | grep -v __pycache__   # 期望 0
# 3. 门禁
uv run ruff check backend/ scripts/
uv run mypy
uv run python -c "from backend.app import app; print(len(app.routes))"
LLM_MOCK=true uv run pytest tests/ -q --tb=short
# 4. DB 列（本地 PG）
psql postgresql://contextgate:...@localhost:5432/contextgate -c "\d chat_messages"   # 无 emotion 列
```

### 10.10.3 Commit 计划
```bash
git add -A && git commit -m "refactor: rename EmotionalChatException to ContextGateException

Signed-off-by: Joe"
git add -A && git commit -m "refactor: remove emotion subsystem from LLM layer

Signed-off-by: Joe"
# ... 每 Phase 一个 commit，按 10.1 → 10.9 顺序
```

---

## ⚠️ Cursor 全局坑清单（必读）

1. **conda 污染**: 本机 shell 每条命令前会打印 conda activate 报错。**不要用 `sed -i` 原位编辑**——报错可能被写进文件头。编辑用整文件 write_file / python 脚本替换，改完 `head -1 文件` 确认是 shebang。
2. **模块已删的 import**: `backend/emotion_analyzer`、`backend.plugins`、`backend.schemas` 已不存在。任何 `from backend.emotion_analyzer import` 都是死 import——连 try/except 块一起删。
3. **协议字段联动**: `emotion_state`（mcp.py）、`emotion_tag`（skills/base.py + hooks）、`emotion_data`（skills/base.py）是跨文件字段——删之前先 `grep -rn "字段名" backend/` 列出全部引用点，一次删完。
4. **枚举联动**: `IntentType.EMOTION` 删除会断 intent_classifier/rule_engine/intent_service/intent_router——按 10.3 顺序删。
5. **DB 列联动**: `empathy_level`（models.py + personalization_service + legacy.py user_profiles）、`empathy_score`（evaluation_engine + routers/evaluation + legacy.py response_evaluations）——三处联动一次删。
6. **不要删通用安全能力**: crisis_intervention.py 的 HIGH_RISK_KEYWORDS（自残/轻生关键词检测）是 P0 安全功能，**保留**；只删 emotion_strategy 参数和旧话术（2026-08-01 已改）。
7. **每 Phase 跑门禁**: 不等到最后。`uv run ruff check backend/ scripts/` + app import 必跑。
8. **半吊子检查**: 只改注释不改逻辑 = 半吊子，验收会打回。删 emotion 分支时确认 `default` 路径完整（planner 的 goal fallback、_turn 的 Skill 链起点）。

---

## 验收标准

- [x] `grep -rn "emotion\|情绪\|情感\|共情\|empathy" backend/ --include="*.py"`（核心路径）≈ 0；遗留层仅剩已中性化的协议常量（如 EMOTIONAL_SUPPORT 注释标注"遗留"）
- [x] `grep -rn "心语\|XINYU\|情感陪伴" backend/ scripts/` = 0
- [x] ruff / mypy / pytest / app import 全绿
- [x] DB 三张表（chat_messages / memory_items / response_evaluations / user_profiles）无 emotion/empathy 列
- [x] `make verify` 通过
- [x] 每 Phase 一个 commit，`Signed-off-by: Joe`
