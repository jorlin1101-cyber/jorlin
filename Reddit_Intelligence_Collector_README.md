# Engine 1 - Reddit 情报采集器工作流说明

这份 README 对应你的 n8n 工作流：

```text
http://localhost:5678/workflow/aOUAKkEsQnXGAs8U
```

工作流名称：

```text
Reddit 情报采集器
```

工作流 ID：

```text
aOUAKkEsQnXGAs8U
```

## 1. 这个工作流是做什么的

这个工作流是你的 Engine 1，负责从 Reddit 发现入境游相关内容，并把有价值的帖子写入 Notion。

它的核心目标是：

```text
抓 Reddit 帖子列表
-> 对每条帖子查重
-> 抓原帖和评论区
-> 整理成 AI 可读文本
-> 用 DeepSeek 判断是否是有价值的入境游痛点
-> 记录处理历史
-> 如果相关，就写入入境游内容库
```

一句话理解：

```text
Engine 1 负责“发现痛点”和“初步筛选”。
Engine 2 再负责“搜索解决方案”和“研究总结”。
```

## 2. 当前工作流概览

| 项目 | 当前值 |
| --- | --- |
| Workflow ID | `aOUAKkEsQnXGAs8U` |
| Workflow Name | `Reddit 情报采集器` |
| 当前节点数 | 19 个 |
| 触发方式 | Schedule Trigger，每 6 小时一次 |
| 当前工作流状态 | 未启用自动运行时，可手动 Execute 测试 |
| Reddit RSS 来源 | `http://127.0.0.1:8787/reddit/travelchina.rss?sort=new` |
| Reddit 评论抓取 | 本地代理 `http://127.0.0.1:8787/reddit-comments` |
| AI 模型 | DeepSeek |
| 记录库 | `Reddit数据记录库` |
| 内容库 | `入境游内容库` |

## 3. 运行前必须确认

### 3.1 n8n 必须启动

浏览器能打开：

```text
http://localhost:5678
```

### 3.2 本地 Reddit 代理必须启动

这个工作流不直接请求 Reddit，而是请求你的本地代理：

```text
http://127.0.0.1:8787
```

当前会用到两个接口：

```text
http://127.0.0.1:8787/reddit/travelchina.rss?sort=new
http://127.0.0.1:8787/reddit-comments
```

如果 `RSS Read` 报 403、502、连接失败，优先检查这个本地代理是否还在运行。

### 3.3 Notion 凭证必须可用

当前使用的 Notion 凭证：

```text
Notion account
```

会写入两个 Notion 数据库：

```text
Reddit数据记录库
入境游内容库
```

### 3.4 DeepSeek 凭证必须可用

当前 AI 节点使用：

```text
DeepSeek account
```

如果 AI 节点报鉴权错误，优先检查这个凭证。

## 4. 整体流程图

```text
Schedule Trigger
-> Feed URL List
-> Loop Over Items
-> RSS Read
-> Limit
-> Build Local Proxy URL
-> Loop Over Items2
-> Check Record DB
-> IF Record Missing
-> Check Content DB
-> IF Content Missing
-> Fetch Reddit Via Local Proxy
-> Format Reddit Post And Comments
-> AI 判断 + 格式化1
-> Code 解析AI JSON1
-> 存处理记录
-> Filter 筛选相关内容1
-> 存入notion内容库
-> 回到 Loop Over Items2
```

AI 节点下方还有一个模型节点：

```text
AI 判断 + 格式化1
└── DeepSeek Chat Model1
```

这个模型节点不是主线节点，而是给 AI 节点提供大模型能力的子节点。

## 5. 这个工作流里有两个循环

### 第一个循环：Loop Over Items

作用：

```text
逐个处理 RSS 源。
```

现在只有一个 RSS 源：

```text
r/travelchina new
```

所以这个循环目前看起来不明显。

如果以后你增加多个 RSS 源，比如：

```text
r/travelchina new
r/chinalife search alipay
r/travel search china
```

第一个循环就会逐个跑这些源。

### 第二个循环：Loop Over Items2

作用：

```text
逐个处理 RSS 源里抓到的 Reddit 帖子。
```

这是更关键的循环。

每一条 Reddit 帖子都会经过：

```text
查记录库
查内容库
抓评论
AI 判断
写 Notion
```

处理完一条后，再回到 `Loop Over Items2` 处理下一条。

## 6. 查重逻辑说明

这个工作流做了双重查重：

```text
第一层：查 Reddit数据记录库
第二层：查 入境游内容库
```

为什么要双重查重：

```text
记录库：防止同一个 Reddit 帖子被重复处理
内容库：防止已经入库的内容再次写入内容库
```

查重用到的字段：

```text
dup_key
reddit_id
source_url
```

其中：

```text
reddit_id 是 Reddit 帖子 ID
source_url 是 Reddit 原帖链接
dup_key 优先使用 reddit_id，如果没有 reddit_id 就使用 source_url
```

## 7. 每个节点详细说明

### 节点 1：Schedule Trigger

节点类型：

```text
n8n-nodes-base.scheduleTrigger
```

节点作用：

```text
定时启动整个工作流。
```

当前配置：

```json
{
  "rule": {
    "interval": [
      {
        "field": "hours",
        "hoursInterval": 6
      }
    ]
  }
}
```

小白理解：

```text
如果工作流启用，它会每 6 小时自动跑一次。
```

测试阶段建议：

```text
先不要急着启用自动运行。
先手动 Execute Workflow，确认每一步都稳定。
```

### 节点 2：Feed URL List

节点类型：

```text
Code
```

节点作用：

```text
生成要抓取的 Reddit RSS 源列表。
```

当前只配置了一个 RSS 源：

```text
http://127.0.0.1:8787/reddit/travelchina.rss?sort=new
```

当前代码：

