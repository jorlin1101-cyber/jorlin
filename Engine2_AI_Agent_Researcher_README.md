# 安全脱敏声明

本文档和对应 Engine 2 workflow JSON 已按公开分享模板处理：

- Notion API Key、DeepSeek API Key、OpenAI API Key、Gemini API Key、Tavily API Key 等真实密钥不应出现在仓库中；如果曾误提交，请立即撤销并重新生成。
- Notion Database ID 已替换为 `YOUR_NOTION_CONTENT_DATABASE_ID` 等占位符。
- n8n Credential ID 已替换为 `YOUR_NOTION_CREDENTIAL_ID` / `YOUR_DEEPSEEK_CREDENTIAL_ID` / `YOUR_OPENAI_CREDENTIAL_ID` 等占位符。
- Tavily 建议通过 `TAVILY_API_KEY` 环境变量配置，不要把真实 key 写入 workflow JSON 或 README。
- 本地路径、本地代理地址、个人账号、客户信息和真实私密字段内容均已用示例或占位符替代。
- 导入 n8n 后，请在本地重新选择自己的 Notion、DeepSeek、OpenAI/Gemini 等凭证，并替换数据库 ID、字段名和环境变量。

# Engine 2 - AI Agent Researcher 工作流说明

这份文档说明本地 n8n 工作流 `Engine 2 - AI Agent Researcher` 的用途、运行条件、每个节点的配置、数据如何流动，以及常见报错怎么排查。

工作流地址：

```text
http://localhost:5678/workflow/engine2_ai_researcher
```

如果打开后停在登录页，先登录 n8n。登录成功后，浏览器会自动跳回这个工作流。

## 1. 这个工作流做什么

Engine 1 已经负责从 Reddit 找入境游相关帖子，并把有价值的痛点写入 Notion 的入境游内容库。

Engine 2 的任务是继续处理这些痛点：

```text
从 Notion 找待调研痛点
-> 每次只取 1 条
-> 整理字段
-> 标记为 Researching
-> 交给 AI Agent 用 Tavily 搜索资料
-> AI 输出固定 JSON
-> Code 节点整理成 Notion 可写入的文本
-> 回写 Notion
```

一句话理解：

```text
Agent 只负责研究和输出 JSON，Notion 回写由普通 Notion 节点完成。
```

这样比让 Agent 直接改 Notion 更稳，也更适合测试阶段。

## 2. 当前工作流信息

| 项目 | 当前值 |
| --- | --- |
| n8n Workflow ID | `engine2_ai_researcher` |
| n8n Workflow Name | `Engine 2 - AI Agent Researcher` |
| 当前处理数量 | 每次最多 1 条 |
| Notion 数据库 | 当前入境游内容库 |
| Notion Database ID | `YOUR_NOTION_CONTENT_DATABASE_ID` |
| Notion 凭证 | `YOUR_NOTION_CREDENTIAL_NAME` |
| AI 模型 | DeepSeek |
| DeepSeek 凭证 | `YOUR_DEEPSEEK_CREDENTIAL_NAME` |
| 搜索工具 | Tavily API，通过 HTTP Request Tool 调用 |

## 3. 为什么没有用专门的 Tavily 节点

我检查了你当前本地 n8n 安装包，里面没有可用的独立 Tavily Tool 节点。

所以现在使用的是：

```text
HTTP Request Tool
-> POST https://api.tavily.com/search
```

它本质上还是在调用 Tavily，只是用 HTTP 工具节点实现。

以后如果你升级 n8n 后，在 AI Agent 的工具节点里能直接搜到 Tavily Tool，可以把现在的 `Tavily Search Tool` 替换成官方 Tavily Tool。

## 4. 运行前必须确认的东西

### 4.1 n8n 已启动

浏览器能打开：

```text
http://localhost:5678
```

### 4.2 Tavily API Key 已经在环境变量里

当前 Tavily 工具节点用的是：

```text
{{ $env.TAVILY_API_KEY }}
```

也就是说，n8n 启动时必须能读到这个环境变量。

如果 Tavily 节点报 `401` 或 `Unauthorized`，优先检查这个环境变量。

### 4.3 Notion 入境游内容库里要有待处理内容

当前筛选条件会找这些 Notion 页面：

