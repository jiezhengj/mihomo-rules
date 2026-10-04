---
description: "001-meta-rules-unification 的实施任务清单"
---

**Feature**：`001-meta-rules-unification`
**输入**：[spec.md](spec.md)、[plan.md](plan.md)、`rule-design.md`
**性质**：从零重写，不是迁移。旧配置只提供个人例外、平台差异、模板简化三类参考。

# 阶段 1：保留旧配置

- [X] T001 建立 `archive/meta-rules-unification-baseline/`，把根目录 12 份历史配置逐字节复制进去；根目录源文件保持不变。
- [X] T002 确认根 `.gitignore` 能忽略新旧目录下的个人配置与个人归档；用 `git check-ignore` 验证，个人文件不出现在 `git status`。

# 阶段 2：资源同质性筛选

这是设计的核心阶段。先筛资源，再写配置。

- [X] T003 从 [MetaCubeX/meta-rules-dat](https://github.com/MetaCubeX/meta-rules-dat) 拉取候选资源目录，按目录逐层枚举（recursive tree 会被截断，`asn/` 就有 9.3 万文件）。列出 `geo/geosite` 1904 个类别与 `geo/geoip` 260 个类别。
- [X] T004 对候选资源做全量后缀语义比对，判定每个集合的成员是否落在同一路由出口。核心问题：是否混了「需翻墙 / 不需翻墙」，是否混了「大流量 / 非大流量」。
- [X] T005 按判定结果取舍资源：同质的前提下取覆盖最全的（如 `category-ads-all` 含 `category-ads` 全部 850 条，取全）；纯子集不单列（如 `category-media-cn`、`category-netdisk-cn` 100% 落在 `geolocation-cn` 内）；确实不同质的整体弃用（如 `category-games` 268 条落在境内集合、`steam` 下载与社区混杂）。**后续修订**：`category-media` 与 `category-ai-chat-!cn` 最初因不同质被弃，后经确认按「宿主内容类型」放宽 `ROUTE_NORMAL`↔`ROUTE_HEAVY` 这条软门槛后重新采用（详见 T020 / T024 与 spec.md 决策 1、4），不再弃用。
- [X] T006 裁定重合集合的顺序：`game_download` 与 `gfw` 的 3 条 Akamai 共享主机、`geolocation-cn` 与 `gfw` 的 2 条受限站点，均由顺序解决，使两类成员各得其所。
- [X] T007 把全部判定依据、弃用理由、顺序反事实写入 `rule-design.md`。抽样不冒充全量，`.mrs` 与 `.list` 的等价边界如实标注。

# 阶段 3：配置实现

- [X] T008 写 Desktop 单机场公开模板，作为参考实现：判定链、策略组、provider 声明、规则顺序一次到位。
- [X] T009 由参考实现派生其余 7 份公开模板：Desktop 双机场、Docker 单/双、iOS 单/双、Android 单/双。双机场含 `AUTO_1` / `AUTO_2` 独立逃生入口；iOS 节点过滤用负向先行断言。
- [X] T010 生成 4 份本地个人配置：复用旧配置的真实订阅 provider 块。个人规则只放**适用**的平台——Tailscale、UU 远程、绿联 NAS 三组规则是桌面/网络存储用途，仅插入 Desktop 个人配置的「不翻墙」层（`geo_cn_ip` 之后、任何翻墙规则之前）共 48 条；Docker / iOS / Android 个人配置不含个人规则。同步保留个人 DNS fake-ip-filter 条目（仅 Desktop）。
- [X] T011 按旧配置落地四平台差异：Desktop `tun.stack: mixed` 且仅监听本机；Docker 关 TUN、开 LAN、Sniffer 开；iOS/Android `stack: gvisor`。平台差异只动平台设置，不改路由哲学。
- [X] T012 简化公开模板：只保留平台设置、订阅 provider、策略组、规则资源、兜底；不留旧配置的历史积累。

# 阶段 4：验证

- [X] T013 用 Mihomo v1.19.32 对 12 份配置逐份执行 `-t -f`，全部退出码 0。
- [X] T014 静态复核：每条 `RULE-SET` 都有声明；业务规则不直连 `AUTO_*`；`ROUTE_NORMAL` 为 `AUTO_STRICT → AUTO_GENERAL`、`ROUTE_HEAVY` 为 `AUTO_GENERAL → AUTO_STRICT`；`MATCH,PROXY` 是最后一条；首条是广告拦截。**后续修订**：候选链按配置类型区分——个人配置追加 `AUTO_BACKUP` 末位、公开模板合并为单一 `AUTO`，最终态见 T019 / T034 与 spec.md 验收 4。
- [X] T015 隐私边界：8 份公开模板不含 Tailscale / UU 远程 / 绿联 NAS 标识；公开模板只用占位订阅，个人配置用真实订阅且被 Git 忽略。
- [X] T016 归档一致性：`archive/meta-rules-unification-baseline/` 的 12 份与根目录历史源文件逐字节一致。
- [X] T017 把验证结果、执行边界、未测事项写入 `validation.md`。
- [X] T018 写 `README.md`：文件矩阵、判定链、平台差异、公开/个人边界、`AUTO_GENERAL` 的 YOUR_HEAVY_KEYWORD 命名适配说明。

# 阶段 5：按已确认决策修订配置

本阶段的决策已在 [spec.md](spec.md) 与 `rule-design.md` 中确认并留痕，配置文件亦已按此修订完成，以下各项均已勾选。

- [X] T019 个人配置去掉 `AUTO_1`/`AUTO_2`，改为 `AUTO_BACKUP`（`use: [sub_nodes_backup]`）追加在 `PROXY`、`ROUTE_NORMAL`、`ROUTE_HEAVY` 末位。**三池互斥**：`AUTO_STRICT`/`AUTO_GENERAL` 只吃 `sub_nodes_primary`（按节点名 `YOUR_HEAVY_KEYWORD` 分池，YOUR_HEAVY_KEYWORD 是 PRIMARY_SUB 下的节点、与 BACKUP_SUB 无关），`AUTO_BACKUP` 只吃 `sub_nodes_backup`。早期曾照搬老配置让前两池同时吃两份订阅，导致 BACKUP_SUB 节点混入、`AUTO_BACKUP` 失去独立性，已订正。
- [X] T020 合并 media：以 `category-media` 替换现有 8 个流媒体单集合，整体走 `ROUTE_HEAVY`。
- [X] T021 补 CDN 层：域名层 `category-cdn-!cn` / `akamai` / `fastly` 与 IP 层 `geoip.cloudflare` / `cloudfront` / `fastly` 全部走 `ROUTE_HEAVY`；置于 `game_download` 之后。
- [X] T022 补社交图片视频：`category-social-media-!cn` 走 `ROUTE_HEAVY`，覆盖 twimg / twvid / fbcdn，不加 Twitter 专属规则。
- [X] T023 补网盘：`category-netdisk-!cn` 走 `ROUTE_HEAVY`。小集合（`openstreetmap`/`organicmaps`/`mapbox`）按「取覆盖最全」原则不采用。
- [X] T024 补 AI 会话层：`category-ai-chat-!cn` 走 `ROUTE_NORMAL`，排在所有大流量层之前。
- [X] T024b 补 `google` → `ROUTE_NORMAL`：1075 条中 530 条不命中 `gfw`，无前置规则可接，不加会落到 `MATCH,PROXY` 取决于手动选择。内容类型以交互为主。
- [X] T025 补国内可用系统服务直连：`apple`、`microsoft` 走 `DIRECT`，排在 `gfw` 之后。`apple-update` 21/21 全在 `apple` 内不单列；`win-update` **不是** `microsoft` 子集（仅覆盖 358/364，余 6 条共享 CDN 边缘主机），Desktop/Docker 单列 `win_update`，移动端不引入。
- [X] T026 策略组更名：`ROUTE_AI` → `ROUTE_NORMAL`（普通流量）、`ROUTE_GLOBAL` → `ROUTE_HEAVY`（大流量）。早期节点按适用场景分纯净/普通/大流量三组，`ROUTE_AI` 原先只针对 AI、`ROUTE_GLOBAL` 表达走全球节点；合并后语义与名字脱节。新名直接编码判定链的分类轴（是不是大流量），并延续历史词汇。文档与 12 份配置已同步替换，`AUTO_*` 未受影响，改名后 12/12 解析通过。
- [X] T027 加 QUIC 阻断：`AND,((NETWORK,UDP),(DST-PORT,443))` → `REJECT`，排在个人例外之后。
- [X] T028 按 rule-design.md 的最终顺序重排 12 份配置的规则段，并重跑解析与结构检查。

# 阶段 6：平台差异与模板简化

老配置分析后的平台裁剪与模板简化，已确认并落地。

- [X] T029 iOS 正则兼容：个人配置的节点过滤与 `YOUR_HEAVY_KEYWORD` 分池在 iOS 上用负向先行断言 `filter: "(?i)^(?!.*…).*$"` 代替 `exclude-filter`，其余平台用 `exclude-filter`。
- [X] T030 订阅节点过滤仅个人配置的 BACKUP_SUB 订阅需要：过滤 `香港|HK|HongKong|Hong Kong|过滤|剩余|套餐`（BACKUP_SUB 有假节点、香港节点不好用）；PRIMARY_SUB 订阅无此问题，不过滤；公开模板用占位订阅，不做此假设。
- [X] T031 移动端场景裁剪：iOS/Android 去 `game_download`（不做 Steam/Epic 下载）；iOS 另去 CDN IP 层 1033 条 CIDR 以守 15 MB 内存约束，Android 保留。`microsoft` 四平台均保留。
- [X] T032 Docker 定位为 NAS 上的局域网代理（供游戏机、电视、手机），非 TUN、`allow-lan: true`、控制器 `0.0.0.0:9090`；保留 `game_download`；无 Desktop 个人例外。
- [X] T033 Docker 开 `sniffer.override-destination: true`：无 TUN 时局域网设备常直连 IP，需从 TLS SNI 反推域名才能命中 `game_download`，否则游戏下载会被误翻墙。其余平台保持 `false`（有 TUN + fake-ip）。DNS 保持原样，不影响 fake-ip。
- [X] T034 模板 AUTO 组简化：不按节点协议类型拆分（老模板按 VLESS/VMess 拆属想当然，协议新旧与纯净/大流量无必然关系），合并为单一 `AUTO` 组。个人配置保留 `AUTO_STRICT` / `AUTO_GENERAL`（按 `YOUR_HEAVY_KEYWORD` 分）+ `AUTO_BACKUP`。
- [X] T035 重跑 12 份解析与结构检查，并把平台差异矩阵、裁剪依据写入 `validation.md` 与 `README.md`。

- [X] T036 两轮独立评审的阻塞项修复：①补 `win_update` → `DIRECT`（Desktop/Docker）——`microsoft` 只覆盖 `win-update` 的 358/364，余 6 条共享 CDN 边缘主机（`*.akadns.net`/`*.nsatc.net`）原会被翻墙（5 条经 `+.akadns.net` 进 CDN 层、1 条落 `MATCH,PROXY`），违反「不应翻墙的绝不翻墙」；②Desktop 补 `external-controller: 127.0.0.1:9090`——原取空值实测不启动 RESTful API，CORS 块失效、面板无法切换策略组；③spec 验收 4 按「个人配置/公开模板」分别表述并补决策 8，plan 与 template-contract 同步；④validation 的「全部通过」「20 个 provider」「20 条规则完全一致」改为逐份实测数字；⑤tasks T005 标注 `category-media`/`category-ai-chat-!cn` 后经放宽软门槛重新采用；⑥contract 旧名替换；⑦rule-design 补 `category-game-platforms-download` 判定行、REJECT 代表场景、`win-update` 缺口推演、google/blogspot 与 CDN 交叠计数更正；⑧删 README/rule-design 全局主标题与重复章节；⑨配置注释去品牌名。

- [X] T037 第三、四轮独立评审修复：①交叠计数改为双向并列（资源侧 / gfw 侧），判据句订正为看 gfw 侧——原「正向计数是判据」推导错误，DIRECT 前置误伤的是 gfw 侧被 DIRECT 集覆盖的成员；README/plan/rule-design 三处散文的「1/15/2」「2/1/15」混合方向统一为 gfw 侧 1/17/2；②`microsoft` 计数统一为去重口径并披露上游 `.list` 重复行（apple 1792/1790、microsoft 748/746、game-download 492/491、entertainment 2129/2127）；③spec 判定链第 2 步「不应翻墙？否 → DIRECT」歧义订正为「需翻墙？否 → DIRECT」；④`win-update` 6 条缺口落点精确为 5 条进 CDN 层、1 条落 `MATCH,PROXY`；⑤README 策略组表被段落截断已修复；⑥rule-design 首节改 H1；⑦T014/T010 措辞对齐；⑧隐私扫描口径与未测事项分类补注；⑨iOS 去 CDN IP 层的分类后果补入已知软故障。

- [X] T038 iOS 按 Stash 适配（目标客户端不是 Mihomo）：①三份 iOS 配置的规则集由 `format: mrs` 改为 `payload:` YAML、URL 换 `.yaml`——`stash.wiki/rules/rule-set` 规定 Stash 只认 payload YAML，`.mrs` 是 Mihomo 专有编译格式；已核对 `.yaml` 与 `.list` 同数据同计数且无 keyword 条目，21 个 provider 的 `.yaml` 在固定 commit 均存在；②去掉 `sniffer` 与 `tun` 块（Stash 自行识别 TLS/HTTP、由 app 的 VPN profile 管理）；③QUIC 逻辑规则 `AND,(…)` 注释保留（逻辑规则是 Mihomo/Clash Premium 扩展，Stash 支持与否未证实，保守停用）；④`fallback`/`url-test` 保留（Clash 核心组类型）但列为待实测；⑤DNS 不做无法验证的变更，与 Stash 推荐写法的差异已披露。文档同步更新 README / validation / rule-design。

- [X] T039 模板去掉 ROUTE 层。原 T034 让 `ROUTE_NORMAL`/`ROUTE_HEAVY` 均 fallback 到单一 `AUTO`，两者行为完全相同，是纯转发的冗余层。改为：模板只有 `AUTO`（节点池，自动选优）与 `PROXY`（手动逃生口兼最终兜底）两个组，业务规则直接指向 `AUTO`；双机场模板另加 `AUTO_1`/`AUTO_2`。个人配置仍保留 `ROUTE_NORMAL`/`ROUTE_HEAVY`——那里 `AUTO_STRICT`/`AUTO_GENERAL` 是两个真实节点池，两条链首选顺序不同，才有区分意义。字面上这使模板的业务规则直指 `AUTO`，与「不直连 `AUTO_*`」有出入；该规矩的用意是别把业务流量焊死在某个节点池，模板只有一个池，故不构成实质违反。文档已同步（README / validation / rule-design / spec）。

# 交付状态

本 Feature 已完成，进入交付：

- 任务 **39 项全部勾选**，每项均有证据（实测数据、复现命令或文档章节）。
- 12 份配置全部通过 Mihomo v1.19.32（`mihomo-windows-amd64.exe`）`-t -f` 解析。
- 三轮独立无上下文复评收到 13 条阻塞项，全部经独立复现后修复；第四轮文档侧与配置侧均 PASS。
- 实测发现并修复两项设计偏差：`geolocation-cn` 误用为境内域名集合（应为 `cn`，5,385 → 111,224 条）；`AUTO_STRICT`/`AUTO_GENERAL` 误吃两份订阅（应仅 PRIMARY_SUB，与 `AUTO_BACKUP` 互斥）。
- 已知未测事项与已知软故障均记录在 `validation.md`，未将「未测」记为「通过」。
- 一处**已知且无法由顺序解决**的取舍：`gfw` 与三层大流量集合、`cn` 构成三方牵制（`gfw < cn < 大流量 < gfw` 为环），当前采用唯一「零打不开、零境内走代理」的排列，代价是 70 条大流量落 `ROUTE_NORMAL`。推演与三种取舍的代价见 `rule-design.md`，待用户实测后决定是否取另一种。

# 依赖

- `T001 → T002`：先归档，再确认忽略边界。
- `T003 → T004 → T005 → T006 → T007`：资源盘点、同质性判定、取舍、顺序、留痕。
- `T007 → T008`：判定依据确定后才写配置。
- `T008 → T009 → T010 → T011 → T012`：参考实现、矩阵派生、个人例外、平台差异、模板简化。
- `T012 → T013 → T014 … T018`：全部落地后统一验证。

# 未纳入本次

以下事项缺少运行环境或当前意图确认，不计入本次完成范围，也不在配置里臆造：

- 目标客户端/容器的真实启动、UI 独立切换、Docker 管理入口连通性。
- 远程 provider 的在线下载、更新与真实连接匹配。
- iOS Network Extension 15 MB 运行时内存实测。
- 非大流量与大流量翻墙业务的当前代理意图确认（现按 `gfw → ROUTE_NORMAL` 作为需翻墙主体，已知流媒体大流量先切到 `ROUTE_HEAVY`；若后续确认更多大流量场景，再按三类误分流门槛引入细分资源）。
