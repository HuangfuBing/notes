---
type: learning
created: "2026-09-01"
updated:
area: learning
status: active
tags: []
---
# notes

## 记录

1.
出行规划场景
复杂规划-多工具/多种类型工具（8种）-多约束（用户请求包含2~8个约束）-长程调用，数十次工具（long-horizon）
经典的数学数据集 DAPO-math，Countdown（数字组合游戏）；数学有标准答案，可以使用RLVR来设计奖励函数
数据集构建/清洗数据，数学数据比较容易，但是这个过程在实际场景下是最花时间的
--**算法的大部分工作是造数据和洗数据**

1.为什么要选择这个场景
1.1 大部分工作是造数据 + 洗数据；
1.2 数学场景距离实际业务远
--面向地图厂商；几乎所有电动车的智慧座舱的业务；移动端路线规划APP（路枢）/旅行规划；

2.场景的难点
2.1 拆解约束
-地点：从哪出发，重点是哪；
-住宿：必须有一天晚上住在黄山；
-饮食：菜馆评分、菜系
-预算：6000
-6处景点：途径点6个，不兜圈，行程3天两夜；
-筛选出3个适合拍照打卡的
-出行方式：高铁

3.技术方案：
![[Pasted image 20260901163428.png]]
1.种子数据（Queries）：
公司里可以从用户数据里捞，前提是公司中已经有一套相关的产品，只是产品可能偏老；from 数据工程师。
if 产品刚开始启动，则获取不到真实数据，属于cold-start。
两条途径：
1.1 购买标注好的训练数据；
1.2 人工去写；（1.数据量300-400即可；2.不要一个人写，多个人写）
注意：只需要有query

2.拆分好训练/测试：
注意：对query（种子数据）做拆分，一会儿要做泛化；不拆分会出现泄露，有交点，出现指标虚高的情况，无法评估系统真实的能力。

3.expansion
拆分后，针对训练/测试，或者只训练，做泛化。如果测试够就可以只泛化训练。
query -> dsv4/claude/gpt -prompt-> 形式相同内容不同的queries(约束都考虑到,但是内容不同)
注意:对于无限扩增数据,可以做但是没必要,数据的数量不重要,重要的是数据的质量.
这部分得到的还是query,没有answer;

4.利用比较厉害的商业的大模型(例如claude/gpt)诱导answer
考虑点:成本/效率
MiniMax-2.7 行程规划上效果不错,coding和别家有差距;实测这个任务上和glm/kimi差不多
**这里有很多考虑,不一定是MiniMax**

用Qwen3-4B去蒸馏,结果不会超过被蒸馏的模型;if better,可以组建大模型公司了...

4.1 Environment
Google search(多方案), Route Planning, POI Search, Flights

**这一步得到answers**,做一个react loop调用商业大模型

5.数据清洗
garbage-in, garbage-out
花尽可能多的时间做数据清洗
使用三级数据清洗策略:
-规则过滤
-格式清洗:tool calls, answer是否闭合, 有无孤儿工具
-大模型打分: LLM as Judge, 对蒸馏后的数据, 从path到answer进行综合打分, 从质量, 可执行性等进行打分, 过滤掉低分case

6.数据切分
冷启动数据(SFT) - RL数据(做on-policy强化学习) - 验证数据(和seed测试集保持一致)

7.训练
SFT -> GRPO
冷启动学到输出格式/基本输出范式
on-policy RL让模型在环境中探索,根据reward调节轨迹
使用ms-swift分布式训练框架,支持异步训练,rollout和训练解耦
**重要: reward设计**

8.预测评估
LLM-as-Judge 标准做法
针对验证数据,用多个大模型交叉打分,取多个得分的平均值;
避开前面用过的大模型打分,避免引入bias;
最后打分的结果和没有训练过的大模型做对比

4.数据准备
![[Pasted image 20260901170553.png]]
1.测试集数据量足够
-规模太大测试时间太长；
-测试集难度本身足够

2.使用多进程并发调用；批量调用
设计进程池；根据QPS限制情况对并发度做增减