```text
status = 📥 To Research
或 status = 待寻找解决方案
或 status 为空

并且：
pain_point_summary  不为空
```

注意：你的字段名里有一个历史遗留细节：

```text
pain_point_summary 
```

这个字段名最后有一个空格。工作流已经按这个真实字段名配置好了，不要随便删掉这个空格，除非同步修改 n8n 节点。

### 4.4 Notion 里已创建 Engine 2 字段

当前已经创建并校验过这些字段：

| 字段名 | 类型 | 用途 |
| --- | --- | --- |
| `research_query` | Text | AI 搜索用的查询词 |
| `solution_summary` | Text | 解决方案总结 |
| `step_by_step_solution` | Text | 分步骤解决方案 |
| `common_mistakes` | Text | 常见错误 |
| `source_links` | Text | 来源链接 |
| `local_chengdu_angle` | Text | 成都/四川本地角度 |
| `draft_angle` | Text | 后续内容创作角度 |
| `image_brief` | Text | 配图/搜图提示 |
| `confidence` | Select | 可信度：`high` / `medium` / `low` |
| `missing_info` | Text | 缺失信息 |
| `claim_check_notes` | Text | 事实核查备注 |
| `engine_2_time` | Date | Engine 2 处理时间 |
| `error_message` | Text | 错误信息 |

`status` 字段当前支持这些状态：

```text
已发布
已生成内容
已有解决方案
待寻找解决方案
📥 To Research
🔍 Researching
📝 To Draft
🧐 Needs Manual Check
⚠️ Error
📱 Ready to Review
🚀 Published
```

## 5. 整体连接顺序

主流程如下：

```text
Manual Trigger
-> Get To Research Items
-> Limit Research Count
-> Loop Each Pain Point
-> Normalize Engine1 Notion Item
-> Has Pain Point
-> Update Status Researching
-> Research AI Agent
-> Normalize Agent Research Output
-> Update Research Result
-> Wait Before Next Item
-> 回到 Loop Each Pain Point
```

AI Agent 下面还挂了 3 个子节点：

```text
Research AI Agent
├── DeepSeek Chat Model
├── Tavily Search Tool
└── Parse Agent Research JSON
```

这 3 个节点不是普通主线节点，而是 AI Agent 的能力组件。

## 6. 每个节点详细说明

### 节点 1：Manual Trigger

节点类型：

```text
n8n-nodes-base.manualTrigger
```

节点作用：

```text
手动启动整个工作流。
```

当前配置：

```text
没有额外参数。
```

小白理解：

```text
你点 Execute Workflow，这个节点就会启动。
```

为什么先用 Manual Trigger：

```text
测试阶段不要一开始就定时跑，先手动确认每一步稳定。
```

后期稳定后，可以换成 Schedule Trigger，比如每天中午 12:00 自动跑。

### 节点 2：Get To Research Items

节点类型：

```text
Notion
```

节点名称：

```text
Get To Research Items
```

节点作用：

```text
从入境游内容库里找需要 Engine 2 调研的痛点。
```

核心配置：

| 配置项 | 当前值 |
| --- | --- |
| Resource | `databasePage` |
| Operation | `getAll` |
| Database ID | `YOUR_NOTION_CONTENT_DATABASE_ID` |
| Return All | `false` |
| Limit | `10` |
| Simple | `true` |
| Filter Type | `json` |
| Credential | `YOUR_NOTION_CREDENTIAL_NAME` |

当前筛选 JSON：

```json
{
  "and": [
    {
      "or": [
        {
          "property": "status",
          "select": {
            "equals": "📥 To Research"
          }
        },
        {
          "property": "status",
          "select": {
            "equals": "待寻找解决方案"
          }
        },
        {
          "property": "status",
          "select": {
            "is_empty": true
          }
        }
      ]
    },
    {
      "property": "pain_point_summary ",
      "rich_text": {
        "is_not_empty": true
      }
    }
  ]
}
```

小白理解：

```text
这个节点只拿“还没研究、但已经有痛点总结”的 Notion 页面。
```

为什么 Limit 是 10：

```text
这是 Notion 入口最多先抓 10 条候选。
真正进入 AI 的数量后面还会被 Limit Research Count 限制成 1 条。
```

### 节点 3：Limit Research Count

