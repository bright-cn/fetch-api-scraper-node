# 使用 Bright Data Fetch API 在 Node.js 中将 URL 转换为 Markdown

使用 Node.js 和 JavaScript 将任意 URL 转换为 Markdown、文本、HTML 或 JSON-LD，供 LLM 和 RAG 使用。可绕过机器人拦截。无依赖项。基于 [Bright Data Fetch API](https://docs.brightdata.com/products/scrapers/fetch-api/introduction?utm_source=github&utm_medium=referral&utm_content=fetch-api-scraper-node) 构建。新账户每月可获得 5,000 个免费额度。

[![实时检查](https://github.com/bright-cn/fetch-api-scraper-node/actions/workflows/live.yml/badge.svg)](https://github.com/bright-cn/fetch-api-scraper-node/actions/workflows/live.yml) [![上次验证](https://img.shields.io/badge/last%20verified-9%20Nov%202026-brightgreen)](https://github.com/bright-cn/fetch-api-scraper-node/actions/workflows/live.yml) <!-- verified: rewritten by the daily run -->

[快速入门](#如何使用-nodejs-将-url-转换为-markdown) · [任意网站](#抓取任意网站包括拦截机器人的网站) · [CLI](#批量将-url-转换为-markdown-的-cli) · [格式](#为-llm-将网站转换为-markdown文本html-和-json-ld) · [数据](#fetch-api-数据字段json) · [错误](#fetch-api-抓取工具错误) · [MCP 和 Claude Code](#fetch-api-mcp-服务器和-claude-code-技能) · [免费额度](#fetch-api-免费吗) · [文档](https://docs.brightdata.com/products/scrapers/fetch-api/introduction?utm_source=github&utm_medium=referral&utm_content=fetch-api-scraper-node) · [支持](#支持)

在 JavaScript 中为 LLM 将任意 URL 转换为 Markdown：发送网页 URL，即可获得页面的 Markdown、纯文本、原始 HTML 和 JSON-LD 结构化数据，以及页面标题和最终 URL。可从任意国家/地区请求，绕过机器人拦截，无需浏览器且没有依赖项。基于 [Bright Data Fetch API](https://docs.brightdata.com/products/scrapers/fetch-api/introduction?utm_source=github&utm_medium=referral&utm_content=fetch-api-scraper-node) 构建。

本文还提供：用于处理 URL 列表的单命令 CLI，以及完全不需要 JavaScript 的 [Bright Data CLI](#fetch-api-mcp-服务器和-claude-code-技能)。

## 如何使用 Node.js 将 URL 转换为 Markdown

只需 Node 20 或更高版本，无需安装任何内容：这里使用内置的 `fetch`。由于它使用顶层 `await`，请将其保存为 `.mjs` 文件。

```bash
export BRIGHTDATA_API_TOKEN=YOUR_API_KEY
```

从 [Bright Data 控制面板](https://www.bright.cn/cp/setting/users)获取令牌。还没有账户？[创建账户](https://www.bright.cn/cp/start)；新账户可[每月获得 5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。

```javascript
const API = "https://api.brightdata.com/datasets/v3";
const HEADERS = {
  Authorization: `Bearer ${process.env.BRIGHTDATA_API_TOKEN}`,
  "Content-Type": "application/json",
};

async function call(path, body) {
  const init = body === undefined ? { headers: HEADERS } : { method: "POST", headers: HEADERS, body: JSON.stringify(body) };
  const response = await fetch(API + path, init);
  if (!response.ok) throw new Error(`HTTP ${response.status}: ${await response.text()}`);
  return response.json();
}

const page = { url: "https://en.wikipedia.org/wiki/Web_scraping", country: "US" };
const job = await call("/trigger?dataset_id=gd_m6gjtfmeh43we6cqc&include_errors=true", [page]);
while (!["ready", "failed"].includes((await call(`/progress/${job.snapshot_id}`)).status)) {
  await new Promise((done) => setTimeout(done, 5_000));
}
const [record] = await call(`/snapshot/${job.snapshot_id}?format=json`);
if (record.error) throw new Error(record.error);
console.log(record.page_title, "|", record.markdown.length, "characters of Markdown");
console.log(record.markdown.split("\n").find((line) => line.startsWith("# ")));
```

```
Web scraping - Wikipedia | 68572 characters of Markdown
# Web scraping
```

[按页面](https://www.bright.cn/pricing/web-scraper)计费。

该代码片段中的两项设置不是可选的。

`include_errors=true` 会让 API 将无法抓取的页面、404 页面或不存在的主机报告为包含 `error` 键的行。API 默认关闭此选项，关闭时失败页面看起来与没有页面相同。

`country` 是可选项。请以大写形式发送；参阅[从其他国家/地区抓取页面](#从其他国家地区抓取页面)。

## 抓取任意网站，包括拦截机器人的网站

Fetch API 可以抓取任何公共网站，包括会拦截机器人的网站。它通过 Bright Data 的解锁功能处理请求：你无需使用无头浏览器、配置代理或破解 CAPTCHA。2026 年 11 月 9 日，它在 12 秒内返回了一个 Zillow 搜索页面，该网站会拦截大多数爬虫工具，并获得了 9,308 个字符的 Markdown 和页面的 JSON-LD。

唯一的凭据是 Bright Data API 密钥，通过 `BRIGHTDATA_API_TOKEN` 设置。此软件包及其命令也能找到由 `bdata login` 存储的密钥。

## 批量将 URL 转换为 Markdown 的 CLI

此仓库中的命令可同时对多个 URL 运行快速入门流程，并写入一个 JSON 文件。

```bash
npm install -g github:bright-cn/fetch-api-scraper-node
fetch-api-scraper https://example.com https://en.wikipedia.org/wiki/Web_scraping --country US
```

```
Fetching 2 page(s) from US
Usually under a minute each. Billed per page, 5,000 free credits a month.
asking  https://example.com...
got     https://example.com: "Example Domain", 156 characters of Markdown
asking  https://en.wikipedia.org/wiki/Web_scraping...
got     https://en.wikipedia.org/wiki/Web_scraping: "Web scraping - Wikipedia", 68,572 characters of Markdown

Saved 2 pages as JSON to pages.json
```

在终端中，`asking` 行会替换为原地更新的加载动画和计时器，让你知道任务正在运行。

```
--country CC   two-letter country to request the pages from, such as US
--out PATH     output file, default pages.json
```

该命令会在 `BRIGHTDATA_API_TOKEN` 中查找令牌；如果你曾运行过一次 `npx -p @brightdata/cli bdata login`，它也会在已存储的登录信息中查找，供你以及该终端中的任何编码智能体使用。它不会读取 `.env` 文件；Node 会通过 `node --env-file=.env` 为其加载该文件。

如需按 URL 获取 `ok` 和 `error`，而不是原始行，请导入它而非直接运行。此软件包使用 ESM，因此请使用 `import`，不要使用 `require`。`scrape` 不会因某个错误 URL 而拒绝；请先检查 `ok`，再读取 `page`：

```javascript
import { scrape } from "@brightdata/fetch-api-scraper-node";

for (const outcome of await scrape(["https://example.com", "https://example.com/this-page-does-not-exist"])) {
  if (outcome.ok) console.log(`${outcome.url}: ${outcome.page.page_title}`);
  else console.log(`${outcome.url} failed: ${outcome.error}`);
}
```

```
https://example.com: Example Domain
https://example.com/this-page-does-not-exist failed: The navigation resulted in a dead page (404 status code)
```

## 为 LLM 将网站转换为 Markdown、文本、HTML 和 JSON-LD

Fetch API 是一个只有一种输入的抓取工具，相关说明见 [Fetch API 文档](https://docs.brightdata.com/products/scrapers/fetch-api/introduction?utm_source=github&utm_medium=referral&utm_content=fetch-api-scraper-node)。下面每个代码片段都是完整的：完成上面的 `npm install` 后，直接按原样粘贴即可。它们每周一都会在 Actions 中运行，每隔一天还会运行一次较小的检查，顶部徽章显示的是最新结果。

| 你的需求 | 方法 |
| --- | --- |
| 将页面转换为供 LLM 使用的 Markdown | `await markdown(url)` |
| 获取多个 URL 的所有格式 | `await pages([url, url])` |
| 从其他国家/地区获取页面 | `await pages(url, { country: "DE" })` |

`pages` 接受一个 URL 或一个数组。一个数组只对应一个任务，而不是每个 URL 对应一个任务。每个页面是一条记录，API 无法抓取的页面会作为包含 `error` 键的行返回。

### 为 LLM 将网页转换为 Markdown

```javascript
import { writeFile } from "node:fs/promises";

import { markdown } from "@brightdata/fetch-api-scraper-node";

const text = await markdown("https://docs.python.org/3/tutorial/index.html");
await writeFile("tutorial.md", text, "utf8");
console.log(text.length, "characters saved to tutorial.md");
const heading = text.split("\n").find((line) => line.startsWith("# "));
console.log(heading.split("[")[0]); // keep the text, drop the pilcrow link after it
```

```
15117 characters saved to tutorial.md
# The Python Tutorial
```

`markdown` 是整个页面，包括导航和侧边栏，而不只是文章。API 返回错误行而非页面时，`markdown` 会拒绝。

### 获取原始 HTML 和最终 URL

```javascript
import { pages } from "@brightdata/fetch-api-scraper-node";

const [page] = await pages("http://example.com");
console.log("asked for: http://example.com");
console.log("landed on:", page.final_url);
console.log(page.page_html.length, "characters of HTML,", page.html2text.length, "of plain text");
```

```
asked for: http://example.com
landed on: http://example.com/
577 characters of HTML, 156 of plain text
```

`final_url` 是页面重定向后的最终地址。`html2text` 是页面的纯文本，`page_html` 是 HTML。

### 从页面提取 JSON-LD 结构化数据

```javascript
import { pages } from "@brightdata/fetch-api-scraper-node";

const [page] = await pages("https://en.wikipedia.org/wiki/Web_scraping");
const found = JSON.parse(page.ld_json);
const article = Array.isArray(found) ? found[0] : found;
console.log(article["@type"], "|", article.name, "|", article.dateModified.slice(0, 10));
```

```
Article | Web scraping | 2026-11-01
```

`ld_json` 是页面的 schema.org 数据，以 JSON 字符串形式提供，仅在页面包含此类数据时存在。在 Wikipedia 上，它可解析为对象列表。

### 从其他国家/地区抓取页面

```javascript
import { pages } from "@brightdata/fetch-api-scraper-node";

for (const country of ["US", "DE", "JP"]) {
  const [page] = await pages("https://ipinfo.io/country", { country });
  console.log(country, "->", page.markdown.trim());
}
```

```
US -> US
DE -> DE
JP -> JP
```

https://ipinfo.io/country 会返回请求来源国家/地区，因此可显示 Fetch API 从哪里抓取页面。2026 年 11 月 9 日，10 次此类请求中有 8 次来自指定国家。JP 有一次、FR 有一次来自西班牙，而记录中的 `country` 仍会原样返回指定国家，因此它不能证明页面是从哪里抓取的。

### 在一个任务中抓取多个 URL

```javascript
import { pages } from "@brightdata/fetch-api-scraper-node";

const urls = ["https://example.com", "https://www.zillow.com/homes/Seattle,-WA_rb/"];
for (const page of await pages(urls)) {
  console.log(page.page_title.slice(0, 50), "|", page.markdown.length, "characters");
}
```

```
Seattle WA Real Estate - Seattle WA Homes For Sale | 9308 characters
Example Domain | 156 characters
```

第二个 URL 是 Zillow 搜索页面，它与 example.com 在同一个任务中抓取，并成功绕过了机器人拦截。

### 立即触发，稍后获取

对于很长的 URL 列表，不要让进程阻塞一小时。触发任务，保存快照 ID，准备就绪后再获取。快照可供下载 30 天。

```javascript
import { FETCH_API, fetchSnapshot, status, trigger } from "@brightdata/fetch-api-scraper-node";

const snapshot = await trigger(FETCH_API, [{ url: "https://example.com" }]);
console.log("snapshot:", snapshot);
let state;
while (!["ready", "failed"].includes((state = await status(snapshot)))) {
  await new Promise((done) => setTimeout(done, 5_000));
}
console.log("status:", state);
console.log("fetched:", (await fetchSnapshot(snapshot))[0].page_title);
```

```
snapshot: sd_mw9can7x779moa1vk
status: ready
fetched: Example Domain
```

对于数据集 `gd_m6gjtfmeh43we6cqc`，实时 API 在 2026-11-09 接受的输入字段：

| 输入 | 必填 | 作用 |
| --- | --- | --- |
| `url` | 是 | 要抓取的页面 |
| `country` | 否 | 用于抓取页面的两位国家/地区代码，须大写 |

`bdata pipelines info gd_m6gjtfmeh43we6cqc collect_by_url` 会输出 API 验证的输入字段。

## Fetch API 数据字段（JSON）

大多数人希望从页面获取的字段：

```
page_title  markdown  final_url  html2text  page_html  ld_json  country
```

代码没有硬编码任何字段列表。API 返回的所有内容都会写入每条记录和命令生成的文件中。

<!-- fields:start -->
每天根据已发布的架构 [scrapers-full.json](https://docs.brightdata.com/scrapers-full.json) 重新生成，因此不会过时。一条记录包含适用于它的字段，以及 `timestamp` 和 `input`；示例文件的第一个页面有 9 个键。

<details>
<summary>全部 8 个页面字段，<code>gd_m6gjtfmeh43we6cqc</code></summary>

| 字段 | 类型 | 描述 |
| --- | --- | --- |
| `markdown` | html2markdown | 提取 Markdown 格式的内容，保留标题、列表和链接等基本格式，同时移除复杂的 HTML 结构。 |
| `url` | url | url |
| `final_url` | url | 最终 URL |
| `html2text` | html2text | 仅提取页面中的原始文本，移除所有 HTML 标签和格式。 |
| `page_html` | html2html | 提取页面的完整 HTML 源代码，包括所有标签、脚本和样式。 |
| `ld_json` | html2ldjson | 提取 JSON-LD 格式的结构化元数据，常用于 SEO 和富媒体摘要（例如 schema.org 数据）。 |
| `page_title` | text | 页面标题 |
| `country` | country | 输入的国家/地区 |

</details>
<!-- fields:end -->

<details>
<summary>真实输出文件的开头，来自 <code>fetch-api-scraper https://example.com</code></summary>

```json
{
  "generated_at": "2026-11-09T14:11:43.269Z",
  "pages": [
    {
      "url": "https://example.com",
      "page": {
        "markdown": "This domain is for use in documentation examples without needing permission. This is not a service; avoid relying on it for testing and monitoring purposes.",
        "url": "https://example.com/",
        "final_url": "https://example.com/",
        "html2text": "This domain is for use in documentation examples without needing permission. This is not a service; avoid relying on it for\ntesting and monitoring purposes.",
        "page_html": "<!doctype html><html lang=en><head><meta charset=utf-8><link rel=icon href=data:,><meta name=viewport content=\"width=device-width,initial-scale=1\"><title>Example Domain</title><style>html{color-scheme:light dark;background:light-dark(#eee,#222)}body{font:16px/1.6 system-ui,sans-serif;max-width:26em;margin:auto;padding:25vh 2em 2em;text-align:center}</style></head><body><p>This domain is for use in documentation examples without needing permission. This is not a service; avoid relying on it for testing and monitoring purposes.</p><script src=/s.js></script></body></html>\n",
        "page_title": "Example Domain",
        "country": null,
        "timestamp": "2026-11-09T14:11:34.180Z",
        "input": {
          "url": "https://example.com"
        }
      }
    }
  ]
  ...
```

包含一个页面及其所有字段的完整文件位于 [examples/sample_output.json](examples/sample_output.json)。

</details>

## Fetch API 抓取工具错误

| 你看到的内容 | 含义 |
| --- | --- |
| `API token required but not found.` | 发出任何请求前以状态码 2 退出。请设置令牌或运行 `bdata login`。 |
| `failed  URL: The navigation resulted in a dead page (404 status code)` | 以状态码 1 退出。页面不存在。 |
| `failed  URL: Crawler error: Navigation failed ... Could not resolve host` | 以状态码 1 退出。域名不存在。API 花了 147 秒才返回此结果。 |
| `failed  URL: timeout: snapshot ... not ready after 900s` | 以状态码 1 退出。请重新运行，或稍后使用 `fetchSnapshot()` 获取快照。 |
| `failed  URL: HTTP 401: ...` | 令牌已设置，但不正确。 |

任何失败都会以状态码 1 退出，因此可安全地使用运行结果控制脚本。

在函数中，相同情况如下所示：

| 你看到的内容 | 含义 |
| --- | --- |
| `TokenMissing` | 没有 `BRIGHTDATA_API_TOKEN`，也没有 `bdata login`。 |
| `ApiError: HTTP 400: ...validation_error...` | API 不接受的输入，例如格式错误的 URL。 |
| 包含 `error` 键的行 | API 对某个 URL 的响应：`dead_page`；如果主机无法解析，则为 `proxy`。其他行不受影响。 |
| `ApiError: timeout: ...` | 任务超过 900 秒等待时间。将 `{ timeoutMs }` 传给 `run`，或使用 `trigger` 并稍后获取。 |

## Fetch API MCP 服务器和 Claude Code 技能

无需 JavaScript，也无需安装任何内容。粘贴这两行；第一行会打开一次浏览器，也可以通过 SSH 和 CI 使用 `bdata login --device`：

```bash
npx -p @brightdata/cli bdata login
npx -p @brightdata/cli bdata pipelines run gd_m6gjtfmeh43we6cqc collect_by_url "https://example.com" --pretty
```

2026 年 11 月 9 日耗时 26 秒，获得 1 条记录、8 个字段。CLI 会省略值为 null 的字段。

`npx skills add brightdata/skills` 会让 Claude Code、Cursor 和 Codex 掌握这些命令和文档，之后即可使用自然语言操作。完整指南：[面向编码智能体的 Bright Data](https://docs.brightdata.com/quickstart-coding-agent)。

为托管式助手使用且没有终端？[Bright Data MCP 服务器](https://github.com/bright-cn/brightdata-mcp#which-tool-to-use)可通过其 `scrapers` 组中的 `search_scrapers` 和 `run_scraper` 运行任何预构建爬虫工具，包括 Fetch API：

    https://mcp.brightdata.com/mcp?token=YOUR_API_TOKEN&groups=scrapers

智能体也可以自行创建账户，无需填写注册表单：[智能体注册](https://brightdata.com/auth.md)。Bright Data 支持的其他所有连接，从 LangChain 到 Zapier 和 n8n，参阅[集成](https://docs.brightdata.com/integrations/introduction)。

## Fetch API 免费吗？

新的 Bright Data 账户可[每月获得 5,000 个免费额度](https://docs.brightdata.com/general/account/billing-and-pricing/free-tier)。Fetch API 按记录计费，每个页面一条记录。[定价](https://www.bright.cn/pricing/web-scraper)列出了费率。

## 支持

此仓库中的错误：[提交 issue](https://github.com/bright-cn/fetch-api-scraper-node/issues)，并参阅 [CONTRIBUTING.md](CONTRIBUTING.md) 了解需要提供哪些信息。有关 API、你的账户或额度的任何问题，请联系 [Bright Data 支持团队](https://brightdata.zendesk.com/hc/en-us/requests/new)。

## 许可证

[Apache 2.0](LICENSE)。

## 使用 Python 将 URL 转换为 Markdown

同一款 URL 转 Markdown 爬虫工具也提供 Python 版本：[bright-cn/fetch-api-scraper](https://github.com/bright-cn/fetch-api-scraper)。

## 更多来自 Bright Data 的开源抓取工具

采用相同方式构建且示例均经过实时验证的其他 Node.js 爬虫工具：

- [适用于 Node.js 的 ChatGPT 抓取工具](https://github.com/bright-cn/chatgpt-scraper-node)
- [适用于 Node.js 的 Amazon 爬虫工具](https://github.com/bright-cn/amazon-scraper-node)
- [适用于 Node.js 的 LinkedIn 抓取工具](https://github.com/bright-cn/linkedin-scraper-node)
