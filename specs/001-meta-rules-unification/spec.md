**Feature Branch**: 未创建

**Created**: 2026-10-03

**Status**: Ready for implementation

**Input**: 用户要求从头重写 Mihomo 分流配置，不迁移旧配置。旧配置只作参考，新配置以 MetaCubeX `meta-rules-dat` 的资源为准重新裁定每类流量该走哪个出口。

# 问题

旧配置是围绕当时上游规则集的形状写出来的，逐项迁移没有意义。真正要回答的是：**每一类流量应该走哪个出口，以及应该用哪个上游资源表达它。**

# 目标

在仓库中新建独立目录，交付 12 份配置（4 平台 × single/dual 公开模板 8 份 + 本地个人配置 4 份），全部按同一条判定链从零设计。

# 分流判定链

流量按以下顺序判定，首匹配生效：

1. **应当拦截？** 是 → `REJECT`
2. **需翻墙？** 否 → `DIRECT`。**不应翻墙的绝不翻墙。**
3. **需翻墙且交互型？** → `ROUTE_NORMAL`，fallback 首选 `AUTO_STRICT`
4. **需翻墙且大流量？** → `ROUTE_HEAVY`，fallback 首选 `AUTO_GENERAL`（YOUR_HEAVY_KEYWORD 大流量节点）
5. **未命中任何规则** → `PROXY` 兜底

# 同质性的两条不同门槛

- **DIRECT ↔ ROUTE_\*** 搞错是硬故障（服务不通，或不该翻墙的被翻墙）。成员必须几乎全部落在同一侧才可用。
- **ROUTE_NORMAL ↔ ROUTE_HEAVY** 搞错是软故障（节点池不理想，但照样能通）。按**宿主的内容类型**判断，不按单次请求大小判断。

举例：CDN 边缘宿主的存在目的就是扛静态资源与媒体，整类算大流量；AI 对话宿主主要承载文字交互，整类算交互型。

# 资源选择原则

## 原则性

在**成员同质**的前提下，取覆盖最全的规则集。不因服务知名、名称熟悉而默认采用细分资源。

同质的含义：该集合的成员必须落在判定链的**同一个出口**。一个集合里既有需翻墙又有不需翻墙的成员，就不能用它绑单一动作。

## 何时引入细分资源

只有出现以下三类误分流，且调整顺序无法解决或会引入新问题时，才引入更细的资源：

1. 不该翻墙的服务被分流到翻墙；
2. 非大流量服务被分流到 `ROUTE_HEAVY`；
3. 大流量服务被分流到 `ROUTE_NORMAL`。

典型情形是大集合内部混了分流原则不同的细分场景。此时引入更细的子集，再用顺序解决。

## 重合集合的取舍

两个集合不需要 100% 包含或重合。接近包含、重合时取边界更干净的一个；除非某个子集确实需要优先套用不同动作，才让它排在前面。

## 资源不限于 rule-set

"规则集"只是方便说法。应在 `meta-rules-dat` 的所有资源形态里找最合适者：GeoSite、GeoIP、分类资源、文本/二进制规则、数据库类别等，按覆盖范围、匹配语义、成员混合程度、维护方式选择。

# 已确认的设计决策

1. **需翻墙的 media 合并，整体走大流量。** 用 `category-media`（海外 media，与 `geolocation-cn` 零交集）作合并目标，不再拆成 youtube / netflix 等单集合。新闻站点与视频分片在同一集合内交织，调序拆不开，按宿主内容类型整体归入大流量，接受新闻页走 YOUR_HEAVY_KEYWORD 的软故障。

2. **需翻墙的图片视频 CDN 合并，走大流量。** 域名层 + IP 层同时加。IP 层用 `geoip.cloudflare` / `geoip.cloudfront` / `geoip.fastly`，覆盖「海外站点把图片视频放在 CloudFlare」这种域名规则抓不到的情形；域名层用 `category-cdn-!cn` / `akamai` / `fastly`，因为在 fake-ip 模式下带 `no-resolve` 的 IP 规则不对域名连接生效。

3. **Twitter 图片视频不需要专属规则。** `category-social-media-!cn` 已含 `+.twimg.com`、`+.twvid.com`、`+.t.co`，同时含 Facebook、VK、LinkedIn 的图床边缘主机。不为单一服务加规则。