节点类型：

```text
Limit
```

节点作用：

```text
把本次真正要处理的数据限制为 1 条。
```

当前配置：

```text
Max Items = 1
```

小白理解：

```text
虽然前面 Notion 可能找到了 10 条，但测试阶段只让 AI 处理第 1 条，避免一次烧太多 API 调用，也方便排错。
```

什么时候改：

```text
一条完整跑通后，可以改成 3。
稳定几天后，再考虑改成 5 或 10。
```

### 节点 4：Loop Each Pain Point

节点类型：

```text
Loop Over Items / Split In Batches
```

节点作用：

```text
逐条处理痛点。
```

当前配置：

```text
Batch Size = 1
```

小白理解：

```text
这个节点像一个排队机。
它一次只放一条 Notion 数据往后走。
后面的节点处理完，再回到这里处理下一条。
```

连接方式：

```text
Loop 分支 -> Normalize Engine1 Notion Item
Done 分支 -> 当前没有接后续节点
```

注意：

```text
最后的 Wait Before Next Item 会连回这个节点。
这是循环能继续处理下一条的关键。
```

### 节点 5：Normalize Engine1 Notion Item

节点类型：

```text
Code
```

节点作用：

```text
把 Engine 1 写入 Notion 的字段整理成 Engine 2 统一使用的字段名。
```

为什么需要这个节点：

```text
Notion 字段可能有英文名、小写名、中文名、历史字段名，后面 AI Agent 不适合直接处理一堆混乱字段。
这个节点先统一清洗一次。
```

主要输出字段：

| 输出字段 | 含义 |
| --- | --- |
| `notion_page_id` | 要回写的 Notion 页面 ID |
| `pain_point_summary_clean` | 清洗后的痛点总结 |
| `hook_clean` | 清洗后的 Hook |
| `category_clean` | 清洗后的分类 |
| `emotion_clean` | 清洗后的情绪 |
| `original_title_clean` | 清洗后的 Reddit 原标题 |
| `original_text_clean` | 清洗后的 Reddit 正文/证据 |
| `source_url_clean` | 清洗后的 Reddit 链接 |
| `is_chengdu_related` | 是否和成都/四川相关 |
| `research_query` | 给 AI/Tavily 用的搜索查询词 |

当前 Code：

```javascript
return items.map(item => {
  const j = item.json;

  function toStr(value) {
    if (value === undefined || value === null) return "";
    if (Array.isArray(value)) return value.join("\n");
    if (typeof value === "object") return JSON.stringify(value);
    return String(value).trim();
  }

  function pick(keys) {
    for (const key of keys) {
      const value = j[key];
      const str = toStr(value);
      if (str) return str;
    }
    return "";
  }

  const notionPageId = j.id || j.page_id || j.notion_page_id || "";

  const painPoint = pick([
    "pain_point_summary ",
    "pain_point_summary",
    "Pain Point Summary",
    "\u75db\u70b9\u603b\u7ed3",
    "\u75db\u70b9",
    "Pain Point"
  ]);

  const hook = pick([
    "hook",
    "Hook",
    "\u7206\u6b3eHOOK\u6807\u9898",
    "\u7206\u6b3e Hook",
    "name",
    "Name",
    "Title"
  ]);

  const category = pick([
    "category",
    "Category",
    "\u5206\u7c7b"
  ]);

  const emotion = pick([
    "emotion",
    "Emotion",
    "\u60c5\u7eea"
  ]);

  const originalTitle = pick([
    "original_title ",
    "original_title",
    "Original Title",
    "Reddit Title",
    "\u539f\u5e16\u6807\u9898",
    "Title",
    "name",
    "Name"
  ]);

  const originalText = pick([
    "original_text",
    "Original Text",
    "Reddit Text",
    "\u539f\u5e16\u6b63\u6587",
    "\u6b63\u6587",
    "Content",
    "evidence"
  ]);

  const sourceUrl = pick([
    "source_url ",
    "source_url",
    "Source URL",
    "URL",
    "link",
    "\u539f\u5e16\u94fe\u63a5"
  ]);

  const combinedText = [
    painPoint,
    hook,
    category,
    originalTitle,
    originalText
  ].join(" ");

  const isChengduRelated =
    /chengdu|sichuan|\u6210\u90fd|\u56db\u5ddd|panda|jiuzhaigou|\u4e5d\u5be8\u6c9f|siguniang|\u56db\u59d1\u5a18\u5c71|tcm|\u4e2d\u533b|tea|\u8336/i.test(combinedText);

  const localExtra = isChengduRelated
    ? "Chengdu Sichuan local practical tips"
    : "China travel practical tips";

  const researchQuery = [
    painPoint,
    category,
    "foreign tourists visiting China",
    "practical solution",
    "official source",
    "latest information",
    localExtra
  ]
    .filter(Boolean)
    .join(" ");

  return {
    json: {
      ...j,
      notion_page_id: notionPageId,
      pain_point_summary_clean: painPoint.slice(0, 2000),
      hook_clean: hook,
      category_clean: category,
      emotion_clean: emotion,
      original_title_clean: originalTitle,
      original_text_clean: originalText.slice(0, 5000),
      source_url_clean: sourceUrl,
      is_chengdu_related: isChengduRelated,
      research_query: researchQuery
    }
  };
});
```

