---
tags:
  - LangGraph
  - DeepResearch
  - Agent
---

# DeepResearch Supervisor：工作流与状态流转

关联笔记：[[Deep Research]]

本文按项目实际代码整理。第一张图展示完整流程与数据流；第二张图展示一个假设问题的运行过程，研究内容、分数和模型决策仅作示例。

## 变量约定

| 缩写 | 变量 | 含义 |
|---|---|---|
| H | `state["supervisor_messages"]` | 当前流程内维护的 Supervisor 消息历史 |
| D | `state["draft_report"]` | 当前草稿 |
| C | `state["active_critiques"]` | 累积的红队批评 |
| M | Supervisor 中的局部变量 `messages` | 本轮发送给模型的输入列表 |

State 是节点读写的数据容器；M 是从 State 中挑选信息后组装的模型输入。模型不会自动读取整个 State。

## 完整工作流与数据流

大框是真正的 LangGraph 节点，框内是节点内部的处理步骤。节点返回 update 后，由 LangGraph 合并进 State，再执行下一节点。

```mermaid
flowchart TD
    INIT["Draft 阶段完成<br/>输入：research_brief、初稿 D<br/>H 中已有初稿和研究简报"]

    subgraph SUP["节点：supervisor"]
        S1["读取 H 和 research_iterations"]
        S2["构建本轮输入 M<br/>系统提示词 + 历史 H"]
        S3["每轮调用模型前遍历 C<br/>筛选 addressed 为 false 的批评<br/>追加到本轮 M"]
        S4["若 needs_quality_repair 为 true<br/>向 M 追加低分修复提醒"]
        S5["调用 Supervisor 模型<br/>返回 AIMessage，可能包含 tool_calls"]
        S6["返回 Command 更新<br/>H 追加 AIMessage<br/>research_iterations 加 1<br/>needs_quality_repair 置 false"]
        S1 --> S2 --> S3 --> S4 --> S5 --> S6
    end

    INIT --> S1

    subgraph TOOLS["节点：supervisor_tools"]
        T0["读取更新后的 State<br/>取 H 最后一条 AIMessage"]
        STOP{"轮数达到 15<br/>或没有 tool_calls<br/>或包含 ResearchComplete？"}
        FIN["从 H 提取全部 ToolMessage 内容<br/>写入 notes<br/>返回 goto END"]
        THINK["执行本轮 think_tool 请求，如有<br/>回显结果暂存 tool_messages"]
        RES["执行本轮 ConductResearch 请求，如有<br/>并发调用 Researcher 子图<br/>压缩结果暂存 tool_messages<br/>原始笔记暂存 all_raw_notes"]
        HASREF{"本轮是否请求<br/>refine_draft_report？"}
        REF["从当前 State 读取简报和 D<br/>从历史 H 的 ToolMessage 提取 findings<br/>交给 Writer 生成 new_draft"]
        EVAL["Evaluator 评估 new_draft<br/>计算三项分数的平均值"]
        PREP["准备更新<br/>D 替换为 new_draft<br/>quality_history 追加评分<br/>均分低于 6 时设置修复标志<br/>评分和理由暂存为 ToolMessage"]
        U1["返回 Command<br/>H 追加 tool_messages<br/>raw_notes 追加原始笔记<br/>goto supervisor"]
        U2["返回 Command<br/>提交新草稿、评分、修复标志<br/>H 追加 tool_messages<br/>raw_notes 追加原始笔记<br/>goto red_team"]
        T0 --> STOP
        STOP -->|"是"| FIN
        STOP -->|"否"| THINK
        THINK --> RES --> HASREF
        HASREF -->|"否"| U1
        HASREF -->|"是"| REF --> EVAL --> PREP --> U2
    end

    S6 -->|"LangGraph 合并 update，传入新 State"| T0
    U1 -->|"LangGraph 合并 update，开始下一轮"| S1

    subgraph RED["节点：red_team"]
        R0["读取更新后的 D、research_brief<br/>以及 critique_nums"]
        R1{"批评数达到 3<br/>或草稿为空<br/>或草稿长度不足 50？"}
        R2["调用红队模型检查新草稿"]
        R3{"回复包含 PASS<br/>或回复长度不足 20？"}
        R4["返回空更新"]
        R5["返回状态更新<br/>C 追加 Critique，addressed=false<br/>critique_nums 加 1<br/>H 追加批评 SystemMessage"]
        R0 --> R1
        R1 -->|"是"| R4
        R1 -->|"否"| R2 --> R3
        R3 -->|"是"| R4
        R3 -->|"否"| R5
    end

    U2 -->|"LangGraph 合并 update，红队读到新草稿"| R0
    R4 -->|"固定边：回到 Supervisor"| S1
    R5 -->|"合并更新后，回到 Supervisor"| S1
    FIN --> OUT["Supervisor 子图结束<br/>主图使用 notes、D、research_brief<br/>生成最终报告"]
```

