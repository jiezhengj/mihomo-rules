- **Slug**: steam-download-proxy
- **Tested**: 2026-10-09T03:27:00+08:00
- **Assessment**: [诊断](assessment.md)
- **Fix**: [修复记录](fix.md)
- **Result**: partial

# 检查结果

| 检查 | 结果 | 边界 |
|---|---|---|
| 12 份配置 YAML、发布分支与周期 | pass | 243 个声明均使用 meta，86400 秒 |
| 39 个上游资源下载 | pass | gh contents/git blobs；不是核心 HTTP 定时刷新 |
| 9 份隔离 Mihomo -t | pass | 测试副本替换节点和动作，实际加载原生 MRS |
| 3 份 iOS payload 解析 | pass | 未在 Stash 启动 |
| Desktop 隔离实际连接首匹配 | pass | 16 个 CONNECT，原规则顺序、实际 MRS；测试代理组指向 DIRECT |
| Steam 原始游戏下载复现 | not-run | 未将配置导入活动客户端，未触发下载 |
| HTTP provider 主动/定时刷新 | not-run | 不能用 GitHub API 下载代替此项 |

# 核心证据

8 个 cache1/3/5/7/9/11/13/15-lax2.steamcontent.com 均记录 match RuleSet(steam) using DIRECT。

store.steampowered.com、steamcommunity.com、steamvideo-a.akamaihd.net、shared.fastly.steamstatic.com 均 match RuleSet(gfw) using ROUTE_NORMAL；steambroadcast.akamaized.net 为 RuleSet(cdn_nc) using ROUTE_HEAVY。steam-chat.com、s.team、steamusercontent.com 为 RuleSet(steam) using DIRECT。

隔离策略组的实际出口是 DIRECT，因此日志中的 ROUTE_NORMAL[DIRECT] 仅证明代理策略组选择，未证明真实代理链可用。系统 DNS 返回活动客户端 fake-IP，本轮不据此判断真实 CDN IP 归属或各地区 IP 抢先匹配效果。隔离进程退出，未更改活动客户端配置。

# 建议

配置修复已经实施，关键规则选择通过。保留 partial，待导入客户端后重做 Steam 下载并核查 provider 刷新再关闭缺陷。未覆盖的宿主、上游集合变化和不同实际 CDN IP 仍须观察。
