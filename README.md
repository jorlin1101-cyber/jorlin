# Engine 1 - Reddit 情报采集器 README

这份 README 对应 n8n 工作流：

```
http://localhost:5678/workflow/aOUAKkEsQnXGAs8U
```

工作流 ID：

```
aOUAKkEsQnXGAs8U
```

## 1. 工作流描述

这个工作流是你的 Engine 1，负责从 Reddit 发现入境游相关内容，并把有价值的帖子写入 Notion。

它的核心目标是：

```
抓 Reddit 帖子列表
-> 对每条帖子查重
-> 抓原帖和评论区
-> 整理成 AI 可读文本
-> 用 DeepSeek 判断是否是有价值的入境游痛点
-> 记录处理历史
-> 如果相关，就写入入境游内容库
```

## 2. 当前工作流概览

```markdown
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
```

## 3. 运行前必须确认

### 3.1 n8n 必须启动

浏览器能打开：

```
http://localhost:5678
```

### 3.2 本地 Reddit 代理必须启动

这个工作流不直接请求 Reddit，而是请求你的本地代理：

```
http://127.0.0.1:8787
```

当前会用到两个接口：

```
http://127.0.0.1:8787/reddit/travelchina.rss?sort=new
http://127.0.0.1:8787/reddit-comments
```

如果 `RSS Read` 报 403、502、连接失败，优先检查这个本地代理是否还在运行。

### 3.3 Notion 凭证必须可用

当前使用的 Notion 凭证：

```
Notion account
```

会写入两个 Notion 数据库：

```
Reddit数据记录库
入境游内容库
```

### 3.4 DeepSeek 凭证必须可用

当前 AI 节点使用：

```
DeepSeek account
```

如果 AI 节点报鉴权错误，优先检查这个凭证。

## 4. 整体流程图

```
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

## 5. 这个工作流里有两个循环

### 第一个循环：Loop Over Items

循环作用：

```
逐个处理 RSS 源。
```

现在只有一个 RSS 源：

```
r/travelchina new
```

### 第二个循环：Loop Over Items2

作用：

```
逐个处理 RSS 源里抓到的 Reddit 帖子。
```

每一条 Reddit 帖子都会经过：

```
查记录库
查内容库
抓评论
AI 判断
写 Notion
```

处理完一条后，再回到 `Loop Over Items2` 处理下一条。

## 6. 查重逻辑说明

这个工作流做了双重查重：

```
第一层：查 Reddit数据记录库
第二层：查 入境游内容库
```

为什么要双重查重：

```
记录库：防止同一个 Reddit 帖子被重复处理
内容库：防止已经入库的内容再次写入内容库
```

查重用到的字段：

```
dup_key
reddit_id
source_url
```

其中：

```
reddit_id 是 Reddit 帖子 ID
source_url 是 Reddit 原帖链接
dup_key 优先使用 reddit_id，如果没有 reddit_id 就使用 source_url
```

## 7. 每个节点详细说明

#### 节点 1：Schedule Trigger

节点类型：

```
n8n-nodes-base.scheduleTrigger
```

节点作用：

```
Schedule Trigger可以设置按照每月、每周、每天、每时或者自定义来进行时间触发，本项目设置为每隔6小时进行工作流触发
```

当前配置：



#### 节点 2：Feed URL List

节点类型：

```
Code
```

节点作用：

```
生成要抓取的 Reddit RSS 源列表。
```

当前只配置了一个 RSS 源：

```
http://127.0.0.1:8787/reddit/travelchina.rss?sort=new
```

代码为

```jsx
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

