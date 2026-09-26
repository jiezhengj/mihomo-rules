# 基本信息

- **Slug**: fix-rules-order-gfw-ai
- **Created**: 2026-09-26T21:20:30+08:00
- **Source**: pasted text
- **Verdict**: valid
- **Severity**: high

# 问题报告

用户反馈：
> 现在有个问题，gfw的顺位太高，所以ai相关的网络服务，都不走ai策略组了。
> 同时你整体检查一下整个顺序，还有哪些问题，向我报告，不要执行修改。

后续架构讨论中用户决策明确：
1. 彻底移除 Telegram 的单独规则（包括外部规则集及内核内置规则），Telegram 无需单独规则配置，随 GFW 列表及兜底策略分流。
2. 彻底移除 `ROUTE_SPEED` 业务策略组与 `AUTO_SPEED` 底层原子测速组；原 speed 对应的高速/低延迟协议节点（Shadowsocks、Hysteria2、TUIC、WireGuard 等）全面进入 `AUTO_STRICT`（仅排除 VMess、HTTP、Socks 等明文或老旧协议）。
3. 彻底移除 `system_ota` 规则集与路由规则：该规则集将必须直连（苹果、国行安卓）与必须翻墙（Google Pixel 等）的域名混杂在一起，绑定 `DIRECT` 存在逻辑冲突；且苹果系统更新与国行安卓更新早已分别被前置的 `apple` 与 `GEOSITE,CN` 规则无损直连，`system_ota` 属于多余且有副作用的冗余规则。

# 缺陷现象

1. 在 Desktop 与 Docker 运行环境下，访问 OpenAI (`chatgpt.com`, `openai.com`)、Claude (`claude.ai`, `anthropic.com`)、Google Gemini (`gemini.google.com`) 等 AIGC 生产力服务，以及 Google 搜索和 TMDB 刮削时，流量未能进入专设的 `ROUTE_AI` 纯净高防降权节点池，而是被全量导流至普通常规节点池 `ROUTE_GLOBAL`。
2. 全系配置中，`ROUTE_SPEED` 与 `AUTO_SPEED` 策略组仅有 Telegram 一项服务使用，且因单独引入规则造成维护与内存冗余。
3. 境内域名在 Fake-IP 模式下存在因 IP 规则倒置导致的额外上游 DNS 查询延迟。
4. `system_ota` 规则集顺位在各平台不一致（移动端在 GFW 前，桌面端在 GFW 后），且因其直连与代理需求混杂导致海外原生安卓机型 OTA 断网。

# 复现与验证

1. 打开 [mihomo_config_desktop_single_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_desktop_single_template.yaml) 或 [mihomo_config_desktop.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_desktop.yaml)，定位至 `rules:`。
2. 观察第 4.2 节：
   ```yaml
   - RULE-SET,gfw,ROUTE_GLOBAL
   - GEOSITE,gfw,ROUTE_GLOBAL
   ```
3. 观察第 6 节：
   ```yaml
   - RULE-SET,sukka_ai,ROUTE_AI
   - RULE-SET,google,ROUTE_AI
   - RULE-SET,tmdb,ROUTE_AI
   ```
4. 模拟请求 `https://chatgpt.com`：
   - 规则自顶向下匹配；
   - 在第 4.2 节匹配到 `RULE-SET,gfw` 或 `GEOSITE,gfw`；
   - 触发分流动作 `ROUTE_GLOBAL`；
   - 匹配链条立即终止，无法触达第 6 节 `RULE-SET,sukka_ai,ROUTE_AI`。

# 涉及代码路径

