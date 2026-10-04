# 研究问题与证据边界

- **标识符**：meta-rules-unification
- **研究阶段**：Assessment `research`；仅记录证据，不作投入结论或方案设计。
- **创建日期**：2026-10-03（Asia/Hong_Kong）
- **整体证据置信度**：medium
- **范围**：直接研究 MetaCubeX `meta-rules-dat` 资源与 Mihomo 实现，支持从 `rule-providers` 开始的整个规则段重新设计；同时审阅 8 份公开模板、4 份私有配置的脱敏汇总及相关项目文档。旧 12 份配置既可直接作为 `rule-providers` 之前全部配置内容的重建参考，也可用于说明旧规则现状、后续提取指定本地规则、平台差异和模板简化线索；旧 provider 仅作为历史证据，不形成新规则的逐项对应或设计输入。新规则从流量意图出发：需翻墙的小流量走 `ROUTE_AI`，需翻墙的大流量走 `ROUTE_GLOBAL`，不应翻墙的走 `DIRECT`，应屏蔽的走 `REJECT`，最后以 `PROXY` 兜底。
- **固定版本**：GeoSite/GeoIP 数据取 `meta-rules-dat` commit `cb3e075825906a4dcc2feff84e3d6cc5691d9416`；README 取 `4178770badecb1b349fbcd62c737e0d7a2079729`；Mihomo 实现取 `88dcbf7f1614a67c3b36b848ee3592dfa92ada36`。[来源：MetaCubeX 资源目录、README 与 Mihomo 固定版本，见“来源”]
- **隐私处理**：4 份私有配置只用于核对 provider 键名、总数、behavior/format 计数、规则顺序和 action 种类；本稿不记录其 URL、订阅值、节点名或其他私有字段。私有文件均被 Git 忽略；该事实仅作边界记录，不证明其内容或运行状态。[来源：脱敏配置统计；`git status` / `git ls-files`]

# 用户与需求信号

- 旧项目诊断记录了一次桌面 Epic 下载流量经代理发出的事件，并描述当时相关的旧配置行为。记录明示问题修复未实施、验证未运行；它只作为特定配置版本的历史证据，不确认 Epic 的新路由意图，也不规定新规则的动作、资源选择或次序。[来源：`.specify/bugs/epic-download-routing/assessment.md:8-27`、`:37-57`]（置信度：high）
- 仓库的网络规则原则要求按规则组覆盖、成员混合用途、重叠和自上而下首匹配次序评估，并明确禁止仅凭集合名称整组改路由；除 Fake-IP 兼容和特定桌面网络敏感程序外，不为单一服务加域名/IP/进程规则。[来源：`AGENTS.md:82-88`]（置信度：high）
- 已知规则段边界为 `rule-providers`：此前的所有配置内容均可直接以旧配置为参考重建。新规则的 fallback 首选组已指定为 `AUTO_STRICT` 与 `AUTO_GENERAL`；尚待核对目标平台配置中组声明是否一致，并厘清私有配置重建所需的用户特定内容和脱敏方式。

# 旧配置及项目证据

以下是旧模板、个人配置的脱敏统计及项目文档证据。`rule-providers` 之前的所有配置内容可以直接参考旧配置重建；从 `rule-providers` 开始的 provider 定义和后续路由规则仅作为旧设计证据，不自动决定新规则骨架。新规则骨架形成后，还按用户指定范围参考旧配置提取三类本地规则、平台差异和模板简化线索。