```javascript
return [
  {
    json: {
      feed_url: "http://127.0.0.1:8787/reddit/travelchina.rss?sort=new",
      source_name: "r/travelchina new"
    }
  }
];
```

带注释解释版：

```javascript
// 返回一个数组，数组里的每一项都会变成 n8n 后续节点的一条 item
return [
  {
    json: {
      // RSS Read 节点后面会读取这个字段
      feed_url: "http://127.0.0.1:8787/reddit/travelchina.rss?sort=new",

      // 给这个来源起一个人能看懂的名字
      source_name: "r/travelchina new"
    }
  }
];
```

如果以后要加更多源，可以改成：

```javascript
return [
  {
    json: {
      feed_url: "http://127.0.0.1:8787/reddit/travelchina.rss?sort=new",
      source_name: "r/travelchina new"
    }
  },
  {
    json: {
      feed_url: "http://127.0.0.1:8787/reddit/travelchina/search.rss?q=alipay&sort=new",
      source_name: "r/travelchina alipay"
    }
  }
];
```

### 节点 3：Loop Over Items

节点类型：

```text
Loop Over Items / Split In Batches
```

节点作用：

```text
逐个处理 Feed URL List 里的 RSS 源。
```

当前配置：

```json
{
  "options": {}
}
```

小白理解：

```text
它像一个队列。
Feed URL List 输出几个 RSS 源，它就一个一个放给 RSS Read 处理。
```

连接说明：

```text
Loop 分支 -> RSS Read
Done 分支 -> 当前为空
```

注意：

```text
Loop Over Items2 处理完一个 RSS 源里的所有帖子后，会回到这个节点，继续处理下一个 RSS 源。
```

### 节点 4：RSS Read

节点类型：

```text
RSS Feed Read
```

节点作用：

```text
读取上一个节点传来的 RSS 地址，抓 Reddit 帖子列表。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| URL | `={{ $json.feed_url }}` |
| Options | 空 |

小白理解：

```text
它会读取 Feed URL List 里生成的 feed_url。
```

比如当前实际读取：

```text
http://127.0.0.1:8787/reddit/travelchina.rss?sort=new
```

常见输出字段：

```text
title
link
guid
pubDate
isoDate
creator
author
content
contentSnippet
```

常见报错：

```text
403：上游 Reddit 或本地代理被拦
ECONNREFUSED：本地代理 8787 没启动
Invalid RSS：返回的不是 RSS，而是错误页面
```

### 节点 5：Limit

节点类型：

```text
Limit
```

节点作用：

```text
限制每个 RSS 源最多处理多少条帖子。
```

当前配置：

```json
{
  "maxItems": 10
}
```

小白理解：

```text
RSS 可能一次返回很多帖子。
这个节点只保留前 10 条，避免一次跑太多。
```

测试建议：

```text
如果你只想测试一条，可以临时改成 1。
稳定后再改回 10。
```

### 节点 6：Build Local Proxy URL

节点类型：

```text
Code
```

节点作用：

```text
把 RSS 里的 Reddit 帖子信息整理成统一字段，并生成本地评论代理 URL。
```

这个节点还做了一次内存级去重：

```text
同一批 RSS 结果里，如果出现相同 reddit_id 或 source_url，只保留第一条。
```

当前代码：

```javascript
const seen = new Set();
return items
  .map(item => {
  const j = item.json;

  const sourceUrl = j.link || j.guid || "";

  const cleanUrl = sourceUrl
    .split("?")[0]
    .replace(/\/$/, "");

  const redditIdMatch = cleanUrl.match(/comments\/([^/]+)/);
  const redditId = redditIdMatch ? redditIdMatch[1] : "";

  // 这里改成你的本地中转服务接口地址
  const localProxyUrl = `http://127.0.0.1:8787/reddit-comments?url=${encodeURIComponent(cleanUrl)}`;

  return {
    json: {
      source: "Reddit",
      source_name: j.source_name || "",
      reddit_id: redditId,
      dup_key: redditId || cleanUrl,
      original_title: j.title || "",
      source_url: cleanUrl,
      local_proxy_url: localProxyUrl,
      published_at: j.isoDate || j.pubDate || "",
      author: j.creator || j.author || "",
      rss_content: j.content || "",
      rss_snippet: j.contentSnippet || ""
    }
  };
})
 .filter(item => {
   const key = item.json.dup_key;

    if (!key) return false;

    if (seen.has(key)) {
      return false;
    }

    seen.add(key);
    return true;
});
```

带注释解释版：

```javascript
// Set 用来记录这一批里已经见过的帖子，防止同一批重复
const seen = new Set();

return items
  .map(item => {
    // j 是 RSS Read 输出的单条帖子
    const j = item.json;

    // RSS 里通常 link 是原帖链接；如果没有 link，就用 guid 兜底
    const sourceUrl = j.link || j.guid || "";

    // 清理 URL：
    // 1. 去掉 ? 后面的追踪参数
    // 2. 去掉最后多余的 /
    const cleanUrl = sourceUrl
      .split("?")[0]
      .replace(/\/$/, "");

    // 从 Reddit URL 里提取帖子 ID
    // 例如 /comments/1abcxyz/title/ 会提取到 1abcxyz
    const redditIdMatch = cleanUrl.match(/comments\/([^/]+)/);
    const redditId = redditIdMatch ? redditIdMatch[1] : "";

    // 构造本地评论代理 URL
    // encodeURIComponent 是为了把 Reddit URL 安全地放进 query 参数里
    const localProxyUrl = `http://127.0.0.1:8787/reddit-comments?url=${encodeURIComponent(cleanUrl)}`;

    // 输出后续节点统一使用的字段
    return {
      json: {
        source: "Reddit",
        source_name: j.source_name || "",
        reddit_id: redditId,
        dup_key: redditId || cleanUrl,
        original_title: j.title || "",
        source_url: cleanUrl,
        local_proxy_url: localProxyUrl,
        published_at: j.isoDate || j.pubDate || "",
        author: j.creator || j.author || "",
        rss_content: j.content || "",
        rss_snippet: j.contentSnippet || ""
      }
    };
  })
  .filter(item => {
    // dup_key 是当前帖子用于查重的 key
    const key = item.json.dup_key;

    // 没有 key 的帖子不要继续处理
    if (!key) return false;

    // 如果这一批已经见过，就跳过
    if (seen.has(key)) {
      return false;
    }

    // 第一次见到，记录下来，并保留这条
    seen.add(key);
    return true;
  });