图中省略异常分支：工具执行 try 块中发生异常时，当前代码也会返回 END。

## 运行示例：研究大模型客服的成本与隐私

研究请求和改写请求分别放在不同轮次。下面的“写入 State”箭头表示节点返回更新后，由 LangGraph 完成合并，不是节点直接修改共享对象。

```mermaid
sequenceDiagram
    autonumber
    participant ST as 共享 State
    participant S as supervisor 节点
    participant L as Supervisor 模型
    participant T as supervisor_tools 节点
    participant R as Researcher 子图
    participant W as Writer 模型
    participant E as Evaluator
    participant RT as red_team 节点

    Note over ST: 初始：D=D0 初稿，H 含初稿和简报<br/>C 为空，修复标志未设置
    ST->>S: 第 1 轮：读取历史 H
    Note over S: 遍历 C：没有批评<br/>组装本轮模型输入 M
    S->>L: 发送 M
    L-->>S: 请求 ConductResearch：调查成本和隐私
    S->>ST: 追加 AIMessage；轮数=1；修复标志=false
    ST->>T: 读取最后一条 AIMessage
    T->>R: 传入研究主题
    R-->>T: 压缩研究结果：成本资料较全，隐私证据不足
    T->>ST: H 追加研究结果 ToolMessage<br/>raw_notes 追加原始笔记

    ST->>S: 第 2 轮：H 已包含研究结果
    S->>L: 发送系统指令和历史 H
    L-->>S: 请求 refine_draft_report
    S->>ST: 追加 AIMessage；轮数=2；修复标志=false
    ST->>T: 读取 refine 请求
    T->>W: 简报 + 从 H 提取的 findings + D0
    W-->>T: 改写后的 D1
    T->>E: 简报 + D1
    E-->>T: 示例均分 5；理由：隐私分析不足
    T->>ST: D=D1；追加评分历史；修复标志=true<br/>H 追加评分反馈 ToolMessage
    ST->>RT: 读取最新草稿 D1
    RT->>ST: C 追加“缺少私有部署的数据流证据”<br/>addressed=false；critique_nums=1<br/>H 同时追加批评 SystemMessage

    ST->>S: 第 3 轮：读取 H、C、修复标志
    Note over S: 遍历 C，找到未解决批评<br/>向本轮 M 追加批评指导和低分提醒
    S->>L: 历史 H + 临时批评指导 + 临时低分提醒
    L-->>S: 请求 ConductResearch：补查私有部署隐私证据
    S->>ST: 追加 AIMessage；轮数=3<br/>修复标志重置为 false
    ST->>T: 执行补查请求
    T->>R: 研究私有部署的数据流和隐私边界
    R-->>T: 返回补充证据
    T->>ST: H 追加新研究结果；追加原始笔记

    ST->>S: 第 4 轮：读取补充证据
    Note over S: 旧批评 addressed 仍为 false<br/>因此再次注入本轮 M
    S->>L: 历史 H + 未解决批评指导
    L-->>S: 请求 refine_draft_report
    S->>ST: 追加 AIMessage；轮数=4；修复标志=false
    ST->>T: 执行第二次 refine
    T->>W: 简报 + 已累积 findings + D1
    W-->>T: 新草稿 D2
    T->>E: 简报 + D2
    E-->>T: 示例均分 8；理由：隐私分析已补充
    T->>ST: D=D2；追加评分历史<br/>H 追加评分反馈；修复标志保持 false
    ST->>RT: 检查 D2
    RT-->>ST: 示例回复 PASS，返回空更新
    Note over ST: PASS 不会删除旧批评<br/>也不会把旧批评 addressed 改成 true

    ST->>S: 第 5 轮：读取当前状态
    Note over S: 旧批评仍会再次注入
    S->>L: 发送本轮 M
    L-->>S: 示例选择 ResearchComplete
    S->>ST: 追加 AIMessage；轮数=5
    ST->>T: 检测到 ResearchComplete
    T->>ST: 从历史 ToolMessage 提取 notes
    Note over T: END：结束子图，主图随后生成最终报告
```

