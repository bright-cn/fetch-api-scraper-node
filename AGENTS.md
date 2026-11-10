# 供在此仓库中工作的编码智能体使用

更改任何内容前请先阅读本文。以下每一行均已于 2026-11-09 根据实时 API 或 Bright Data CLI 0.4.1 验证。如果 API 参考页面与实时 API 不一致，以本文记录的实时 API 为准。

## 身份验证

- `token()` 先读取 `BRIGHTDATA_API_TOKEN`，然后读取存储在 `brightdata-cli/credentials.json` 中的密钥 `bdata login`（SDK 也读取同一文件）。不支持 `.env`；Node 20 可通过 `node --env-file=.env` 加载该文件。设置变量或运行一次登录命令，是唯一需要人工完成的步骤；切勿将密钥粘贴到代码中。
- README 中的原始 HTTP 快速入门仅读取 `BRIGHTDATA_API_TOKEN`。它无法获取 CLI 登录信息。

## API 的行为方式

- 一个数据集 `gd_m6gjtfmeh43we6cqc`，仅按 URL 采集。输入：`url`，可选 `country`。
- 产品名称是 Fetch API。不要写成“Crawl API”，也不要使用 SDK 的爬虫方法：Bright Data 的文档会避免这两种写法。
- 此软件包使用内置的 `fetch` 调用 `https://api.brightdata.com/datasets/v3`，且没有依赖项。快照下载地址是 `fetchSnapshot`。测试通过 `transport` 模拟网络。
- 每次调用都是异步任务：`/trigger`，轮询 `/progress/<id>`，然后调用 `/snapshot/<id>?format=json`。一个页面耗时 7 到 44 秒；未知主机耗时 147 秒后失败。`TIMEOUT_MS` 为 900000。
- 如果状态轮询因网络错误失败，则会持续重试直至截止时间。HTTP 错误仍会导致拒绝。
- 始终发送 `include_errors=true`。404 会作为错误行返回，即 `dead_page`；无法解析的主机会作为错误行返回，其 error_code 为 `proxy`。
- 2026-11-09 的实际行为：
  - 每条记录包含：`markdown`、`html2text`、`page_html`、`page_title`、`url`、`final_url`、`country`，仅当页面包含相关内容时才会有 `ld_json`。
  - `ld_json` 是 JSON 字符串；在 Wikipedia 上，它可解析为列表。
  - `markdown` 是整个页面，包括导航部分。
  - `country`：请以大写形式发送。省略时，记录中的 `country` 为 null。通过 https://ipinfo.io/country 检查后，10 次请求中有 8 次来自指定国家，另有 2 次（JP 一次、FR 一次）来自西班牙，而记录中的 `country` 仍会原样返回输入值。它不能证明出口国家。
  - 一个受机器人防护的 Zillow 搜索页面在 12 秒内返回。
- 架构可能随时变化，恕不另行通知。切勿硬编码字段列表。
- 完整文档索引，每个条目对应一个 `.md` 页面：https://docs.brightdata.com/llms.txt

## 在此仓库中工作

- `npm test` 可离线运行，无需令牌。`npm run lint` 必须通过。
- CI 会在空白机器上执行 README 自身的命令，并通过准确的 H2 文本查找各部分。重命名标题时，也要在 ci.yml 中同步修改。
- 每周工作流会针对真实 API 执行 README 中的每个围栏代码块，每日工作流则会运行规模较小的实时检查。代码片段必须完整且可单独粘贴运行，显示的输出也必须真实。
- README 中的字段表位于 `<!-- fields:start -->` 和 `<!-- fields:end -->` 之间，每天由 `.github/scripts/field_tables.mjs` 根据 https://docs.brightdata.com/scrapers-full.json 重新生成。不要手动编辑。
- README 顶部的“上次验证”徽章行会由每日运行任务重写。不要手动编辑。
- 标题应便于搜索，例如“适用于 Node.js 的 URL 转 Markdown”和“如何将 URL 转换为 Markdown……”。添加章节时请保持这种模式。
- 保持精简。除状态轮询外，不要添加重试、去重、调度、数据库、异步示例或并发功能。