- 旧项目文档记录了当时的分类动作和规则安排；这些仅作为旧版行为背景，不决定新规则如何分类、选取数据或排列顺序。新规则按用户当前说明的四类流量意图独立组织。[来源：`README.md:95-112`；`LOCAL_ARCH_DESIGN.md:69-84`]（置信度：high）
- 8 份公开模板和 4 份个人配置的脱敏摘要显示，Desktop/Docker 各有 21 个 provider，Android/iOS 各有 17 个；前两者的行为计数为 `classical:8, domain:11, ipcidr:2`，格式计数为 `text:5, yaml:16`，移动端分别为 `classical:5, domain:10, ipcidr:2` 与 `text:3, yaml:14`。四类平台矩阵还记录了移动端裁剪情况；此处仅保存汇总，不列旧 provider 清单或逐项动作。iOS Network Extension 的 15 MB 约束来自架构文档，现有资料未独立测量新设计数据的常驻内存贡献。[来源：公开模板、脱敏私有统计；`LOCAL_ARCH_DESIGN.md:50-57,71-83`]（置信度：配置统计 high；运行内存影响 low）
- 旧项目文档记录了旧配置的分类行为和规则相对次序。这些描述仅用于说明旧配置状态，不定义新规则的分类、动作、数据选择或顺序。[来源：`README.md:95-112`；`LOCAL_ARCH_DESIGN.md:69-84`]（置信度：high）
- 旧模板、README 与历史 fix/test 记录对若干规则类别的先后关系存在可核对的描述；私有配置仅采用脱敏统计，未逐行确认其顺序。此处只记录旧版本证据及其来源，不据此推导新规则顺序。[来源：公开模板 `mihomo_config_desktop_single_template.yaml:306-343`、`mihomo_config_android_single_template.yaml:267-296` 及同结构的其余 6 份模板；`README.md:99-103`；`.specify/bugs/fix-rules-order-gfw-ai/fix.md:6-10,53-96`；`.specify/bugs/fix-rules-order-gfw-ai/test.md:7-17,21-46`；`.specify/bugs/epic-download-routing/assessment.md:18-27`]（置信度：公开模板顺序 high；私有运行实例符合程度 low）

# 上游资源和格式证据

- MetaCubeX 仓库的 `geo/geosite/` 与 `geo/geoip/` 发布单组 `.yaml`、`.list`、`.mrs` 文件；Mihomo provider 对应格式名分别为 `yaml`、`text`、`mrs`。域名资源可使用 `behavior: domain`，CIDR 集合可使用 `behavior: ipcidr`。因此文件格式有适配路径，但文件扩展名本身不能证明成员或行为相等。[来源：MetaCubeX 固定版资源目录；Mihomo `constant/provider/interface.go:134-180`；上游 README 的 MRS 配置示例]
- Mihomo 的 MRS reader 需要相应 domain/ipcidr 策略实现；检查的策略实现暴露 `FromMrs`，而 classical 策略解析规则行并有其自身规则语法。因此，classical/text、classical/yaml 与二进制域名/IP 集合的规则语法不同；仅修改格式声明不能证明规则内容可转换或行为等价。[来源：Mihomo `rules/provider/provider.go:55-65,177-241`、`domain_strategy.go:22-57`、`ipcidr_strategy.go:22-58`、`classical_strategy.go:18-57`]（置信度：high）
- `meta-rules-dat` README 指出数据库包含来自其他上游的集合，也对部分地区和服务类别集合做了替换或合并。这说明该项目会汇集、整理其他数据；具体数据仍须依据其实际内容和已确认的流量意图评估，不能单凭发布者或类别名推导用途。[来源：MetaCubeX README 固定版本第 25-71 行]
- 可选数据形态包括随 Mihomo GeoSite/GeoIP 数据库调用的 `GEOSITE`/`GEOIP` 规则，以及独立 `.yaml`、`.list`、`.mrs` rule-set。`geosite-lite` 和 `geoip-lite` 是精简发布物，README 明确说 lite 的 `cn` 可能不完整。内置数据库组、外置 rule-provider 和本地 classical 列表是不同接入形态，不能只看规则名视为可互换。[来源：MetaCubeX README 固定版本第 5-17、25-71、73-84 行；`mihomo_config_*_template.yaml` 中 `GEOSITE` 与 `RULE-SET` 用法]

以下分开记录旧配置的汇总历史事实与 MetaCubeX 固定快照的数据观察。两部分互不映射：旧版资料只说明其当时的平台统计和规则行为；新规则应从四类流量意图出发，再根据已确认的需求检查相关数据。下列观察用于说明证据边界，不构成新规则的数据选择或设计输入。除特别注明外置信度 medium。