涉及全系 12 份配置文件中的 `proxy-groups:`、`rule-providers:` 与 `rules:` 配置块：
- [mihomo_config_desktop_single_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_desktop_single_template.yaml)
- [mihomo_config_desktop_dual_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_desktop_dual_template.yaml)
- [mihomo_config_desktop.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_desktop.yaml)
- [mihomo_config_docker_single_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_docker_single_template.yaml)
- [mihomo_config_docker_dual_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_docker_dual_template.yaml)
- [mihomo_config_docker.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_docker.yaml)
- [mihomo_config_android_single_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_android_single_template.yaml)
- [mihomo_config_android_dual_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_android_dual_template.yaml)
- [mihomo_config_android.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_android.yaml)
- [mihomo_config_ios_single_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_ios_single_template.yaml)
- [mihomo_config_ios_dual_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_ios_dual_template.yaml)
- [mihomo_config_ios.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_ios.yaml)
以及文档：
- [LOCAL_ARCH_DESIGN.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/LOCAL_ARCH_DESIGN.md)
- [README.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/README.md)
- [.specify/memory/constitution.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/.specify/memory/constitution.md)

# 根因分析

1. **特异性倒置（General over Specific）**：
   在编写 Desktop 与 Docker 配置时，为了防止第 4.3 节的大文件下载直连规则误杀被墙下载源，在第 4.2 节强插了 `gfw` 规则集强制代理。但 GFW 列表囊括了所有主流 AIGC 与 Google 站点，置于前面导致专属策略组被架空。置信度：高（High）。
2. **策略组过度细分与规则冗余**：
   全系设立了 `ROUTE_SPEED` 策略组，但实际自动分流中只有 Telegram 独享该组；且使用外部 `telegramcidr` 规则文件造成了额外的更新和网络开销。既然 Telegram 需享受高可靠抗封锁节点，直接将其并入 `ROUTE_AI`（strict 协议链路）可大幅精简策略组层级。置信度：高（High）。
3. **DNS 解析前置倒置**：
   国内直连区将 `cncidr`（IP-CIDR）置于 `GEOSITE,CN` 之前，违背了 Fake-IP 模式下“域名规则优先以避免不必要 DNS 解析”的设计原则。置信度：高（High）。
4. **外部规则包定义粗放与业务语义冲突 (`system_ota`)**：
   `system_ota` 外部规则集将不同物理网络可达性的厂商（境内可直连的苹果、国产安卓 vs 境内受阻断必须代理的 Google Pixel）强行揉在一个规则集内。配置一律绑定 `DIRECT`，置于 GFW 前会导致 Pixel 断网，置于 GFW 后会导致大包更新走代理跑流量。且其直连主体已被前置的 `apple` 与 `GEOSITE,CN` 完整覆盖，规则本身存在不可调和的逻辑缺陷，应予剔除。置信度：高（High）。

# 修复方案

**最终修复方案**：

1. **彻底精简策略组结构（废弃 SPEED，节点并入 STRICT）**：
   - 全系 12 份配置文件中彻底删除 `ROUTE_SPEED` 业务策略组与 `AUTO_SPEED` 原子测速组。
   - 原 speed 对应的协议节点（Shadowsocks、Hysteria2、TUIC、WireGuard 等）全面进入 `AUTO_STRICT`，修改其排除列表为 `exclude-type: "Vmess|Http|Socks|Socks5"`。
   - 在顶层 `PROXY` 及高层 `ROUTE_AI`、`ROUTE_GLOBAL` 中移除对 `AUTO_SPEED` 的引用。
   - 核心场景策略组精准收束为两大业务组：`ROUTE_AI`（优先 `AUTO_STRICT`）与 `ROUTE_GLOBAL`（优先 `AUTO_GENERAL`）。

2. **彻底移除 Telegram 规则**：
   - 在 `rule-providers:` 中彻底删除 `telegramcidr` 外部订阅。
   - 在 `rules:` 中彻底删除所有 `telegram` 规则；Telegram 流量随常规 GFW 列表或兜底规则自动走代理。

3. **彻底移除 `system_ota` 规则集与路由规则**：
   - 在 `rule-providers:` 中彻底删除 `system_ota` 外部订阅。
   - 在 `rules:` 中彻底删除 `RULE-SET,system_ota,DIRECT`。系统固件更新由更精确的前置直连规则（`apple` 与 `GEOSITE,CN`）无损保障，消除海外机型断网风险并降低内存开销。

