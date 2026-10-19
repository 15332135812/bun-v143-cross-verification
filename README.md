# Bun v1.4.3 多源交叉核验简报

用 Lexmount WebFetch 与 Cloud Browser 对 Bun v1.4.3 官方发布说明和 GitHub 社区 Issue 做交叉核验，产出一份带来源、截图和降级记录的简报。

## 材料

- [brief.md](brief.md) — 核验简报：执行摘要、工作流、交叉核验矩阵、关键发现、信息缺口、附录
- [evidence.json](evidence.json) — 9 条证据记录（URL、方法、状态、warnings、verdict、notes）
- [workflow-log.md](workflow-log.md) — 四阶段运行日志（常规提取、DOM/截图兜底、API 降级、边界探索）
- [bun-releases.json](bun-releases.json) — GitHub Releases 页面解析出的 10 个版本
- [artifacts/](artifacts/) — 截图、DOM 与原始材料

## 核心发现

- #46005：失败消息超过 JSC 最大字符串长度时，`bun:test` matcher 静默通过。
- #45860：`String.replace` 带回调在 1.4.3 中 454 ns，对比 Node 26.12 的 222 ns，慢 2 倍。
- #46050：子进程等待 unref'd IPC channel 时忙等占满一个核心。

## 方法与边界

- 云平台工具：`webfetch-cli extract`、`dump-dom`、`dump-dom --engine http`、`browser-cli`。
- 本地 Agent：URL 构造、JSON 解析、Issue 筛选、严重性判断、矩阵设计、简报撰写。
- 人工：`fetched_at` 赋值、最终审阅。
- v1.4.3 博客正文 86,187 字符，本次检索覆盖约 32,711 字符（38%）；其余未逐字核验。
- 比赛活动页在本次环境中 `extract` 与 `dump-dom` 均返回 502，未获取到比赛要求；该边界见 `brief.md` §7 与 `workflow-log.md` 阶段四。

## 证据边界

- `issue-46005.png`：证明 Issue 真实存在、标题与标签可见；不证明正文技术细节的正确性。
- `api-issue-45860.png`：证明 REST API 返回结构化 JSON；不证明 454 ns/222 ns 的测量方法。
- `search-502-url.png`：证明搜索页在浏览器中可渲染、显示 666 results；不证明 `webfetch` 的 502 不存在，502 由 `evidence.json` #6 的原始错误文本证明。

## 快速校验

```bash
python -I -c "import json; json.load(open('evidence.json', encoding='utf-8')); print('evidence.json: valid JSON')"
```
