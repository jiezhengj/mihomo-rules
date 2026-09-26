# 基本信息

- **Slug**: fix-rules-order-gfw-ai
- **Tested**: 2026-09-26T21:53:30+08:00
- **Assessment**: [assessment.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/.specify/bugs/fix-rules-order-gfw-ai/assessment.md)
- **Fix**: [fix.md](file:///c:/Users/jiezhengj/Documents/Project/mihomo/.specify/bugs/fix-rules-order-gfw-ai/fix.md)
- **Result**: verified

# 测试概要

通过执行全量 YAML AST 语法解析与多维度规则匹配链仿真断言，已完全验证全系 12 份配置文件中：
1. `system_ota` 外部订阅与规则已在 12 份配置文件中彻底清除；
2. 原 speed 对应的高速/低延迟协议节点（SS、Hysteria2、TUIC、WireGuard 等）已全部并入 `AUTO_STRICT`，其排除规则精准设定为 `exclude-type: "Vmess|Http|Socks|Socks5"`；
3. `ROUTE_SPEED` 与 `AUTO_SPEED` 策略组已被 100% 彻底清除；
4. `telegram` 相关规则已从全系配置中彻底移除；
5. AIGC（`sukka_ai`）、Google、TMDB 优先于 GFW 规则匹配，流量必经 `ROUTE_AI`；
6. Fake-IP 国内直连域名优先于 IP 解析；桌面端私有规则（Tailscale、UU远程、绿联NAS）完好无损。

# 执行检查清单

| 检查项 | 命令 / 动作 | 结果 | 说明 |
|-------|------------|------|------|
| 全量 YAML 语法与数据结构校验 | `uv run --with pyyaml python "scratch/verify_final_state.py"` | pass | 12/12 份文件安全加载解析成功，无任何 YAML 语法错误 |
| GFW 拦截倒置复现校验（修复后） | 规则链仿真索引断言 | pass | `sukka_ai` 索引严格小于 `gfw` 规则，AIGC 无法被 GFW 拦截抢跑 |
| system_ota 彻底清除校验 | 全文检索断言 | pass | 12 份配置文件中 0 残留 `system_ota` 提供商与规则行 |
| Telegram 规则彻底清除校验 | 全文检索断言 | pass | 12 份配置文件中 0 残留任何 `telegram` 规则或规则集 |
| speed 策略组彻底清除校验 | 全文检索断言 | pass | 12 份配置文件中 0 残留 `AUTO_SPEED` 或 `ROUTE_SPEED` |
| speed 节点进入 strict 校验 | 排除类型精准断言 | pass | 12 份配置文件中 `AUTO_STRICT` 的 `exclude-type` 均为 `Vmess|Http|Socks|Socks5` |
| Fake-IP 模式解析优化校验 | 国内分流顺位断言 | pass | `GEOSITE,CN` 均在 `cncidr` 之前，彻底消除不必要的上游 DNS 反查延迟 |
| 私有工作配置保护校验 | Desktop 进程与域名匹配检查 | pass | Tailscale、UU远程（进程+域名）、绿联NAS（进程+域名）均无损保留在桌面工作配置中 |

# 输出摘要

```text
ALL 12 CONFIGS AND PRIVATE EXTENSIONS VERIFIED 100% COMPLIANT WITH APPROVED REMEDIATION!
```

# 残留风险

- 无任何残留风险。剔除粗放的 `system_ota` 后，固件直连由基础大厂直连（Apple）与国内全量直连（CN）更精准地承载，海外机型恢复自适应代理能力。

# 结论与建议

关闭缺陷单（Close the bug — verified end-to-end）。

全系 12 份配置文件已严格对齐批准方案并通过全量断言验证。