4. **AI 会话必须避开大流量。** `category-ai-chat-!cn` → `ROUTE_NORMAL`，且排在所有大流量层之前。

5. **国内可用的系统服务直连。** `apple` 与 `microsoft` 未被 `geolocation-cn` 覆盖（均 0 交集），需单独引入为 `DIRECT`。`apple-update` 的 21/21 条全在 `apple` 内，无需单列；**`win-update` 不是 `microsoft` 的子集**——`microsoft` 只覆盖其 358/364 条，余 6 条是挂在共享 CDN 边缘域名（`*.akadns.net`、`*.nsatc.net`）下的主机——5 条被 CDN 层的 `+.akadns.net` 命中进 `ROUTE_HEAVY`、1 条落 `MATCH,PROXY`，总之都被翻墙。故 Desktop / Docker 单列 `win_update` → `DIRECT`，移动端不引入。

6. **QUIC 阻断必须加。** `AND,((NETWORK,UDP),(DST-PORT,443))` → `REJECT`，强制浏览器回落 TCP，否则 TUN/嗅探抓不到 TLS SNI，域名规则无法匹配。排在个人例外之后，使 Tailscale 的 UDP 打洞不被误杀。

7. **策略组末位兜底。** 4 份个人配置均不设 `AUTO_1` / `AUTO_2`；改为 `AUTO_BACKUP`（`use: [sub_nodes_backup]`）作为独立逃生池，追加在 `PROXY`、`ROUTE_NORMAL`、`ROUTE_HEAVY` 的最后一位。公开双机场模板保留 `AUTO_1` / `AUTO_2`，供 UI 独立切换作逃生。

8. **公开模板的 AUTO 组不按节点协议类型拆分。** 老模板把 `AUTO_STRICT` 限定为 VLESS/Trojan/Snell/Hysteria2、`AUTO_GENERAL` 限定为 VMess/HTTP/Socks，但协议新旧与「纯净/大流量」没有必然关系。故模板合并为单一 `AUTO` 组；个人配置因有真实订阅的 `YOUR_HEAVY_KEYWORD` 节点命名依据，保留 `AUTO_STRICT` / `AUTO_GENERAL` 分池。

# 仍需覆盖的大流量类目

以下由维护者提出、经核对后确认为大流量且此前遗漏，全部纳入 `ROUTE_HEAVY`：

| 类目 | 资源 | 备注 |
|---|---|---|
| 海外媒体 | `category-media` | 视频分片端点 |
| 社交图片视频 | `category-social-media-!cn` | 含 twimg / twvid / fbcdn |
| 网盘与文件同步 | `category-netdisk-!cn` | dropbox / mega / onedrive |
| CDN 边缘 | `category-cdn-!cn`、`akamai`、`fastly` + 三个 geoip | 静态资源与媒体 |

软件更新（`apple-update`、`win-update`）属大文件下载，但不需翻墙，走 `DIRECT` 层以跑满本地带宽、节省代理流量。

**小集合不采用**：`openstreetmap`(9) / `organicmaps`(2) / `mapbox`(2) 条目太少，覆盖不了实际地图瓦片流量（Google Maps 瓦片在 `google` 内），按「取覆盖最全」原则弃用。

# Google 的取舍

`google` 1075 条中 545 条命中 `gfw`，530 条不命中。这 530 条是广告/统计基建（`adwords`、`adsense`、`admob`、`app-analytics-services`）、`blogspot.*`、开发工具（`bazel.build`、`angulardart.org`）与 `+.android` TLD，与 `category-media` 零交集、与 `category-cdn-!cn` 仅 1 条交集、与 `geolocation-cn` 零交集——没有任何前置规则能接住它们。

因此 `google → ROUTE_NORMAL` 不是冗余：没有它，这 530 条会落到 `MATCH,PROXY`，路由结果取决于 `PROXY` 的手动选择；有了它，全 Google 确定性走干净节点。内容类型以交互为主，保留 `ROUTE_NORMAL`。

已知软故障：`+.2mdn.net`（Google 广告媒体 CDN）与 Google Photos / Drive / 地图瓦片会走 `AUTO_STRICT` 而非 YOUR_HEAVY_KEYWORD。

# 策略组命名

早期配置把节点按适用场景分成三组：**纯净 / 普通 / 大流量**。`ROUTE_AI` 最初确实只针对 AI 业务，`ROUTE_GLOBAL` 表达的是走全球节点。后来各组按用途合并，两者的实际语义已经和名字脱节：

