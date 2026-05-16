---
title: 脚本不要装聪明：AI 原生系统的语义权边界
date: 2026-05-14
tags:
  - AI
  - LLM
  - AI-native
  - agent-runtime
  - control-plane
  - workflow-smell
---

# 脚本不要装聪明：AI 原生系统的语义权边界

## 引子：AI 原生系统为什么会长回老样子

一个看起来很反直觉的现象是：我们明明一开始就说要做 AI 原生系统，最后却经常在代码里看到一堆 `regex`、`keyword`、`if text contains ...`。

脚本在读自然语言。脚本在猜用户意图。脚本在判断当前目标。脚本在分类需求类型。脚本在决定影响面。脚本在生成验收标准。脚本在拼验证计划。脚本在看见 `ui`、`operator`、`dashboard` 这样的词以后，就悄悄把任务导向某个固定方向。

这不是某个项目的偶发问题，而是用 LLM 开发 AI 原生系统时的高频失败模式。

原因并不神秘：LLM 本身就是在大量既有工程实践上训练出来的。既有工程实践里，绝大多数系统都没有真正会读人话的智能体。所以传统软件设计自然会把人话压成枚举、规则、关键词、状态机和分支逻辑。现在 LLM 有了语义理解能力，但如果我们不主动给它注入新的架构概念，它很容易继续沿用旧世界的默认模式：

```text
自然语言太不稳定，所以先用规则拆一下。
LLM 输出太不稳定，所以先用脚本兜底。
为了测试稳定，先写几个关键词分类。
为了快点跑通，先把常见情况 hardcode。
```

每一句单独看都合理。合起来就会得到一个很荒诞的系统：

```text
LLM 明明会读人话，
但真正决定任务方向的，
却是一段不会理解上下文的脚本。
```

这不是 LLM 太蠢。相反，这是我们没有重新划清边界：在 AI 原生系统里，什么应该由脚本固定，什么应该由 LLM 判断。

如果这条边界不清楚，系统就会自然退化成传统自动化：

```text
脚本负责理解需求；
LLM 负责执行脚本误解出来的 prompt。
```

这不是 AI-native。只是传统控制流外面套了一个 LLM worker。

## 一、真正的问题不是“脚本太多”

讨论这个问题时，很容易走向另一个极端：既然是 AI 原生，那是不是应该少写脚本，多让 LLM 决定？

不是。

AI 原生系统不是不要确定性。恰恰相反，它需要更多确定性，只是确定性应该放在正确的位置。

脚本应该非常强，甚至应该比传统系统更强。它应该负责：

```text
身份
权限
状态
合同
持久化
幂等
安全
证据
执行
回放
```

这些东西不能交给 LLM 临场发挥。

例如：

```text
case_id 是什么？
requirement_id 是什么？
状态文件写在哪里？
路径是否越界？
JSON schema 是否通过？
某个 phase 是否允许从 running 变成 completed？
测试命令是否真的执行？
artifact 是否真的落盘？
证据是否可跳转？
危险命令是否被拦住？
```

这些都应该由脚本固定。脚本越硬越好。

真正的问题不是脚本太多，而是脚本越权。

脚本不应该负责这些：

```text
用户真正想要什么？
哪一段是当前目标？
哪一段只是背景？
哪一段是示例？
哪一段是后续建议？
这个需求属于什么类型？
影响面有哪些？
哪些文件大概率相关？
怎样才算完成？
该跑哪些验证？
这次 review feedback 的本质是什么？
证据是否足以证明验收？
```

这些是开放语义判断。开放语义判断的情况不可枚举，关键词规则迟早会漏，甚至会在长文本里产生方向性误导。

所以第一条原则是：

```text
AI 原生系统不是减少脚本，而是禁止脚本装聪明。
```

脚本要强在确定性边界上，不要强在语义理解上。

## 二、脚本最容易越权的地方

最危险的代码通常长得并不吓人。它往往只是一些 helper：

```python
lowered = text.lower()
if "ui" in lowered:
    requirement_type = "ui_change"
if "deploy" in lowered:
    risk = "high"
if "follow-up" in lowered:
    section = "future"
if "test" in lowered:
    verification_plan.append("run tests")
```