```

主要输出字段：

| 字段 | 含义 |
| --- | --- |
| `source` | 固定为 `Reddit` |
| `source_name` | 来源名称 |
| `reddit_id` | Reddit 帖子 ID |
| `dup_key` | 查重 key |
| `original_title` | RSS 里的标题 |
| `source_url` | 清洗后的 Reddit 链接 |
| `local_proxy_url` | 本地评论代理链接 |
| `published_at` | 发布时间 |
| `author` | 作者 |
| `rss_content` | RSS 正文 |
| `rss_snippet` | RSS 摘要 |

### 节点 7：Loop Over Items2

节点类型：

```text
Loop Over Items / Split In Batches
```

节点作用：

```text
逐个处理 Reddit 帖子。
```

当前配置：

```json
{
  "options": {}
}
```

小白理解：

```text
前面 Limit 最多给它 10 条帖子。
它一次只拿一条往后处理。
处理完再回来拿下一条。
```

连接说明：

```text
Loop 分支 -> Check Record DB
Done 分支 -> Loop Over Items
```

这表示：

```text
如果还有帖子，就继续处理帖子。
如果这个 RSS 源里的帖子处理完了，就回到第一个循环，处理下一个 RSS 源。
```

### 节点 8：Check Record DB

节点类型：

```text
Notion
```

节点作用：

```text
去 Reddit数据记录库 里查这条 Reddit 帖子是否已经处理过。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| Resource | `databasePage` |
| Operation | `getAll` |
| Database | `Reddit Record DB` |
| Database ID | `35dfed01-79aa-801b-8ebc-ce8d72b4a33f` |
| Limit | `1` |
| Filter Type | `json` |
| Credential | `Notion account` |

当前查重条件：

```json
{
  "or": [
    {
      "property": "dup_key",
      "rich_text": {
        "equals": "{{ $('Loop Over Items2').item.json.dup_key }}"
      }
    },
    {
      "property": "reddit_id",
      "rich_text": {
        "equals": "{{ $('Loop Over Items2').item.json.reddit_id }}"
      }
    },
    {
      "property": "source_url",
      "url": {
        "equals": "{{ $('Loop Over Items2').item.json.source_url }}"
      }
    }
  ]
}
```

小白理解：

```text
它会问 Notion：
这个 dup_key、reddit_id 或 source_url 以前有没有记录过？
只要任意一个匹配，就认为处理过。
```

### 节点 9：IF Record Missing

节点类型：

```text
IF
```

节点作用：

```text
判断 Check Record DB 有没有查到记录。
```

当前条件：

```text
{{ $json.id || "" }} equals ""
```

小白理解：

```text
如果 Notion 查到了记录，输出里通常会有 id。
如果没有查到，id 就是空。
```

连接逻辑：

```text
true 分支：没有处理记录 -> 继续查内容库
false 分支：已经处理过 -> 回到 Loop Over Items2，处理下一条
```

这个节点是防重复处理的第一道门。

### 节点 10：Check Content DB

节点类型：

```text
Notion
```

节点作用：

```text
去 入境游内容库 里查这条 Reddit 帖子是否已经入库。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| Resource | `databasePage` |
| Operation | `getAll` |
| Database | `Inbound Travel Content DB` |
| Database ID | `35dfed01-79aa-8061-8a0a-e321a6cf38b3` |
| Limit | `1` |
| Filter Type | `json` |
| Credential | `Notion account` |

当前查重条件：

```json
{
  "or": [
    {
      "property": "reddit_id",
      "rich_text": {
        "equals": "{{ $('Loop Over Items2').item.json.reddit_id }}"
      }
    },
    {
      "property": "source_url ",
      "url": {
        "equals": "{{ $('Loop Over Items2').item.json.source_url }}"
      }
    }
  ]
}
```

注意：

```text
这里的 source_url 字段名后面有一个空格：source_url 
这是你 Notion 内容库里的真实字段名。
不要随便删掉这个空格，除非同步修改 Notion 字段和 n8n 配置。
```

小白理解：

```text
即使记录库里没有，也要再检查内容库。
这样可以避免某些历史内容已经写入内容库，但记录库没记录，导致重复写入。
```

### 节点 11：IF Content Missing

节点类型：

```text
IF
```

节点作用：

```text
判断 Check Content DB 有没有查到内容库记录。
```

当前条件：

```text
{{ $json.id || "" }} equals ""
```

连接逻辑：

```text
true 分支：内容库没有这条 -> 抓 Reddit 评论区
false 分支：内容库已有这条 -> 回到 Loop Over Items2，处理下一条
```

小白理解：

```text
这是防重复写入内容库的第二道门。
```

### 节点 12：Fetch Reddit Via Local Proxy

节点类型：

```text
HTTP Request
```

节点作用：

```text
请求本地 Reddit 代理，获取原帖和评论区 JSON。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| Method | 默认 GET |
| URL | `http://127.0.0.1:8787/reddit-comments` |
| Send Query | `true` |
| Specify Query | `json` |
| Response Format | `json` |

Query 参数：

```json
{
  "url": "{{ $('Loop Over Items2').item.json.source_url }}",
  "limit": 100,
  "depth": 6
}
```

