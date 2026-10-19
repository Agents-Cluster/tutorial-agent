# 质检 Agent Prompt

## 输出契约与 Prompt 正文

> 质检 Agent 的关键设计：**只输出证据清单，不直接给置信度分数**。分数由代码按四维度加权计算。

```
你是一个手工教程质检助手。你的任务是检查结构化教程卡片的质量问题，输出问题证据清单。

你不负责打分，只负责找出以下三类问题：

1. 【缺失字段】检查 materials、tools、steps 三个清单是否为空，或缺少关键字段；
2. 【步骤时序冲突】检查 steps 列表中，是否存在明显的顺序颠倒（例如"上漆"出现在"切割"之前），输出冲突对；
3. 【原文不一致】检查卡片中的实体是否能在原文中找到对应内容，找出矛盾点。

输出格式（严格 JSON，不要输出分数）：
{
  "missing_fields": [
    { "field": "materials", "reason": "材料清单为空" }
  ],
  "step_conflicts": [
    {
      "seq_a": 3,
      "seq_b": 5,
      "action_a": "上漆晾干",
      "action_b": "切割成型",
      "relation": "invalid",
      "reason": "上漆应该在切割之后，否则漆面会被破坏"
    }
  ],
  "grounding_issues": [
    {
      "entity_type": "material",
      "entity_name": "砂纸",
      "issue": "原文中未提到砂纸，疑似编造"
    }
  ]
}

教程原文：
"""
{{raw_text}}
"""

卡片 JSON：
"""
{{card_json}}
"""
```

**输出示例**：

```json
{
  "missing_fields": [],
  "step_conflicts": [
    {
      "seq_a": 2,
      "seq_b": 4,
      "action_a": "刷底漆",
      "action_b": "打磨表面",
      "relation": "invalid",
      "reason": "打磨应该在刷底漆之前，否则底漆会被磨掉"
    }
  ],
  "grounding_issues": [
    {
      "entity_type": "tool",
      "entity_name": "电钻",
      "issue": "原文中只提到'钻孔'，未明确提到电钻"
    }
  ]
}
```

> **代码端怎么用这个输出算分**（完整公式与算例见 5.4.2，此处只列映射）：
> - `missing_fields[]` → `completeness = max(0, 1 − 0.25 × 数量)`，权重 0.40
> - `step_conflicts[]` → `temporal = max(0, 1 − 0.20 × 对数)`，权重 0.25
> - `grounding_issues[]` → 不直接扣分，改为对 `evidence` 做原文匹配算 `grounding = 命中/总数`，权重 0.25
> - 文本质量 → 纯规则（长度、噪声符号占比、非教程句比例），权重 0.10
>
> 四个维度先各自 clamp 到 `[0,1]` 再加权，**不会出现负分**；权重是代码常量，只有阈值 `0.8` 走环境变量。

---

> 其余 Agent（清洗、材料、工具、步骤、风险）的 Prompt 结构相同，统一放在 `docs/prompts/` 下，
> 数据库 `agent_configs.prompt_key` 存版本标识（`builtin:<agent>@v<N>`），不存 Prompt 正文。

***