注意：

```text
代码里的 \u75db\u70b9 这种写法是 Unicode 转义。
它在 JavaScript 运行时等于中文“痛点”。
这样写是为了避免 Windows/PowerShell 编码导致中文变成乱码。
```

### 节点 6：Has Pain Point

节点类型：

```text
Filter
```

节点作用：

```text
检查这一条数据是否真的有痛点总结。
```

当前条件：

```text
{{ $json.pain_point_summary_clean }} is not empty
```

连接逻辑：

```text
true 分支 -> Update Status Researching
false 分支 -> 回到 Loop Each Pain Point
```

小白理解：

```text
如果这一条没有痛点总结，就不要浪费 AI 搜索。
直接跳过，继续下一条。
```

### 节点 7：Update Status Researching

节点类型：

```text
Notion
```

节点作用：

```text
告诉 Notion：这条内容已经进入 Engine 2 调研阶段。
```

核心配置：

| 配置项 | 当前值 |
| --- | --- |
| Resource | `databasePage` |
| Operation | `update` |
| Page ID | `={{ $json.notion_page_id }}` |
| Credential | `YOUR_NOTION_CREDENTIAL_NAME` |

写入字段：

| Notion 字段 | 写入值 |
| --- | --- |
| `status` | `🔍 Researching` |
| `research_query` | `={{ $json.research_query }}` |
| `error_message` | 空 |

小白理解：

```text
只要这条内容走到这里，Notion 里就会先变成“正在研究”。
如果后面 AI 或 Tavily 报错，你也能知道它不是没开始，而是卡在研究阶段。
```

### 节点 8：Research AI Agent

节点类型：

```text
AI Agent
```

节点作用：

```text
让 AI 像研究员一样，根据痛点调用 Tavily 搜索，然后输出固定 JSON。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| Prompt Type | `define` |
| Has Output Parser | `true` |
| Max Iterations | `5` |

这个节点连接了 3 个 AI 子节点：

```text
DeepSeek Chat Model
Tavily Search Tool
Parse Agent Research JSON
```

当前 Prompt：

```text
You are the Research Agent for U Travel, an inbound China travel content system.

U Travel helps foreign travelers understand and experience China like locals, especially Chengdu and Sichuan.

Your job:
Research the travel pain point below, use the Tavily search tool when needed, and produce a practical, source-grounded solution that can later be turned into social media content.

Important rules:
1. You must use Tavily search before giving the final answer.
2. Prefer official sources when possible, such as government, airport, railway, attraction, map, payment platform, airline, or official tourism websites.
3. If official sources are not available, use recent high-quality travel guides, forums, or firsthand traveler reports.
4. Do not invent links.
5. Do not invent prices, policies, opening hours, visa rules, railway rules, or payment rules.
6. If the sources are weak or incomplete, set research_status to "needs_manual_check".
7. If the answer is useful but not fully official, use confidence "medium".
8. Use confidence "high" only when the answer is supported by strong or official sources.
9. Add a Chengdu or Sichuan local angle when relevant.
10. Avoid sales language.
11. Do not say "book a tour", "DM me", "contact me", "I offer tours", or "I can guide you".
12. The final answer must be useful for foreign travelers even if they never buy anything.
13. Output only valid JSON.
14. No Markdown.
15. No code block.

