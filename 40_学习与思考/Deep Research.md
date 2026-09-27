构建自进化和对抗降噪的深度研究智能体

1.
```
AI researcher
1.自主性
2.long-CoT Resoning
3.专业级报告生成
```
2.
```
整体架构-多Agent

1.计划:
user-query -> outline(研究简报)(提纲)
2.起草:
产出报告草稿
3.研究:ReAct循环
	3.1 主Agent:supervisor,找出初稿不足,进行针对性信息搜集/事实验证
	3.2 子Agent:Research Agent
4.修改:基于TTD-DR思想的高效去噪
	红蓝对抗 + 质量评估的双Agent实现多轮迭代稳定去噪
	supervisor收到两方面信息:红方找出的问题;质量评估Agent的多维打分结果/原因
```
5.
```
langgraph>=1.0.5
langchain>=1.2.0
```
6.
```
构造自己的项目时，一般需要自己重写logger
draft agent负责生成研究简报 + 生成报告草稿

deep_researcher_builder = StateGraph(
    AgentState, #整个内部流程使用的状态结构
    input_schema=AgentInputState #外部调用这个流程时的输入结构
)
```

得到初稿
11.
```
supervisor agent -> global agent(能看到所有信息)
输入:报告提纲和草稿, 研究发现, 反馈(红队批评&打分评估) -> supervisor(大脑) -创建研究计划->下一步(Todo list) -- Action(Output)(Planning(Tool Calls))

状态很重要，会专门定义supervisor的state进行状态管理

三个tool(蓝色)
ConductResearch Tool: 需要新信息时委派任务
refine_draft_report: 用已有的研究发现修改draft，由评估器打分、红队提供批评，为下一轮文山提供依据

工具都是做任务分配的，实际做事情的是下面的子Agent


```
![[7b8acc45428a3750e89fb3cd0fb36c99 1.jpg]]
![[e22bf827aa2c9ea4d6aa54d81cbbc8df.jpg]]

12.
```
supervisor agent功能介绍和第三方库引入

文件功能：
用于协调多个Sub-Research-Agent的监督。该模块实现了一种监督者模式，其中：
1. Supervisor Agent协调研究活动并分配任务
2. 多个Sub-Research-Agent独立地处理特定的子主题
3. 结果汇总并压缩，用于最终报告
Supervisor Agent采用**并行执行方式来提高效率，同时为每个研究主题保持独立的上下文窗口**。

refine_draft_report：
输入研究简报、研究草稿、findings，制定规范输出新的正文

最大迭代次数 15
最大并行sub-agent 3
min_need_repair_score = 6.0 评估低于这个分数发修复提醒；向supervisor发一个草稿质量低的信号

supervisor能拿到所有全局信息
```

```
每个节点能返回对状态的更新
node：接收state的Python函数
node输出：只返回改了哪些字段
更新会合并消息字段
tools取最新的消息字段
```

20.
```
Web搜索工具 Tavily
1.URL过多导致上下文腐烂
2.网页长文档
URL去重 + 抽取摘要 + 格式化 -> 返回干净的Notes

压缩Findings -- supervisor需要清晰 完整的摘要
需要compress-research节点
1.think filter
2.filters + strctures -> 简洁精炼的摘要;
3.原始notes返给supervisor做存档,并不会用于后续研究
```

```
代码:
# Add nodes to the graph

agent_builder.add_node("llm_call", llm_call)

agent_builder.add_node("tool_node", tool_node)

agent_builder.add_node("compress_research", compress_research)

  

# Add edges to connect nodes

agent_builder.add_edge(START, "llm_call")

agent_builder.add_conditional_edges(

    "llm_call",

    should_continue,

    {

        "tool_node": "tool_node", # Continue research loop

        "compress_research": "compress_research", # 返回 final answer

    },

)

agent_builder.add_edge("tool_node", "llm_call") # 继续搜索获得更多结果

agent_builder.add_edge("compress_research", END)
```
![[93bd48e28431a613c3ad6e3be3e67e78.jpg]]![[b34c9d7c42b13d98c13d4c46a15527ac.jpg]]
![[527f4097f9d835f70b79ec0d328e0220.jpg]]

```
研究报告降噪(自进化)
提供反馈代替人工提供refine意见,稳定迭代, 自我修正
redteam 提供批评意见
当前报告草稿 -> reviewed by Red Team -> 找出逻辑漏洞/关键缺失 -> 批评意见 -> 高优先级传递给supervisor
红队输入: 提纲 + 当前报告草稿
原则:
1.逻辑
2.偏题
3.结构/语言可读性

*1* 红队的批评意见动态提示词注入到system prompt中；防止淹没在长上下文中
*2* 写到红队的system prompt中,抓大放小
*3* 设置红队最大意见数；写在supervisor state里

红队作为降噪器

如果红队输出很短，直接判定通过

单次红队意见不超过3条

红队使用更强的模型
```

29.
```
红蓝对抗:定性,给意见
报告质量评估:定量,打多维分数
1.全面性 7
2.准确性 6
3.一致性 8
if分不够,返给supervisor发质量警告

不是独立的LangGraph,而是supervisor内部调用的评估函数
位于Writer改写草稿之后,进入Red Team之前
```

33.
```
深度研究workflow

deep_researcher_builder = StataGraph(AgentSate, input_schema=AgentInputState)

deep_researcher_builder.add_node("write_research_brief", write_research_brief)
deep_researcher_builder.add_node("write_draft_report", write_draft_report)
deep_researcher_builder.add_node("supervisor_agnet", supervisor_agnet)
deep_researcher_builder.add_node("final_report_generation", final_report_generation)

deep_researcher_builder.add_edge(START, "write_research_brief")
deep_researcher_builder.add_edge("write_research_brief", "write_draft_report")
deep_researcher_builder.add_edge("write_draft_report", "supervisor_agnet")
deep_researcher_builder.add_edge("supervisor_agent", "final_report_generation")
deep_researcher_builder.add_edge("final_report_generation", END)

agent = deep_researcher_builder.compile()


```

34.
```
RedTeam使用的模型性能最高

Evaluator会提供评分原因
```