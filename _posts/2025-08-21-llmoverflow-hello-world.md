---
layout: post
title: "Hello, World! LLMOverflow 第一问：当编程的起点变成Prompt，工程师要如何确保终点的可靠？"
date: 2025-08-21 16:00:00 +0800
description: "一篇关于在LLM时代追求100%可靠JSON输出的深度探索，记录了从Prompt工程到Tool Calling的五种尝试与反思，同时也是LLMOverflow.dev的创世之问。"
author: "CharlieBrown-v1"
---

大家好。

在这个被大家习惯性称呼为“LLM元年”的时代，我像许多工程师一样，一头扎进了这个由 Prompt、模型和不确定性构成的新世界。

编程界有一句圣经：“不要重复造轮子”。它背后有一个温暖的假设：你遇到的所有Bug，都一定有前人为你铺平了道路。但在LLM应用这个崭新的、巨大的蓝海领域，我们脚下空无一物，每个人都在重复地、孤独地试错。

这篇文章，记录了我为解决一个看似基础、实则致命的问题所做的全部探索。它没有最终的完美答案，但记录本身，就是 `LLMOverflow.dev` 存在的意义。

**`LLMOverflow` 的第一问：如何确保LLM能 100% 输出我们预定义的 JSON，无论输入如何复杂？**

---

### 一次与“不确定性”的较量

以下是我走过的`弯路`，以及我个人给每条路径的评分。

#### 方向一：提示词工程 + “面向报错编程”的后处理

对于一个新手来说，这是最容易想到的一个方向，我把 Prompt 划分为三个部分：

- 任务相关的指令
- JSON Schema 的格式化描述
- Few-shot 的输出示例

最后，我用 `json_repair` 和 `re` 来清洗那些不符合Schema的输出。

这个方案迭代了无数个版本，每当遇到一个新的corner case，我的后处理代码和Prompt就得再打一个补丁。它能用，但极其脆弱，就像是在维护一台随时会散架的老爷车。

##### 提示词设计
... (任务相关的指令)

# 输出格式要求（必须严格遵守）
严格按照 JSON Schema 格式返回 JSON，禁止输出任何无关内容

## JSON Schema
```json
{
    "steps": [
        {
            "step_idx": "计算步骤的序号，从 **1** 开始（整数类型）",
            "pseudocode": "计算步骤的伪代码",
            "input_params": [
                {
                    "name": "参数的名称",
                    "value": "参数的取值",
                    "purpose": "参数的作用"
                }
            ],
            "input_fields": [
                {
                    "name": "字段的名称",
                    "description": "字段的描述",
                    "purpose": "字段的作用"
                }
            ]
        },
        "..." // 其他计算步骤
  ]
}
```

## 输出示例
```json
{
    "steps": [
      {
          "step_idx": 1,
          "pseudocode": "计算20日对数收益率: log_ret = log(close / lag(close, lookback_period))",
          "input_params": [
              {
                  "name": "lookback_period",
                  "value": "20",
                  "purpose": "回看周期"
              }
          ],
          "input_fields": [
              {
                  "name": "close",
                  "description": "收盘价",
                  "purpose": "计算对数收益率"
              }
          ]
      },
      {
          "step_idx": 2,
          "pseudocode": "计算换手率: turnover_rate = volume / free_float_shares",
          "input_params": [],
          "input_fields": [
              {
                  "name": "volume",
                  "description": "成交量",
                  "purpose": "计算换手率"
              },
              {
                  "name": "free_float_shares",
                  "description": "自由流通股本",
                  "purpose": "计算换手率"
              }
          ]
      },
      {
          "step_idx": 3,
          "pseudocode": "计算换手率加权反转因子: factor = -log_ret * turnover_rate",
          "input_params": [],
          "input_fields": [
              {
                  "name": "log_ret",
                  "description": "对数收益率",
                  "purpose": "计算换手率加权反转因子"
              },
              {
                  "name": "turnover_rate",
                  "description": "换手率",
                  "purpose": "计算换手率加权反转因子"
              }
          ]
      }
  ]
}
```

##### 后处理代码
```python
def validate_json(json_content: str) -> Dict:
    try:
        if isinstance(json_content, str):
            if json_content:
                match = re.search(r'```json(.*?)```', json_content, re.DOTALL)
                if match:
                    json_content = str(match.group(1).strip())
                    data = json.loads(json_content)
                else:
                    data = json.loads(json_content.strip())
            else:
                data = {}
        else:
            raise ValueError(f"非法格式: {type(json_content)}")

        return data
    except json.JSONDecodeError as e:
        print(f'json_content 解析失败: {json_content}')
        print(f'Json 解析错误: {e}')
        raise ValueError("无效的JSON响应")
```

* **评分：7.0 / 10**
* **理由**：可以应对大多数任务场景，但维护成本极高并且探索曲线非常曲折。每一次业务逻辑的微小变动，都可能引发Prompt和后处理代码的连锁崩溃。它完全不是一个工业级的解决方案，更像是一件“手艺品”。

#### 方向二：官方承诺 - 一瓶名为 `response_format` 的“安慰剂”

在方向一的基础上，我加上了OpenAI官方的 `response_format={ "type": "json_object" }`。我曾期望其能一劳永逸。

结果是，它几乎没有带来任何显著的增益。该出错还是得出错。

* **评分：7.1 / 10**
* **理由**：分数比方向一略高0.1分，完全是出于官方承诺所带来的“心理安慰”。至少，我们感觉自己用了“官方推荐”的方法，尽管效果微乎其微。

