# 基本信息

- **Slug**: fix-rules-order-gfw-ai
- **Fixed**: 2026-09-26T21:53:00+08:00
- **Assessment**: [assessment.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/.specify/bugs/fix-rules-order-gfw-ai/assessment.md)
- **Status**: applied

# 修复概要

在用户正式批准的修复方案指导下，Agent 完成了全系 12 份配置文件的标准化重构：彻底清除了定义混乱且存在直连与代理冲突的 `system_ota` 规则集与路由规则，彻底删除了 `ROUTE_SPEED` 业务策略组与 `AUTO_SPEED` 原子测速组，将原 speed 对应的高速/低延迟协议节点（Shadowsocks、Hysteria2、TUIC、WireGuard 等）全面并入 `AUTO_STRICT`，并彻底清除了全系所有 Telegram 独立规则；高敏感业务（AIGC、Google、TMDB）置于通用 GFW 规则与大文件下载之前，彻底消除了 GFW 规则抢跑拦截 AI 流量的问题，全系统一收敛为严格同构的七层白名单流水线。

# 变更清单

| 文件 | 变更类型 | 说明 |
|------|----------|------|
| [mihomo_config_desktop_single_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_desktop_single_template.yaml) | modified | 移除 system_ota、speed 相关组与 telegram 规则，speed 节点并入 AUTO_STRICT，重构规则顺位为标准七层 |
| [mihomo_config_desktop_dual_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_desktop_dual_template.yaml) | modified | 同步重构双机场桌面模板，移除 system_ota、speed 组与 telegram 规则，规整分层 |
| [mihomo_config_desktop.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_desktop.yaml) | modified | 私有桌面工作配置重构，严格保护 Tailscale、UU远程、绿联NAS 等自用进程与域名规则完整无损 |
| [mihomo_config_docker_single_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_docker_single_template.yaml) | modified | 移除 system_ota、speed 组与 telegram 规则，重构规则分层并校正国内域名优先于 IP |
| [mihomo_config_docker_dual_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_docker_dual_template.yaml) | modified | 同步双机场 Docker 模板分层重构与策略组收敛 |
| [mihomo_config_docker.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_docker.yaml) | modified | 同步私有 Docker 配置的分层重构与策略组收敛 |
| [mihomo_config_android_single_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_android_single_template.yaml) | modified | 移除 system_ota、speed 组与 telegram 规则，校正国内域名优先于 IP |
| [mihomo_config_android_dual_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_android_dual_template.yaml) | modified | 同步双机场 Android 模板分层重构与策略组收敛 |
| [mihomo_config_android.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_android.yaml) | modified | 同步私有 Android 配置的分层重构与策略组收敛 |
| [mihomo_config_ios_single_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_ios_single_template.yaml) | modified | 移除 system_ota、speed 组与 telegram 规则，校正国内域名优先于 IP |
| [mihomo_config_ios_dual_template.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_ios_dual_template.yaml) | modified | 同步双机场 iOS 模板分层重构与策略组收敛 |
| [mihomo_config_ios.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_ios.yaml) | modified | 同步私有 iOS 配置的分层重构与策略组收敛 |
| [.specify/memory/constitution.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/.specify/memory/constitution.md) | modified | 架构宪章更新至 1.3.0，剔除 system_ota、speed 组与 telegram 规则条目 |
| [LOCAL_ARCH_DESIGN.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/LOCAL_ARCH_DESIGN.md) | modified | 更新本地设计规范，规则矩阵中移除 system_ota、speed 组与 telegram，底层收敛为 STRICT + GENERAL |
| [README.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/README.md) | modified | 更新项目说明文档，移除 system_ota、speed 与 telegram 描述，反映纯净同构架构 |

# 关键改动说明

## 1. 彻底剔除 `system_ota`

- 从 `rule-providers:` 中完全移除了 `system_ota` 外部订阅。
- 从 `rules:` 中完全移除了 `RULE-SET,system_ota,DIRECT`。苹果系统固件由基础直连层的 `apple` 规则放行，国内厂商固件由 `GEOSITE,CN` 放行，海外原生机型（如 Pixel）随 GFW 与代理自适应，消除断更风险与规则冲突。