这种代码一开始很诱人。它便宜、稳定、容易测，而且能立刻跑通 demo。

但它有一个根本缺陷：它只看词，不懂话。

用户可能在说：

```text
不要把这个做成 UI-only。
下面是一个 Follow-Up Dogfood 示例，不是本轮目标。
这个不是 deployment 需求，只是说明不能碰部署。
测试部分只是旧方案的反例。
```

关键词逻辑看到的是：

```text
UI
Follow-Up
deployment
test
```

它看不到否定、引用、层级、反例、示例、历史、未来计划和当前目标之间的关系。

于是系统会出现一种特别隐蔽的失败：

```text
脚本先误读任务；
LLM 后续非常努力地执行；
最后整个执行链条都朝错误方向收敛。
```

这种失败比普通 bug 更危险。因为后续每一步都可能看起来很认真：

```text
任务被拆解了。
phase prompt 写得很完整。
agent 执行了。
测试也跑了。
review 也做了。
summary 也很顺。
```

但最开始的 current goal 已经错了。

这就是 AI 原生系统里最典型的控制面失败：**智能体没有输给任务难度，而是输给了脚本的早期误读。**

## 三、LLM 不稳定，不等于脚本应该读需求

很多脚本语义逻辑的来源，是对 LLM 不稳定的恐惧。

这种恐惧是对的。LLM 确实会：

```text
幻觉
漏字段
输出坏 JSON
过度解释
被上下文带偏
把假设说成事实
把故事闭合当成交付闭合
```

所以 LLM 不能拥有无约束权力。

但从“LLM 不稳定”不能推出“脚本应该读需求”。正确结论应该是：

```text
LLM 负责语义判断；
脚本负责约束、校验、拒绝和记录。
```

LLM 可以说：

```json
{
  "current_objective": "本轮只修控制面 intake 的语义拆解",
  "non_current_sections": ["Follow-Up Dogfood 示例"],
  "executable_tasks": ["新增 management intake contract", "注入 phase prompt"],
  "acceptance_criteria": ["示例中的 Requirement 不得成为 current goal"],
  "scope_guard": ["不扩 UI v1", "不改外部 methodology repo"]
}
```

脚本应该检查：

```text
JSON 是否合法？
字段是否齐全？
枚举是否有效？
路径是否在 repo 内？
requirement_id 是否由系统 canonicalize？
source_refs 是否可信？
风险是否需要人工 gate？
```

如果 LLM 输出坏了，系统应该进入：

```text
blocked
needs_repair
needs_human
retryable_management_intake
```

而不是：

```text
LLM 失败了，那就让 regex 接管吧。
```

这是非常关键的一条。因为 LLM 失败的时候，往往正是输入复杂、语义模糊、最不该让脚本硬猜的时候。

所以第二条原则是：

```text
LLM 失败不能 fallback 到聪明脚本。
```

失败时应该保守停住，而不是用更低级的理解能力继续推进。

## 四、语义权和执行权必须分开

AI 原生系统里需要一个新概念：**语义权**。

语义权指的是：谁有资格解释自然语言输入，并把它变成结构化任务真相。

例如：

```text
谁判断当前目标？
谁判断非目标？
谁判断验收标准？
谁判断风险理由？
谁判断影响面？
谁判断 review gap 是否关闭？
```

传统系统没有真正的语义智能，所以这些判断通常被产品文档、人类、规则引擎、表单字段或工作流系统提前消化。到了代码里，需求已经变成枚举和状态。

AI 原生系统不同。它经常直接面对 raw human text。用户不会天然给出完美 JSON。用户会给一段混合了目标、背景、情绪、示例、反例、后续计划和隐含约束的自然语言。

这时系统必须决定：谁来解释这段话？

答案应该是 LLM。但这不代表 LLM 可以拥有执行权。

执行权包括：

```text
写状态
创建 case
关闭 phase
放行命令
声明完成
修改权限边界
决定 canonical id
覆盖真实路径
```

这些应该属于脚本。

所以更精确的分工是：

```text
LLM 拥有语义建议权；
脚本拥有执行裁决权。
```

或者更短：

```text
LLM 不碰权力，脚本不装聪明。
```

这句话背后是一套架构：

