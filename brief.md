# Bun v1.4.3 多源交叉核验简报：官方宣传与社区现实的落差

## 1. 执行摘要
**核验环境声明**：时间线设定为 2026-10；数据来源分为真实抓取、部分检索、推测三类；核验截止时间为 2026-10-19（UTC）；全文级断言未完成的范围为 Bun v1.4.3 博客正文中未被本次检索覆盖的部分（按 32,711/86,187 字符口径约 62%）。

本简报利用 Agent 与 Lexmount 云平台的 Web Fetch 及 Cloud Browser 能力，对 Bun v1.4.3 版本的官方发布说明与社区真实反馈进行了交叉核验。
核验发现，虽然官方宣称了一系列性能提升与安全修复，但在发布后的 9 天内，社区接连爆出**严重逻辑缺陷（测试静默通过）** 与 **特定 API 性能回退（慢 2 倍）**。这表明官方在边缘场景的测试覆盖存在盲区，升级需谨慎。

## 2. 工作流与可核验性说明
为获取最真实的证据链，本工作流采用了**多级降级与容错策略**：
1. **常规提取**：优先使用 `webfetch-cli extract` 抓取官方博客和 Issue 搜索页。
2. **DOM 探测与可视化**：GitHub Releases 页面遭遇正文提取失败（`thin_content`），通过 `dump-dom` 提取完整 HTML，并用 `browser-cli` 截图交叉验证。
3. **API 降级兜底**：当 GitHub 网页端返回 502 超时（`lightmount_lite context deadline exceeded`）时，直接向 GitHub REST API 发起 `dump-dom --engine http` 请求，绕过网页渲染限制，成功提取结构化 Issue 数据。

## 2.1 本地 vs 云平台分工

| 环节 | 云平台 | 本地/人工 |
| :--- | :--- | :--- |
| 网页抓取 | `webfetch-cli extract` | 构造 URL、决定抓取顺序 |
| DOM 渲染与截图 | `dump-dom`、`browser-cli` | 指定目标 URL 与落盘路径、检查截图 |
| API 降级 | `dump-dom --engine http` | 判断 502 后切换、构造 REST API URL 与查询参数 |
| JSON 解析 | — | 解析 `html` 字段中的 JSON、提取 issue 字段 |
| Issue 筛选与严重性判断 | — | 按标题中的版本号与具体数值、open 状态、评论数筛选；判定 #46005 为严重逻辑缺陷、#45860 为性能回退 |
| 矩阵设计 | — | 设计“官方声明 / 社区反馈 / 结论”三维度矩阵 |
| 简报撰写 | — | 撰写执行摘要、发现详情与附录 |
| 时间戳记录 | — | 人工赋值 `fetched_at` |

本工作流中，云平台负责“取到原始材料”，本地 Agent 负责“判断与组织”，两者不可互相替代。

## 2.2 从 666 条到 3 条：筛选路径

GitHub REST 搜索接口返回 `total_count=666`，请求参数为 `per_page=10&sort=created&order=desc`，因此实际只查看了前 10 条，未翻页。#45860 不在这 10 条内，它来自同一查询更早一次网页搜索页 `extract` 的 25 条列表（按列表顺序计为第 18 条）；把它写成“从前 10 条选出”会与原始记录不符。

1. 搜索 API 返回 `total_count=666`，`per_page=10`，按 `created desc` 取前 10 条。
2. 从前 10 条中选出 #46050（第 3 条，标题描述 CPU busy-spin，0 评论）与 #46005（第 8 条，正文明确写 `bun 1.4.2 and 1.4.3`，1 条评论）；#45860 从网页搜索页 25 条列表中选出（标题含 `Bun 1.4.3`、`454 ns`、`222 ns`）。
3. 对 3 条逐一请求 `api.github.com/repos/oven-sh/bun/issues/{number}` 及其 `/comments`；#46005 的评论 ID 为 `6258946358`，#45860 与 #46050 的评论数为 0。

上述筛选维度为事后归纳，会话中未记录当时的逐条打分。

## 3. 交叉核验矩阵