4. **重构统一的七层规则顺位（全系 12 份配置严格同构）**：
   - **第 1 层：核心防御与本地直连**
     - QUIC 阻断 (`REJECT`)
     - 进程与私网直连 (`applications`, `private`, `lancidr`, `GEOIP,lan`)
     - 广告拦截 (`reject`)
     - 基础大厂直连 (`icloud`, `apple`)
   - **第 2 层：国内服务绝对直连（域名严格优于 IP）**
     - `RULE-SET,direct,DIRECT`
     - `GEOSITE,CN,DIRECT`
     - `RULE-SET,cncidr,DIRECT`
     - `GEOIP,CN,DIRECT`
   - **第 3 层：本地专属大带宽直连**
     - 游戏下载切片直连 (`game_download`, `steamstatic.com`) [仅 Desktop/Docker]
   - **第 4 层：高敏感专享代理业务（特异性最高，最高优先级）**
     - AIGC 生产力、敏感服务与影视刮削 (`sukka_ai`, `google`, `tmdb` [tmdb仅Desktop/Docker]) -> `ROUTE_AI`
     - 国际流媒体音视频 (`global_media`) -> `ROUTE_GLOBAL`
   - **第 5 层：通用受限被阻断站点（GFW 代理层）**
     - `RULE-SET,gfw,ROUTE_GLOBAL`
     - `GEOSITE,gfw,ROUTE_GLOBAL`
     - `RULE-SET,proxy,ROUTE_GLOBAL`
     *(由于排在第 4 层之后，绝不抢跑 AI 与流媒体；由于排在第 6 层下载之前，确保被封锁下载源不被直连误杀)*
   - **第 6 层：未被阻断的软件镜像与大文件直连**
     - `sukka_download_domain,DIRECT` [仅 Desktop/Docker]
     - `sukka_download_non_ip,DIRECT` [仅 Desktop/Docker]
   - **第 7 层：常规境外 CDN、泛域名与白名单兜底**
     - `sukka_cdn_domain,ROUTE_GLOBAL`
     - `sukka_cdn_non_ip,ROUTE_GLOBAL`
     - `cloudflare,ROUTE_GLOBAL`
     - `tld-not-cn,PROXY`
     - `MATCH,PROXY`

5. **同步更新架构文档与宪章**：
   - 更新 [LOCAL_ARCH_DESIGN.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/LOCAL_ARCH_DESIGN.md)、[README.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/README.md) 以及 [.specify/memory/constitution.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/.specify/memory/constitution.md) 中关于场景化策略组与规则矩阵的表述，移除 `ROUTE_SPEED`、`AUTO_SPEED`、`system_ota` 与 Telegram 独立规则描述。

# 自动化验证计划

编写独立验证脚本执行全量断言：
1. 断言全系 12 份配置文件均不再包含 `ROUTE_SPEED` 与 `AUTO_SPEED`。
2. 断言全系 12 份配置文件 `AUTO_STRICT` 的 `exclude-type` 精准对齐 `"Vmess|Http|Socks|Socks5"`（原 speed 节点并入 strict）。
3. 断言全系 12 份配置文件 `rule-providers:` 与 `rules:` 彻底清除所有 `telegram` 规则与规则集。
4. 断言全系 12 份配置文件 `rule-providers:` 与 `rules:` 彻底清除所有 `system_ota` 规则与规则集。
5. 断言 `sukka_ai` 规则位于 `gfw` 之前，且目标为 `ROUTE_AI`。
6. 断言 `GEOSITE,CN` 顺位高于 `cncidr`。
7. 在包含 `sukka_download_*` 的配置中，断言 `gfw` 顺位高于 `sukka_download_*`。
8. 断言所有 12 份 YAML 配置格式完全有效。

# 风险与考量

- 4 份私有配置（`mihomo_config_*.yaml`）中的 Tailscale、UU远程、绿联NAS 规则不受影响，保持原样。

# 遗留问题

无。