小白理解：

```text
n8n 不直接请求 Reddit。
n8n 请求本地代理。
本地代理再去抓 Reddit 评论区。
```

参数含义：

| 参数 | 含义 |
| --- | --- |
| `url` | Reddit 原帖链接 |
| `limit` | 最多抓 100 条评论 |
| `depth` | 最多抓 6 层评论深度 |

常见报错：

```text
ECONNREFUSED：本地代理没启动
502：代理抓 Reddit 失败
403：Reddit 拦截了代理请求
JSON parse error：代理返回的不是 JSON
```

### 节点 13：Format Reddit Post And Comments

节点类型：

```text
Code
```

节点作用：

```text
把代理返回的 Reddit 原帖和评论整理成 AI 可读的输入文本。
```

它会做这些事情：

```text
清理 HTML 标签
清理 Reddit 的 [deleted] / [removed]
提取原帖标题和正文
提取评论
按评论分数排序
只保留前 20 条有用评论
拼成 ai_input 给 DeepSeek
```

当前代码：

```javascript
function cleanText(text) {
  return String(text || "")
    .replace(/<[^>]*>/g, " ")
    .replace(/&nbsp;/g, " ")
    .replace(/&amp;/g, "&")
    .replace(/&lt;/g, "<")
    .replace(/&gt;/g, ">")
    .replace(/&#32;/g, " ")
    .replace(/\[deleted\]/gi, "")
    .replace(/\[removed\]/gi, "")
    .replace(/\s+/g, " ")
    .trim();
}

return items.map(item => {
  const j = item.json;
  const post = j.post || {};

  const title = cleanText(post.title || "");
  const postText = cleanText(post.selftext || "");
  const sourceUrl = j.source_url || "";
  const redditId = post.id || "";
  const subreddit = post.subreddit || "";
  const author = post.author || "";
  const score = post.score || 0;
  const numComments = post.num_comments || 0;
  const createdUtc = post.created_utc || "";

  const rawComments = Array.isArray(j.flat_comments)
    ? j.flat_comments
    : Array.isArray(j.comments)
      ? j.comments
      : [];

  const usefulComments = rawComments
    .map(c => ({
      author: c.author || "",
      score: c.score || 0,
      depth: c.depth || 0,
      body: cleanText(c.body || c.text || "")
    }))
    .filter(c => c.body.length > 10)
    .sort((a, b) => b.score - a.score)
    .slice(0, 20);

  const commentsText = usefulComments
    .map((c, index) => {
      return `Comment ${index + 1} | score: ${c.score} | author: ${c.author} | depth: ${c.depth}\n${c.body}`;
    })
    .join("\n\n---\n\n");

  const aiInput = `
Reddit Source:
r/${subreddit}

Post Title:
${title}

Post Text:
${postText || "No post body."}

Top Comments:
${commentsText || "No useful comments found."}

Post Score:
${score}

Number of Comments:
${numComments}

Source URL:
${sourceUrl}
  `.trim();

  return {
    json: {
      source: "Reddit",
      source_name: subreddit ? `r/${subreddit}` : "",
      reddit_id: redditId,
      dup_key: redditId || sourceUrl,
      source_url: sourceUrl,
      original_title: title,
      original_text: postText,
      author,
      post_score: score,
      comments_count: rawComments.length,
      useful_comments_count: usefulComments.length,
      top_comments_text: commentsText.slice(0, 8000),
      ai_input: aiInput.slice(0, 12000),
      created_utc: createdUtc
    }
  };
});
```

带注释解释版：

```javascript
// 清理 Reddit 文本，避免 HTML 标签、实体字符、删除标记干扰 AI 判断
function cleanText(text) {
  return String(text || "")
    .replace(/<[^>]*>/g, " ")       // 去掉 HTML 标签
    .replace(/&nbsp;/g, " ")        // HTML 空格
    .replace(/&amp;/g, "&")         // &amp; 还原成 &
    .replace(/&lt;/g, "<")          // &lt; 还原成 <
    .replace(/&gt;/g, ">")          // &gt; 还原成 >
    .replace(/&#32;/g, " ")         // HTML 空格编码
    .replace(/\[deleted\]/gi, "")   // 去掉 deleted 评论
    .replace(/\[removed\]/gi, "")   // 去掉 removed 评论
    .replace(/\s+/g, " ")           // 多个空白合并成一个空格
    .trim();                        // 去掉首尾空格
}

return items.map(item => {
  // j 是本地 Reddit 代理返回的 JSON
  const j = item.json;

  // post 是原帖对象
  const post = j.post || {};

  // 原帖标题和正文
  const title = cleanText(post.title || "");
  const postText = cleanText(post.selftext || "");

  // 原帖元信息
  const sourceUrl = j.source_url || "";
  const redditId = post.id || "";
  const subreddit = post.subreddit || "";
  const author = post.author || "";
  const score = post.score || 0;
  const numComments = post.num_comments || 0;
  const createdUtc = post.created_utc || "";

  // 代理可能返回 flat_comments，也可能返回 comments
  // 优先使用 flat_comments，因为它已经被拉平成一维列表，更方便处理
  const rawComments = Array.isArray(j.flat_comments)
    ? j.flat_comments
    : Array.isArray(j.comments)
      ? j.comments
      : [];

  // 整理评论：
  // 1. 提取 author / score / depth / body
  // 2. 去掉太短的评论
  // 3. 按 score 从高到低排序
  // 4. 只保留前 20 条
  const usefulComments = rawComments
    .map(c => ({
      author: c.author || "",
      score: c.score || 0,
      depth: c.depth || 0,
      body: cleanText(c.body || c.text || "")
    }))
    .filter(c => c.body.length > 10)
    .sort((a, b) => b.score - a.score)
    .slice(0, 20);

  // 把评论拼成一段文本，给 AI 阅读
  const commentsText = usefulComments
    .map((c, index) => {
      return `Comment ${index + 1} | score: ${c.score} | author: ${c.author} | depth: ${c.depth}\n${c.body}`;
    })
    .join("\n\n---\n\n");

  // 这是最终喂给 AI 的完整输入
  const aiInput = `