```jsx
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

feed_url：代表你想要访问的链接，本链接由于访问reddit的1个sub，所以对应1个url，

127.0.0.1:8787：由于reddit不能直接通过n8n访问RSS，所以本项目通过建立中转代理服务来解决，此码代表我本机建立的中转代理服务地址；

q=China%20travel&restrict_sr=1&sort=new：%20表示空格，更好的写法为encodeURIComponent("Chengdu travel tips")，它会自动变成Chengdu%20travel%20tips

- 更好维护版本如下：
    
    ```jsx
    const feeds = [
      {
        subreddit: "travel",
        keyword: "China travel",
        source_name: "r/travel China travel"
      },
      {
        subreddit: "solotravel",
        keyword: "China",
        source_name: "r/solotravel China"
      }
    ];
    
    return feeds.map(feed => {
      const keyword = encodeURIComponent(feed.keyword);
    
      return {
        json: {
          feed_url: `http://127.0.0.1:8787/reddit/${feed.subreddit}/search.rss?q=${keyword}&restrict_sr=1&sort=new`,
          source_name: feed.source_name
        }
      };
    });
    ```
    
    其中const feeds = [ 指的是建立一个叫feeds的列表
    
    return feeds.map(feed => {  意思是： 把 `feeds` 这个列表里的每一个链接，都转换成 n8n 能识别的格式；map指的是逐个处理
    
    第二个return是返回给n8n的数据
    

#### 节点 3：Loop Over Items

节点类型：

```
Loop Over Items 
```

节点作用：

```
逐个处理 Feed URL List 里的 RSS 源。
```

注意：

```
Loop Over Items2 处理完一个 RSS 源里的所有帖子后，会回到这个节点，继续处理下一个 RSS 源。
```

#### 节点 4：RSS Read

节点类型：

```
RSS Feed Read
```

节点作用：

```
读取上一个节点传来的 RSS 地址，抓 Reddit 帖子列表。
```

当前配置：

![image.png](image%201.png)

常见输出字段：

```
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

#### 节点 5：Limit

节点类型：

```
Limit
```

节点作用：

```
限制每个 RSS 源最多处理多少条帖子。
```

当前配置：

![image.png](image%202.png)

keep可以用于设置保存最前面几条还是最后面几条

#### 节点 6：Build Local Proxy URL

节点类型：

```
Code
```

节点作用：

```
把 RSS 里的 Reddit 帖子信息整理成统一字段，并生成本地评论代理 URL。
```

这个节点还做了一次内存级去重：

```
同一批 RSS 结果里，如果出现相同 reddit_id 或 source_url，只保留第一条。
```

当前代码：

```jsx
const seen = new Set();//set可以理解为一个内容不重复的库

return items          //把每一条items处理后，返回给下个节点
  .map(item => {
    // j 是 RSS Read 输出的单条帖子
    const j = item.json;

    // RSS 里通常 link 是原帖链接；如果没有 link，就用 guid 兜底
    const sourceUrl = j.link || j.guid || "";

    // 清理 URL：
    const cleanUrl = sourceUrl
      .split("?")[0]       //将？后的内容分解开，取第0段
      .replace(/\/$/, "");  //删除最后的/

    // 从 Reddit URL 里提取帖子 ID
    // 例如 /comments/1abcxyz/title/ 会提取到 1abcxyz
    const redditIdMatch = cleanUrl.match(/comments\/([^/]+)/);
    const redditId = redditIdMatch ? redditIdMatch[1] : "";

    // 构造本地评论代理 URL
    // encodeURIComponent 是为了把 Reddit URL 安全地放进 query 参数里
    const localProxyUrl = `http://127.0.0.1:8787/reddit-comments?url=${encodeURIComponent(cleanUrl)}`;
    //``用于把变量写入字符串
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

```markdown
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
```

#### 节点 7：Loop Over Items2

节点类型：

```
Loop Over Items
```

节点作用：

```
逐个处理 Reddit 帖子。
```

#### 节点 8：Check Record DB

节点类型：

```
Notion
```

节点作用：

```
去 Reddit数据记录库 里查这条 Reddit 帖子是否已经处理过。查询dup_key、reddit_id、source_url是否有出现过。
```

当前配置：

![image.png](image%203.png)

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

#### 节点 9：IF Record Missing

节点类型：

```
IF
```

节点作用：