```text
Raw human text
  -> LLM Management Intake
  -> strict JSON contract
  -> script validates / canonicalizes / gates
  -> phase runner consumes contract
  -> worker LLM executes within contract
  -> script captures evidence / test results / state
  -> reviewer LLM judges semantic sufficiency
  -> script records verdict and lifecycle
```

这里每一层都有清楚职责。

LLM 可以读懂复杂语义，但它的输出必须变成合同。脚本不负责理解人话，但它负责保证合同合法、边界安全、证据真实、状态可回放。

## 五、为什么 LLM 会反复写出旧范式

还有一个更深层的问题：就算我们想做 AI-native，为什么 LLM 还是经常帮我们写回传统模式？

因为 LLM 继承的是既有工程文化。

既有工程文化默认：

```text
输入要规范化。
自然语言要转成枚举。
业务类型要分类。
风险要规则化。
测试要稳定。
系统要可预测。
fallback 要尽量继续跑。
```

这些经验本身没有错。它们是在没有通用语义智能的时代发展出来的优秀实践。

但 AI 原生系统改变了一个前提：系统内部现在有一个能处理开放语义的组件。

如果我们不明确告诉 LLM：

```text
这里不是传统 rule engine。
这里有语义判断者。
不要用 keyword 冒充理解。
不要为了测试稳定把开放语义压成几个词。
确定性应该用于约束 LLM，而不是替代 LLM。
```

它就会自然生成旧世界的代码。

甚至人类工程师也会这样做。因为旧模式有短期收益：

```text
开发快
单测稳
demo 顺
依赖少
成本低
出错看起来可控
```

而新模式一开始更麻烦：

```text
要设计 contract
要 mock runtime
要处理坏 JSON
要处理 timeout
要校验 schema
要 canonicalize 身份
要记录 blocked state
要写对抗性语义用例
```

所以如果没有明确的架构语言和 review 标准，系统会自动滑回旧模式。

这就是为什么 AI-native 不是一句口号，而是一组需要反复注入的概念：

```text
语义权
执行权
合同化输出
保守失败
证据绑定
非当前内容隔离
脚本不装聪明
LLM 不碰权力
```

没有这些概念，LLM 和人都会本能地回到老工程实践。

## 六、判断一段逻辑该不该写死

可以用一个简单测试：

```text
这个判断是否依赖开放语义？
```

如果答案是是，就默认不该脚本写死。

例如：

```text
“这是不是本轮目标？”
“这是不是示例？”
“这是不是反例？”
“用户是不是在否定这个方向？”
“这个验收标准是否已经被证据证明？”
“这个 review feedback 是不是 scope drift？”
```

这些都依赖开放语义，应该由 LLM 给结构化判断。

再问第二个问题：

```text
这个判断是否涉及安全、身份、状态、权限或真实证据？
```

如果答案是是，就应该由脚本固定。

例如：

```text
“这个路径是否在允许目录内？”
“这个状态迁移是否合法？”
“这个 JSON 是否符合 schema？”
“这个测试命令是否真的返回 0？”
“这个 artifact 文件是否存在？”
“这个 requirement_id 是否是系统生成的？”
```

这些都不应该交给 LLM 自由发挥。

所以可以得到一张很实用的表：

| 问题类型 | 应该谁负责 |
|---|---|
| 解释 raw requirement | LLM |
| 区分当前目标/未来建议/示例/反例 | LLM |
| 生成任务拆解/验收/验证建议 | LLM |
| 判断证据语义上是否足够 | LLM reviewer |
| 校验 JSON schema | 脚本 |
| canonicalize id/path | 脚本 |
| 状态机流转 | 脚本 |
| 执行测试命令 | 脚本 |
| 捕获 stdout/stderr/returncode | 脚本 |
| 权限和危险操作 gate | 脚本 |
| owner-facing 解释的语义内容 | LLM 生成，脚本渲染 |
| UI 排版、链接、转义、显示状态 | 脚本 |

判断标准不是“LLM 能不能做”，而是“这件事本质上是不是开放语义”。

## 七、测试也会把系统带偏

AI 原生系统里还有一个不容易察觉的问题：测试会奖励错误的确定性。

关键词规则非常好测：

