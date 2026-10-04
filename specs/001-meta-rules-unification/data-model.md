本 Feature 交付 Mihomo 配置，不新增运行时数据库或程序实体。以下实体是配置设计、场景审阅和文档记录中的逻辑数据模型。

# 流量场景

| 字段 | 含义 | 校验约束 |
| --- | --- | --- |
| `scenario_id` | 稳定的场景标识 | 在场景表中唯一 |
| `operation` | 用户实际执行的业务操作 | 描述具体操作，不以服务名或集合名替代 |
| `block_intent` | `block`、`allow`、不适用或待确认 | 首先判断是否应拦截；应拦截的操作预期走 `REJECT`；仅用于兜底的抽象场景可标不适用 |
| `proxy_intent` | `proxy`、`direct`、不适用或待确认 | 仅对不应拦截的操作判定；不应代理的操作必须走 `DIRECT` |
| `size_class` | `small`、`large`、不适用或待确认 | 仅描述已确认需要代理操作的预期传输规模；此字段选择 ROUTE fallback 组。批量载荷、长时媒体或实际承载图片/视频载荷的 CDN 操作可作为 large 候选特征，但特征名称本身不足以分类；操作规模不适用时填 `N/A`，依据不足时填待确认 |
| `expected_action` | `ROUTE_NORMAL`、`ROUTE_HEAVY`、`DIRECT`、`REJECT`、`PROXY` 或待确认 | 按顺序：应拦截 → `REJECT`；未拦截且不需代理 → `DIRECT`；需代理的非大流量 → `ROUTE_NORMAL`；需代理的大流量 → `ROUTE_HEAVY`；未命中前述规则的最终兜底 → `PROXY` |
| `basis` | 拦截和代理意图依据 | 需要可审阅的用户/维护者依据；禁止仅以旧动作或集合标签推断 |
| `scale_basis` | 该具体操作归入 `small` 或 `large` 的传输特征、判断理由及依据来源 | 仅在已确认需要代理且规模适用时填写；描述操作的预期载荷/持续传输特征及为何支持该分类。不能仅以服务名、CDN 身份或集合名分类；不设数值阈值或运行时字节切换。已确认规模不适用时填 `N/A`；代理意图或规模依据尚待确认时，将 `size_class` 标为待确认，并记录尚缺信息 |
| `first_match` | 配置中首次匹配的规则及动作 | 必须与已确认的 `expected_action` 一致；未确认场景不可标为通过 |
| `status` | 待确认、已确认、已验证、未覆盖 | 已验证要求可追溯的规则和验证证据 |

需要代理的非大流量操作可作为 `ROUTE_NORMAL` 基准；需要代理的大流量操作可作为 `ROUTE_HEAVY` 基准。候选链按配置类型区分，且只列出该配置实际定义的候选组：

- **4 份个人配置**：`ROUTE_NORMAL: AUTO_STRICT → AUTO_GENERAL → AUTO_BACKUP`；`ROUTE_HEAVY: AUTO_GENERAL → AUTO_STRICT → AUTO_BACKUP`。`AUTO_BACKUP` 为末位逃生候选。
- **8 份公开模板**：不拆节点池，`ROUTE_NORMAL` 与 `ROUTE_HEAVY` 均 fallback 到单一 `AUTO` 组。模板用占位订阅，无法预设 `YOUR_HEAVY_KEYWORD` 之类的节点命名；老模板按节点协议类型拆分的做法已废弃。

策略组名称不代表业务类别。维护者须点名 REJECT 样例和依据后，该路径才可记为已验证。Desktop 游戏内容、补丁或分发切片下载按通用操作范围审阅 DIRECT 意图与规则组覆盖，不另设服务品牌验收项或例外。

# 规则数据集合

| 字段 | 含义 | 校验约束 |
| --- | --- | --- |
| `resource_id` | 本设计内稳定引用 | 唯一；避免将旧 provider 标识机械改名后充当新设计依据 |
| `source` / `revision` | 上游仓库与固定版本 | 可复现；上游更新必须显式审阅 |
| `resource_path` | 资源路径或数据库类别标识 | 必须在对应发布形态中存在 |
| `match_space` | domain、IP/CIDR 或 Mihomo Geo* 数据库对象 | 与 `behavior` 和配置匹配语义一致 |
| `format` | `yaml`、`text`、`mrs` 或适用的内置数据库形态 | 与 Mihomo 支持的 provider 类型匹配，不由扩展名单独推断 |
| `intended_action` | 目标业务动作 | 必须有场景意图依据；不确定或混杂时不得整组绑定单一相反动作 |
| `member_evidence` | 代表成员与审阅范围 | 标出样本/全量边界；不得把抽样说成完整覆盖证明 |
| `known_limits` | 混合用途、缺口、格式或平台限制 | 已知限制必须进入说明 |
| `overlaps` | 与其他选用集合的交叠 | 记录精确条目、后缀匹配或 CIDR 包含的检查口径及其限制 |

