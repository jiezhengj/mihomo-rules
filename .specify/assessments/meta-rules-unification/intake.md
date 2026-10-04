- **标识符**：meta-rules-unification
- **创建日期**：2026-10-03
- **来源**：用户提供的文本；MetaCubeX `meta-rules-dat`（仓库 URL 在此阶段登记为允许主机，尚未抓取）
- **类型**：改进

# 原始想法

> 这是重写，不是迁移：旧 12 份配置原样归档，在独立新目录创建 12 份新配置。`rule-providers` 之前的所有配置内容，都可以直接参考旧 12 份配置重建，无须从头重新思考；从 `rule-providers` 开始的整个规则段，包括 provider 定义与其后的规则顺序，则依据 MetaCubeX `meta-rules-dat` 和本轮明确的分流原则重新设计。
>
> 用户给出的新规则原则：规则集依据实际流量用途选动作，必要时拒绝混合用途集合并评估重叠、首匹配；`PROXY` 只作兜底。需翻墙的小流量走 `ROUTE_AI`，需翻墙的大流量走 `ROUTE_GLOBAL`，不应翻墙的走 `DIRECT`，应屏蔽的走 `REJECT`。`ROUTE_AI` 的 fallback 首选 `AUTO_STRICT`；`ROUTE_GLOBAL` 的 fallback 首选 `AUTO_GENERAL`（大流量节点）。
>
> 规则骨架确立后，可参照旧配置处理 Tailscare、UU 远程、绿联 NAS 三类本地规则，以及 Desktop、Docker、iOS、Android 平台差异和模板简化。交付范围包括 8 份公开模板、4 份个人配置和相关文档，规则数据资源来自 MetaCubeX `meta-rules-dat`。
>
> 后续澄清：research 可以收录并分析旧配置规则作为现状证据；撰写新规则时须忘掉旧 provider 分类和细节，不得逐项对应重写。应从上述常见流量意图重新组织新规则。

# 重新表述

保留并归档原 12 份配置，在独立新目录创建 8 份公开模板、4 份个人配置及相关文档。`rule-providers` 之前的所有配置内容都可以直接参考旧 12 份配置重建；从 `rule-providers` 开始的整个规则段，则忘掉旧 provider 的分类和逐项对应关系，从目标流量行为重新设计：需要翻墙的小流量走 `ROUTE_AI`，需要翻墙的大流量走 `ROUTE_GLOBAL`，不应翻墙的走 `DIRECT`，应屏蔽的走 `REJECT`，最后以 `PROXY` 兜底。旧配置只在明确范围内用于重建 provider 前内容、补入指定的 Tailscare、UU 远程、绿联 NAS 规则，以及判断 Desktop、Docker、iOS、Android 差异和模板简化。

# 来源与背景

- **提出者**：用户
- **触发原因**：用户澄清新目录和 12 份配置都要重写，`rule-providers` 之前的所有配置内容可以直接参考旧 12 份配置重建；从 `rule-providers` 开始的整个规则段须按用户描述的四类流量意图和 MetaCubeX 资源重新设计，research 中旧 provider 汇总仅作现状证据，撰写新规则时不得逐项对应。

# 用户补充范围与原则

- 工作类型是重写：现有模板和个人配置归档保存，在独立新目录中重建 12 份新配置。`rule-providers` 之前的所有配置内容直接参考旧配置重建；`rule-providers` 及其后的规则部分才以 MetaCubeX `meta-rules-dat` 和场景、动作原则重新设计。
- 新规则围绕四类流量意图独立组织：需翻墙的小流量走 `ROUTE_AI`，需翻墙的大流量走 `ROUTE_GLOBAL`，不应翻墙走 `DIRECT`，应屏蔽走 `REJECT`，最后由 `PROXY` 兜底。
- 研究可记载旧规则并分析 MetaCubeX 资源，但新规则撰写时不得按旧 provider 逐项对应或迁移；旧规则只用于明确的配置重建和本地/平台差异参考。
- 对新设计实际考虑的资源按用途检查纯度；若集合混有需要翻墙与不应翻墙的成员，应评估不使用该集合。若实际检查的资源集合重叠，应按四类目标动作考虑首匹配影响，不为保留旧例外而强行对应。
- 评估规则集的先后顺序及首匹配影响。
- 新配置中，分类规则使用 `REJECT`、`DIRECT`、`ROUTE_AI` 或 `ROUTE_GLOBAL`；`PROXY` 只作为兜底。`ROUTE_AI` fallback 首选 `AUTO_STRICT` 节点组；`ROUTE_GLOBAL` fallback 首选 `AUTO_GENERAL` 节点组。分类规则不得直接指向 `AUTO_XXX`。
- 直接从 MetaCubeX `meta-rules-dat` 审视 `rule-providers` 及其后规则所需的资源和内容；不把旧 provider 分类机械改名为新分类，也不预设只用某一种资源形式。`rule-providers` 之前的所有配置内容可直接参考旧配置重建。
- 新规则骨架确定后，再参照旧配置补入 Tailscare、UU 远程、绿联 NAS 三类规则；随后参考旧配置确定 Desktop、Docker、iOS、Android 四个平台之间的新配置差异，并简化模板。
- 最终新建独立目录，交付 8 份公开模板、4 份个人配置及相关文档；旧文件保持原样归档。

# 初步待确认问题

- [NEEDS CLARIFICATION: 小流量与大流量如何在实际流量意图中划分；哪些业务应分别纳入这两类？]
- [NEEDS CLARIFICATION: 不同目标动作对应的集合发生重叠时，哪些场景优先级必须固定，特别是 AI 流量、其他需翻墙的大流量与屏蔽流量？]
- [NEEDS CLARIFICATION: Epic 下载作为历史案例是否继续要求直连，还是按其实际翻墙需求与流量大小归入四类场景？]
- [NEEDS CLARIFICATION: 新规则骨架确立后，Tailscare、UU 远程、绿联 NAS 旧规则在四个平台分别如何适配？]
- [NEEDS CLARIFICATION: 12 份新配置与相关文档的最终公共/个人目录结构、个人配置如何脱敏和保密？]