```text
输入包含 bug -> type=bugfix
输入包含 ui -> surface=operator_ui
输入包含 deploy -> risk=high
```

这些测试便宜、快速、稳定。它们会给团队一种错觉：系统很可靠。

但这种可靠性只覆盖了词面，不覆盖语义。

真正应该测的是：

```text
“不要改 UI” 不应该被识别成 UI 需求。
“下面是后续示例” 不应该成为 current goal。
“这个 deploy 只是禁止事项” 不应该触发部署任务。
“旧方案里的测试计划” 不应该成为本轮验证计划。
“反例中的 Requirement:” 不应该被当作真实 Requirement。
```

这些测试不再验证关键词，而是验证语义边界。

所以 AI-native 的测试也要换脑子：

```text
不要只测 happy path 分类；
要测脚本是否被禁止解释语义；
要测 LLM contract 的 schema 和边界；
要测复杂文本里的非当前内容隔离；
要测 LLM 失败时是否保守停住；
要测身份和路径是否由脚本覆写；
要测 review gap 是否真的进入下一轮中心上下文。
```

换句话说，测试不能只让系统“跑起来”。测试要防止系统“用旧方式跑起来”。

## 八、隐藏最深的失败：prompt 层重新解释合同

即使 intake 已经交给 LLM，系统仍然可能在后面重新滑回脚本语义。

最常见的位置是 prompt 拼装层。

例如：

```text
如果 requirement_type == ui_change，就在 phase prompt 里强调 UI-only。
如果 affected_surfaces 包含 operator_ui，就限制 agent 只看 UI。
如果 risk == low，就自动执行。
```

这些逻辑看起来只是在“优化 prompt”。但如果这些字段来自旧的 keyword classifier，或者 prompt layer 自己又加了关键词判断，它就会再次成为隐藏产品经理。

正确做法是：

```text
phase prompt 只消费 management contract；
不要重新解释 raw text；
不要根据 legacy keyword field 注入方向性 bias；
如果 contract 不够清楚，就回到 intake repair；
不要在 runner 里偷偷补语义。
```

展示层也是一样。

UI 可以渲染状态、链接、时间、按钮、证据列表。但 UI 不应该根据关键词自己总结“这个 case 为什么卡住”“下一步该做什么”“风险是什么”。这些解释应该由负责语义理解的 LLM 生成结构化 summary，然后 UI 只负责展示。

否则系统会出现多个语义中心：

```text
intake 解释一遍；
runner 解释一遍；
review 解释一遍；
UI 再解释一遍。
```

多个语义中心必然漂移。

AI 原生系统应该努力做到：

```text
语义判断集中产生；
合同化保存；
下游只消费合同；
发现不清楚就回到语义层修正。
```

## 九、AI 原生系统的新工程纪律

如果要防止这类失败反复出现，需要把它变成工程纪律，而不是靠临场提醒。

第一条纪律：

```text
任何读取 raw text 后做 goal/scope/type/surface/risk/verification 判断的代码，默认可疑。
```

这不代表绝对不能有规则。极窄安全拦截可以有，例如检测明确危险命令、路径越界、权限请求。但开放语义分类不应该由脚本承担。

第二条纪律：

```text
LLM 输出的语义判断必须合同化。
```

不要只让 LLM 写一段自然语言 summary。它应该输出结构化合同，例如：

```text
current_objective
non_goals
scope_guard
non_current_sections
executable_tasks
acceptance_criteria
target_files
risk_assessment
verification_plan
open_questions
confidence
```

第三条纪律：

```text
脚本必须 canonicalize 身份和边界。
```

LLM 不应该决定真实 `case_id`、`requirement_id`、`brief_path`、状态目录、证据路径。它可以引用，但脚本必须校验、覆写或拒绝。

第四条纪律：

```text
LLM 失败必须保守。
```

坏 JSON、缺字段、timeout、schema invalid，都应该进入可观察的 blocked state。不要让 regex fallback 继续执行复杂任务。

第五条纪律：

```text
review 要问“谁拥有语义权？”
```

每次 code review 遇到这样的逻辑，都应该停一下：