Reddit Source:
r/${subreddit}

Post Title:
${title}

Post Text:
${postText || "No post body."}

Top Comments:
${commentsText || "No useful comments found."}

Post Score:
${score}

Number of Comments:
${numComments}

Source URL:
${sourceUrl}
  `.trim();

  // 输出统一字段，供 AI、Notion 节点使用
  return {
    json: {
      source: "Reddit",
      source_name: subreddit ? `r/${subreddit}` : "",
      reddit_id: redditId,
      dup_key: redditId || sourceUrl,
      source_url: sourceUrl,
      original_title: title,
      original_text: postText,
      author,
      post_score: score,
      comments_count: rawComments.length,
      useful_comments_count: usefulComments.length,

      // 限制长度，避免后面 Notion 或 AI 输入过长
      top_comments_text: commentsText.slice(0, 8000),
      ai_input: aiInput.slice(0, 12000),

      created_utc: createdUtc
    }
  };
});
```

主要输出字段：

| 字段 | 含义 |
| --- | --- |
| `reddit_id` | Reddit 帖子 ID |
| `source_url` | Reddit 原帖链接 |
| `original_title` | 清理后的标题 |
| `original_text` | 清理后的正文 |
| `author` | 作者 |
| `post_score` | 原帖分数 |
| `comments_count` | 抓到的评论数量 |
| `useful_comments_count` | 被保留给 AI 的评论数量 |
| `top_comments_text` | 前 20 条高价值评论文本 |
| `ai_input` | 给 AI 判断用的完整输入 |

### 节点 14：AI 判断 + 格式化1

节点类型：

```text
Basic LLM Chain
```

节点作用：

```text
让 DeepSeek 判断这条 Reddit 帖子是否值得进入入境游内容库，并输出 JSON。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| Prompt Type | `define` |
| Batching Batch Size | `1` |
| Delay Between Batches | `1000` ms |

主 Prompt：

```text
你是一个专门研究入境游需求的内容分析助手。

请阅读下面 Reddit 帖子和评论区，判断它是否值得进入我的“入境游内容选题库”。

判断标准：
1. 是否包含外国游客来中国旅行的痛点、疑问、担忧或决策困难
2. 是否包含评论区给出的实用解决方案
3. 是否可以改写成社交媒体内容
4. 如果只是普通风景分享、照片分享、历史介绍、没有明确问题，则判定为不相关

输入内容：
{{ $json.ai_input }}

请只输出 JSON，不要 Markdown，不要解释，不要代码块。

输出格式：
{
  "is_relevant": true,
  "content_type": "痛点/解决方案/经验分享/普通分享",
  "category": "支付/交通/住宿/签证/景点/语言/网络/安全/路线/其他",
  "pain_point_summary": "一句话总结用户痛点，如果没有则为空字符串",
  "solution_summary": "一句话总结评论区或帖子中的解决方案，如果没有则为空字符串",
  "content_angle": "这个帖子可以做成什么内容角度，如果不相关则为空字符串",
  "hook": "一个英文社交媒体标题，如果不相关则为空字符串",
  "evidence": "引用帖子或评论中的关键依据"
}
```

System Message：

```text
You are an inbound China travel intelligence analyst for U Travel.

Your job is to analyze Reddit posts and decide whether they contain useful pain points, questions, confusions, fears, objections, or practical solutions from foreign travelers visiting China.

U Travel focuses on helping foreign visitors experience China like locals, especially Chengdu and Sichuan. The brand tone is local, practical, friendly, non-salesy, and experience-based.

Important rules:
1. Output only valid JSON.
2. Do not use Markdown.
3. Do not wrap the JSON in code blocks.
4. Do not output null.
5. Always output the same JSON structure.
6. If the post is irrelevant, set "is_relevant" to false and explain the reason in "reject_reason".
7. For irrelevant posts, still fill every required string field with an empty string or short reason, scores with 1, and hook_titles with 3 short placeholders.
8. Avoid commercial language. Do not write "book a tour", "DM me", "contact me", or "I can guide you".
9. Hook titles should sound natural for Reddit, not like AI marketing copy.
10. Focus on real travel pain points: visa, payment, transport, food, language, apps, itinerary, safety, culture shock, TCM, local experiences, Chengdu, Sichuan, western China, accommodation, SIM/eSIM, translation, and first-time China travel.
11. Use enum values exactly when choosing content_type, category, emotion, and travel_stage.
```

小白理解：

```text
这个节点会读 ai_input。
如果它认为帖子是有价值的入境游痛点，就输出 is_relevant: true。
如果只是普通分享，就输出 is_relevant: false。
```

注意：

```text
当前 Prompt 要求 AI 输出一些字段，但 System Message 里还提到了 reject_reason、scores、hook_titles、emotion、travel_stage。
这些字段没有在后续 Code 节点中使用。
当前工作流真正使用的是：
is_relevant
content_type
category
pain_point_summary
solution_summary
content_angle
hook
evidence
```

### 节点 15：DeepSeek Chat Model1

节点类型：

```text
DeepSeek Chat Model
```

节点作用：

```text
给 AI 判断节点提供大模型。
```

当前配置：

```json
{
  "options": {}
}
```

凭证：

```text
DeepSeek account
```

小白理解：

```text
AI 判断 + 格式化1 本身只是链条节点。
真正的大模型能力来自这个 DeepSeek Chat Model1。
```

当前没有设置：

```text
temperature
maxTokens
responseFormat
```

可选优化：

```text
后期如果 AI JSON 不稳定，可以把 temperature 调低，比如 0.2。
如果 DeepSeek 节点支持 JSON 模式，也可以设置 responseFormat = json_object。
```

### 节点 16：Code 解析AI JSON1

节点类型：

```text
Code
```

节点作用：

```text
解析 AI 输出的 JSON，把字段整理成后续 Notion 节点能使用的结构。
```

为什么需要这个节点：

```text
AI 有时会输出纯 JSON。
有时会包在 ```json 代码块里。
有时前后会多写几句话。
这个节点负责尽量提取里面真正的 JSON。
```

