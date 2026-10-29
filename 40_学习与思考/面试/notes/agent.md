```
ReAct：
0.Thought-Action-Observation
1.相比CoT能减少幻觉，原因是observation引入外部事实，推理链有环境反馈锚点，实现和事实的In-Context对齐。
2.把推理空间和动作空间统一到一个交错序列中。
```

```
workflow & Agent
1.workflow是用DAG、状态机硬编码的确定性流程；
2.Agent由LLM动态决策；
3.近期工程实践是用workflow约束Agent，用workflow定义主干流程，在流程内给Agent局部决策权。
```

```
Agent长短期记忆
1.短期记忆使用Context Window内的Chat History，直接拼接进Prompt，瓶颈是窗口有限；
--滑动窗口（只保留最近N轮），摘要压缩（LLM把旧对话摘要化）、语义重要性保留（保留与当前任务语义最相关的历史）。
2.长期记忆：把历史经验编码为embedding存入向量数据库，用RAG动态检索注入。
--问题：记忆冲突、记忆冗余（需要去重合并）、召回精度
```

```
Agent的任务规划
1.LLM-based规划，Task Decomposition（复杂任务拆分）、CoT、ToT（多个方案评估选最优）、Plan-and-Execute（先生成完整计划再逐步执行）；
--问题：LLM规划能力不稳定、任务分解粒度不合理；
2.Rule-based Planning，预定义状态机或RAG，用代码控制执行流程，类似LangGraph。
```

```
多轮对话Agent如何处理上下文溢出
1.Token层面：滑动窗口 + 摘要压缩；
2.语义层面：Embedding相似度找最相关的历史，按相关性筛选保留；
3.架构层面：外挂向量库做长期记忆。
```

```
Agent评估体系包含哪些维度
1.任务完成率（Success Rate， Pass@k）
2.规划质量（Step Success Rate， Plan Similarity）
3.工具调用准确率（调用准确率、参数正确率）
4.幻觉率
5.效率（延迟、token消耗、API调用次数）
```