## 旧配置历史事实

旧配置在 Desktop/Docker 模板中使用 21 个 provider，在 Android/iOS 模板中使用 17 个；模板文档和规则行还记录了当时的平台差异、行为类型、文件格式以及规则先后次序。前述统计为脱敏汇总，不在此列出 provider 名称与原动作的逐项组合，也不表达新规则的类别、动作、次序或数据选择。来源：四类平台公开模板的 `rule-providers` 和模板规则行；`LOCAL_ARCH_DESIGN.md:50-84`。私有配置仅采用脱敏统计，不展开个人规则内容。（置信度：high）

## MetaCubeX 固定快照的数据观察

以下是固定数据版本中抽样或计算所得的事实记录，用于说明已查内容的边界，不构成新规则资源清单或设计输入。每项都受所注明的数据范围和检查方法限制。

- 一份类别样本包含游戏服务、通用 CDN 和对象存储域名；该样本只能说明抽取条目内容多样，不能说明整个类别的用途或完整成员范围。[来源：MetaCubeX 固定版类别样本；`LOCAL_ARCH_DESIGN.md:73`]（置信度：high，限于样本）
- 媒体与 CDN 类别样本中可见媒体托管、流媒体、支持服务和共享 CDN 域名；两份样本还有后缀形式可能相交的条目。样本反映所检查条目的构成，不能据此决定路由动作。[来源：MetaCubeX 固定版媒体与 CDN 类别样本]（置信度：high，限于样本）
- 对两份固定版独立数据中的 YAML 条目按大小写归一化后，得到 30 条完全相同的文本项，说明文本层面存在精确交集；这不是运行时完整交集，后缀匹配可能扩大实际重叠范围。[来源：固定版 YAML 文件本地交集统计]（置信度：high，限于计算方法）
- MetaCubeX README 说明某广告类别仅使用域名且没有额外补充域名；这说明该数据的收录形式，不足以证明任何用途下的完整覆盖。[来源：MetaCubeX README 固定版本第 54-59 行；对应固定版数据文件]（置信度：high）
- GeoSite 域名与 GeoIP CIDR 属于不同匹配空间。固定版保留地址数据样本包含保留地址范围、回环、链路本地、文档网段及组播；类似类别名不代表资源类型或覆盖范围相同。[来源：MetaCubeX 固定版 GeoIP 样本；Mihomo provider 实现]（置信度：high）
- 本次所查的不同地区标记 GeoSite 类别与相应 GeoIP 数据代表不同地区范围或数据类型；没有测量两者的完整成员交集。[来源：MetaCubeX 固定版 GeoSite/GeoIP 资源目录及 README]（置信度：medium）
- README 将一个仅含于 lite GeoSite 数据库的类别列入其目录；本次检查的独立 GeoSite 目录中未发现对应名称的 `.yaml`、`.list` 或 `.mrs` 文件。内置数据库类别与独立 rule-set 是不同发布形态。[来源：MetaCubeX README 固定版本第 5-17、73-84 行；固定版 GeoSite 目录]（置信度：medium）

# 顺序行为与重叠证据

- Mihomo 的路由规则按配置中的先后顺序匹配；旧项目诊断将特定连接未到达当时位于后面的规则归因于首匹配行为。该案例记录的是旧配置版本，修复未实施、验证未运行；它不确认 Epic 的新路由意图，也不推导新规则应采用的动作、数据或次序。[来源：`.specify/bugs/epic-download-routing/assessment.md:8-27`、`:37-51`；旧配置次序记录：`mihomo_config_desktop_single_template.yaml:294-336`]
- 旧模板按平台使用不同的规则组合，旧版基础规则、分类规则和兜底规则之间存在特定次序；Desktop/Docker 与移动端的规则规模不同。这里只记录旧模板结构及平台差异，不将其作为新规则的排序依据。[来源：模板规则行；`LOCAL_ARCH_DESIGN.md:50-84`]（置信度：high）
- 固定快照中观察到不同类别样本具有可重叠的域名后缀；按后缀规则语义可判断部分样例可能交叠。此判断是匹配语义推论，并非对所有成员计算出的完整交集，也不决定未来规则动作。[来源：MetaCubeX 固定版 YAML 样例]（置信度：medium）
- [NEEDS CLARIFICATION: 新规则方向已定为四类流量意图；后续是否需要对依据已确认意图选定的数据做全量成员审计、后缀匹配、CIDR 包含和数据间交集分析？本研究中的历史配置事实与 MetaCubeX 观察不构成旧新规则的逐项映射。]