| 核验维度 | 官方声明（Release Notes） | 社区真实反馈（Issues） | 核验结论与影响 |
| :--- | :--- | :--- | :--- |
| **正确性 / 测试** | 在本次核验检索的 32,711 字符（约占博客正文 86,187 字符的 38%）中未出现 `bun:test` 相关表述；其余约 62% 正文未逐字核验，不能排除存在相关表述。 | **[#46005]** 若失败消息长度超过 JSC 最大限制（2^31-1），`throw_rendered` 抛出空异常，导致断言静默通过（False Positive）。 | **致命逻辑缺陷**：严重影响 CI/CD 的可靠性。根因指向 `expect.rs` 的异常处理边界。[E#8] |
| **特定 API 性能** | 宣称全局提速：JSON.parse 快 1.5x，Promise 处理大幅优化。 | **[#45860]** `String.replace` 配合回调函数在 1.4.3 中耗时 454 ns，比 Node 26.12（222 ns）**慢 2 倍**。 | **性能倒退**：在本次核验检索的 32,711 字符（约占博客正文 86,187 字符的 38%）中未出现 `String.replace`；其余约 62% 正文未逐字核验，不能排除存在相关表述。目前该 Issue 处于 `needs triage`。[E#9] |
| **版本兼容性** | 修复大量 Node.js 兼容性问题。 | 社区报告了与 1.4.3 相关的运行时异常（如 #46050 的 IPC 通道空转）。 | 需针对特定运行时场景做灰度测试。[E#7][#46050 正文] |

## 4. 关键发现详情

### 4.1 严重逻辑缺陷：`bun:test` 静默通过（Issue #46005）
- **现象**：当断言失败生成的错误消息过大（超过 2147483647 字符）时，本应触发的失败被系统当作“通过”处理。`expect.assertions()` 也无法捕获。
- **根因**：在底层 Rust 实现中，`throw()` 遇到空 `create_error_instance` 时未正确设置异常状态，导致 JS 侧认为没有挂起异常。根因分析来自评论者 aron-intframe（评论 ID 6258946358），非 Issue 作者 lo5。

### 4.2 性能回退：`String.replace` 慢 2 倍（Issue #45860）
- **现象**：在 AWS Signature V4 库的百分号编码逻辑中发现，正则替换带回调的性能不及 Node.js。

## 5. 信息缺口与待核验项
- **官方响应**：截至核验时，会话中未获取到相关 PR 的合并状态（未发起该查询）。
- **修复版本预测**：预计在 v1.4.4 中修复（推测，非官方确认；依据仅为 v1.4.x 系列平均发布间隔，非 PR 状态）。

## 6. 附录：数据来源与核验材料
- **官方事实源**：[Bun v1.4.3 博客](https://bun.sh/blog/bun-v1.4.3)
- **官方安全修复**：[Bun v1.4.3 博客 · Security 段](https://bun.sh/blog/bun-v1.4.3)（CVE-2026-48618）
- **社区严重缺陷**：[GitHub Issue #46005](https://github.com/oven-sh/bun/issues/46005) [截图: artifacts/issue-46005.png]（证明 Issue 真实存在、标题与标签可见；不证明正文技术细节的正确性）
- **社区性能回退**：[GitHub Issue #45860](https://github.com/oven-sh/bun/issues/45860) [截图: artifacts/api-issue-45860.png]（证明 REST API 返回结构化 JSON；不证明 454 ns/222 ns 的测量方法）
- **社区运行时异常**：[GitHub Issue #46050](https://github.com/oven-sh/bun/issues/46050)（该 Issue 无独立正文截图；其标题可见于 search-502-url.png 的搜索结果列表；正文与状态由 API 返回，见 evidence.json #7）
- **搜索页（502 URL）**：[GitHub Issues 搜索 · is:issue 1.4.3](https://github.com/oven-sh/bun/issues?q=is%3Aissue+1.4.3) [截图: artifacts/search-502-url.png]（证明该搜索页在浏览器中可渲染、显示 666 results；**不证明 webfetch 的 502 不存在，502 由 evidence.json #6 的原始错误文本证明**）

## 7. 可复用性：从 Bun v1.4.3 到任意 {repo}/{version}

1. **官方声明提取**：用 `webfetch-cli extract` 抓 `{repo}` 的 release blog，记录标题、日期、正文长度与 warning。
2. **版本列表兜底**：`extract` 触发 `thin_content` 时改用 `dump-dom` 获取完整 HTML，再解析版本号、日期与链接。
3. **社区搜索降级链**：先抓网页搜索页；遇到 502 时改用 `api.github.com/search/issues?q=...&per_page=10`，记录 `total_count` 与实际查看条数。
4. **Issue 深挖**：对候选 Issue 请求 `api.github.com/repos/{repo}/issues/{n}` 及其 `/comments`，保存正文、状态、标签与评论。
5. **交叉核验矩阵**：按“官方声明 vs 社区 Issue vs 结论”三维度组织，逐条标注证据编号（本地 Agent 完成）。
6. **证据落盘**：在 `evidence.json` 记录 `url`、`method`、`status`、`warnings`、`verdict`、`notes`（本地写盘，时间戳人工赋值）。

本模板已在 Bun v1.4.3 上验证；泛化到其他项目时，需替换 repo 名与 version 关键词，并重新评估搜索页 502 的发生率。

**适用边界**：本模板的降级链（`extract` → `dump-dom` → REST API）依赖目标站点存在结构化 API 端点。对无 API 替代的纯前端渲染页面（例如本次抓取比赛活动页时 `extract` 与 `dump-dom` 均返回 502），降级链在第二级终止，需人工介入。