资源选择遵循“原则性”：按已确认意图优先采用覆盖最完整且边界适用的资源，不因服务知名或名称熟悉而默认引入 Apple、Google 等细分资源。只有具体证据表明不应代理流量被代理、非大流量代理操作进入 `ROUTE_HEAVY`、或大流量代理操作进入 `ROUTE_NORMAL`，且调整顺序无法解决或会引入新的路由问题时，才考虑更细资源。触发记录须包括具体操作及意图、资源成员/匹配语义、当前首匹配规则和动作、期望动作、候选顺序调整及其无法解决问题或引入其他误分流的依据。样本证据注明抽样范围和未覆盖边界，抽样不得表述为全量证明。候选范围包括 MetaCubeX `meta-rules-dat` 中所有适用资源形态，不限于规则集。

被实际引用的每个规则数据集合必须存在声明；未采用的候选资源不得遗留悬空引用。

# 路由规则与策略组

| 字段 | 含义 | 校验约束 |
| --- | --- | --- |
| `position` | `rules` 中从前到后的次序 | 逐一评估首匹配结果和冲突场景 |
| `matcher` | 规则类型及匹配目标 | 与规则提供者行为、平台能力和数据格式一致 |
| `target` | 路由动作或策略组 | 业务目标只能为 `ROUTE_NORMAL`、`ROUTE_HEAVY`、`DIRECT`、`REJECT`；禁止直接使用 `AUTO_XXX` |
| `fallback_chain` | 容灾策略组次序 | 按配置类型：个人配置为 `ROUTE_NORMAL`：`AUTO_STRICT`、`AUTO_GENERAL`、`AUTO_BACKUP`，`ROUTE_HEAVY`：`AUTO_GENERAL`、`AUTO_STRICT`、`AUTO_BACKUP`；公开模板为单一 `AUTO`。仅列出该配置实际定义的候选组，未定义的策略组不得引用 |
| `terminal_fallback` | 最后一条未命中规则 | 必须是最后一条 `MATCH,PROXY` |

路由意图先裁定是否拦截，再裁定未拦截操作是否需要代理；需要代理时按操作规模选择 fallback 组：非大流量使用 `ROUTE_NORMAL`，大流量使用 `ROUTE_HEAVY`。两组只定义不同的首选和容灾顺序，不由名称推断业务类别。集合交叠先按原则性意图和安全顺序解析；只有顺序无法修复或会引入新误分流，且有证据属于“不应代理流量进入代理、非大流量进入 `ROUTE_HEAVY`、大流量进入 `ROUTE_NORMAL`”之一时，才评估更细资源。

# 配置矩阵与文件边界

| 维度 | 允许值/数量 | 校验约束 |
| --- | --- | --- |
| `platform` | Desktop、Docker、iOS、Android | 平台裁剪遵循 Constitution |
| `subscription_mode` | single、dual | 每个平台各一份公开模板 |
| `visibility` | public、local-only | public 不得包含个人字段；local-only 被 Git 忽略 |
| `config_file` | 8 public + 4 local | 8 模板均为平台×模式笛卡尔积；个人配置每平台一份 |
| `archive_baseline` | 原始 12 份文件 | 归档内容与执行归档前的字节内容相同，个人材料仍不跟踪 |
| `platform_constraints` | 平台相关约束记录 | 包含移动端资源裁剪、iOS 正则兼容、Docker 网络边界和 Desktop 能力 |
| `runtime_memory_status` | `unmeasured` 或有证据的实测状态 | 未依获准口径设备实测前不得声称达到 15 MB 运行时上限 |

配置矩阵建议位于 ``，归档基线建议位于 `archive/meta-rules-unification-baseline/`；最终文件命名须符合 Constitution 的公开模板命名和个人本地文件命名规则。

# 关系与状态约束

- 一个流量场景可由一个或多个规则数据集合匹配；所有已确认且实际匹配的集合须通过规则顺序解析成一个首匹配动作。
- 一个规则数据集合可被多个平台模板引用，或因平台资源裁剪而不被引用；平台差异必须说明。
- 每个公开配置属于唯一平台和订阅模式；每个本地个人配置属于唯一平台且始终 local-only。
- `待确认 → 已确认 → 已验证` 为场景证据状态推进；没有实际规则审阅证据时不得跳到已验证。覆盖不足可记录为 `未覆盖` 并提出规则组级选项，不得用单服务例外掩盖。
- 修改任一共享规则集合、动作或通用顺序时，必须复核全部 8 份公开模板，并同步检查适用的 4 份个人配置。