#### 方向三：业界神话 - `tool_calling` 的“貌合神离”

Agent 社区内，将 `tool_calling` 视为 JSON 格式化输出的最先进技术，怀揣着这一调研结果，我满怀希望地手写了一个 `tool_calling` 逻辑，期望LLM能够精准调用。

但现实给了我沉重一击。LLM完全没有调用我定义的工具，而是直接输出一个**像是**在调用工具的**字符串**：

```json
{
    "tool_code": "xxx_schema",
    "content": "xxx"
}
```

除此之外，它还会自己“发明”一些我从未在Schema中定义的Key。我严重怀疑这个手写的 `tool_calling` 逻辑有Bug，但面对这个巨大的黑盒，我完全不知道Ground Truth究竟是什么，调试无从下手，最终出于项目进度考虑，只能放弃。

* **评分：4 / 10**
* **理由**：在我的复杂场景下，它不仅没解决问题，反而引入了更诡异、更无法理解的新问题。这是一个典型的“理论可行，实践崩溃”的典型案例。

#### 方向四：优雅地误入歧途 - `任务解耦`

我尝试将任务拆分为两个阶段：
1. LLM只管完成任务，不增加任何格式限制，使用纯文本作答
2. 使用另一个LLM将阶段一的输出格式化为JSON

这是一个看似非常优雅、符合软件工程“单一职责”原则的设想。**但实践下来，效果极其糟糕！**

* 第一阶段的LLM由于卸下了格式枷锁，彻底放飞自我，输出大段难以解析和验证的文本
* 第二阶段的LLM由于缺失了任务背景，在格式化时产生了严重的歧义，完全无法保证内容的准确性

* **评分：2 / 10**
* **理由**：方向性的错误。是一种试图用战术上的勤奋（拆分任务），来掩盖战略上的疏忽（LLM的上下文依赖性），结果是两头落空，效果远不如方案一。

#### 方向五：最后的希望 - LangChain + `with_structured_output`

为了兼容不同的LLM，我用LangChain重写了底层的LLM接口，并最终摸索到了 `with_structured_output` 这个功能。这是目前唯一能与方向一“掰手腕”的方案。

但它也给了我一个反直觉的“惊喜”：**我原以为有了它，就可以让Prompt完全专注于任务本身，格式由Schema保证。但事实是，我依然需要像方向一那样，提供一个包含指令、格式、示例的“三段式”Prompt。** 否则，它也无法准确理解我的意图。

然后，它依旧不是100%可靠。模型还是存在：不调用工具、直接返回思考过程或者调用工具出错直接返回None的情况。这些错误随机、难以复现，就像幽灵一样漂浮在我的项目上空。
但是通过重试机制，已经可以将这些错误控制在可接受范围内。

* **评分：8.5 / 10**
* **理由**：这是我目前能找到的、在工程化和效果之间平衡得最好的方案。它承认了LLM的不可靠，并试图在框架层提供一种约束。但它离“100%可靠”的圣杯，依旧遥远。

---

### 这不仅仅是JSON，而是LLM时代的困惑 (It's more than JSON!)

回顾这次探索的历程，我发现我面对的其实不仅仅是一个简单的JSON格式问题，而是所有与LLM打交道的工程师共同的困惑：

* **什么样的提示词才是最好的？** 面对LLM这个黑盒（神经网络）构建的黑盒，不同任务、不同需求，其正确的打开方式是什么？
* **如何评估一个方案的优劣？** LLM是一个“随机求解器”，同样的问题它会给出不同的方案。我们能否在投入大量时间试错前，预知哪个方案更有前景？更值得尝试？（即使是目前看起来的SOTA：`方向五` 仍然无法做到100%可靠，是否存在100%可靠的方案呢？）
* **社区里流传的各种Tricks，哪些是真理，哪些只是幸存者偏差？** 我们需要一个方法来量化它们的有效性。
* **如何终结“大海捞针”式的探索？** 在无数个社区、论坛、Discord频道之间切换，大海捞针式地寻找解决方案，体验实在太糟糕了！
* **我们如何找到LLM的“母语”？** 能否量化不同数据结构对LLM的友好度？我反复迭代优化后得到的 `计算步骤` Schema，它真的是最优解吗？

**以上种种，就是 `LLMOverflow.dev` 存在的意义！**

我们来到了一个前所未见的蓝海，试错肯定在所难免。但在试错之后，我们的经验和教训不应该像水蒸气一样消散在空气中。那些我们花费无数个深夜调试出来的Prompt、我们总结出的宝贵经验，应该被沉淀下来，变成后来者的阶梯。

`LLMOverflow` 想成为的，正是开往这些“康庄大道”的枢纽。

1. **引入类似StackOverflow的评分/点赞机制**：让社区来筛选出最有价值的方案，为你省去在无意义方向上的试错时间。
2. **汇聚整个Agent开发社区的力量**：分享你的经验，在这里是一件利己利他的举手之劳。
3. **成为LLM时代的“自带评分的开发百科全书”**：通过优质的筛选和评优机制，确保你能在这里以最小的时间成本找到最可靠的答案。
4. **to be continued...**

这仅仅是个开始。

**我的“创世之问”抛给了大家：关于“确保LLM输出100%符合Schema的JSON”，你是否也有自己的独门秘籍，或是同样惨痛的教训？**

欢迎诸君带着代码与思考，来到 `LLMOverflow.dev`，让我们一起，点燃这个LLM时代的星星之火！