## 关键变量的读写路径

| 变量 | 谁写入 | 谁读取、什么时候读取 | 更新方式 |
|---|---|---|---|
| `supervisor_messages` | Supervisor 写决策；工具节点写结果；红队写批评 | Supervisor 每轮组装输入；工具节点取最后一条决策 | add_messages 合并，新消息通常追加 |
| `draft_report` | Draft 写初稿；工具节点写新草稿 | Writer、红队、最终报告节点；Evaluator 直接接收局部 new_draft | 覆盖旧稿 |
| `active_critiques` | 红队发现问题时 | Supervisor 每轮调用模型前遍历 | 追加 |
| `needs_quality_repair` | 工具节点评分低于 6 时置 true | Supervisor 下一轮调用前检查 | Supervisor 提醒后提交 false |
| `quality_history` | 工具节点评分后 | 当前 Supervisor 不直接向模型发送整份评分历史 | 追加；一轮多个 refine 时局部 updates 只保留最后一条 |
| `research_iterations` | Supervisor 决策后加一 | 工具节点检查是否达到 15 | 覆盖计数 |
| `critique_nums` | 红队记录有效批评时加一 | 红队检查是否达到 3 | 覆盖计数 |
| `findings` | 工具节点执行 refine 时生成 | Writer | 局部变量，不是这里的 State 字段 |
| `tool_messages` | 工具节点内部暂存结果 | 节点返回时合并到消息历史 | 返回前尚未写入 State |

## 关键细节：当前实现与设计意图的区别

### 1. 每轮什么时候遍历批评？

每次进入 supervisor，在调用模型之前执行：

```python
critiques = state.get("active_critiques", [])
unaddressed = [c for c in critiques if not c.addressed]
```

如果筛选结果非空，就把批评指导追加到本轮局部输入 M。这次追加本身不会更新历史 H。

### 2. 红队批评有两条路径进入模型

红队返回时，一方面往 C 追加 Critique，另一方面往 H 追加批评 SystemMessage。因此同一意见既在历史中出现，也会因 addressed=false 被下一轮再次注入。

### 3. addressed 没有闭环更新

当前代码创建批评时设置 addressed=false，但没有把它改成 true 的实现。即使新草稿已解决问题，或者红队回复 PASS，旧批评仍会继续被注入。PASS 只是让当前红队节点返回空更新，不会清空旧字段。

### 4. Refine 的正文和反馈走不同路径

```text
new_draft → state["draft_report"] → 红队、下一次 Writer、最终报告
评分和理由 → ToolMessage → supervisor_messages → 下一轮 Supervisor 模型
```

Supervisor 模型不会自动收到整份新草稿。红队意见也没有作为专门参数直接传给 refine；它先影响 Supervisor 决策，再可能通过补充研究等路径影响改写。

### 5. 同一轮的研究结果，不能立即被该轮 refine 从 State 读到

本轮研究结果先放在局部 tool_messages 中，节点返回后才合并进 H。但 refine 的 findings 从进入节点时的 H 提取。因此同一回复同时请求 ConductResearch 和 refine 时，后者读不到本轮刚完成但尚未提交的研究结果。

类似地，同一轮多个 refine 都从原始 state 读取 draft_report，不会自动以此前局部 new_draft 为输入；最终状态更新主要保留最后一次改写。

### 6. findings 并不只有研究资料

get_notes_from_tool_calls 实际提取全部 ToolMessage，没有按工具名筛选。因此 findings 和最终 notes 可能同时包含研究结果、think_tool 回显、评分反馈。

### 7. 低分阈值与提醒文字不一致

实际判断是平均分低于 6；Supervisor 的提醒文字却写“低于 7/10”。图按实际判断阈值 6 绘制。

## 对照源代码

- 状态定义：`deep_research/states/supervisor.py`
- 调度、工具执行与图构建：`deep_research/agents/supervisor.py`
- 红队节点：`deep_research/agents/red_team_agent.py`
- 批评字段：`deep_research/states/critique.py`
- 改写工具：`deep_research/tools/tool.py`

项目目录：`D:\workspace\video\01丁师兄大模型项目_基于自进化+对抗降噪的DeepResearch Agent项目实战\代码\project-Deep_Research`