```
判断Reddit数据记录库有没有查到记录。
```

节点配置：

![image.png](image%204.png)

连接逻辑：

```
true 分支：没有处理记录 -> 继续查内容库
false 分支：已经处理过 -> 回到 Loop Over Items2，处理下一条
```

这个节点是防重复处理的第一道门。

#### 节点 10：Check Content DB

节点类型：

```
Notion
```

节点作用：

```
去 入境游内容库 里查这条 Reddit 帖子是否已经入库。
```

当前配置：

![image.png](image%205.png)

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

```
即使记录库里没有，也要再检查内容库。
这样可以避免某些历史内容已经写入内容库，但记录库没记录，导致重复写入。
```

#### 节点 11：IF Content Missing

节点类型：

```
IF
```

节点作用：

```
判断 Check Content DB 有没有查到内容库记录。
```

当前配置：

![image.png](image%206.png)

连接逻辑：

```
true 分支：内容库没有这条 -> 抓 Reddit 评论区
false 分支：内容库已有这条 -> 回到 Loop Over Items2，处理下一条
```

#### 节点 12：Fetch Reddit Via Local Proxy

节点类型：

```
HTTP Request
```

节点作用：

```
请求本地 Reddit 代理，获取原帖和评论区 JSON。
```

当前配置：

![image.png](image%207.png)

Query 参数：

```jsx
{
  "url": "{{ $('Loop Over Items2').item.json.source_url }}",
  "limit": 100,  //最多抓 100 条评论
  "depth": 6  //最多抓 6 层评论深度
}
```

参数含义：

```markdown
| 参数 | 含义 |
| --- | --- |
| `url` | Reddit 原帖链接 |
| `limit` | 最多抓 100 条评论 |
| `depth` | 最多抓 6 层评论深度 |
```

- 节点参数解释
    
    **Method参数**
    
    | Method | 意思 | 常见用途 |
    | --- | --- | --- |
    | GET | 获取数据 | 搜索、读取、查询 |
    | POST | 提交数据 | 创建内容、发送内容 |
    | PUT | 整体更新数据 | 更新一整条记录 |
    | PATCH | 部分更新数据 | 修改某几个字段 |
    | DELETE | 删除数据 | 删除记录 |
    
    **Query Parameters  查询参数：**
    
    如地址：
    
    ```jsx
    http://127.0.0.1:8787/reddit/travel/search.rss?q=China%20travel&restrict_sr=1&sort=new
    ```
    
    其中Query Parameters为：
    
    | 参数名 | 参数值 | 意思 |
    | --- | --- | --- |
    | q | China travel | 搜索关键词是 China travel |
    | restrict_sr | 1 | 只在当前 subreddit 内搜索 |
    | sort | new | 按最新排序 |
    
    **Headers 请求头：**
    
    你告诉服务器的一些“身份信息、格式信息、请求说明“
    
    | Header 名称 | 作用 |
    | --- | --- |
    | User-Agent | 告诉服务器你是谁，比如浏览器、爬虫、n8n |
    | Content-Type | 告诉服务器你提交的数据是什么格式 |
    | Authorization | 用来放 API Key、Token 等身份验证信息 |
    | Accept | 告诉服务器你希望接收什么格式的数据 |
    
    **Body 请求体：**
    
    GET 请求一般不需要 Body，POST、PUT、PATCH 请求经常需要 Body。
    
    适合：提交给 AI、创建数据库记录、调用 API。
    
    ```jsx
    {
      "title": "Chengdu travel pain point",
      "status": "To Research"
    }
    ```
    
    URL = 我要去哪里
    Method = 我要做什么
    Query Parameters = 我要查什么条件
    Headers = 我是谁、我带什么格式、我有没有权限
    Body = 我要正式提交的内容
    

#### 节点 13：Format Reddit Post And Comments

节点类型：

```
Code
```

节点作用：

```
把代理返回的 Reddit 原帖和评论整理成 AI 可读的输入文本。
```