Traveler pain point:
{{ $('Normalize Engine1 Notion Item').item.json.pain_point_summary_clean }}

Hook from Engine 1:
{{ $('Normalize Engine1 Notion Item').item.json.hook_clean }}

Category:
{{ $('Normalize Engine1 Notion Item').item.json.category_clean }}

Emotion:
{{ $('Normalize Engine1 Notion Item').item.json.emotion_clean }}

Original Reddit title:
{{ $('Normalize Engine1 Notion Item').item.json.original_title_clean }}

Original Reddit text:
{{ $('Normalize Engine1 Notion Item').item.json.original_text_clean }}

Original source URL:
{{ $('Normalize Engine1 Notion Item').item.json.source_url_clean }}

Suggested search query:
{{ $('Normalize Engine1 Notion Item').item.json.research_query }}

Please return the final answer in the required JSON structure.
```

小白理解：

```text
这个节点不是普通问答。
它会拿痛点、原帖、分类、搜索词，然后自己决定 Tavily 搜索什么，最后按固定 JSON 格式返回。
```

### 节点 9：DeepSeek Chat Model

节点类型：

```text
DeepSeek Chat Model
```

节点作用：

```text
给 Research AI Agent 提供大模型能力。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| Model | `deepseek-chat` |
| Temperature | `0.2` |
| Max Tokens | `2500` |
| Response Format | `json_object` |
| Timeout | `360000` ms |
| Max Retries | `2` |
| Credential | `YOUR_DEEPSEEK_CREDENTIAL_NAME` |

小白理解：

```text
Temperature 越低，AI 越稳，不容易乱发挥。
Engine 2 是研究任务，不是写爆款文案，所以设置为 0.2。
```

注意：

```text
DeepSeek 在工具调用稳定性上可能不如 OpenAI。
如果 Agent 不调用 Tavily、或者工具调用报错，后续可以把这个节点换成 OpenAI Chat Model。
```

### 节点 10：Tavily Search Tool

节点类型：

```text
HTTP Request Tool
```

节点作用：

```text
作为 AI Agent 的搜索工具，调用 Tavily 搜索实时网页资料。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| Method | `POST` |
| URL | `https://api.tavily.com/search` |
| Authentication | `none` |
| Send Headers | `true` |
| Send Body | `true` |
| Body Type | `json` |
| Optimize Response | `true` |
| Expected Response Type | `json` |
| Include Fields | `selected` |
| Fields | `answer,results.title,results.url,results.content,results.score` |

Headers：

| Header | Value |
| --- | --- |
| `Authorization` | `=Bearer {{ $env.TAVILY_API_KEY }}` |
| `Content-Type` | `application/json` |

Body：

```json
{
  "query": "{query}",
  "search_depth": "basic",
  "topic": "general",
  "max_results": 5,
  "include_answer": true,
  "include_raw_content": false,
  "include_images": false
}
```

Placeholder：

| Name | Type | Description |
| --- | --- | --- |
| `query` | `string` | Search query for researching the travel pain point. Use English. Include China travel, foreign tourists, official source, and Chengdu/Sichuan terms when relevant. |

小白理解：

```text
AI Agent 会自己填 {query}。
比如它可能会搜索：
Alipay foreign tourists China official guide 2026
```

为什么只保留部分返回字段：

```text
Tavily 返回内容可能很多。
这里只把 answer、title、url、content、score 给 AI，减少 token 浪费。
```

### 节点 11：Parse Agent Research JSON

节点类型：

```text
Structured Output Parser
```

节点作用：

```text
约束 AI Agent 最终必须输出固定 JSON 格式。
```

当前配置：

| 配置项 | 当前值 |
| --- | --- |
| Schema Type | `manual` |
| Auto Fix | `false` |

当前 Schema：

