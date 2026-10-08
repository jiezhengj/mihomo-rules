# Summary

**Date**: 2026-10-04 | **Spec**: [spec.md](spec.md)

从零重写 12 份 Mihomo 配置。不做旧规则迁移；旧配置仅提供个人例外、平台差异、模板简化三类参考。设计核心是对 `meta-rules-dat` 的资源做**同质性筛选**，再按判定链排首匹配顺序。

# Technical Context

**格式**: Mihomo YAML 配置。

**核心**: Mihomo Meta v1.19.32（Windows amd64），用于 `-t -f` 解析验证。

**上游**: 唯一来源为 MetaCubeX `meta-rules-dat`。独立 `geo/geosite`、`geo/geoip` provider 必须订阅 `meta` 持续发布分支并按 `interval: 86400` 刷新，禁止锁死 commit、日期快照或不可更新的 tag。Mihomo 使用适用 MRS，Stash 使用适用 YAML；逐项核查资源路径、格式和刷新加载。

**布局**:

```
archive/meta-rules-unification-baseline/   # 原 12 份历史配置，只读
            # 新 12 份配置 + 3 份说明
```

个人配置与个人归档由根 `.gitignore` 的文件名规则忽略。

# 设计决策

## 1. 同质性是准入判据，但有两条不同门槛

`DIRECT ↔ ROUTE_*` 搞错是硬故障，成员必须几乎全部落在同一侧才可用；`ROUTE_NORMAL ↔ ROUTE_HEAVY` 搞错是软故障，按宿主的内容类型判断，不按单次请求大小判断。CDN 边缘整类算大流量，AI 对话宿主整类算交互型。

## 2. 判定链顺序即设计产物

```
REJECT → DIRECT → ROUTE_NORMAL → ROUTE_HEAVY → MATCH,PROXY
```

「不应翻墙的绝不翻墙」是硬约束，DIRECT 判定必须在任何翻墙判定之前。交互型先于大流量切分，使 AI 会话避开大流量节点。

## 3. 顺序解决重合

- `game_download` 排在 `gfw` 之后、CDN 层之前：491 条里 3 条是 Akamai 共享边缘主机，让 `gfw` 先命中；而 `category-cdn-!cn` 的 `+.fastly-edge.com` 会覆盖回归主机 `egdownload.fastly-edge.com`，故 CDN 层必须在 `game_download` 之后。
- `geolocation-cn` / `apple` / `microsoft` 排在 `gfw` 之后：按 gfw 侧计数分别有 2 / 1 / 17 条与 `gfw` 交叠，让 `gfw` 先命中。
- CDN 层排在 `gfw` 之后：让位给上一条取舍，6 个 CDN 边缘主机（去重后；原始交叠 7 条，`akamaihd.net` 在 `category-cdn-!cn` 与 `akamai` 两集合重复计数）落入 `ROUTE_NORMAL` 属已知软故障。

## 4. 细分是修复不是优化

`category-media` 混了新闻页与视频分片，按宿主内容类型整体归入大流量，接受新闻页走大流量节点的软故障——这是放宽规模门槛后的明确取舍，不是疏漏。`category-social-media-!cn` 已含 twimg / twvid，不为 Twitter 单独加规则。

## 5. CDN 必须域名层 + IP 层同时加

fake-ip 模式下带 `no-resolve` 的 IP 规则不对域名连接生效。IP 层抓 IP 直连与已解析连接，域名层抓 fake-ip 下的域名连接，两者缺一不可。

## 6. 策略组

- 公开模板：`ROUTE_NORMAL` 与 `ROUTE_HEAVY` 均 fallback 到单一 `AUTO` 组（全部节点 url-test）。模板用占位订阅，无法预设节点命名，故不拆节点池；老模板按节点协议类型（VLESS/Trojan vs VMess/HTTP/Socks）拆分属想当然，已去除。
- 个人配置：`ROUTE_NORMAL: AUTO_STRICT → AUTO_GENERAL → AUTO_BACKUP`；`ROUTE_HEAVY: AUTO_GENERAL → AUTO_STRICT → AUTO_BACKUP`。
- `AUTO_GENERAL` = `filter: "(?i)YOUR_HEAVY_KEYWORD"`，大流量节点池（仅个人配置）
- `AUTO_STRICT` = `exclude-filter: "(?i)YOUR_HEAVY_KEYWORD"`，其余节点池（仅个人配置）
- 4 份个人配置均末位追加 `AUTO_BACKUP`（`use: [sub_nodes_backup]`）作独立逃生池，不设 `AUTO_1`/`AUTO_2`；公开双机场模板保留 `AUTO_1`/`AUTO_2` 供 UI 独立逃生

# 平台差异

| 平台 | TUN | stack | LAN | 控制器 | 其它 |
|---|---|---|---|---|---|
| Desktop | 开 | `mixed` | 否 | 本机 + CORS | — |
| Docker | 关 | — | 是 | `0.0.0.0:9090` | Sniffer 开 |
| iOS | 开 | `gvisor` | 否 | — | 节点过滤用负向先行断言 |
| Android | 开 | `gvisor` | 否 | — | — |

# Constitution Check

- 文件边界：公开模板占位订阅、个人配置真实订阅且被忽略 — PASS
- 容灾路由：业务规则指向 `ROUTE_*`，fallback 首选符合要求 — PASS
- 平台裁剪：四平台差异按旧配置落地，iOS 正则兼容 — PASS
- 协议分层与场景路由：动作由流量意图决定，不从旧 provider 动作推导 — PASS
- 单/双机场：每平台两种模式，双机场有独立逃生入口 — PASS
- 配置校验与敏感信息：12 份逐一解析，公开内容扫描 — PASS
- 旧文件归档：原 12 份保留在 `archive/meta-rules-unification-baseline/`，逐字节一致 — PASS

# Complexity Tracking

无宪章偏离。
