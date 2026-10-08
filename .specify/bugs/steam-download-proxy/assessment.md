- **Slug**: steam-download-proxy（沿用本会话用户已确认的标识；旧记录已删除，本次重新诊断）
- **Created**: 2026-10-09T02:38:39+08:00
- **Source**: 用户连接记录、本机 Steam 日志、通过 gh 读取的固定版本 MetaCubeX 数据
- **Verdict**: valid
- **Severity**: medium
- **阶段边界**: 仅诊断；未修改配置；方案确认前不进入 fix；自动目标保持暂停。

# 症状与现场证据

用户报告 shared.fastly.steamstatic.com:443 下载 12.1 MB，命中 RuleSet(gfw)，走 ROUTE_NORMAL / AUTO_STRICT；cache7-lax2.steamcontent.com:443 下载 18.8 MB，命中 Match，走 PROXY / ROUTE_HEAVY / AUTO_GENERAL。不复述代理节点标识。

游戏下载应 DIRECT，商店、社区、视频允许代理。只读查看本机 Steam content_log.txt 的 90–129 行，AppID 281990 更新明确使用 lax2 SteamCache 节点下载 depot 281991/281992。152 行记录 cache7-lax2 下载 19654944 Bytes、23 Hits / 0 Misses。shared.fastly 暂无 depot 请求证据，不能用流量大小判定用途。

# 复现与检查

1. 用户进行 Steam 游戏下载/更新，现场记录 cache7-lax2 命中 Match 代理。
2. 当前 Desktop 配置下载组使用固定版本 cb3e075825906a4dcc2feff84e3d6cc5691d9416 的 category-game-platforms-download.mrs，规则位于 gfw 后、通用 CDN 前，最终 MATCH,PROXY。
3. 通过 gh api contents 接口读取上述固定版本 category-game-platforms-download.list、steam.list、gfw.list，Base64 解码后按精确域名和 +. 后缀语义核查。命令退出 0。

本次未启动隔离核心或触发新下载；用户现场与当前规则数据支持诊断，不是修复后复现。list 的检查不能直接证明同名 MRS 的实际加载匹配结果。

# 相关路径

- [Desktop 配置](../../../mihomo_config_desktop.yaml)：262–267 行下载 provider；388 行 gfw；408 行下载 DIRECT；432 行代理兜底。
- [项目级约束](../../../AGENTS.md)：82–92 行规定唯一来源；105 行禁止单服务规则绕过。
- [Feature 来源决策](../../../specs/001-meta-rules-unification/research.md)：19 行规定从 MetaCubeX 的全部适用资源形态选择。
- 本机证据：C:/Program Files (x86)/Steam/logs/content_log.txt，90–129、152 行。

# 根因与覆盖核查

置信度高：下载组漏掉 cache7-lax2，gfw 也不覆盖它，因此落入代理兜底。缺失成员不能通过只调整下载组顺序解决。

| 域名 | 下载组 | steam | gfw |
|---|---|---|---|
| cache7-lax2.steamcontent.com | 不匹配 | +.steamcontent.com | 不匹配 |
| shared.fastly.steamstatic.com | 不匹配 | +.steamstatic.com | +.steamstatic.com |
| store.steampowered.com | 不匹配 | +.steampowered.com | +.store.steampowered.com |
| steamcommunity.com | 不匹配 | +.steamcommunity.com | +.steamcommunity.com |
| steamvideo-a.akamaihd.net | 不匹配 | 精确成员 | +.akamaihd.net |
| steambroadcast.akamaized.net | 不匹配 | 精确成员 | 不匹配 |
| steamusercontent.com | 不匹配 | +.steamusercontent.com | 不匹配 |

这是代表域名检查，非全部成员用途或全量交叠证明。把整个 steam 组放在 gfw 后 DIRECT，虽然会保留前五项中受 gfw 覆盖的代理域，仍会将直播、用户内容等未被 gfw 覆盖的成员直连，不能视为边界干净的下载补充。

# 更新机制与修复持久性