```json
{
  "type": "object",
  "additionalProperties": false,
  "properties": {
    "research_status": {
      "type": "string",
      "enum": [
        "solved",
        "needs_manual_check"
      ]
    },
    "confidence": {
      "type": "string",
      "enum": [
        "high",
        "medium",
        "low"
      ]
    },
    "solution_summary": {
      "type": "string"
    },
    "step_by_step_solution": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "minItems": 3,
      "maxItems": 8
    },
    "common_mistakes": {
      "type": "array",
      "items": {
        "type": "string"
      },
      "minItems": 1,
      "maxItems": 5
    },
    "local_chengdu_angle": {
      "type": "string"
    },
    "source_links": {
      "type": "array",
      "items": {
        "type": "object",
        "additionalProperties": false,
        "properties": {
          "title": {
            "type": "string"
          },
          "url": {
            "type": "string"
          },
          "why_useful": {
            "type": "string"
          },
          "source_type": {
            "type": "string",
            "enum": [
              "official",
              "travel_guide",
              "forum",
              "map",
              "news",
              "other"
            ]
          }
        },
        "required": [
          "title",
          "url",
          "why_useful",
          "source_type"
        ]
      },
      "minItems": 1,
      "maxItems": 5
    },
    "missing_info": {
      "type": "string"
    },
    "draft_angle": {
      "type": "string"
    },
    "image_brief": {
      "type": "string"
    },
    "claim_check_notes": {
      "type": "string"
    }
  },
  "required": [
    "research_status",
    "confidence",
    "solution_summary",
    "step_by_step_solution",
    "common_mistakes",
    "local_chengdu_angle",
    "source_links",
    "missing_info",
    "draft_angle",
    "image_brief",
    "claim_check_notes"
  ]
}
```

小白理解：

```text
它像一个表格模板。
AI 不能随便输出一段文章，必须按这些字段输出。
```

为什么 `autoFix = false`：

```text
测试阶段先让错误暴露出来，方便知道 DeepSeek 输出哪里不稳定。
后期如果经常只是小格式问题，可以考虑打开自动修复。
```

### 节点 12：Normalize Agent Research Output

节点类型：

```text
Code
```

节点作用：

```text
把 AI Agent 的 JSON 输出整理成 Notion 能写入的普通文本。
```

为什么需要它：

```text
AI 输出里的 step_by_step_solution、common_mistakes、source_links 是数组。
Notion 的普通 Text 字段不能直接写数组，所以要转成多行文本。
```

当前 Code：

```javascript
const base = $('Normalize Engine1 Notion Item').item.json;
let ai = $input.first().json;

function cleanJsonString(str) {
  return String(str)
    .replace(/^```json/i, "")
    .replace(/^```/i, "")
    .replace(/```$/i, "")
    .trim();
}

if (ai.output) {
  if (typeof ai.output === "string") {
    try {
      ai = JSON.parse(cleanJsonString(ai.output));
    } catch (error) {
      ai = {
        research_status: "needs_manual_check",
        confidence: "low",
        solution_summary: "",
        step_by_step_solution: [],
        common_mistakes: [],
        local_chengdu_angle: "",
        source_links: [],
        missing_info: "AI Agent output could not be parsed as JSON.",
        draft_angle: "",
        image_brief: "",
        claim_check_notes: String(error.message || error)
      };
    }
  } else if (typeof ai.output === "object") {
    ai = ai.output;
  }
}

const stepByStep = Array.isArray(ai.step_by_step_solution) ? ai.step_by_step_solution : [];
const mistakes = Array.isArray(ai.common_mistakes) ? ai.common_mistakes : [];
const links = Array.isArray(ai.source_links) ? ai.source_links : [];

const sourceLinksText = links.map((s, index) => {
  return [
    `${index + 1}. ${s.title || "Untitled source"}`,
    `URL: ${s.url || ""}`,
    `Type: ${s.source_type || "other"}`,
    `Why useful: ${s.why_useful || ""}`
  ].join("\n");
}).join("\n\n");

const finalStatus =
  ai.research_status === "solved" && ai.confidence !== "low"
    ? "📝 To Draft"
    : "🧐 Needs Manual Check";

return [
  {
    json: {
      ...base,
      ...ai,
      step_by_step_solution_text: stepByStep.map((s, i) => `${i + 1}. ${s}`).join("\n"),
      common_mistakes_text: mistakes.map((s, i) => `${i + 1}. ${s}`).join("\n"),
      source_links_text: sourceLinksText,
      final_status: finalStatus
    }
  }
];
```

状态判断规则：

```text
如果 research_status = solved
并且 confidence 不是 low
-> final_status = 📝 To Draft

