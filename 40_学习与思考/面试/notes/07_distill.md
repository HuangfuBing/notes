1.reward model
```
--取<eos>对应的reward，实践证明效果更好、策略更简单、流程更稳定

--reward模型训练：直接在LLM输出[B, S, V] --Linear(V, 1)--> [B, S, 1] --gather--> [B, S]

--reward损失函数和DPO一致，学习偏序关系；sigmoid好处是自带margin

---pairwise loss:
embedding -> transformer -> [2 * batch_size, seq_len, dim] --Linear(dim, 1)--.squeeze(-1)--> [2 * batch_size, seq_len,] --gather(dim=1, index=eos_tokens).squeeze(1) --> [batch_size,] + [batch_size,] --> pairwise loss
-eos_token.shape -> [B,]

--奖励模型的经验
1.复合奖励：
1.1 例如deepseek，使用多模型加权
1.2 业界都采用复合奖励，防止reward hacking
2.RLVR：
可验证的奖励，强任务依赖，二元奖励无法形容细粒度的对错
3.reward shaping：不只是总奖励，而是人为设计一些额外的奖励/惩罚，更容易让模型学到想要的行为
3.1 多维度打分
3.2 把一个大的目标分解成子目标
4.reward hacking
4.1 复合奖励；
4.2 KL散度约束；
4.3 过程监督，对CoT打分

--PRM和ORM有什么不同
PRM缺点：
1.复杂；
2.+-难以区分细粒度区别
3.奖励等权，部分过程可能更关键
ORM：
奖励稀疏
```

2.PPO
```

```