当前 Desktop 下载 provider 为 `type: http`、`format: mrs`，`path: ./rule_providers/game-download.mrs`，`interval: 86400`。其 URL 固定到 commit `cb3e075825906a4dcc2feff84e3d6cc5691d9416`，公开 Desktop 模板采用同样设置。

必须区分两件事：定时刷新会请求该固定 commit 的资源，不会自动跟随 MetaCubeX 最新版本；本地缓存也不是可维护的修复源。直接修改缓存文件不能保证经刷新、重新下载或重建环境后保留，因此排除为修复方案。当前固定 URL 阻断上游更新，是本 bug 合并治理的独立缺陷；必须将全部规则订阅改为 meta 持续发布分支。

可接受的方案必须说明持久修改落在哪里，以及刷新和重新下载后为何仍有效。上游修正方向需要实际生成来源、可审阅补丁、产物版本、配置引用更新范围及验证证据。目前这些尚未齐备，不能声称方案可实施，不得进入 fix。

# 已查证的生成链与资源边界

通过 gh 读取 MetaCubeX master commit `4178770badecb1b349fbcd62c737e0d7a2079729` 的 `.github/workflows/run.yml`：构建检出 `v2fly/domain-list-community`，将 community/data 编译为 geosite.dat，再由 MetaCubeX/meta-rules-converter 生成 meta 分支资源。工作流没有独立维护或覆写游戏下载分类的步骤；只修改 meta 分支生成产物会在重新构建时丢失。

读取 v2fly commit `e84921279c9142434b43cb84ec6c17f987e025df` 的 `data/category-game-platforms-download`：第 131 行将 Valve 缓存段标为“游戏下载/社区实况直播”，后面是逐个 full: 主机枚举，完全没有 lax2。该注释证明分类不能被视为纯下载用途；不证明每一个 lax2 节点实际承载直播。

读取最新 MetaCubeX meta commit `32bbb6c45c63a491c571f98cd9d863d7fd6771dc` 的下载 list：仍无 lax2 或 steamcontent 宽后缀。通过逐层 git tree 枚举完整 geosite 目录（5713 个文件，truncated=false）：相关业务资源只有下载分类及其 @cn、steam 及其 @cn（各有 list/mrs/yaml）；没有独立 Steam 下载或直播子组。此枚举限定在 geosite 目录，不宣称已证明任意 IP 数据或所有逻辑组合均不可能；但 IP/连接条件未经用途证据支持，不能提供所需业务区分。

来源证据：