3.expansion prompt
```
QUERY_EXPANSION_PROMPT = """

你是一个经验丰富的文字改写专家，对中国的旅游城市，交通，景点路线等非常熟悉。你的任务是根据输入的用户关于旅游的问题，扩充生成5个新的问题。

  

以下是一些生成新问题的要求：

1.给的用户问题只是参考，要输出完全新的问题，不能跟原有的句子意思相同或相近。

2.新的问题里面出现的地名，景点，必须是真实的，问题是符合逻辑的，请结合你的只旅游知识来进行改写，可以适当替换里面出现的地点/城市。

3.(可选)如果有必要，可以在提问里面增加一些约束，例如价格，距离，评分，交通方式，预算，途经点等。

  

请直接输出改写后的句子，每一个句子是一行，不要输出理由或者其他无关内容。

  

用户问题：{}

"""
```
关键：
1.用户问题作为参考，输出完全新的问题；
2.真实；
3.若必要，则增加约束，例如价格、评分、距离；
4.重试机制，对于QPS限制/失败调用，对失败请求做重试

代码实战：
```
1.修改env后print environ仍然显示默认值
--强制override，load_dotenv默认override=False
```


7.扩展后做数据蒸馏
MiniMax-M2.7 两个接口：high-speed/标准版；high-speed性能有损失
先把Environment搭好，和后面做on-policy RL是一个环境
八种工具：
网页搜索/网页内容抓取/POI搜索/路径规划/航班搜索/周边搜索

块1 网页搜索
每个tools继承于qwen-agent的base tool，协议统一、一致。
网页搜索-Google 2500 - 10r
网页抓取-Tavily 1000 - 10r
平替firecrawl，一个key实现网页搜索 + 抓取；speed比二者更快 8w-10r

块2
Route/POI/Around Search/Weather Predict 高德Key
5个接口，每个接口配1w，逆地址解析

块3
航班/火车查询，在线接口很贵 1w 500
故使用大模型mock的方式，让大模型随机模拟不同距离/时间的航班价格

8.方案选型，做技术选型时要重点考虑哪些因素
成本、效率、效果
国内大模型蒸馏首选claude；蒸馏的模型差不多是被蒸馏模型前一个版本的水平