- `ROUTE_AI` 现在承接的是**所有交互型、非大流量**的翻墙流量（AI 会话、Google 检索与文档、博客、开发工具等），不再限于 AI。
- `ROUTE_GLOBAL` 现在承接的是**大流量**（视频、CDN、网盘），和「全球」无关。

因此更名为：

| 旧名 | 新名 | 中文 | 语义 |
|---|---|---|---|
| `ROUTE_AI` | `ROUTE_NORMAL` | 普通流量 | 交互型、非大流量，首选 `AUTO_STRICT` |
| `ROUTE_GLOBAL` | `ROUTE_HEAVY` | 大流量 | 持续大载荷，首选 `AUTO_GENERAL`（YOUR_HEAVY_KEYWORD） |

命名直接编码判定链的分类轴（是不是大流量），并延续历史词汇「普通 / 大流量」，与 `AUTO_STRICT`(纯净节点) / `AUTO_GENERAL`(大流量节点) 的分工不撞名。

节点组名不变：`AUTO_STRICT` / `AUTO_GENERAL` / `AUTO_BACKUP`。

# 交付范围

## 12 份配置

| 平台 | Single 模板 | Dual 模板 | 个人配置 |
|---|---|---|---|
| Desktop | ✅ | ✅ | ✅ |
| Docker | ✅ | ✅ | ✅ |
| iOS | ✅ | ✅ | ✅ |
| Android | ✅ | ✅ | ✅ |

公开模板使用订阅占位符；个人配置保存真实订阅，按 `.gitignore` 忽略。

## 三个个人例外

从旧配置中提取并保留在本地个人配置：Tailscale、UU 远程、绿联 NAS。它们是本地远程访问/穿透用途，动作是 `DIRECT`，属于「不翻墙」层，不进公开模板，也不影响通用规则选择。

## 平台差异

参考旧配置确定四平台差异，仅限平台约束，不改路由哲学：

- Desktop：TUN 开启，`stack: mixed`（TCP system / UDP gvisor），仅监听本机
- Docker：TUN 关闭，允许 LAN 访问，控制器暴露给容器网络，Sniffer 开启
- iOS：TUN 开启，`stack: gvisor`，节点过滤使用负向先行断言以兼容 iOS 正则
- Android：TUN 开启，`stack: gvisor`

## 模板简化

公开模板只保留：平台设置、订阅 provider、策略组、规则资源、兜底。不保留旧配置的历史积累。

# 验收

1. 12 份配置均由 Mihomo 接受。
2. 每条被引用的资源都有声明，无悬空引用。
3. 业务规则只指向 `REJECT` / `DIRECT` / `ROUTE_NORMAL` / `ROUTE_HEAVY`，不直连 `AUTO_*`。
4. 候选链按配置类型区分，均只引用各自文件中实际定义的组：
   - **4 份个人配置**：`ROUTE_NORMAL` 候选 `AUTO_STRICT → AUTO_GENERAL → AUTO_BACKUP`，`ROUTE_HEAVY` 为 `AUTO_GENERAL → AUTO_STRICT → AUTO_BACKUP`，`AUTO_BACKUP` 恒在末位。
   - **8 份公开模板**：**不设 ROUTE 层**。只有一个 `AUTO` 节点池，业务规则直接指向它，`PROXY` 作手动逃生口兼最终兜底。模板只有一个池，普通流量与大流量落到同一组，再包 `ROUTE_NORMAL` / `ROUTE_HEAVY` 只是纯转发；模板用占位订阅也无法预设 `YOUR_HEAVY_KEYWORD` 节点命名。老模板按节点协议类型拆分属想当然，已去除（见决策 8）。若日后拆出两个池，再把规则改指 `ROUTE_*`。
5. `MATCH,PROXY` 是每份配置的最后一条。
6. 公开模板不含个人订阅、节点、私有策略组或内网字段；个人配置及个人归档均被 Git 忽略。
7. 每个进入配置的资源都有同质性判定依据，包括被弃用资源的理由。

# 非目标

- 不迁移旧规则，不做旧 provider 到新 provider 的映射。
- 不按服务品牌设验收项、分类或优先级。
- 不为单一服务新增单条域名/IP/进程规则。
- 不做目标客户端/设备的运行时验证（订阅、UI、内存实测另行安排）。