否则
-> final_status = 🧐 Needs Manual Check
```

小白理解：

```text
AI 觉得资料足够，就进入下一步写草稿。
AI 觉得资料不够，或可信度低，就留给你人工检查。
```

### 节点 13：Update Research Result

节点类型：

```text
Notion
```

节点作用：

```text
把 AI 研究结果回写到原来的 Notion 页面。
```

核心配置：

| 配置项 | 当前值 |
| --- | --- |
| Resource | `databasePage` |
| Operation | `update` |
| Page ID | `={{ $json.notion_page_id }}` |
| Credential | `YOUR_NOTION_CREDENTIAL_NAME` |

字段映射：

| Notion 字段 | 写入表达式 |
| --- | --- |
| `status` | `={{ $json.final_status }}` |
| `research_query` | `={{ $json.research_query }}` |
| `solution_summary` | `={{ $json.solution_summary }}` |
| `step_by_step_solution` | `={{ $json.step_by_step_solution_text }}` |
| `common_mistakes` | `={{ $json.common_mistakes_text }}` |
| `source_links` | `={{ $json.source_links_text }}` |
| `local_chengdu_angle` | `={{ $json.local_chengdu_angle }}` |
| `draft_angle` | `={{ $json.draft_angle }}` |
| `image_brief` | `={{ $json.image_brief }}` |
| `confidence` | `={{ $json.confidence }}` |
| `missing_info` | `={{ $json.missing_info }}` |
| `claim_check_notes` | `={{ $json.claim_check_notes }}` |
| `engine_2_time` | `={{ $now }}` |
| `error_message` | 空 |

小白理解：

```text
这是最终落库节点。
AI 不直接碰 Notion，最后由这个普通 Notion 节点统一写回。
```

### 节点 14：Wait Before Next Item

节点类型：

```text
Wait
```

节点作用：

```text
每处理完一条，等 2 秒再处理下一条。
```

当前配置：

```text
Resume = timeInterval
Amount = 2
Unit = seconds
```

小白理解：

```text
这是为了不要连续太快请求 Notion、DeepSeek 和 Tavily。
```

连接方式：

```text
Wait Before Next Item
-> Loop Each Pain Point
```

## 7. 一条数据完整跑完后会发生什么

假设 Notion 里有一条数据：

```text
status = 待寻找解决方案
pain_point_summary  = 外国游客不知道怎么用支付宝坐地铁
```

运行后：

```text
1. Get To Research Items 找到它
2. Limit Research Count 只保留这 1 条
3. Loop Each Pain Point 开始处理
4. Normalize Engine1 Notion Item 生成 research_query
5. Has Pain Point 判断痛点不为空
6. Update Status Researching 把 status 改成 🔍 Researching
7. Research AI Agent 调用 Tavily 搜索
8. Parse Agent Research JSON 约束输出格式
9. Normalize Agent Research Output 把数组转成文本
10. Update Research Result 写回 Notion
11. Wait 2 秒
12. 回到 Loop，检查是否还有下一条
```

最后 Notion 的状态通常会变成：

```text
📝 To Draft
```

或者：

```text
🧐 Needs Manual Check
```

## 8. 如何手动测试

### 第一步：先准备一条 Notion 测试数据

在入境游内容库里找一条你想测试的痛点，把状态改成：

```text
📥 To Research
```

或者保留旧状态：

```text
待寻找解决方案
```

并确认这个字段不为空：

```text
pain_point_summary 
```

### 第二步：打开工作流

打开：

```text
http://localhost:5678/workflow/engine2_ai_researcher
```

### 第三步：点 Execute Workflow

测试阶段不要改 Limit，保持：

```text
Limit Research Count = 1
```

### 第四步：看每个节点输出

建议按顺序点开这些节点看输出：

```text
Get To Research Items
Normalize Engine1 Notion Item
Update Status Researching
Research AI Agent
Normalize Agent Research Output
Update Research Result
```

最关键的是 `Normalize Engine1 Notion Item` 输出里必须有：

```text
notion_page_id
pain_point_summary_clean
research_query
```

如果 `notion_page_id` 为空，后面无法回写 Notion。

## 9. 常见问题排查

### 问题 1：Get To Research Items 没有数据

优先检查 Notion：

```text
status 是否等于 📥 To Research 或 待寻找解决方案？
pain_point_summary  是否为空？
字段名 pain_point_summary 后面是不是有一个空格？
```

### 问题 2：Update Status Researching 报错

常见原因：

```text
notion_page_id 为空
Notion 凭证失效
status 选项不存在
research_query 字段不存在
```

先看前一个 Code 节点输出里有没有：

```text
notion_page_id
```

### 问题 3：Tavily Search Tool 报 401

通常是 API Key 问题。

检查：

```text
n8n 启动环境里是否有 TAVILY_API_KEY
Tavily key 是否过期
Tavily key 是否复制错
```

### 问题 4：Tavily Search Tool 报 429

通常是请求太多或额度限制。

处理方式：

```text
保持 Limit Research Count = 1
Wait 可以从 2 秒改成 5 秒或 10 秒
检查 Tavily 套餐额度
```

### 问题 5：Research AI Agent 不调用 Tavily

当前 Prompt 已经写了：

```text
You must use Tavily search before giving the final answer.
```

如果仍然不调用，可能是 DeepSeek 工具调用不稳定。

处理方式：

```text
先重跑一次
如果多次不调用，后续把 DeepSeek Chat Model 换成 OpenAI Chat Model
```

### 问题 6：Structured Output Parser 报错

说明 AI 没有按 Schema 输出。

常见原因：

```text
DeepSeek 输出了 Markdown
DeepSeek 少了某个 required 字段
source_links 为空
step_by_step_solution 少于 3 条
common_mistakes 为空
```

处理方式：

```text
先看 Research AI Agent 的输出
如果只是格式小问题，可以后期考虑把 Parse Agent Research JSON 的 autoFix 打开
```

### 问题 7：Update Research Result 报错

常见原因：

```text
Notion 字段类型不匹配
confidence 不是 high / medium / low
status 不是 Notion 已有选项
某个文本太长
```

优先看 `Normalize Agent Research Output` 输出里的：

```text
final_status
confidence
solution_summary
step_by_step_solution_text
source_links_text
```

## 10. 什么情况下可以扩大处理数量

当前：

```text
Limit Research Count = 1
```

建议升级节奏：

```text
第 1 阶段：1 条，手动测试
第 2 阶段：3 条，观察 2-3 天
第 3 阶段：5 条，确认成本和稳定性
第 4 阶段：改成 Schedule Trigger 自动跑
```

不要一开始就跑 10 条以上，因为每条都会调用：

```text
DeepSeek
Tavily
Notion Update
```

## 11. 什么时候进入 Engine 3

Engine 2 完成后，Notion 状态会决定下一步：

```text
📝 To Draft
```

表示资料基本够，可以进入 Engine 3 做内容草稿。

```text
🧐 Needs Manual Check
```

表示 AI 认为资料不足、来源不强、或者可信度低，需要你人工检查。

## 12. 当前设计原则

这套工作流刻意没有让 Agent 直接更新 Notion。

当前边界是：

```text
AI Agent：
只负责搜索、判断、总结、输出 JSON

Notion 节点：
负责状态变更和结果回写

Code 节点：
负责字段清洗和格式转换
```

这样做的好处：

```text
更安全
更容易排错
更适合新手理解
不会让 Agent 越权乱改数据库
```

## 13. 最小可用测试清单

每次你怀疑工作流有问题，可以按这个清单检查：

```text
1. n8n 是否已启动
2. 是否能打开 http://localhost:5678
3. Notion 里是否有 status = 📥 To Research 的数据
4. pain_point_summary  是否不为空
5. Limit Research Count 是否还是 1
6. DeepSeek 凭证是否可用
7. TAVILY_API_KEY 是否可用
8. Research AI Agent 是否真的调用了 Tavily
9. Normalize Agent Research Output 是否生成 final_status
10. Update Research Result 是否成功写回 Notion
```

## 14. 备份说明

在修正 `Normalize Engine1 Notion Item` 编码问题前，已经备份过本地 n8n 数据库：

```text
C:\\path\\to\\your\\workspace\n8n-local\n8n-data\.n8n\database.sqlite.backup-before-engine2-readme-normalize-fix-20260512-142939
```

如果后面你误改工作流，仍然可以从备份恢复。

