# Epic 下载路由诊断

- **Slug**: epic-download-routing
- **Created**: 2026-10-03T11:13:37+08:00
- **Source**: 用户粘贴的运行时连接信息与问题描述
- **诊断结论**: valid（原诊断）
- **Severity**: high（原始用户影响）
- **状态**: 已关闭；后续由 `meta-rules-unification` Assessment 承接更广泛的规则来源议题
- **修复状态**: 未实施
- **验证状态**: 未运行；不能视为已修复

用户反馈 Desktop 上 Epic 游戏下载仍经代理，并提供连接：`egdownload.fastly-edge.com:443`，下载 193 MB，链路 `ROUTE_GLOBAL / AUTO_GENERAL / JMS-1423733@c2YOUR_HEAVY_KEYWORD.portablesubmarines.com:443`，命中规则 `RuleSet(proxy)`。用户要求从上游规则集和规则顺序中找通用解决办法，不接受为单个服务加规则。

预期是 Epic 游戏下载命中现有上游游戏下载/大文件下载规则并直连。当前连接却被代理规则截获，产生了实际的大流量代理消耗。

# 复现与证据

1. 在 Desktop 运行配置中查看 `rules:`。
2. `game_download,DIRECT` 位于规则前部，但该上游规则当前不含 `egdownload.fastly-edge.com`。
3. `RULE-SET,proxy,ROUTE_GLOBAL` 位于 `sukka_download_domain,DIRECT` 之前。
4. Mihomo 自上而下使用首个匹配规则；用户日志中的 `RuleSet(proxy)` 与此顺序一致，连接因此到不了后面的下载直连规则。

上游核对：

- Blackmatrix7 当前 `GameDownload.yaml` 包含 `download.epicgames.com`、`download2/3/4.epicgames.com`、`fastly-download.epicgames.com` 等域名，但不含本次主机名；文件头标注更新时间为 2025-06-06。
- Sukka `Source/domainset/game-download.conf` 明确包含 `.egdownload.fastly-edge.com`。Sukka 的生成脚本将该游戏域名集并入 `domainset/download` 输出，因此现有配置订阅的 `https://ruleset.skk.moe/Clash/domainset/download.txt` 应覆盖本次主机名。
- Sukka 对 Mihomo 的规则集格式说明将 `/Clash/domainset/` 定义为 `behavior: domain`、`format: text`；当前 provider 声明与此相符。

# 涉及路径

- `mihomo_config_desktop.yaml:422-446` — 当前游戏下载规则、GFW、`proxy` 与 Sukka 下载规则的顺序。
- `mihomo_config_desktop_single_template.yaml:319-343` — 单订阅 Desktop 模板存在相同顺序。
- `mihomo_config_desktop_dual_template.yaml:348-373` — 双订阅 Desktop 模板存在相同顺序。
- `mihomo_config_docker.yaml:340-365` — Docker 的相同规则排列也受同类影响。
- `README.md:99-103` — 当前下载优先级的设计说明没有反映 `proxy` 先于 Sukka 下载组造成的截获。

# 根因判断

**置信度：高。** 本次域名已由 Sukka 上游游戏下载子集覆盖，并通过 Sukka 的下载域名集发布。私有 Desktop 配置和两个 Desktop 模板却把覆盖面更大的 `RULE-SET,proxy,ROUTE_GLOBAL` 放在 Sukka 下载直连组之前。运行日志恰好报告命中 `RuleSet(proxy)`，因此直接根因是规则首匹配顺序，而非缺少单服务规则。Blackmatrix7 的游戏规则作为单独的补充组仍可保留，但不能覆盖这个新域名。

# 建议修复

**首选：调整现有规则组顺序，并沿用上游组。** 保持精确的 `game_download,DIRECT` 在前；保持 `gfw` 与 `GEOSITE,gfw` 在通用下载直连之前，避免被墙的软件镜像被误直连；然后把 `sukka_download_domain,DIRECT` 和 `sukka_download_non_ip,DIRECT` 移到 `proxy,ROUTE_GLOBAL` 之前。其后再执行通用 `proxy` 和 CDN 规则。这样 Sukka 上游新增的游戏下载域名可以命中直连，仍由更前面的 GFW 组处理已知受阻站点，且无需维护某个服务的单独域名规则。

Blackmatrix7 `GameDownload` 适合作为语义清晰、范围较窄的游戏下载优先组；Sukka `download` 覆盖更广的软件安装包和大文件下载，并在生成时并入 Sukka 的 `game-download` 数据。二者职责有重叠，可以保留当前 Blackmatrix7 组作为先行规则，再使用 Sukka 下载组补充更广覆盖。不要把 Sukka 下载组放到 GFW 之前，因为那会削弱当前为被墙下载源设计的保护。

**备选：** 仅把 `sukka_download_domain` 提到 `proxy` 前面而保留 `sukka_download_non_ip` 原位置，改动更小；但两个同属下载业务的组顺序会分散，不如整体移动两个 Sukka 下载组清楚。

**可能涉及文件：** 私有 Desktop 配置、Desktop 单订阅/双订阅模板，以及 Docker 配置中相同规则段；README 和架构矩阵也应同步说明规则优先级。若只要求修复当前 Desktop，平台范围可以限于 Desktop 三份文件。

**验证建议：** 更新规则提供方后，在 Mihomo 的连接详情确认该主机名命中 `sukka_download_domain`（或其编译后的 RuleSet）并选择 `DIRECT`；同时确认 GFW 下载站仍命中前置代理规则。当前诊断阶段未运行配置或连接验证。

# 风险与待确认

- Sukka `download` 是通用软件与大文件域名集，移动到 `proxy` 前会使其中所有域名直连；这与其现有 DIRECT 设计一致，但比 Epic 专属规则覆盖更广。
- 规则 provider 每日更新一次；配置载入时还需确保成功拉取并激活上游规则文件。当前日志显示命中 `proxy`，而上游明确覆盖该域名，所以顺序仍是已观测问题的首要修复点。
- 若调整后运行日志仍命中 `proxy`，需检查运行实例是否确实加载了修改后的配置、`sukka_download_domain` 是否更新成功，以及 RuleSet 是否被其他覆写规则替代。