当前代码：

```javascript
function extractJson(text) {
  const raw = String(text || "").trim();

  const withoutCodeFence = raw
    .replace(/^```json\s*/i, "")
    .replace(/^```\s*/i, "")
    .replace(/```$/i, "")
    .trim();

  try {
    return JSON.parse(withoutCodeFence);
  } catch (e) {
    const match = withoutCodeFence.match(/\{[\s\S]*\}/);
    if (match) {
      return JSON.parse(match[0]);
    }
    throw new Error("AI 输出不是有效 JSON: " + raw);
  }
}

return items.map(item => {
  const j = item.json;

  const aiRaw =
    j.text ||
    j.output ||
    j.response ||
    j.result ||
    j.message ||
    "";

  const parsed = extractJson(aiRaw);

  return {
    json: {
      ...j,
      ai_raw: aiRaw,
      is_relevant: Boolean(parsed.is_relevant),
      content_type: parsed.content_type || "",
      category: parsed.category || "",
      pain_point_summary: parsed.pain_point_summary || "",
      solution_summary: parsed.solution_summary || "",
      content_angle: parsed.content_angle || "",
      hook: parsed.hook || "",
      evidence: parsed.evidence || ""
    }
  };
});
```

带注释解释版：

```javascript
// 从 AI 输出文本里提取 JSON
function extractJson(text) {
  // 把输入转成字符串，并去掉首尾空格
  const raw = String(text || "").trim();

  // 如果 AI 输出了 ```json ... ```，先把代码块标记去掉
  const withoutCodeFence = raw
    .replace(/^```json\s*/i, "")
    .replace(/^```\s*/i, "")
    .replace(/```$/i, "")
    .trim();

  try {
    // 第一种情况：AI 输出本身就是干净 JSON
    return JSON.parse(withoutCodeFence);
  } catch (e) {
    // 第二种情况：AI 前后多写了文字，就尝试提取第一个 {...}
    const match = withoutCodeFence.match(/\{[\s\S]*\}/);
    if (match) {
      return JSON.parse(match[0]);
    }

    // 如果还是解析不了，就抛出错误，让 n8n 显示问题
    throw new Error("AI 输出不是有效 JSON: " + raw);
  }
}

return items.map(item => {
  // j 是 AI 节点输出
  const j = item.json;

  // 不同 n8n/LLM 节点可能把 AI 文本放在不同字段里
  // 这里按常见字段顺序兜底查找
  const aiRaw =
    j.text ||
    j.output ||
    j.response ||
    j.result ||
    j.message ||
    "";

  // 解析 AI JSON
  const parsed = extractJson(aiRaw);

  // 输出后续节点需要的字段
  return {
    json: {
      ...j,
      ai_raw: aiRaw,
      is_relevant: Boolean(parsed.is_relevant),
      content_type: parsed.content_type || "",
      category: parsed.category || "",
      pain_point_summary: parsed.pain_point_summary || "",
      solution_summary: parsed.solution_summary || "",
      content_angle: parsed.content_angle || "",
      hook: parsed.hook || "",
      evidence: parsed.evidence || ""
    }
  };
});
```

主要输出字段：

| 字段 | 含义 |
| --- | --- |
| `ai_raw` | AI 原始输出 |
| `is_relevant` | 是否相关 |
| `content_type` | 内容类型 |
| `category` | 分类 |
| `pain_point_summary` | 痛点总结 |
| `solution_summary` | 解决方案总结 |
| `content_angle` | 内容角度 |
| `hook` | 英文标题 |
| `evidence` | 关键依据 |

### 节点 17：存处理记录

节点类型：

```text
Notion
```

节点作用：

```text
把这条 Reddit 帖子写入 Reddit数据记录库，表示它已经被处理过。
```

重要：

```text
无论 AI 判断相关还是不相关，都会先写处理记录。
这样下次不会重复分析同一个帖子。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| Resource | `databasePage` |
| Database | `Reddit数据记录库` |
| Database ID | `35dfed01-79aa-801b-8ebc-ce8d72b4a33f` |
| Simple | `false` |
| Credential | `Notion account` |

字段映射：

| Notion 字段 | 写入值 |
| --- | --- |
| `is_relevant` | `={{ $json.is_relevant }}` |
| `name` | `={{ $('Format Reddit Post And Comments').item.json.original_title }}` |
| `dup_key` | `={{ $('Format Reddit Post And Comments').item.json.dup_key }}` |
| `original_title` | `={{ $('Format Reddit Post And Comments').item.json.original_title }}` |
| `reddit_id` | `={{ $('Format Reddit Post And Comments').item.json.reddit_id }}` |
| `source_url` | `={{ $('Format Reddit Post And Comments').item.json.source_url }}` |
| `source_name ` | `={{ $('Format Reddit Post And Comments').item.json.source_name }}` |
| `processed_at  ` | `2026-05-11T16:47:14` |

注意：

```text
source_name 后面有一个空格。
processed_at 后面有两个空格。
这是 Notion 字段名里的真实空格。
```

另一个注意点：

```text
processed_at 当前是固定时间：2026-05-11T16:47:14。
如果你希望记录真实处理时间，后期建议改成：
{{ $now }}
```

### 节点 18：Filter 筛选相关内容1

节点类型：

```text
Filter
```

节点作用：

```text
只让 AI 判断为相关的内容进入入境游内容库。
```

当前条件：

```text
{{ ($json.is_relevant === true || $json.is_relevant === "true").toString() }} equals true
```

小白理解：

```text
如果 AI 输出 is_relevant = true，就继续写内容库。
如果 AI 输出 is_relevant = false，就跳过内容库，回到 Loop Over Items2 处理下一条。
```

连接逻辑：

```text
true 分支 -> 存入notion内容库
false 分支 -> Loop Over Items2
```

### 节点 19：存入notion内容库

节点类型：

```text
Notion
```

节点作用：

```text
把 AI 判断为相关的帖子写入入境游内容库。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| Resource | `databasePage` |
| Database | `入境游内容库` |
| Database ID | `35dfed01-79aa-8061-8a0a-e321a6cf38b3` |
| Credential | `Notion account` |

字段映射：

| Notion 字段 | 写入值 |
| --- | --- |
| `name` | `={{ $json.hook }}` |
| `author` | `={{ $('Format Reddit Post And Comments').item.json.author }}` |
| `category` | `={{ $json.category }}` |
| `comments_count` | `={{ $('Format Reddit Post And Comments').item.json.comments_count }}` |
| `content_type` | `={{ $('Code 解析AI JSON1').item.json.content_type }}` |
| `evidence` | `={{ $('Code 解析AI JSON1').item.json.evidence }}` |
| `hook` | `={{ $('Code 解析AI JSON1').item.json.hook }}` |
| `original_title ` | `={{ $('Format Reddit Post And Comments').item.json.original_title }}` |
| `pain_point_summary ` | `={{ $('Code 解析AI JSON1').item.json.pain_point_summary }}` |
| `post_score` | `={{ $('Format Reddit Post And Comments').item.json.post_score }}` |
| `reddit_id` | `={{ $('Format Reddit Post And Comments').item.json.reddit_id }}` |
| `solution_summary` | `={{ $('Code 解析AI JSON1').item.json.solution_summary }}` |
| `source_name ` | `={{ $('Format Reddit Post And Comments').item.json.source_name }}` |
| `source_url ` | `={{ $('Format Reddit Post And Comments').item.json.source_url }}` |

注意：

```text
original_title、pain_point_summary、source_name、source_url 这些字段名后面都有空格。
这是当前 Notion 库里的真实字段名。
```

写入成功后：

```text
这条内容会进入入境游内容库。
后续 Engine 2 可以继续筛选这些内容，做解决方案研究。
```

## 8. 一条帖子完整跑完会发生什么

假设 RSS 抓到一条 Reddit 帖子：

```text
Title: Can I use Alipay as a foreigner in China?
URL: https://www.reddit.com/r/travelchina/comments/xxxx/...
```

完整流程是：

```text
1. Build Local Proxy URL 提取 reddit_id 和 source_url
2. Loop Over Items2 开始逐条处理
3. Check Record DB 查记录库
4. 如果记录库已有，跳过
5. 如果记录库没有，Check Content DB 查内容库
6. 如果内容库已有，跳过
7. 如果内容库也没有，Fetch Reddit Via Local Proxy 抓评论
8. Format Reddit Post And Comments 整理给 AI 的 ai_input
9. AI 判断 + 格式化1 判断是否相关
10. Code 解析AI JSON1 解析 AI 输出
11. 存处理记录 写入 Reddit数据记录库
12. Filter 筛选相关内容1 判断 is_relevant
13. 如果 relevant，存入notion内容库
14. 回到 Loop Over Items2 处理下一条
```

## 9. 如何手动测试

### 第一步：确认本地代理启动

浏览器或 HTTP 工具能打开：

```text
http://127.0.0.1:8787/reddit/travelchina.rss?sort=new
```

### 第二步：打开工作流

```text
http://localhost:5678/workflow/aOUAKkEsQnXGAs8U
```

### 第三步：临时把 Limit 改成 1

如果只是测试，建议：

```text
Limit -> Max Items = 1
```

这样只处理一条 Reddit 帖子，方便排查。

### 第四步：手动运行

点击：

```text
Execute Workflow
```

### 第五步：按顺序检查输出

重点检查这些节点：

```text
RSS Read
Build Local Proxy URL
Check Record DB
Check Content DB
Fetch Reddit Via Local Proxy
Format Reddit Post And Comments
AI 判断 + 格式化1
Code 解析AI JSON1
存处理记录
存入notion内容库
```

如果 `Code 解析AI JSON1` 输出里有这些字段，就说明 AI 判断阶段基本正常：

```text
is_relevant
content_type
category
pain_point_summary
solution_summary
hook
evidence
```

## 10. 常见问题排查

### 问题 1：RSS Read 报 403

可能原因：

```text
Reddit 拦截请求
本地代理没有正确伪装请求头
代理服务异常
```

处理方式：

```text
先在浏览器打开 http://127.0.0.1:8787/reddit/travelchina.rss?sort=new
如果浏览器也打不开，先修本地代理
```

### 问题 2：RSS Read 只有 1 条输入

这是正常的。

原因：

```text
Loop Over Items 当前一次只把 1 个 RSS 源交给 RSS Read。
不是只抓 1 条帖子。
真正的帖子数量看 RSS Read 的输出和 Limit 节点。
```

### 问题 3：后面每次只处理一条帖子

这是 Loop Over Items2 的正常行为。

原因：

```text
Loop Over Items2 一次只放一条帖子给后面的查重、评论抓取、AI 判断节点。
处理完再回到 Loop Over Items2 继续下一条。
```

### 问题 4：Fetch Reddit Via Local Proxy 报 502

可能原因：

```text
本地代理抓 Reddit 评论失败
Reddit 暂时拦截
帖子 URL 异常
评论区访问失败
```

处理方式：

```text
复制 Loop Over Items2 输出里的 source_url
手动访问：
http://127.0.0.1:8787/reddit-comments?url=你的Reddit链接
```

### 问题 5：AI 节点输出不是 JSON

报错节点通常是：

```text
Code 解析AI JSON1
```

可能原因：

```text
DeepSeek 输出了 Markdown
DeepSeek 多写了解释
DeepSeek 少输出字段
输入太长导致截断
```

处理方式：

```text
先看 AI 判断 + 格式化1 的输出
确认它是不是纯 JSON
```

可选优化：

```text
给 DeepSeek Chat Model1 设置 temperature = 0.2
如果支持 JSON 模式，开启 responseFormat = json_object
```

### 问题 6：Notion 查重没有生效

重点检查：

```text
Check Record DB 的 dup_key / reddit_id / source_url 是否有值
Check Content DB 的 reddit_id / source_url 是否有值
Notion 字段名是否和节点配置一致
字段名后面的空格是否被改掉
```

尤其是内容库字段：

```text
source_url 
```

后面有一个空格。

### 问题 7：不相关内容也进了内容库

重点检查：

```text
Code 解析AI JSON1 输出的 is_relevant 是什么
Filter 筛选相关内容1 的条件是否为 true
AI 是否错误判断了相关性
```

如果 AI 判断太宽松，需要调整 `AI 判断 + 格式化1` 的 Prompt。

### 问题 8：相关内容没有进内容库

重点检查：

```text
AI 是否输出 is_relevant: true
Code 解析AI JSON1 是否正确解析
Filter 筛选相关内容1 是否走 true 分支
存入notion内容库 是否报字段类型错误
```

## 11. 当前值得注意的配置

### 11.1 Limit 当前是 10

```text
Limit -> Max Items = 10
```

这表示每个 RSS 源最多处理 10 条帖子。

测试时可以改成：

```text
1
```

### 11.2 记录库 processed_at 当前是固定时间

当前：

```text
2026-05-11T16:47:14
```

建议后期改成：

```text
{{ $now }}
```

这样 Notion 里会记录真实处理时间。

### 11.3 DeepSeek 当前没有细调参数

当前：

```json
{
  "options": {}
}
```

如果 JSON 稳定性不够，建议后期设置：

```text
temperature = 0.2
maxTokens = 1500 或 2000
responseFormat = json_object
```

### 11.4 本地代理是核心依赖

这个工作流能不能抓评论，关键不在 n8n，而在：

```text
http://127.0.0.1:8787/reddit-comments
```

如果 Reddit 抓取出问题，先查代理服务。

## 12. 字段来源关系

### RSS 阶段生成

来自 `RSS Read` 和 `Build Local Proxy URL`：

```text
feed_url
source_name
reddit_id
dup_key
original_title
source_url
local_proxy_url
published_at
author
rss_content
rss_snippet
```

### 评论抓取阶段生成

来自 `Fetch Reddit Via Local Proxy` 和 `Format Reddit Post And Comments`：

```text
original_text
post_score
comments_count
useful_comments_count
top_comments_text
ai_input
created_utc
```

### AI 阶段生成

来自 `AI 判断 + 格式化1` 和 `Code 解析AI JSON1`：

```text
is_relevant
content_type
category
pain_point_summary
solution_summary
content_angle
hook
evidence
ai_raw
```

### Notion 写入阶段使用

写入记录库：

```text
is_relevant
name
dup_key
original_title
reddit_id
source_url
source_name
processed_at
```

写入内容库：

```text
name
author
category
comments_count
content_type
evidence
hook
original_title
pain_point_summary
post_score
reddit_id
solution_summary
source_name
source_url
```

## 13. 推荐的新手调试顺序

如果你不知道哪里坏了，不要从最后开始看。

按这个顺序：

```text
1. RSS Read 是否有多条帖子
2. Build Local Proxy URL 是否有 reddit_id 和 source_url
3. Check Record DB 是否返回空或已有记录
4. IF Record Missing 分支是否正确
5. Check Content DB 是否返回空或已有记录
6. IF Content Missing 分支是否正确
7. Fetch Reddit Via Local Proxy 是否拿到 post 和 comments
8. Format Reddit Post And Comments 是否生成 ai_input
9. AI 判断 + 格式化1 是否输出 JSON
10. Code 解析AI JSON1 是否解析出 is_relevant
11. 存处理记录 是否成功
12. Filter 筛选相关内容1 是否走正确分支
13. 存入notion内容库 是否成功
```

## 14. 工作流安全边界

这个工作流里 AI 只做判断和格式化。

AI 不直接写 Notion。

真正写 Notion 的只有两个普通 Notion 节点：

```text
存处理记录
存入notion内容库
```

这样设计比较安全：

```text
AI 不能越权乱改数据库
写入字段都能在 n8n 节点里清楚看到
哪里出错更容易定位
```

## 15. 和 Engine 2 的关系

Engine 1 结束后，如果 AI 判断相关，会写入：

```text
入境游内容库
```

Engine 2 会继续从这个内容库里找待调研内容。

一般关系是：

```text
Engine 1：发现痛点
Engine 2：研究解决方案
Engine 3：生成内容草稿
```

所以这个工作流的输出质量会直接影响后面的 Engine 2。

如果 Engine 1 误判太多，Engine 2 就会浪费时间研究低价值内容。

## 16. 最小可用检查清单

每次运行前快速检查：

```text
1. n8n 已启动
2. 本地 Reddit 代理 8787 已启动
3. RSS 地址能打开
4. Notion account 可用
5. DeepSeek account 可用
6. Limit 测试阶段最好是 1
7. Check Record DB 查重字段正常
8. Check Content DB 查重字段正常
9. AI 输出是 JSON
10. Notion 内容库没有重复写入
```

