- **Slug**: steam-download-proxy（沿用上下文）
- **Fixed**: 2026-10-09T03:27:00+08:00
- **Assessment**: [诊断](assessment.md)
- **Status**: applied

# 变更

全部 12 份根目录配置的 237 个既有 provider 从固定版本改为 meta 发布分支，保持路径、格式和 interval: 86400。Desktop/Docker 六份新增 steam.mrs，全部既有规则之后、MATCH 之前 RULE-SET,steam,DIRECT；共 243 个声明、39 个不同资源。Android/iOS 不新增游戏下载行为。未修改活动客户端。

修改前已保存完整配置基线并完成语义对照；用户随后要求清理备份，2026-10-09 已删除本地副本、下载资源、隔离配置、运行日志和临时探针，个人配置副本未提交。历史归档模板的删除一并提交。设计说明、README 同步持续更新政策和 Steam 剩余业务直连范围。

# 对诊断的说明

以诊断末尾用户确认后的实施契约为准，取代旧单域名、上游修改提案及禁止用户内容直连的反例要求。用户已接受聊天、部分用户内容、短链等直连；没有第二来源或单域名规则。未依赖上游接纳提案。

# 本地检查

通过 gh contents/git blobs 获取 meta 的 39 个完整资源。9 份 Mihomo 配置在隔离副本中替换订阅节点与动作进行 -t 检查，全部 exit 0；3 份 iOS 的 YAML payload 解析通过。这不是原始订阅节点启动验证或 Stash 实测。

隔离运行 Mihomo v1.19.32，实际 MRS、原始规则顺序及动作名称，对 16 个域名发送 CONNECT：8 个 lax2 均 steam→DIRECT；商店、社区、steamvideo、shared.fastly 均 gfw→ROUTE_NORMAL；steambroadcast 为 cdn_nc→ROUTE_HEAVY；聊天、短链、用户内容为 steam→DIRECT。测试策略组只有 DIRECT，验证规则选择而非真实代理成功。测试进程已结束。

# 后续验证

尚未验证客户端原始游戏下载、实际 HTTP provider 定时/主动刷新或 Stash 启动。资源成功获取并加载不能替代客户端持续刷新验证；测试结果应为 partial，勿将本 bug 标记端到端关闭。