# 数据、约束与证据质量

- MetaCubeX README 说明 `geosite-lite`/`geoip-lite` 是精简数据库，Lite `cn` 可能不完整；lite 资源在地区集覆盖与体积上存在差异。[来源：MetaCubeX README 固定版本第 39-52、66-71 行]（置信度：high）
- iOS Network Extension 的 15 MB 限制以及移动端裁剪策略来自项目架构文档；当前没有本机/手机上的内存 profile、MRS 映射后的 RSS 或首次载入峰值数据，不能据此推算新设计能否满足限制。[来源：`LOCAL_ARCH_DESIGN.md:50-57`]（置信度：约束来源 high，运行内存影响 low）
- 当前项目允许的路由 action 是 `REJECT`、`DIRECT`、`PROXY`、`ROUTE_XXX`，规则 action 不应指向 `AUTO_XXX`。本次脱敏汇总中的私有配置 action 均属于允许类型。[来源：`.specify/memory/constitution.md` 路由原则；脱敏私有汇总]（置信度：high）
- `meta-rules-dat` 来源包含多个上游，且部分数据库类别经历合并/替换；来源合并记录描述了数据整理过程；发布者身份本身不能证明更新频率、源完整性、授权状态或与本地流量意图一致。[来源：MetaCubeX README 固定版本第 25-71 行]（置信度：high）
- 本次上游访问通过 `gh` CLI 获取固定版 MetaCubeX/Mihomo 的元数据和文件内容；GitHub 页面抓取器拒绝了 GitHub URL 的连接，因此研究结论基于该 CLI 返回的固定版本内容。[来源：本次研究执行记录]（置信度：high）

# 数据限制与潜在风险

- 已检查的 MetaCubeX 类别样本显示，其中可能包含共享资源、CDN 或支持域名；类别名称本身不能证明成员用途。此处是对数据解释限制的记录，不选定任何新规则资源或动作。[来源：MetaCubeX 固定版游戏、媒体与 CDN 类别样本；`AGENTS.md:84-88`]（置信度：high）
- 本次检查的 MetaCubeX 数据中未发现通用下载专用集合，且游戏类别样本不专指下载内容。此结果仅限本次检查范围，不构成新规则要求。Epic 的新路由意图尚未确定，旧诊断中的预期行为也未被确认为新规则要求。[来源：`LOCAL_ARCH_DESIGN.md:73-75`；`.specify/bugs/epic-download-routing/assessment.md:23-27`；本次 MetaCubeX 数据检查]（置信度：high）
- GeoSite 域名、GeoIP CIDR、classical/process 规则以及路由 action 彼此不同；其匹配对象和语义不能由扩展名或统一结构推定为等价。[来源：Mihomo provider interface；固定版 GeoIP 私有地址样本；`AGENTS.md:84-88`]（置信度：high）
- 旧规则与 MetaCubeX 数据在范围、格式和业务场景上的差别不能由名称对应消解。`rule-providers` 之前的配置内容可参考旧配置重建；之后的新规则依据用户指定原则，从流量意图出发形成。旧配置仅在明确范围内支持指定本地规则和平台信息的整理。[来源：`LOCAL_ARCH_DESIGN.md:73-83`；MetaCubeX README 固定版本第 25-71 行]（置信度：medium）
- 尚无新设计规则数据在目标设备上的加载/内存数据及长期误分流度量；这些资料会影响平台差异和验证深度，不能由旧 provider 数量推断。[来源：本次研究未取得这些项目度量]（置信度：medium）