## 2. 原 speed 节点并入 strict，废除 speed 策略组

- 全系 12 份配置彻底删除 `ROUTE_SPEED` 与 `AUTO_SPEED`。
- `AUTO_STRICT` 的 `exclude-type` 精简为：
  ```yaml
  exclude-type: "Vmess|Http|Socks|Socks5"
  ```
  囊括所有现代高效强对抗协议（VLESS, Trojan, Snell, Shadowsocks, Hysteria2, TUIC, WireGuard 等）。
- 顶层 `PROXY` 及高层 `ROUTE_AI`、`ROUTE_GLOBAL` 的 proxies 列表中同步移除 `AUTO_SPEED`。

## 3. 彻底移除 Telegram 独立规则

- 从 `rule-providers:` 中删除 `telegramcidr`，从 `rules:` 中删除全部 `telegram` 规则，流量由常规出海通道自然承载。

## 4. 全平台统一七层同构顺位

```yaml
# 1. 基础防护与直连区 (QUIC拦截、私网、大厂直连)
# 2. 国内服务直连区 (域名规则严格优于 IP-CIDR)
  - RULE-SET,direct,DIRECT
  - GEOSITE,CN,DIRECT
  - RULE-SET,cncidr,DIRECT
  - GEOIP,CN,DIRECT

# 3. 平台特化定制区 (如 Desktop 的私有进程、游戏下载；移动端严格精简)
# 4. 高敏感业务代理区 (AIGC、Google、TMDB 统一走 ROUTE_AI / AUTO_STRICT)
  - RULE-SET,sukka_ai,ROUTE_AI
  - RULE-SET,google,ROUTE_AI
  - RULE-SET,tmdb,ROUTE_AI

# 5. 国际多媒体区 (流媒体走 ROUTE_GLOBAL)
  - RULE-SET,global_media,ROUTE_GLOBAL

# 6. GFW 代理区 (位于特定业务后，大文件下载前)
  - RULE-SET,gfw,ROUTE_GLOBAL
  - GEOSITE,gfw,ROUTE_GLOBAL
  - RULE-SET,proxy,ROUTE_GLOBAL

# 7. 兜底直连与代理 (大文件下载直连、通用 CDN 直连、兜底 MATCH)
  - RULE-SET,sukka_download_domain,DIRECT
  - RULE-SET,sukka_download_non_ip,DIRECT
  - RULE-SET,cloudflare_cidr,DIRECT
  - MATCH,ROUTE_GLOBAL
```

# 本地验证

- **自动化逻辑与架构断言检查**：
  - 运行自动化验证脚本 [scratch/verify_final_state.py](file:///c:/Users/jiezhengj/.gemini/antigravity/brain/0b86b487-3522-44f7-adaf-6ea14d4d2908/scratch/verify_final_state.py)：
    - 验证 12 份配置文件通过 `yaml.safe_load` 结构解析。
    - 验证 `system_ota` 在 12 份配置文件中 0 残留。
    - 验证 `telegram` 在 12 份配置文件中 0 残留。
    - 验证 `AUTO_SPEED` 与 `ROUTE_SPEED` 在 12 份配置文件中 0 残留。
    - 验证 `AUTO_STRICT` 的 `exclude-type` 精准对齐 `"Vmess|Http|Socks|Socks5"`。
    - 验证 `sukka_ai` 严格在 `gfw` 之前。
    - 验证 `GEOSITE,CN` 严格在 `cncidr` 之前。
    - 验证私有配置 [mihomo_config_desktop.yaml](file:///c:/Users/jiezhengj/Documents/Project/mihomo/mihomo_config_desktop.yaml) 中 Tailscale、UU远程、绿联NAS 等自定义规则完好无损。
  - 断言结果：全量测试项通过（`ALL 12 CONFIGS AND PRIVATE EXTENSIONS VERIFIED 100% COMPLIANT WITH APPROVED REMEDIATION!`）。

# 偏差说明

无任何偏差。实施动作与用户批准的 [assessment.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/.specify/bugs/fix-rules-order-gfw-ai/assessment.md) 完全一致。

# 后续建议

运行 `/speckit-bug-test slug=fix-rules-order-gfw-ai` 产出正式测试报告并关闭缺陷单。