- [MetaCubeX 构建工作流](https://github.com/MetaCubeX/meta-rules-dat/blob/4178770badecb1b349fbcd62c737e0d7a2079729/.github/workflows/run.yml)
- [分类源数据](https://github.com/v2fly/domain-list-community/blob/e84921279c9142434b43cb84ec6c17f987e025df/data/category-game-platforms-download)
- [最新下载文本产物](https://github.com/MetaCubeX/meta-rules-dat/blob/32bbb6c45c63a491c571f98cd9d863d7fd6771dc/geo/geosite/category-game-platforms-download.list)

# 修复方案（Proposed Remediation）

**合并治理两项缺陷：A. 12 份配置的规则订阅被 commit 锁死；B. Steam lax2 下载节点覆盖缺口。A 恢复持续更新；B 的单域名提案已被用户拒绝；正在评估 steam 规则组的顺序及组合，已确认覆盖下载但附带额外业务直连，尚无满足全部边界的获批方案；上游提案仅为非必需备选。不得因 A 修复完成就宣称 B 已解决。**

## A：恢复所有平台规则订阅持续更新

静态审计 12 份配置：Desktop/Docker 各 22 个、Android 各 19 个、iOS 各 16 个 rule-provider，全部锁到 cb3e075…，均设置 86400 秒刷新。合计 237 个声明、38 个不同资源路径；节点 proxy-provider 不属于该锁版缺陷。设计将研究快照变成运行时订阅版本，是根因。

将全部 rule-provider URL 的版本段改为 `meta`，保留唯一来源、资源路径、格式、path、interval、动作和顺序。公开模板与个人配置同时修复，iOS 保留 YAML、其余平台保留各自既有格式。通过逐层 git tree 枚举已验证 meta 分支中 38 个路径全部存在，geoip/geosite 均 truncated=false；早先递归 tree 被截断的 MISSING 输出无效，不作为缺失证据。

预计修改范围：12 份配置；README.md、rule-design.md、validation.md；Feature plan/research/data-model/contracts/quickstart 中仍要求运行时固定版本或逐次人工批准更新的约束；AGENTS.md 已按用户明确要求改为持续更新硬约束。历史研究证据保留事实，不再作为运行时锁版要求。

验证要求：解析全部配置；逐个核查 237 个声明唯一来源及 meta URL、刷新周期和引用；下载 38 个实际资源，按目标核心/格式验证加载；在隔离核心主动刷新 provider，记录结果并重测关键路由场景。平台未实际运行必须披露，不能以 Mihomo 测试冒充 Stash 验证。刷新后仍无 lax2 的事实作为 B 未解决证据。

## B：规则组方案重新评估（用户拒绝单域名例外）

用户明确不接受单域名、单主机规则，不打破项目原则。下面的单域名提案撤回，不得实施；允许直播直连仍有效，但不等于允许其他业务直连。

现有规则的纯顺序调整不能修复 cache7-lax2：在本次最新 meta 数据的全部现有 domain provider 中，该主机不属于任何集合。该结论仅限域名匹配，不把 fake-IP 地址误当上游 IP 数据覆盖证据。

可落地的规则组候选确实存在：新增 MetaCubeX steam.mrs provider，订阅 meta 分支，置于现有 gfw、游戏下载、业务代理、CDN 域名/IP 层之后、MATCH 之前，动作 DIRECT。steam.list 含 +.steamcontent.com，故能接住 lax2 和后续同类节点；现有 gfw 接住 store.steampowered.com、steamcommunity.com 和 steamvideo-a.akamaihd.net，CDN 层接住其他已覆盖的媒体宿主。此方案不依赖上游改分类、不添加单域名条件、也不引入第二来源。

然而它还会将原本 MATCH→PROXY 的 steam-chat.com、s.team、steamusercontent.com、以及未被 gfw 全部覆盖的 steampowered.com 子域等变为 DIRECT；属于超出直播让步的行为变化，当前不能认定为获批方案。60 个 steam 条目的代表宿主匹配检查不是其所有子域用途的全量证明，现有规则包含个人例外/IP 条件，最终必须由核心验证。

已检查 steam@cn、category-games、geolocation-!cn、category-communication、cn 等候选：steam@cn 不含 steamcontent；category-games 和 geolocation-!cn 同时包含 steamcontent 与上述聊天/用户内容，作为先行代理会一起抢走下载；category-communication 和 cn 未含所列 Steam 反例。steam 减 steam@cn 可排除一部分用户内容，却仍会包含聊天、短链等，不能仅靠这一组合宣称满足业务边界。

结论：否定“上游没有任何资源可用”的绝对说法；steam 组能覆盖下载，但整组顺序方案有额外直连范围。当前未找到只让下载/直播直连并保留其余 Steam 代理的已验证组合；未穷举 MetaCubeX 所有条目或所有逻辑表达式，不能声称所有资源组合绝对不可能。不得把这些候选限制写成已完成核心测试。

## 已撤回的单域名提案（仅留治理记录）

用户已明确允许 Steam 直播直连，要求修复不依赖上游接受提案。首选改为在现有游戏下载 DIRECT 层增加 `DOMAIN-SUFFIX,steamcontent.com,DIRECT`，保留原 game_download 以处理其他分发地址；不将完整 steam 组直连。现有 gfw、商店、社区网页、独立视频及共享静态规则保持原动作和顺序。覆盖 cache7-lax2 及该后缀下未来缓存节点；共享该域的直播也直连，这项代价已获用户接受。其他下载域的未来覆盖不作保证。

这一条是本地单服务域名例外，当前 AGENTS.md 禁止。用户允许牺牲直播不自动等于批准规则例外；在实际修改配置前须明确审阅并批准这条例外。获批后只修改 Desktop/Docker 六份配置的游戏下载层，并同步记录限定例外，不扩大唯一来源许可或允许任意服务规则。域名后缀是路由条件，不引入第二套下载规则数据或本地 payload。

证据：meta 分支 steam.list 有 `+.steamcontent.com`，而下载组仅枚举缓存节点；gfw.list 对商店和社区分别有 `+.store.steampowered.com`、`+.steamcommunity.com`。完整 steam.list 同时包含 steamvideo、steamusercontent、steamstatic 等业务，故整组 DIRECT 会超出直播让步，拒绝采用。

验证：在实际核心重测八个 lax2 节点及未枚举的后缀测试主机，要求命中新后缀规则 DIRECT；商店、社区、独立视频的首匹配和代理出口应保持；shared.fastly 仍排除修改。刷新及重新下载 provider 后重测，证明不依赖缓存补丁。域名缺失或无法识别的连接不能保证命中，须披露运行时限制。

以下上游提案降为非必需备选，不能作为本项目修复完成条件。

## B 的非必需上游备选：补齐已验证的 lax2 缓存主机

这是修复已知下载节点覆盖缺口的最小方案，不是保证所有未来节点或将同一节点的下载与直播按用途分开的方案。域名规则对同一主机的下载和直播连接不能区分；若用户要求连共享缓存上的直播也必须代理，本方案不足，不能宣称满足该要求。商店、社区网页、独立视频/直播域名不因本补丁新增直连覆盖；实际全规则首匹配仍须验证。

## 源数据修改形态

目标为 v2fly `data/category-game-platforms-download` 的 Valve 缓存枚举段，补充本机 Steam 日志实际出现的八个主机，保持现有 full: 精确匹配风格，不新增 steamcontent 后缀，不修改共享静态或整个 steam 分类：

```text
full:cache1-lax2.steamcontent.com
full:cache3-lax2.steamcontent.com
full:cache5-lax2.steamcontent.com
full:cache7-lax2.steamcontent.com
full:cache9-lax2.steamcontent.com
full:cache11-lax2.steamcontent.com
full:cache13-lax2.steamcontent.com
full:cache15-lax2.steamcontent.com
```

这是对外部维护源的补丁提案形态，不是在本项目添加单服务规则或复制外部规则。证据材料只附 depot 更新、节点及必要统计，去掉代理节点、个人路径和无关日志。准备或发布上游 Issue/PR 是分开的动作，发布必须取得用户明确授权；不在本机项目外直接修改其他工作区。

## 生效链路与持久性

1. 上游源分类接受修正，MetaCubeX 构建拉取该修正。
2. 检查实际 MetaCubeX 固定 commit 的下载 MRS，确认八个主机已包含且无未经评估的分类扩张。不能仅凭 PR 合并或 list 内容认为 MRS 已修好。
3. 本项目所有规则订阅已按 A 改用 meta 发布分支后，由正常刷新获取修正。不得再锁定 game_download 或其他 provider；记录实际刷新结果与覆盖证据。
4. 保持 game_download 位于 gfw 后、通用 CDN 前及 DIRECT 动作。不增加 provider，不引入第二直接来源。
5. 客户端重新加载新配置、刷新 provider 并重建下载连接后验证。清除隔离环境中的旧缓存并重新从该固定 URL 获取，再重复路由检查，证明修复不依赖手改缓存。

## 预计修改文件（Files likely to change）

- 上游提案：v2fly/domain-list-community 的 `data/category-game-platforms-download`；MetaCubeX 生成产物由其构建生成，不手改生成分支。上游合并及发布时间不由本项目控制。
- 本项目 12 份配置：四个平台各自的个人配置、single_template、dual_template，全部 rule-provider 改用 meta 发布分支；Steam 覆盖由该分支持续更新获取。
- 本项目 README.md、rule-design.md、validation.md 及上述 Feature 设计文件：准确记录唯一直接来源、meta 持续发布分支、有效刷新周期、八个节点的已知覆盖及验证限制。
- 当前 bug 目录内 fix/test 阶段的正式记录与隔离验证材料；实施前保存被忽略文件完整基线，不记录个人订阅。

## 备选与拒绝理由

- 只升级到当前最新 MetaCubeX commit：查证仍无 lax2，不能修复。
- 编辑本地 game-download.mrs：缓存刷新和重新下载不保证保留，不能修复源。
- steam 在 gfw 后整组 DIRECT：会抢走不命中 gfw 的视频、直播、用户内容，范围不合要求。
- steamcontent 后缀加入分类：上游明确标注缓存混合用途，未经用途分离证据不能采用。撤回先前会话对此宽后缀的推荐。
- 下载组与 steam 的交集：cache7-lax2 不在下载组，交集仍漏；并集/steam 减 gfw 又会包含混合用途，不解决业务边界。
- steam@cn：不能覆盖 lax2 地区节点。
- 本地单服务规则、其他来源或进程整体 DIRECT：违反当前约束或混淆业务，不实施。

上游是否接受提案不再阻断首选本地方案。若用户不批准限定域名例外，不能擅自实施；继续报告当前约束下的缺口，不采用缓存补丁或第二来源。

# 后续验证与风险

- 采用资源须验证实际 MRS/文本加载及核心首匹配，而非仅检查 list。
- 八个已列出的 lax2 节点应在实际核心中命中 game_download → DIRECT；未列出的未来节点不属于此最小补丁的覆盖承诺。
- 以 store.steampowered.com、steamcommunity.com、steamvideo-a.akamaihd.net、steambroadcast.akamaized.net、steamusercontent.com 为反例，要求新版本不新增 DIRECT 首匹配。若当前基线反例已直连，先记录既有行为，不把它伪装成补丁满足代理要求的证据。
- 同一 lax2 主机上若实际承载直播，本补丁也会使该连接直连，必须列入风险和用户审阅事项；域名级验证不能证明内容用途分离。
- 用户明确暂不处理 shared.fastly.steamstatic.com，其用途由用户后续观察，不作为本次修复或验收目标。
- 检查适用配置的解析、provider 引用、来源、交叠和顺序；被忽略文件修改前保存完整基线。
- 工作区测试不能证明已导入客户端生效；真实下载路由和速度需单独验证。
- shared.fastly 暂无 depot 用途证据，保留现有代理。

# 用户确认后的实施契约（优先于上文历史候选）

用户接受聊天、部分用户内容、短链等剩余 Steam 业务直连，并认可商店、社区网页和视频继续代理。采用完整 Steam 组放在现有域名/IP 规则之后、MATCH 之前 DIRECT；仅 Desktop/Docker 六份有游戏下载层的配置新增 steam provider 和引用，Android/iOS 不扩展游戏下载行为。全部 12 份配置的现有 rule-provider URL 改用 meta，继续每日刷新。不得增加单域名/单主机规则，不改第二来源，不发布上游提案，不修改活动客户端配置。

验证要求更新：八个 lax2 节点应命中 steam→DIRECT；store.steampowered.com、steamcommunity.com、steamvideo-a.akamaihd.net、shared.fastly.steamstatic.com 保持代理；steam-chat.com、s.team、steamusercontent.com 等剩余业务允许直连。域名缺失、IP 先匹配及未覆盖的视频宿主仍须分别说明，不能保证所有用途只凭集合名分离。新增 steam 使用原生 MRS，路径独立；全部 12 份保持原资源路径/格式/周期，核查实际下载加载与主动刷新。修复记录应明确本契约取代上文单域名、上游提案和旧反例验收要求。

方案已获用户在本会话确认，进入 fix，无需重复询问；记录变更与验证边界。