# 待后续阶段确认的证据边界

- [NEEDS CLARIFICATION: 新规则设计时是否需要检查仅由预编译数据库提供的类别？可检查的数据形式包括 `GEOSITE`/`GEOIP` 内置数据库和独立 rule-set，具体范围应由已确认的流量意图决定。]
- [NEEDS CLARIFICATION: 私有工作配置中 `rule-providers` 前的用户特定差异、启动参数及平台资料，哪些需要随 `rule-providers` 之前的配置内容一并重建；如纳入，采用何种脱敏边界？]
- [NEEDS CLARIFICATION: 历史修复记录与旧模板顺序存在差异；该冲突是否只记录为旧基线差异，或需要后续从运行实例核实？]
- [NEEDS CLARIFICATION: 新规则顺序评估要覆盖哪些地区、Fake-IP 模式、DNS 解析策略和共享 CDN 误分流场景？]
- [NEEDS CLARIFICATION: 后续验证是否需要测量规则数据库压缩体积、不同 Mihomo 客户端的下载/解压/更新峰值，尤其 iOS Network Extension？]
- [NEEDS CLARIFICATION: 固定版 MetaCubeX commit 是否作为后续新规则研究基线；进入后续实际撰写时是否重新核对上游更新？]

# 来源

下列网络来源只访问了 `github.com`，符合 intake URL policy 的 allowlisted host 分支；未从其他未识别 host 抓取内容。文件中的项目本地引文使用仓库相对路径和行号，不是网络 URL。

- `https://github.com/MetaCubeX/meta-rules-dat/tree/cb3e075825906a4dcc2feff84e3d6cc5691d9416/geo/geosite` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/meta-rules-dat/tree/cb3e075825906a4dcc2feff84e3d6cc5691d9416/geo/geoip` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/meta-rules-dat/blob/4178770badecb1b349fbcd62c737e0d7a2079729/README.md` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/meta-rules-dat/blob/cb3e075825906a4dcc2feff84e3d6cc5691d9416/geo/geosite/category-ai-%21cn.yaml` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/meta-rules-dat/blob/cb3e075825906a4dcc2feff84e3d6cc5691d9416/geo/geosite/gfw.yaml` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/meta-rules-dat/blob/cb3e075825906a4dcc2feff84e3d6cc5691d9416/geo/geosite/category-cdn-%21cn.yaml` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/meta-rules-dat/blob/cb3e075825906a4dcc2feff84e3d6cc5691d9416/geo/geosite/category-media.yaml` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/meta-rules-dat/blob/cb3e075825906a4dcc2feff84e3d6cc5691d9416/geo/geosite/category-games.yaml` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/meta-rules-dat/blob/cb3e075825906a4dcc2feff84e3d6cc5691d9416/geo/geosite/category-ads-all.yaml` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/meta-rules-dat/blob/cb3e075825906a4dcc2feff84e3d6cc5691d9416/geo/geoip/private.yaml` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/meta-rules-dat/blob/cb3e075825906a4dcc2feff84e3d6cc5691d9416/geo/geosite/google.yaml` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/mihomo/blob/88dcbf7f1614a67c3b36b848ee3592dfa92ada36/constant/provider/interface.go` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/mihomo/blob/88dcbf7f1614a67c3b36b848ee3592dfa92ada36/rules/provider/provider.go` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/mihomo/blob/88dcbf7f1614a67c3b36b848ee3592dfa92ada36/rules/provider/domain_strategy.go` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/mihomo/blob/88dcbf7f1614a67c3b36b848ee3592dfa92ada36/rules/provider/ipcidr_strategy.go` （host: `github.com`；policy: allowlisted）
- `https://github.com/MetaCubeX/mihomo/blob/88dcbf7f1614a67c3b36b848ee3592dfa92ada36/rules/provider/classical_strategy.go` （host: `github.com`；policy: allowlisted）
