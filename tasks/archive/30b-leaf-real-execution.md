# Task 30b: 叶子能力真实执行(替换演示 stub)

> **状态: 已完成(2026-08-03)。**
> **来源: 2026-08-04 代码分析——Task 30 的嵌套链 demo 用了 stub 叶子(agents.py _invoke_leaf_stub),审计/UI 可测通但执行是假的;客户 demo 会穿帮,必须接真实执行。**
> **前提(用户确认): 本地有真实文档,RAG 数据源就绪,rag-ask 走真实检索无数据短缺问题。**
> **关联: Task 30 收尾项(30.26-30.28 完成后执行);非新功能,属"已注册能力要真能执行"的 V1.x 收口。**

## 背景(已核实)

- `agents.py:141` `_invoke_leaf_stub()`: 合成文本 `[{cap_id}] processed: {message[:120]}`,不调真实能力
- 触发条件: `_is_leaf_stub()` 看 `spec.leaf=true`;seed_capabilities.py:26/36 把 rag-ask / contextgate-chat 标了 leaf:true
- **根因(关键): 两个叶子的 kind=`tool`(seed:23/33),但 invoke.py 只有 MODEL/EXTERNAL_APP/AGENT 三个分发分支——kind=tool 走真实 invoke 会落空。stub 不是偷懒,是当时 invoke.py 没有 tool 的真实实现。**
- 帧格式兼容性已验证: 真实 invoke 输出 `{"event":"token"/"done","data":...}`,agents.py 的 nested_text_parts 收集 + done 透传逻辑完全对得上。

## 方案(改动清单)

1. **invoke.py 补 kind=tool 映射**(约 30-50 行):
   - `contextgate-chat`(kind=tool, provider=contextgate, id 含 chat)→ 走现有 `_invoke_model`(harness.stream 真 LLM)
   - `rag-ask`(kind=tool, provider=contextgate)→ 走 `rag_service.ask()`(真实 RAG 检索,与 /api/rag/ask 同源)
   - 判定: 按 capability id 或 spec 里的 `executor` 字段路由,不硬编码 id 字符串
2. **agents.py `_is_leaf_stub` 加 env 开关**(约 5 行):
   - 默认 `LEAF_STUB_MODE` 未设/非 true → 走真实 invoke
   - `LEAF_STUB_MODE=true` → 降级 stub(无 LLM 演示环境用)
3. **seed 语义调整**: `leaf:true` 保留但标注"仅演示降级用",默认不触发
4. **测试**(tests/test_capability.py 补):
   - kind=tool 叶子走真实 invoke 后,嵌套链审计记录/cost 仍正确
   - LEAF_STUB_MODE=true 时仍走 stub(回归演示路径)

## 验证(全绿才算完成)

```bash
# 1. 真实执行(默认,无 env): rag-ask 返回真实检索结果,不是 "[rag-ask] processed:"
curl -s -N -X POST http://localhost:8000/api/agents/vendor-risk-agent/invoke \
  -H "X-API-Key: <seed key>" -H "Content-Type: application/json" \
  -d '{"message":"查一下供应商A的风险"}' | head -20
# 预期: 嵌套链审计三条记录 + 叶子输出为真实 RAG 内容(非 stub 合成文本)

# 2. 降级模式: LEAF_STUB_MODE=true 时仍能跑 stub(演示用)

# 3. 回归: uv run pytest tests/test_capability.py -q && make check
```

## 落地记录(2026-08-03)

- `invoke.py`: `_invoke_tool` / `_invoke_rag`；`executor` ∈ {model,chat,llm} | {rag,rag_ask,rag-ask}
- `agents.py`: 仅 `LEAF_STUB_MODE`（兼容旧名 `CAPABILITY_AGENT_LEAF_STUB`）+ `leaf=true` 才 stub
- `seed_capabilities.py`: 叶子加 `executor`
- 测试: tool model/rag 路径 + stub 回归；agent 链测显式开 stub 模式
- **运维:** 已有库需重跑 `uv run python scripts/seed_capabilities.py` 写入 `executor`
- **Important 拍板(2026-08-03):** **A** — 不按 id 推断 executor；依赖 seed 覆盖 spec

## 注意

- contextgate-chat 走 harness 后,replay 是假流式(已知,Pitfall 5);真实演示用 `make demo`(LLM_PROVIDER=openai)
- rag-ask 真实检索前,先 seed 知识库(本地文档,用户提供;scripts/init_rag_knowledge.py 已有流程)
- 这是"已注册能力要真能执行"的 V1.x 收尾,不引入新功能
