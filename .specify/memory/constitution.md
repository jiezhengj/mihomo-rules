<!--
Sync Impact Report:
- Version change: 1.2.0 -> 1.3.0
- Modified principles:
  - 原则 2: 容灾兜底优先于原子切片 (Disaster Recovery over Atomic Selection) — 彻底废除 Telegram 单独规则定义，Telegram 流量随 GFW 列表及兜底分流；ROUTE_AI 专供 AIGC、Google 等高敏感业务。
  - 原则 3: 真实物理画像与去伪存真裁剪 (Device Profiling & Hardware-Aware Pruning) — 全系彻底剔除 system_ota 外部规则集，消除直连与翻墙冲突，固件更新由 apple 与 GEOSITE,CN 前置无损保障。
  - 原则 4: 协议分层与场景化分流 (Protocol Tiering & Scenario-Based Routing) — 原 speed 对应节点（Shadowsocks、Hysteria2、TUIC 等）全面并入 AUTO_STRICT，彻底移除 AUTO_SPEED 策略组，底层收敛为 STRICT 与 GENERAL 双层测速池。
- Added sections:
  - 无 (None)
- Removed sections:
  - system_ota 规则订阅与引用
- Follow-up TODOs:
  - 无 (None)
-->

# 核心原则

## 一、严格文件边界与同构发布 (Strict File Boundary & Isomorphic Release)

本项目严格区分“公开发布”与“个人自用”两套文件边界，所有代码与配置变动必须恪守以下隔离要求：

1. **开源公开模板文件 (`*_template.yaml`，共 8 份)**：
   - 托管于 Git 版本控制中，面向外部社区用户。
   - 必须保持绝对中立、通用且开箱即用。
   - 订阅链接统一使用占位符 `YOUR_SUBSCRIPTION_LINK_HERE`。
   - 绝对禁止硬编码任何个人私有订阅链接、私有 IP/网段、商业软件专属规则（如 UU 远程、绿联 NAS 等）、特定私有策略组或私有节点过滤关键字（如 `s801`）。
   - 必须保持与私有自用配置在策略组名称、协议分层与容灾逻辑上的完全同构镜像。

2. **个人私有工作配置 (`mihomo_config_*.yaml`，无 `_template` 后缀，共 4 份)**：
   - 仅用于本地设备真实运行，必须被 `.gitignore` 严密忽略。
   - 严禁通过 Git 暂存、提交或推送至远端仓库。

## 二、容灾兜底优先于原子切片 (Disaster Recovery over Atomic Selection)

业务路由规则层（`rules`）严禁直接绑定 `AUTO_XXX` 原子测速切片组，必须统一收束至具备容灾回退（fallback）能力的 `ROUTE_XXX` 策略组：

1. `AUTO_STRICT` 与 `AUTO_GENERAL` 属于底层协议单点测速池（`url-test`），不具备跨协议回退能力；直连原子组在节点失效时将导致直接断网。
2. 业务规则必须指向具备层级 fallback 链条的策略组：
   - `ROUTE_AI`：优先保障纯净与极速协议池（`AUTO_STRICT`），异常时自动降级 fallback，承载 AIGC、Google 等高敏感生产力业务。
   - `ROUTE_GLOBAL`：优先常规/低倍率流量（`AUTO_GENERAL`），异常时自动降级 fallback，承载通用出海与多媒体流量；Telegram 及常规被墙流量由 GFW 规则与全局兜底承载。

## 三、真实物理画像与去伪存真裁剪 (Device Profiling & Hardware-Aware Pruning)

配置设计必须源于真实物理设备场景与第一性原理，坚决剔除臆想与纸面冗余：

1. **全系无用规则彻底剔除**：全平台模板剔除测速规则（`speedtest`）、PT Tracker 拦截规则（`private_tracker`）与系统固件规则（`system_ota`），避免常驻内存与无意义 DNS 浪费；Telegram 无需单独规则组。
2. **移动端（iOS / Android）内存防线**：坚决剔除主机游戏下载（`game_download`、`sukka_download_*`）与影视刮削（`tmdb`），严格守住 iOS Network Extension 15MB 内存红线，杜绝后台崩溃重启。
3. **桌面端（Desktop）性能特化**：采用 mixed 协议栈（TCP 系统原生、UDP gvisor），集成游戏与大文件切片直连。
4. **Docker / NAS 局域网特化**：关闭破坏宿主机网络的 TUN 模式，仅暴露必要端口；必须开启 Sniffer 流量嗅探，杜绝局域网纯 IP 请求误判走代理。

## 四、协议分层与场景化分流 (Protocol Tiering & Scenario-Based Routing)

底层节点必须按协议属性自动归类并池化管理：

1. `AUTO_STRICT`：聚合纯净强对抗与低延迟极速协议节点（VLESS、Trojan、Snell、Shadowsocks、Hysteria2、TUIC、WireGuard 等），排除老旧与明文协议（VMess、HTTP、Socks）。
2. `AUTO_GENERAL`：通用、低倍率或专属特化节点池（VMess 或大流量特化节点）。
3. 业务场景按需匹配最适协议池，实现大流量低倍率消耗、敏感业务高纯净保活。

## 五、开箱即用与单机场一键逃生 (Out-of-the-Box & Emergency Egress)

1. **单机场模板 (`*_single_template.yaml`)**：用户填入 1 个订阅链接即可直接启动，无需理解或修改策略组语法。
2. **双机场模板 (`*_dual_template.yaml`)**：底层跨双机场聚合测速优选；顶层提供独立的 `AUTO_1` 与 `AUTO_2` 策略，支持单机场大规模故障时的 UI 一键逃生切换。

# 平台架构与物理画像约束

模板覆盖 4 类运行环境 × 2 种机场模式，共 8 份标准矩阵：

1. 桌面端：`mihomo_config_desktop_single_template.yaml` / `mihomo_config_desktop_dual_template.yaml`
2. 安卓端：`mihomo_config_android_single_template.yaml` / `mihomo_config_android_dual_template.yaml`
3. 苹果端：`mihomo_config_ios_single_template.yaml` / `mihomo_config_ios_dual_template.yaml`
4. Docker 端：`mihomo_config_docker_single_template.yaml` / `mihomo_config_docker_dual_template.yaml`

iOS 平台节点正则过滤必须保持负向先行断言兼容写法，防止解析异常。

# 开发工作流与配置校验标准

1. **配置语法校验**：任何模板改动必须保证 YAML 语法合法且符合 Mihomo 内核规范。
2. **矩阵一致性检查**：当修改通用路由策略或协议分层逻辑时，必须同步核验并更新全系 8 份模板，严禁产生平台碎片化偏差。
3. **敏感信息拦截**：提交前必须执行敏感信息核查，杜绝私有 token、真实机场 URL、内网 IP 泄露至开源模板。

# 治理规程

1. **宪章效力**：本宪章为本项目最高技术与架构准则，后续所有 Feature 设计、规范制定与代码变更均须符合本宪章。
2. **修订规程**：重大架构变更（原则增删或定义重构）须递增 MAJOR 版本；新增约束或场景扩展递增 MINOR 版本；文字微调与澄清递增 PATCH 版本。
3. **合规审查**：每次 PR 与代码审查必须对照核心原则逐项核对。

**Version**: 1.3.0 | **Ratified**: 2026-09-26 | **Last Amended**: 2026-09-26