9.性能优化，如何提高环境调用的效率和稳定性？
数据蒸馏：
	1.效率：Batch调用，而不是串行调用；
		由于很多大模型原生支持batch调用，即微调阶段训练过tool-calls，这样每次发一批tool calls，速度取决于最慢的那个tool call。
		 使用协程实现
	2.稳定性
		 让环境稳定比搭建环境难度更大
		 模拟环境做的足够稳定、足够scaling，决定了能在多大规模/高保真下生成稳定的agent轨迹；这个环境一旦做成，reward/数据就成了副产品；如Dsec沙箱
		 在Reasoning时代，强调SFT + RL数据的质量；
		 在Agentic时代，重心转移到谁的沙箱（仿真环境）体系更稳健，即infra
		 **本项目中保证稳定性**
		 1.服务方（tools）：重试机制；异常回复反馈给调用方，下次可以重试参数；key额度耗尽，启用备用key切换/**多key轮询**（亦可解决并发限制）；长度（对抓取到的信息做压缩/上下文管理）
		 2.调用方：上下文长度、调用工具次数、工具调用失败、答案tag没有闭合
蒸馏好后导出rollouts

10.
json_repair 修复不标准的json
if firecrawl fail， use beautifulsoup4
使用协程，量级低于线程

firecrawl
search 根据key搜索网页
scrape 根据url抓取内容

tool的description太长，system prompt对4B模型的负担太重

lines：url和title拼接成一个字符

网页内容 markdown

firecrawl 98%

13.
POI Point Of Interest
美食/街道；按照省-市-区层级结构组织成数据库
poi_search：address/region

双层并行，工具方并行["xx", "yy"]，调用方并行（batch）

14.
路径规划：从A点到B点

15.
中心点周边搜索：以某个POI点为圆心，精确搜索附近的科教文卫等；
POI搜索：直接搜关键词，能搜到城市，但是不能精确定位到POI点；
description尽量短：system prompt对长文本很敏感

properties：
location的radius范围内的keyword，可以对region进行限定

16.
天气查询：输入city，默认返回七天内天气，返回字段很多，只要核心的：日期、日/夜间天气、气温风向/风力

summary：导航-POI（按文本搜索地点，返回地址、经纬度、商业信息。）、周边搜索（以圆心+半径搜索周边地点。）、路径规划、逆地址解析（通过经纬度反解城市名称）、天气

17.18.
火车票查询&机票查询，用大模型模拟火车票/航班班次
火车票查询：输入日期、起终点
异步openai调用模型api，并发
格式：输入、输出格式约定、真实性规则、few-shot（示例交互）

19.react loop实现大模型推理路径蒸馏
实现react并不复杂，但是要考虑边界

每轮最大tokens 1w；总最大token数20w；最大调用轮数50轮，一般10轮，至多20轮
到最大token的80%，强制输出答案

协程执行时使用asyncio.run

一条轨迹包含推理 + answer
对tool call的解析失败集中在单双引号输出错配
max token是student model的token，用其tokenizer计算token

20.
在大规模调用之前一定要做单测
1 - 2 min可以得到一条trace，3 min偏长

模型发回的response，经OpenAI Compatible server直接返回json格式的tool call；得到的tool response直接塞回原来的completions，然后整体发给model
初始得到的内容是结构化的协议；导出数据集时变成role-content这样的形式

22.数据清洗
规则过滤（hard rules） - 格式清洗 - LLM-as-Judge
规则过滤：
1.不调用工具
2.调用超最大次数；打出50/80/90/95分位点，按照这个设置最大调用次数
--调用次数过多的样本：工具调用失败/工具调用冗余
--保留太长上下文的样本，会增加显存压力，被迫提高上下文长度
3.无答案/答案是全英文（语言幻觉）（例如deepseek，语言一致性是后训练的目标）
4.答案太简短

格式清洗 稳定训练协议
1.答案和工具互斥
--要么调用工具，要么输出答案，不要既调用工具又输出答案；
---例如content“我先查询下···”，tool_calls也不为空。这会让模型学到调用工具时输出自然语言
---清洗后将content置空或者改成"<think/think>"
2.删除废话轮，删除对目标无贡献的中间assistant文本；
--降低训练成本，去掉增加上下文而不带来信息量的token
--assistant 轮只保留两类：包含可解析的 `<tool_call>`，或者包含有效的 `<answer>`；其他 assistant 轮整轮删除。
3.规范答案的位置：只要在answer内的结果，去掉前后的解释/客套话，保证结果好抽取、可比较
4.删除失败工具轮，避免模型学习到工具调用失败后强行生成正确答案
5.**统一OpenAI格式**
线上训练的格式不是OpenAI解析出的结构化的格式，将json格式换成xml的格式（《tag》《tag》），即follow qwen的格式
--格式不一致会导致：
---1.复读机；2.tag不闭合

3.LLM-as-Judge
质量层面进行筛选
推理质量的好坏没办法通过代码判断
一致、可复现的质量打分；用指令遵循能力比较强的模型
gpt/ds-v4/gpt-oss-120B（judge/数据生成效果很好）
结果：保留/降权保留/剔除

评分维度：（1-5分）
1.任务相关性；
--答非所问；用户需求遗漏；误解目的地、时间、预算等关键信息；和query无关的冗余内容
2.答案完整性；
--问答是否包含足够可执行的信息
3.事实安全性；
--回答是否存在无依据的论点；对于易变信息是否鲁棒
4.工具使用合理性；
--调用是否合理；是否重复调用；是否工具调用失败后编造答案
5.格式质量；
--结构是否清晰，是否符合《answer》《/answer》的最终答案协议

任务相关性最重要，所以权重最高

评分 + verdict（pass/borderline/fail） + issues【问题1， 问题2】， + brief_comment（一句话总结

1137/1143
904 条 pass + 177 条 borderline = 1081 条 （881条做sft，200条做rl）（850/881做训练，31条做验证）
较低的3 vs 4，主要是事实安全性和格式

29.对Agent数据的特殊检查
1.工具调用是否和query相关；
2.是否先检索信息再生成结论；
3.是否存在重复搜索或重复路线规划；
4.工具返回失败后是否仍强行编造答案；
5.最终答案是否正确吸收工具返回信息；

human in the loop；

30.数据这块未来的优化方向：
1.按样本难度分层
--根据工具调用次数/任务复杂度/query长度，对难度进行分层；例如分成简单（1-3次tools）/中等（4-10）/复杂（10以上）样本；课程学习
2.对borderline样本降权
--pass样本权重1.0，borderline：0.5，failed踢掉了（权重是0）；既能保证多样性，又能减少低质量样本对模型的负面影响
3.构建高质量子集
--做sft warmup，做高质量评估测试集（用来选各种参数），做奖励模型的正样本（训练小模型打分）
4.对实时信息加入保守表达要求
--训练时在system prompt中强调，对于实时信息提醒以官方为准，减少打分偏差，不要打分那么紧；10.1采样，10.4训练，日期不同对于同一query结果不一样，
5.保留数据血缘信息
--1.样本ID，原始ID；2.中间的数据；3.每个维度judge的分维度score；4.被过滤的原因；
--均可保存，方便溯源，分析数据为什么差

**数据清洗：让Agent数据既符合模型协议，又有高的工具调用质量**
--协议一致，训练/测试/线上线下；
--格式清洗 -- 去掉异常样本； LLM-as-Judge -- 判断哪些样本值得学

31.
SFT冷启动

同参数量base不如instruct，8B/9B比4B的base版本强，后者base不能用

要让模型学到完整的行为链路，需要做到：
1.判断用户问题是否需要调用工具；
2.根据任务类型选择合适工具；
3.生成稳定、可解析的结构化tool_calls；
4.读取并整合工具返回结果；
5.在信息充分后停止工具调用；
--小模型在上下文很长是不懂何时停止；要么很快停止（2-3轮工具），要么上下文很长时不停止
6.使用answer。。。/answer输出最终答案；

loss低不代表效果就好，需要看下clod start后格式、整体的工具调用能力如何

32.冷启动参数
```
ms-swift 支持分布式训练；支持异步采样
LoRA SFT；LoRA rank=128；不使用全参是因为800多条，全参可能导致数据量不够导致灾难性遗忘；子u后高rank效果和全参效果差距不大；
4卡GPU DDP；
精度 bf16（qwen默认）
2个epoch，数据过2遍
per-device batch size 1；总体相当于4
target modules：all-linear
开启gradient checkpointing 显存更低
max length 32768；**关键**；多轮调用 + 返回很长；阶段后信息损失，模型学不到预期行为；现在样本最长20000多，平均1w5到1w8

```

33.冷启动训练经验总结
1.system prompt规范化与小模型踩坑
同样一套系统提示词，大模型可以正常工作；小模型会出现**严重的**格式漂移
在在线的RL阶段尤为明显
原因：
1.1 大模型通常具备更强的上下文辨别能力，能分清在现有提示词中哪些内容是工具定义、哪些是禁止输出、哪些是few-shot；能分清在多轮对话中所处的位置和状态。小模型容易出现局部模式匹配（不是完整理解上下文含义，而是只能记住最近出现过的模式片段）；
--例如写了复杂的与、或、非prompt，大模型能理解，小模型会认为格式限制过强。（所以后续会对小模型专门做system prompt调优，它不是越冗长越好，而是越清晰越好）
--小模型对系统提示词特别敏感，低效果可能来源于此，而不是reward设计；
2.低学习率导致学不到工具协议；
--锁不住格式，loss下降慢
--解决方案：lr逐渐提高，从1e-6抬到5e-6，最后抬到8e-6；再往上loss抬，有点overfit。
--看loss看eval的loss
3.模型不会稳定收口；
--会持续探索；
--解决方案：
1.在answer中保留answer的答案协议；在推理中加force-answer-after-turns，MAX - 1轮后告诉模型不要调用工具；
2.repeat tools设定调用阈值，如果反复调用参数、返回值相同的tool，强制插入信息终止。防止大模型复读机。
4.工具失败导致答案风险
解决方案：
1.过滤fail tool；
2.system prompt加入更保守表达；
3.在GRPO阶段的reward里面，对工具失败使用保守/不保守回答给予奖励/惩罚，进行调优。

system prompt调优前：不知道什么时候停止；不调用工具

调优过程不能只看loss，loss低不一定效果好，loss高效果一定不好。
**调完直接去环境里rollout，就能很明显地知道效果咋样**

lr从小到大慢慢调，直到接近拐点

lora rank从8开始调，到32，128，一步一步达到最后效果

35.
训练不复杂，被封装在框架里；难点在数据和流程。
world 训练时的进程集合
使用share GPT格式，conversions；

ms框架/其它框架默认，工具回答/query/甚至思考部分，都可以mask掉

注意看tokenize前后的内容是否符合预期

37.38.
adapter 3G -- 模型大小7G
预测时单卡即可
SFT的train/val使用的是COLDSTART_SYSTEM_PROMPT，背景 + HARD LIMIT + tools列表

transformers/vllm init都能返回model对象

工具重复调用-陷入复读机；使用signiture来比较和上一个调用是否重复

prompt中卡死每一轮只允许输出工具调用

baseline工具调用轮数会短一些，40min/83条；训练过的1h左右
渲染出emoji/md表格，一般需要大参数量的模型才有这种能力

通过冷启动，给强化学习提供良好的基础

39.
冷启动取得的成果和后续

1.明显的提升：
```
1.工具调用稳定性提升； -- 更稳定生成结构化<tool_call></tool_call>；根据情况选择工具
2.协议一致性提升； -- 
3.多轮任务闭环能力提升；
4.最终答案格式更稳定；
5.为GRPO提供更强初始化；
```
2.典型问题： SFT本质是offline的fit，SFT难以解决，需要online RL
```
1.少数样本最终答案缺少完整<answer>标签；
2.个别复杂问题中工具调用次数偏多；
3.部分样本仍存在重复搜索或过度查询；
4.工具失败时，模型有时没有足够保守；
5.何时停止调用工具并最终回答的能力，仍需进一步强化
```
3.后续RL奖励需要重点覆盖：
```
1.工具调用与query的相关性；
2.重复工具调用惩罚； --根据签名统计重复调用
3.工具失败后的保守回答；
4.最终<answer>标签完整性；
5.LLM-as-udge对最终答案质量的综合评分
```


40.在线RL

1.RL的目标 ：在已有的Agent能力上，优化policy输出质量
```
1.中间轮是否应该继续调用工具；
2.工具调用是否和用户需求相关；
3.是否避免重复工具调用和无效搜索；
4.什么时候停止工具调用并输出答案；
5.最终答案是否真正吸收工具结果，输出高质量的规划内容。
```
2.RL训练环境与整体链路
```
# 异步：rollout和训练解耦
ms-swift
全参GRPO 强化学习用LoRA微调效果会大打折扣

agentic rl中的rollout涉及多轮交互，故需要定义multi-turn多轮的scheduler
```
3.多轮工具调用scheduler设计
```
1.需要维护完整trajectory，不能只有答案；
2.每轮需要判断assistant输出的是工具调用还是答案；if tool call，还要把response写回message
3.需要返回reward plugin需要的完整trajectory

**训练/推理 rollout 尽量复用**，尤其是tool call解析的逻辑要一样
```

41.
single-turn-rollout/multi-turn-rollout
核心处理step方法/check finished方法

42.
reward函数设计
```
（一般都）采用组合reward，单一reward容易被reward hacking
reward：
process_step
tool_schema
answer_tag
stage_aware
tool_efficiency
llm_judge -- 只看最终答案，上述奖励才评价完整轨迹；if LLM_judge考虑工具调用，打分不稳定

内容最重要，因为是直接呈现给用户的
```

43.
reward 见ppt
```
process step
奖励点：
--中间阶段希望看到：
	继续调用工具
	工具调用JSON可解析
	工具名和参数结构合理
	工具返回后能继续正确响应
--最终阶段希望看到：
	不再调用工具；
	只输出最终答案；
	答案被<answer></answer>包裹；

惩罚点：
	中间轮空转，只输出自然语言但不调用工具；
	同一轮同时出现tool_calls和answer
	输出其它除了<tool_call>的工具调用标签
	输出不可解析JSON
	提前直接回答；
	到达最大轮数仍没有最终答案

tool_schema
奖励点
	多个工具调用都能被解析 -- 容易出现格式不一致，例如多工具参数混用；或把字典写成list
	参数字段和工具定义一致
惩罚点
	使用不符合协议的格式，例如openai风格：{"type": "function", "function":...}
	在普通文本中夹带工具请求

answer_tag 检查最终答案的边界
奖励点：
	标签外没有多余文本；
	最终答案只出现一次；
惩罚点：
	同时输出工具调用和answer
	
stage_aware  奖励阶段意识 考察多轮对话中模型是否知道自己当前的状态
奖励点：
	初始阶段
		第一次成功工具返回之前，不输出最终答案
	信息收集阶段
		拿到工具返回后继续基于信息缺口补查
	收口阶段
		信息足够后及时收口
		不在收口阶段继续无意义搜索
惩罚点：
	未查证就回答
	中间轮输出答案
	答案轮输出工具
	同一轮既调用工具又输出答案
	最终轮无答案

tool_efficiency 奖励工具效率：有效但不重复的工具调用
--让模型认识到多工具调用并不是一个好的策略
奖励点
	工具参数有差异，能带来新信息
	没有大量重复query
惩罚点
	连续多次相同search query
	重复规划同一origin/destination
	同一POI反复查询
	工具返回失败后仍机械重复
	为了满足多工具调用而无效调用

llm_judge：最终答案质量评估
--只针对answer打分，不针对process；中间过程针对结构化reward
多维度打分，针对(query, answer, ref_answer)
答案是否吸收工具返回中的事实
```

44.reward代码实现
ORM是reward的基类
```
1.课程是reward，按照step去调整reward权重；按照进程逐渐提升内容（llm-as-judge）所占的比例
--三个阶段
2.LLM-as-Judge需要写明参考答案和Agent答案的日期可能不同
--一个实验：如果评测时把路径也考虑进去，即PREDICTION PATH/GOLD PATH，结果波动很大，效果也没有直接打分答案好。
--原因：要达到好的答案，路径有很多条，训练时限制探索空间并不好
```

46.课程式reward
参考ppt的三阶段权重
```
structed vs llm_judge 0.4 -- 0.6，防止一开始模型输出结构比较好，导致看reward看不出生成质量是否有改善
早期更倾向结构化reward，后期更倾向于答案质量打分

思路：
1.降低格式reward上限 -- 格式reward加起来不要超过0.3；
2.不要让只得到格式reward就能得到高分 -- 早期模型只靠格式不能拿满分
3.由于优化目标是内容，llm judge的权重一开始就要高（0.6） -- 中期基本格式都是满分
4.后期llm jugde的权重逐步提升
```

47.训练时参数配置
```
初始化模型 sft checkpoint-420 merged
全参训练
rollout backend： vllm server mode （不使用colocate，后者rollout和训练在同一张卡上分时复用，显存压力更大，更容易OOM）（使用HHTP调用vllm server，使用的是网络不是进程，从而实现异步采样）
训练GPU 6卡 由于显存足够，使用分布式训练；
--每个GPU都会保存模型，会导致大量内存占用；一次ckpt需要60G，本身 + 6卡就是60G，只保存两个ckpt
Rollout GPU 2卡 张量并行
ZeRO-3 + CPU offload
Max steps 200 控制在100到200步即可，模型收敛 + 控制成本
lr 1e-6
--瓶颈不在于llm推理，而在于工具调用
per device batchsize 1
generations 6
generation batch size 6
T 0.9
Max Length 50000
--cold start时使用32768，如果还沿用这个设置，有时会出现输入超过32768的情况，输出没有多余token，vllm会报错输出长度<0

```
曲线波动原因
```
1.query难度大
2.工具返回结果质量差异
3.LLM-judge 噪声
4.格式错误影响总分
```

48.
2.5 - 3h
```
memory utilization 0.8 vllm
最大13轮，蒸馏数据中统计出来
上下文长，train batch size per device = 1，刚开始30G，后面高。调高的话会OOM
vllm enable prefix caching 把相同的prompt缓存起来
RL的LoRA没有全参效果好
num iterations 1，指每个rollout的结果只用一次，实际上是on-policy的GRPO

```

50.
可视化绘图与分析
```
可视化时关心：
loss 在强化学习中loss上涨正常，因为loss是代理损失，如果loss > 0，说明模型在探索新策略，fit数据
学习率 cosine decay
**reward** 
reward std 正常情况：0，05附近，有时候方差会大，对于特别难的样本会有大方差
train speed 45s左右
num turns 7轮居多，最少5轮，最高8轮
```

51.Agentic-RL训练中遇到的真实问题和解决方案
1.训练中遇到的问题与解决方案
```
1.OOM和显存压力
--GRPO > SFT；例如多个模型；
--上下文长
解决：
--vLLM server mode，rollout和训练解耦；
--ZeRO3
--grad-ckpt
--vllm显存利用率到0.8
2.训练rollout进程残留
--端口被占用
解决：
--检查GPU利用率以及端口是否被占用
3.多轮工具调用导致单步耗时很长
--先做简单的smoke test，例如3步做一次reward，不要一开始就把完整step过一遍；
--completions的log打开，方便定位轨迹的问题
--一些解析不出来/反爬的网站，直接拉黑名单
4.工具API限流影响reward
--只要不是模型的问题，适当宽容
--对重复调用失败的工具做fallback；例如POI调用失败后用web search
5.模型过早输出最终答案 -- 尤其是query看起来能用常识回答时
--在system prompt中保留tool first的内容
--惩罚首次工具调用前回答
--奖励最终answer而不是中间的answer
--用schduler观察是否过早输出答案（？）
6.reward过高导致曲线不明显
--降低format reward比重
--提高content比重
```
2.RL训练Trick总结
```
1.先做rollout smoke，再做正式训练
--成本高，容易失败
--towards：trainer/rollout/reward plugin/answer
--一步一步阶段性来，例如3 steps看rollout，50 step看answer
2.reward要留出上升空间
--否则后续训练很难看出改善
3.不要奖励工具调用次数本身
--奖励unique的信息增益
4.Judge只评final answer，过程交给结构reward
5.system prompt不要和reward打架
--不要写最小调用轮次，与或非写少点；写基本格式和最大轮次即可
6.工具失败要单独建模
--真实环境工具一定会失败，可做些fallback
7.max turns要和completion length limit配合
--后者cll是每轮的最大上限，不设全体上限，否则一开始几步用完预算，后面无法生成
8.小模型RL更需要强协议边界

```

53.LLM-as-Judge
两种方式：
1.绝对分 -- 每条样本打分，所有样本平均
--没有标准答案（蒸馏出来的不是标准答案）
2.相对分 -- **采用这个**
--base - pred1；sft - pred2；RL - pred3
--用没有出现过的llm，pred2 vs pred1，看下谁赢 
参与评分：
1.base
2.SFT后
3.RL后
```
1.过程与结果分离的评估机制
--过程30%，结果70%
2.路径评测维度
--去掉<think>
3.答案评价维度
4.评分范围可量化
--0-10
--分开维度打分再加权
--输出一个json，like数据清洗打分
5.多LLM交叉打分
--避免引入bias，使用gpt 5.4和google gemini进行打分；
--国内模型做数据处理、蒸馏；国外模型打分
```

代码
```
1.同时评过程/结果
2.相对评分，绝对评分有漂移
3.匹配度/可行性/细节丰富度/清晰度

prompt中加入工具解释

考虑到prompt中A/B顺序可能有影响，所以base/pred分别放在A/B、B/A上计算一次分数，求平均分，消除bias

差0.8分，6.94 vs 6.11，对打分保守的gpt来说是很大差距

14b base模型不如4b instruct

交叉打分：gpt/gemini打分求平均 效果提升了10%，基于sft - RL

模型scaling的效果大于数据的scaling
--模型本身 + 算力带来的提升

```

55.项目总结
思路、成本、踩坑、优化思路、trick
实践经验
每一个简历的技术点/名词要很清楚