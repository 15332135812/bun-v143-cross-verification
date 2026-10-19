# Lexmount 工作流运行日志

## 阶段一：常规提取测试
- 对 `bun.sh/blog/bun-v1.4.1` 使用 `webfetch-cli extract`，返回 200，无警告，成功提取。先用 v1.4.1 博客验证 extract 稳定性，再抓 v1.4.3。
- 对 `github.com/oven-sh/bun/releases` 使用 `extract`，返回 200，但触发 `thin_content` 警告（仅 19 字符）。

## 阶段二：能力升级与兜底
- 对 releases 页面使用 `dump-dom`，成功获取 539KB 完整 HTML，解析出 10 个版本的 JSON 数据。
- 对 releases 页面使用 `browser-cli` 创建会话并截图；提取失败是否由复杂布局导致未经验证（未做 DOM 结构诊断）。

## 阶段三：社区核验与 API 降级
- 对 Issues 搜索页后续 3 次请求（extract×2 + dump-dom×1）均返回 502 超时（context deadline exceeded）；同一搜索页更早一次 extract 成功（200 / 5,655 字符 / 25 条）。
- 切换至 GitHub REST API，使用 `dump-dom --engine http` 绕过网页渲染，成功拉取 #46005 和 #45860 的 Issue 数据。

## 阶段四：边界探索（比赛活动页抓取）

- 对 https://browser.lexmount.cn/events/real-task-challenge 使用 `webfetch-cli extract`，返回 502（`lightmount_domstable ... context deadline exceeded`）。
- 改用 `dump-dom` 重试，同样返回 502；两次均无可用页面内容返回。
- 与阶段三的 GitHub 搜索页 502 相比：引擎不同（`lightmount_domstable` vs `lightmount_lite`），但失败模式相同（context deadline exceeded）。
- 结论：本工作流的多级降级（extract → dump-dom → REST API）对"有结构化 API 端点"的目标有效；对"纯前端渲染、无 API 替代"的页面，降级链在第二级即终止，无法兜底。该边界已写入 brief §7。