# Mihomo (Clash Meta) 精细化白名单路由配置模板

这是一个专为 Mihomo (Clash Meta) 设计的高阶配置文件集合。包含了分别针对桌面端 (Desktop)、Android、iOS 客户端以及 Docker 代理容器的专属**配置模板**。本仓库的设计初衷是提供一套**极度干净、可控且智能**的网络分流体系。

## 设计思路：白名单模式与强制兜底

本配置模板的核心哲学是 **“白名单模式 (White-list)”**：
1. **预设明确直连：** 局域网、国内 IP (基于 GeoIP)、国内域名 (基于 GeoSite)、以及特殊应用直连。
2. **预设明确拦截：** 针对特定的广告、隐私追踪（基于 Reject 规则集），以及特殊的抗干扰需求（如拦截 QUIC 以避免浏览器 UDP 穿透直连）。
3. **未匹配的统一走代理：** 在规则的最末尾设置 `- MATCH,PROXY` 作为兜底。只要不是明确直连的流量，全部送入代理。这避免了日常“漏网之鱼”导致的网络阻断，完美解决各类小众国外网站、冷门客户端无法连通的问题。

---

## 一、致谢：远程规则集的来源

本模板的智能分流高度依赖于开源社区维护的远程规则集（Rule Providers）。向以下项目的维护者表示诚挚的感谢：

*   **[Loyalsoldier/clash-rules](https://github.com/Loyalsoldier/clash-rules)**：提供了绝大多数的基础路由规则，包括：
    *   `reject.txt`: 广告与隐私追踪域名。
    *   `icloud.txt`, `apple.txt`, `google.txt`: 科技巨头的直连与代理规则。
    *   `proxy.txt`, `direct.txt`, `private.txt`: 常见的需代理域名、需直连域名及局域网域名。
    *   `gfw.txt`, `tld-not-cn.txt`: 被墙域名及非中国大陆顶级域名。
    *   `telegramcidr.txt`: Telegram 专属 IP 段。
    *   `cncidr.txt`, `lancidr.txt`: 中国大陆及局域网 IP 段。
    *   `applications.txt`: 经典应用程序特征。
*   **[blackmatrix7/ios_rule_script](https://github.com/blackmatrix7/ios_rule_script)**：
    *   `Cloudflare.yaml`: 提供了 Cloudflare 的精准 IP 规则，用于解决 CDN 流量代理异常的问题。

---

## 二、为什么采用这些分流策略？

不同的远程规则集在我们的模板中被分配了不同的策略（Action），这是基于实际网络体验的深度考量：

*   **`RULE-SET,reject,REJECT`**：将已知的广告、追踪器、恶意域名直接丢弃。这样可以在代理客户端层面实现全设备（甚至全局域网，若是旁路由）的广告屏蔽，节省带宽。
*   **`RULE-SET,cloudflare,cloudflare`（特殊 IP 规则）**：为什么我们需要专门匹配 Cloudflare 的 IP？因为许多网站使用 Cloudflare CDN，配合 HTTP/3 (QUIC) 时，传统的域名嗅探可能会失效，导致流量漏匹配而直连，最终无法访问。通过将其 IP 强制指向专属代理组，确保这些流量稳定穿越。
*   **`GEOIP,CN,DIRECT` / `RULE-SET,cncidr,DIRECT`**：基于 IP 的国内直连。即使一个国内域名没有被收录在直连列表，只要解析出的 IP 在中国大陆境内，一律直连，保证国内应用的速度。
*   **`AND,((NETWORK,UDP),(DST-PORT,443)),REJECT` (QUIC 拦截)**：这是强制兜底策略。浏览器遇到 UDP 443 阻断时，会自动无感降级到 TCP 443 进行 TLS 握手。这一举措彻底根除了因运营商 UDP 劣化或代理 UDP 穿透失败导致的视频缓冲、网页无限加载问题。

---

## 三、为什么 Desktop 配置更复杂？

您可能会发现，`mihomo_config_desktop_template.yaml` 相比移动端和 Docker 版本更加复杂。主要体现在代理组（Proxy Groups）的设计上：

**Desktop 引入了 `AUTO_STRICT`、`AUTO_SPEED`、`AUTO_GENERAL` 这样的分层容灾策略。**

*   **桌面端的网络环境往往更苛刻**：在办公或高强度查阅资料时，我们既需要**极低的延迟**（用于游戏、网页秒开），又需要**极高的稳定性**（不断线）。
*   **多维度 fallback**：
    *   `AUTO_STRICT` 组被设计为剔除那些容易断流的协议，只选用最稳健的节点。
    *   `AUTO_SPEED` 则可能放宽协议限制，通过 `url-test` 选择绝对延迟最低的节点。
*   **建议使用原则**：如果使用者（或 AI Agent）要调整 Desktop，建议将最核心的生产力流量（如 Google、GitHub）绑定到 `AUTO_STRICT`，将流媒体等大流量分配给 `AUTO_SPEED`，其余常规请求扔给 `AUTO_GENERAL`。

---

## 四、不同配置文件之间的差异与考量

除了 Desktop 之外，Android、iOS 和 Docker 的配置也有微妙的区别：

1.  **Docker 版 (旁路由/网关模式)**：
    *   作为整个局域网的网关，通常不开启 TUN 模式，而是依赖内核路由或 iptables 转发。因此它的规则更倾向于大吞吐量和纯 IP 级别的稳健路由，不过多依赖本地设备的特定协议嗅探。
2.  **Android / iOS (移动端)**：
    *   移动端经常在蜂窝网络和 WiFi 之间切换，IP 会频繁变动。因此移动端配置强烈依赖 `sniffer`（嗅探器）。
    *   配置中开启了极具针对性的 DNS 与 TLS 嗅探，以便在手机某些流氓 App 强行使用 DoH (DNS over HTTPS) 解析出裸 IP 发起连接时，Mihomo 能从流量中“扒出”真实域名，重新正确分流，防止隐私泄露。
    *   代理组相对简单（直接使用 `AUTO_1`, `AUTO_2`），因为移动端算力和续航有限，不宜运行过多的并发测速 (url-test) 分组。

---

## 五、给 AI Agent (AI 智能体) 的食用指南

如果你是一个 AI Agent（且具备代码/文件编辑能力，即 Skills），人类用户可能会要求你：“帮我把我的机场链接导入到这些配置中”。你可以遵循以下思路与人类互动：

### 交互与执行流程
1.  **主动询问订阅数量**：
    > “请问您有几个机场订阅链接需要导入？我可以帮您分别注入到 `sub_nodes_1` 和 `sub_nodes_2` 中。”
2.  **获取链接并修改配置**：
    *   读取对应的 `*_template.yaml`，找到 `proxy-providers` 块。
    *   将 `url: "YOUR_SUBSCRIPTION_LINK_HERE"` 替换为用户提供的真实链接。
    *   如果只有一个机场，则删除 `sub_nodes_2` 及相关引用，或者将其留空。
3.  **高级定制 (分配节点)**：
    > “需要我帮您针对特定应用分配节点吗？比如让 Telegram 固定走 `AUTO_1`，让流媒体固定走 `AUTO_2`？”
    *   如果用户同意，请在 `rules:` 列表中，找到对应的 `RULE-SET,telegramcidr`，将其后的 `PROXY` 改为 `AUTO_1`。
4.  **保存输出**：
    *   将修改好的内容**另存为**去掉 `_template` 后缀的文件（例如 `mihomo_config_ios.yaml`），以此保护模板不被破坏。