```text
这段代码是在执行合同，还是在解释需求？
如果是在解释需求，为什么不是 LLM contract？
如果必须写死，第一性原理是什么？
失败时是停住，还是继续硬猜？
```

这类问题应该成为 AI-native 项目的 review checklist。

## 十、一个更好的默认架构

更 AI-native 的默认架构可以很简单：

```text
1. 用户给 raw text。
2. LLM intake agent 读取 raw text，输出严格 JSON 管理合同。
3. 脚本校验合同，覆写身份，检查路径和权限。
4. 如果合同坏了，进入 blocked/repair，不继续执行。
5. phase runner 只消费管理合同，不重新读 raw text 猜方向。
6. worker LLM 在合同内执行。
7. 脚本执行测试、保存证据、记录状态。
8. reviewer LLM 判断证据是否语义上关闭验收。
9. 脚本记录 verdict、关闭或打回生命周期。
```

这个架构里，LLM 和脚本都很重要，但它们的错误边界不同。

LLM 的错误边界是：

```text
理解错
漏掉约束
输出格式坏
过度自信
```

脚本负责把这些错误变成可见失败，而不是让它们偷偷获得执行权。

脚本的错误边界是：

```text
规则过死
语义越权
错误 fallback
状态漂移
证据未绑定
```

LLM 负责提供语义判断，但不能替脚本保证真实性。

这就是双方的正确关系：

```text
LLM 提供判断；
脚本提供制度。
```

## 十一、迁移旧系统时先找这些味道

如果一个系统已经写了一堆传统逻辑，可以先不急着重构。先找味道。

最明显的味道是：

```text
text.lower()
regex section parser
keyword classifier
if "xxx" in text
requirement_type == ...
affected_surfaces contains ...
prompt branch based on guessed type
fallback classifier
UI summary based on keyword
test asserts keyword -> semantic category
```

看到这些不要立刻删。先问：

```text
它是在做确定性校验，还是在做开放语义判断？
```

如果它是在做确定性校验，可以留下，甚至加强。

如果它是在做开放语义判断，就迁到 LLM contract：

```text
旧逻辑：keyword -> requirement_type
新逻辑：LLM intake -> requirement_type + rationale + confidence
脚本：校验枚举、记录来源、低 confidence 时 block 或请求 repair
```

迁移时还有一个重要动作：不要只替换实现，要补失败测试。

例如：

```text
Follow-Up 示例不得成为 current goal。
否定式 deployment constraint 不得触发 deployment 任务。
反例里的 Requirement 不得覆盖真实 Requirement。
LLM 返回错误 requirement_id 时脚本必须覆写或拒绝。
LLM timeout 不得落回 deterministic semantic parsing。
```

这些测试不是在测 LLM 聪不聪明，而是在测控制面有没有守住职责边界。

## 十二、最短口号

这整套东西可以压成一句话：

```text
脚本管确定性，LLM 管开放语义；LLM 不碰权力，脚本不装聪明。
```

这句话看起来简单，但它背后其实是在替 AI 原生系统建立新的工程直觉。

旧直觉是：

```text
自然语言不可靠，所以要把它尽快压成规则。
```

新直觉是：

```text
自然语言需要智能理解，所以要让 LLM 先把它变成合同；
合同需要确定性约束，所以要让脚本严格校验和执行。
```

旧直觉是：

```text
fallback 要尽量继续跑。
```

新直觉是：

```text
语义层失败时，继续跑比停住更危险。
```

旧直觉是：

```text
测试稳定说明系统可靠。
```

新直觉是：

```text
测试也可能把错误的语义模型固化下来。
```

旧直觉是：

```text
脚本能处理常见情况就够了。
```

新直觉是：

```text
开放语义的边界情况不是例外，而是常态。
```

AI 原生系统真正难的地方，不是“怎么调用 LLM”。调用 LLM 很容易。难的是重新设计组织智能和确定性的边界。

如果边界错了，LLM 会被降级成执行器，脚本会被抬成伪管理层。系统表面上越来越自动化，实际上越来越容易在最早的语义入口处跑偏。

如果边界对了，LLM 可以做它擅长的事：理解、判断、拆解、解释、修正。脚本可以做它擅长的事：固定、校验、执行、记录、拒绝。

这才是 AI-native 该有的工程形态。