它会做这些事情：

```
清理 HTML 标签
清理 Reddit 的 [deleted] / [removed]
提取原帖标题和正文
提取评论
按评论分数排序
只保留前 20 条有用评论
拼成 ai_input 给 DeepSeek
```

- 当前代码：
    
    ```jsx
    // 清理 Reddit 文本，避免 HTML 标签、实体字符、删除标记干扰 AI 判断
    function cleanText(text) {           //清洗文本函数
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
    
      // 清洗原帖标题和正文并分别写入变量
      const title = cleanText(post.title || "");
      const postText = cleanText(post.selftext || "");
    
      // 原帖元信息
      const sourceUrl = j.source_url || "";
      const redditId = post.id || "";
      const subreddit = post.subreddit || "";
      const author = post.author || "";
      const score = post.score || 0;
      const numComments = post.num_comments || 0;  //原贴评论数量
      const createdUtc = post.created_utc || "";  //原贴发布时间
    
      // 代理可能返回 flat_comments，也可能返回 comments
      // 优先使用 flat_comments，因为它已经被拉平成一维列表，更方便处理
      // Array.isArray判断某个东西是不是数组
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
      //用于统一所有评论的字段，以便后面filter、sort使用
        .map(c => ({
          author: c.author || "",         //c为任意定义的变量
          score: c.score || 0,          //用：是创建对象字段
          depth: c.depth || 0,
          body: cleanText(c.body || c.text || "")  //提取评论正文
        }))
        .filter(c => c.body.length > 10) //删除正文长度小于等于 10 的评论。
        .sort((a, b) => b.score - a.score)  //这行是按 score 从高到低排序。
        .slice(0, 20);  //保留排序后的前 20 条评论
    
      // 把评论拼成一段文本，给 AI 阅读
        const commentsText = usefulComments
        .map((c, index) => {
          return `Comment ${index + 1} | score: ${c.score} | author: ${c.author} | depth: ${c.depth}\n${c.body}`;
        })
        .join("\n\n---\n\n");
      //生成文本为
      //Comment 1 | score: 10 | author: userA | depth: 0
    //  I recommend staying near Wulingyuan.
    //   ---
    //  Comment 2 | score: 5 | author: userB | depth: 1
    //Tianmen Mountain needs a full day.
    
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

```markdown
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
```

#### 节点 14：AI 判断 + 格式化

节点类型：

```
Basic LLM Chain
```

节点作用：

```
让 DeepSeek 判断这条 Reddit 帖子是否值得进入入境游内容库，并输出 JSON。
```

当前配置：

![image.png](image%208.png)

主 Prompt：

```
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

```
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

System Message = 给 AI 设定“身份、规则、边界”

Prompt / User Message = 给 AI 当前这一次要处理的“具体任务和输入内容”

注意：

```
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

#### 节点 15：DeepSeek Chat Model1

节点类型：

```
DeepSeek Chat Model
```

节点作用：

```
给 AI 判断节点提供大模型。
```

当前配置：

![image.png](image%209.png)

注意：在Credentials中输入正确的API

规定输出格式为JSON，方便后续分类

#### 节点 16：存处理记录

节点类型：

```
Notion
```

节点作用：

```
把这条 Reddit 帖子写入 Reddit数据记录库，表示它已经被处理过。
```

重要：

```
无论 AI 判断相关还是不相关，都会先写处理记录。
这样下次不会重复分析同一个帖子。
```

当前配置：

首先在Credential中加入notion的API

![image.png](image%2010.png)

配置如下：

其中Title可以任取

![image.png](image%2011.png)

对于notion数据库进行字段映射，其中Key Name or ID 能够自动进行下拉选择：

![image.png](image%2012.png)

详细映射如下：

```markdown
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
```

#### 节点 17：Filter 筛选相关内容1

节点类型：

```
Filter
```

节点作用：

```
只让 AI 判断为相关的内容进入入境游内容库。
```

当前条件：

```
{{ ($json.is_relevant === true || $json.is_relevant === "true").toString() }} equals true
```

理解：

```
如果 AI 输出 is_relevant = true，就继续写内容库。
如果 AI 输出 is_relevant = false，就跳过内容库，回到 Loop Over Items2 处理下一条。
```

连接逻辑：

```
true 分支 -> 存入notion内容库
false 分支 -> Loop Over Items2
```

#### 节点 18：存入notion内容库

节点类型：

```
Notion
```

节点作用：

```
把 AI 判断为相关的帖子写入入境游内容库。
```

当前配置：

![image.png](image%2013.png)

字段映射：

```markdown
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
```

写入成功后：

```
这条内容会进入入境游内容库。
后续 Engine 2 可以继续筛选这些内容，做解决方案研究。
```

## 8. 一条帖子完整跑完会发生什么

假设 RSS 抓到一条 Reddit 帖子：

```
Title: Can I use Alipay as a foreigner in China?
URL: https://www.reddit.com/r/travelchina/comments/xxxx/...
```

完整流程是：

```
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

```
http://127.0.0.1:8787/reddit/travelchina.rss?sort=new
```

### 第二步：打开工作流

```
http://localhost:5678/workflow/aOUAKkEsQnXGAs8U
```

### 第三步：临时把 Limit 改成 1

如果只是测试，建议：

```
Limit -> Max Items = 1
```

这样只处理一条 Reddit 帖子，方便排查。

### 第四步：手动运行

点击：

```
Execute Workflow
```

### 第五步：按顺序检查输出

重点检查这些节点：

```
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

## 10. 常见问题排查

### 问题 1：RSS Read 报 403

可能原因：

```
Reddit 拦截请求
本地代理没有正确伪装请求头
代理服务异常
```

处理方式：

```
先在浏览器打开 http://127.0.0.1:8787/reddit/travelchina.rss?sort=new
如果浏览器也打不开，先修本地代理
```

### 问题 2：Fetch Reddit Via Local Proxy 报 502

可能原因：

```
本地代理抓 Reddit 评论失败
Reddit 暂时拦截
帖子 URL 异常
评论区访问失败
```

处理方式：

```
复制 Loop Over Items2 输出里的 source_url
手动访问：
http://127.0.0.1:8787/reddit-comments?url=你的Reddit链接
```

### 问题 3：相关内容没有进内容库

重点检查：

```
AI 是否输出 is_relevant: true
Filter 筛选相关内容1 是否走 true 分支
存入notion内容库 是否报字段类型错误
```

## 11. 当前值得注意的配置

### 11.1 Limit 当前是 6

```
Limit -> Max Items = 6
```

这表示每个 RSS 源最多处理 6条帖子。

测试时可以改成：

```
1
```

### 11.2 记录库 processed_at 当前是固定时间

当前：

```
2026-05-11T16:47:14
```

建议后期改成：

```
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

```
temperature = 0.2
maxTokens = 1500 或 2000
responseFormat = json_object
```

### 11.4 本地代理是核心依赖

这个工作流能不能抓评论，关键不在 n8n，而在：

```
http://127.0.0.1:8787/reddit-comments
```

如果 Reddit 抓取出问题，先查代理服务。

## 12. 字段来源关系

### RSS 阶段生成

来自 `RSS Read` 和 `Build Local Proxy URL`：

```
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

```
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

```
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

```
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

```
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

```
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

## 14. 和 Engine 2 的关系（后续会出）

Engine 1 结束后，如果 AI 判断相关，会写入：

```
入境游内容库
```

Engine 2 会继续从这个内容库里找待调研内容。

一般关系是：

```
Engine 1：发现痛点
Engine 2：研究解决方案
Engine 3：生成内容草稿
```

所以这个工作流的输出质量会直接影响后面的 Engine 2。

如果 Engine 1 误判太多，Engine 2 就会浪费时间研究低价值内容。

## 15. 最小可用检查清单

每次运行前快速检查：

```
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
