
## Chat Completions API

`Chat Completions (/v1/chat/completions)` 是第一代API。

```http
POST https://api.openai.com/v1/chat/completions
Authorization: Bearer $OPENAI_API_KEY
Content-Type: application/json
```

参数说明：

- model：模型ID
- message：消息内容

| role        | 一句话作用                                                  |
| ----------- | --------------------------------------------------------- |
| `system`    | 定义模型的**最高级行为规则和身份设定**。                        |
| `developer` | 定义**模型规则**，业务逻辑、输出格式和约束要求。                |
| `user`      | 用户问题或需求。                                 |
| `assistant` | 历史回答。                  |
| `tool`      | 工具执行结果。   |

```json
{
  "messages": [
    {
      "role": "system",
      "content": "你是一个专业的 Java 后端专家"
    },
    {
      "role": "user",
      "content": "如何优化 Elasticsearch 查询？"
    },
    {
      "role": "assistant",
      "content": "可以从索引、mapping、查询条件几个方面优化..."
    }
  ]
}
```

- stream
- max_completion_tokens
- **tools**：调用工具说明
- tool_choice：
- parallel_tool_calls：
- response_format：

- **Temperature**：控制输出的随机性，越高越有创造性，越低越稳定。
- **Top_k**：限制模型每次只能从概率最高的 K 个候选词中选择。
- **Top_p**：限制模型只能从累计概率达到 p 的候选词中选择。





请求参数：

```jsonc
{
  "model": "你的实际模型ID", // 指定使用的模型，必填

  "messages": [ // 对话消息列表，定义整个上下文
    {
      "role": "developer", // 消息角色，developer 表示开发者指令
      "name": "system_rules", // 可选，消息参与者名称
      "content": "你是一个中文智能客服，回答要简洁准确。" // 消息内容
    },
    {
      "role": "user", // 消息角色，user 表示用户消息
      "name": "customer", // 可选，用户或参与者名称
      "content": "请查询一下北京今天的天气。" // 用户实际输入内容，也可以写成多模态 content 数组
    }
  ],

  "temperature": 0.2, // 控制输出随机性，通常越低越稳定，具体是否支持取决于模型
  "top_p": 1.0, // 控制 nucleus sampling，通常与 temperature 二选一调整
  "max_completion_tokens": 500, // 限制模型最多生成多少个 completion token
  "n": 1, // 一次生成多少个候选答案，1 表示只生成一个
  "stream": false, // 是否使用流式输出，false 一次性返回，true 边生成边返回
  "stream_options": { // 流式输出的附加配置，仅在 stream=true 时有意义
    "include_usage": true, // 流式结束时是否额外返回 token 使用量
    "include_obfuscation": false // 是否使用流式数据 obfuscation，一般无需修改
  },

  "stop": ["<END>"], // 模型生成到指定字符串时停止输出
  "presence_penalty": 0, // 根据内容是否已经出现过进行惩罚，用于鼓励引入新内容
  "frequency_penalty": 0, // 根据 token 出现频率进行惩罚，用于减少重复
  "logprobs": true, // 是否返回生成 token 的 log probability
  "top_logprobs": 3, // 每个 token 返回多少个最高概率候选，通常需要配合 logprobs=true

  "tools": [ // 提供给模型调用的工具列表
    {
      "type": "function", // 工具类型，这里表示函数工具
      "function": { // 函数工具的详细定义
        "name": "get_weather", // 函数名称，模型调用工具时会返回这个名字
        "description": "根据城市查询当前天气", // 告诉模型这个工具是什么、什么时候应该使用
        "parameters": { // 定义函数参数的数据结构，本质上是 JSON Schema
          "type": "object", // 函数参数整体是一个 JSON 对象
          "properties": { // 定义函数有哪些参数
            "city": { // 定义 city 参数
              "type": "string", // city 参数必须是字符串
              "description": "城市名称，例如北京" // 对模型解释这个参数的含义
            }
          },
          "required": ["city"], // 指定哪些函数参数是必填的
          "additionalProperties": false // 是否禁止传入未定义的额外参数
        },
        "strict": true // 是否严格按照 parameters 中的 JSON Schema 调用函数
      }
    }
  ],

  "tool_choice": "auto", // 控制是否调用工具：none=不调用，auto=模型自己决定，required=必须调用
  "parallel_tool_calls": true, // 是否允许模型一次产生多个并行工具调用

  "response_format": { // 限制模型输出格式
    "type": "json_object" // 要求模型输出合法 JSON；更严格时可以使用 JSON Schema
  },

  "prediction": { // 预测输出，用于已知大部分输出内容、只需要模型少量修改的场景
    "type": "content", // prediction 类型，这里表示内容预测
    "content": "" // 提供给模型的预期输出内容
  },

  "metadata": { // 自定义业务元数据，用于追踪、统计、查询
    "request_type": "weather", // 业务字段示例
    "customer_type": "vip" // 业务字段示例
  },

  "store": true, // 是否保存这次 Chat Completion
  "service_tier": "auto", // 服务层级，auto 表示自动选择
  "user": "customer_123" // 最终用户标识，可用于用户级安全和请求追踪
}
}
```

响应参数（正常返回）：

```jsonc
{
  "id": "chatcmpl_123456", // 本次 Chat Completion 的唯一 ID
  "object": "chat.completion", // 返回对象的类型
  "created": 1760000000, // 创建时间，Unix timestamp，单位秒
  "model": "你的实际模型ID", // 实际使用的模型
  "service_tier": "auto", // 实际使用的服务层级
  "system_fingerprint": "fp_xxxxx", // 后端系统配置 fingerprint，主要用于分析行为变化

  "choices": [ // 模型生成的候选结果列表，n=1 时通常只有一个
    {
      "index": 0, // 当前候选结果的下标，第一个结果为 0

      "message": { // 模型生成的 assistant 消息
        "role": "assistant", // 消息角色，这里表示模型生成的回答
        "content": "{\"city\":\"北京\",\"weather\":\"晴\",\"temperature\":25}", // 模型实际生成的文本
        "refusal": null, // 如果模型拒绝回答，这里可能包含拒答信息
        "annotations": [], // 某些能力产生的注释、引用等附加信息
        "tool_calls": null, // 如果模型需要调用工具，这里会出现工具调用信息
        "audio": null // 如果使用音频输出，这里会包含音频相关信息
      },

      "logprobs": null, // 如果请求 logprobs=true，这里可能包含 token 概率信息
      "finish_reason": "stop" // 模型为什么结束：stop=正常结束，length=达到 token 限制，tool_calls=调用工具等
    }
  ],

  "usage": { // 本次请求的 token 使用量
    "prompt_tokens": 120, // 输入消息消耗的 token 数
    "completion_tokens": 35, // 输出内容消耗的 token 数
    "total_tokens": 155, // 输入 token + 输出 token

    "prompt_tokens_details": { // 输入 token 的进一步统计
      "cached_tokens": 0 // 其中有多少 token 来自缓存
    },

    "completion_tokens_details": { // 输出 token 的进一步统计
      "reasoning_tokens": 0 // 输出中用于 reasoning 的 token 数，是否存在取决于模型
    }
  }
}
```

响应参数（调用函数）：

```jsonc
{
  "id": "chatcmpl_123456", // 本次请求的唯一 ID
  "object": "chat.completion", // 返回对象类型
  "created": 1760000000, // 创建时间
  "model": "你的实际模型ID", // 实际使用的模型

  "choices": [ // 候选结果
    {
      "index": 0, // 第一个候选

      "message": { // 模型生成的 assistant 消息
        "role": "assistant", // 角色是 assistant
        "content": null, // 此时没有直接回答文本，因为模型准备调用工具

        "tool_calls": [ // 模型要求你的程序执行的工具列表
          {
            "id": "call_123", // 这次工具调用的唯一 ID，后面返回工具结果时需要用它关联
            "type": "function", // 工具类型
            "function": { // 具体函数调用信息
              "name": "get_weather", // 模型决定调用的函数名称
              "arguments": "{\"city\":\"北京\"}" // 模型生成的函数参数，是一个 JSON 字符串
            }
          }
        ]
      },

      "finish_reason": "tool_calls" // 表示模型这次不是正常回答，而是要求调用工具
    }
  ],

  "usage": { // 本次调用消耗的 token
    "prompt_tokens": 120, // 输入 token
    "completion_tokens": 20, // 模型生成工具调用所消耗的 token
    "total_tokens": 140 // 总 token
  }
}
```

然后程序执行：

```
weather = get_weather(city="北京")
```

得到结果：

```josnc
{
  "city": "北京",
  "weather": "晴",
  "temperature": 25
}
```

再把结果通过：

```jsonc
{
  "role": "tool", // 表示这是工具执行结果
  "tool_call_id": "call_123", // 必须对应模型刚才返回的 tool_calls[].id
  "content": "{\"city\":\"北京\",\"weather\":\"晴\",\"temperature\":25}" // 工具执行结果
}
```

传回 Chat Completions，模型才会继续生成最终答案。

## Responses API

`Responses API (/v1/responses)` 是 OpenAI 当前推荐的新一代统一响应接口。与传统的 Chat Completions API 不同，Responses API 使用 `input` 作为输入、`output` 作为输出，并将普通消息、函数调用、reasoning、Web Search、File Search 等不同类型的输出统一表示为 `output` items。

```http
POST https://api.openai.com/v1/responses
Authorization: Bearer $OPENAI_API_KEY
Content-Type: application/json
```

## 一、核心参数

| 参数                     | 一句话作用                      |
| ---------------------- | -------------------------- |
| `model`                | 指定使用的模型                    |
| `instructions`         | 定义模型的系统/开发者级指令             |
| `input`                | 给模型的文本、图片、文件或消息输入          |
| `previous_response_id` | 指向上一轮 Response，用于多轮对话      |
| `conversation`         | 指定所属的 Conversation         |
| `max_output_tokens`    | 限制本次 Response 最多生成多少 token |
| `temperature`          | 控制输出随机性                    |
| `top_p`                | 控制 nucleus sampling        |
| `stream`               | 是否使用流式响应                   |
| `stream_options`       | 流式响应的附加配置                  |
| `reasoning`            | 配置 reasoning 模型的推理能力       |
| `text`                 | 配置文本输出格式和 verbosity        |
| `tools`                | 指定模型可以调用的工具                |
| `tool_choice`          | 控制模型选择什么工具                 |
| `parallel_tool_calls`  | 是否允许并行工具调用                 |
| `max_tool_calls`       | 限制一次 Response 中最多调用多少次内置工具 |
| `previous_response_id` | 通过上一个 Response 继续对话        |
| `truncation`           | 上下文超长时采用什么截断策略             |
| `store`                | 是否保存生成的 Response           |
| `include`              | 要求 Response 额外返回某些数据       |
| `metadata`             | 附加业务元数据                    |
| `prompt`               | 使用 Prompt Template         |
| `prompt_cache_key`     | 用于 Prompt Cache            |
| `prompt_cache_options` | 配置 Prompt Cache            |
| `service_tier`         | 指定服务层级                     |
| `safety_identifier`    | 稳定标识最终用户，用于安全控制            |
| `background`           | 是否以后台任务方式运行                |
| `context_management`   | 配置上下文管理                    |
| `moderation`           | 配置输入和输出的 moderation        |


下面是一个比较完整的 Responses API 请求示例：

```jsonc
{
  "model": "你的实际模型ID", // 指定使用的模型
  "instructions": "你是一个专业的 Java 后端专家，回答必须使用中文并尽量简洁。", // system/developer级指令
  "input": [ // 给模型的输入，可以是字符串，也可以是消息、图片、文件等 input items
    {
      "role": "user", // 输入消息的角色
      "content": [
        {
          "type": "input_text", // 输入内容类型，这里表示文本
          "text": "请分析一下 Elasticsearch 查询优化方法。" // 用户输入的实际文本
        }
      ]
    }
  ],
  "previous_response_id": null, // 上一轮 Response 的 ID，用于多轮对话；首次请求通常为空
  "conversation": null, // 所属 Conversation ID；与 previous_response_id 不能同时使用
  "max_output_tokens": 1000, // 本次 Response 最大输出 token 数，包含可见输出和 reasoning token
  "temperature": 0.2, // 控制输出随机性，较低通常更稳定，具体是否支持取决于模型
  "top_p": 1.0, // nucleus sampling 参数，通常与 temperature 二选一调节
  "stream": false, // 是否使用流式响应，false 一次返回完整 Response，true 持续返回事件
  "stream_options": { // 流式响应配置，仅 stream=true 时使用
    "include_obfuscation": false // 是否对流式事件启用 obfuscation，一般业务无需修改
  },
  "reasoning": { // reasoning 模型的推理配置
    "effort": "medium", // 推理强度，例如 none、low、medium、high，具体可用值取决于模型
    "summary": "auto" // 是否生成 reasoning summary
  },
  "text": { // 配置文本输出
    "format": { // 输出格式
      "type": "text" // 普通文本输出；也可以配置 JSON Schema
    },
    "verbosity": "medium" // 输出详细程度
  },
  "tools": [ // 模型可以调用的工具
    {
      "type": "function", // 工具类型，这里是自定义函数
      "name": "get_weather", // 函数名称
      "description": "根据城市查询当前天气", // 告诉模型该工具是什么以及什么时候应该使用
      "parameters": { // 函数参数 JSON Schema
        "type": "object", // 函数参数整体是 JSON 对象

        "properties": { // 定义函数参数
          "city": {
            "type": "string", // city 参数类型为字符串
            "description": "城市名称，例如北京" // 参数说明
          }
        },
        "required": ["city"], // 必填参数
        "additionalProperties": false // 不允许额外参数
      },
      "strict": true // 严格按照 JSON Schema 生成函数参数
    }
  ],
  "tool_choice": "auto", // 控制工具选择：auto=模型自己决定，none=不使用工具，required=必须使用工具，也可以指定具体工具
  "parallel_tool_calls": true, // 是否允许模型并行调用多个工具
  "max_tool_calls": 10, // 本次 Response 最多处理多少次内置工具调用
  "truncation": "disabled", // 上下文超长时的处理方式：disabled=超长直接失败，auto=自动从较早内容开始截断
  "store": true, // 是否保存 Response，默认通常为 true
  "include": [ // 要求额外返回某些信息
    "message.output_text.logprobs" // 示例：要求返回输出文本的 logprobs
  ],
  "metadata": { // 自定义业务元数据
    "request_type": "elasticsearch", // 业务类型
    "customer_type": "vip" // 客户类型
  },
  "prompt": { // 使用 Prompt Template
    "id": "pmpt_123", // Prompt Template ID
    "version": "1", // Prompt Template 版本
    "variables": { // Prompt Template 使用的变量
      "language": "Chinese" // 模板变量
    }
  },
  "prompt_cache_key": "customer-support-v1", // 用于 Prompt Cache 的稳定 key
  "prompt_cache_options": { // Prompt Cache 配置
    "mode": "implicit", // 缓存模式
    "ttl": "30m", // 缓存生命周期
    "comparison_response_id": "resp_xxx" // 用于缓存比较的 Response ID
  },
  "service_tier": "auto", // 服务层级，auto 表示自动选择
  "safety_identifier": "user_hash_123", // 最终用户的稳定安全标识
  "background": false, // 是否将 Response 放到后台运行
  "context_management": [ // 上下文管理配置
    {
      "type": "compaction", // 上下文压缩策略类型
      "compact_threshold": 200000 // 达到阈值后触发上下文管理
    }
  ],
  "moderation": { // moderation 配置
    "model": "你的 moderation 模型", // 使用的 moderation 模型
    "policy": "default" // moderation 策略
  }
}
```

实际普通聊天通常只需要下面几个参数：

```jsonc
{
  "model": "你的实际模型ID", // 使用哪个模型
  "instructions": "你是一个专业的中文技术助手。", // 系统/开发者指令
  "input": "什么是 Elasticsearch？", // 用户输入
  "max_output_tokens": 1000, // 最大输出 token
  "temperature": 0.2 // 输出随机性
}
```

Responses API 也可以把 `input` 直接写成字符串，这是最简单的调用方式。

当模型直接产生普通文本回答时，Response 大致如下：

```jsonc
{
  "id": "resp_123456", // 本次 Response 的唯一 ID
  "object": "response", // Response 对象类型，固定为 response
  "created_at": 1760000000, // Response 创建时间，Unix timestamp
  "status": "completed", // Response 状态：completed=完成
  "error": null, // 生成失败时这里会包含错误信息，正常返回通常为 null
  "incomplete_details": null, // 如果 Response 没有完整生成，这里说明未完成原因
  "instructions": "你是一个专业的中文技术助手。", // 本次 Response 使用的 instructions
  "max_output_tokens": 1000, // 本次 Response 的最大输出 token 限制
  "model": "你的实际模型ID", // 实际使用的模型
  "output": [ // 模型生成的 output item 列表
    {
      "id": "msg_123", // 当前 output item 的唯一 ID
      "type": "message", // output 类型，这里是普通 message
      "status": "completed", // 当前 message 的状态
      "role": "assistant", // 模型角色
      "content": [ // message 的内容列表
        {
          "type": "output_text", // 输出内容类型，这里是文本
          "text": "Elasticsearch 是一个基于 Lucene 的分布式搜索与分析引擎。", // 模型实际输出文本
          "annotations": [] // 引用、注释等附加信息
        }
      ]
    }
  ],
  "parallel_tool_calls": true, // 是否允许并行工具调用
  "previous_response_id": null, // 上一轮 Response ID，首次请求为空
  "reasoning": { // reasoning 配置
    "effort": "medium", // 推理强度
    "summary": "auto" // reasoning summary 配置
  },
  "store": true, // 是否保存本次 Response
  "temperature": 0.2, // 实际使用的 temperature
  "text": { // 实际文本输出配置
    "format": {
      "type": "text" // 普通文本格式
    },
    "verbosity": "medium" // 输出详细程度
  },
  "tool_choice": "auto", // 工具选择策略
  "tools": [], // 本次 Response 使用的工具列表
  "top_p": 1.0, // 实际使用的 top_p
  "truncation": "disabled", // 实际使用的上下文截断策略
  "usage": { // Token 使用量
    "input_tokens": 100, // 输入 token 数
    "input_tokens_details": {
      "cached_tokens": 20, // 输入中命中缓存的 token 数
      "cache_write_tokens": 0 // 写入缓存的 token 数
    },
    "output_tokens": 50, // 输出 token 数
    "output_tokens_details": {
      "reasoning_tokens": 10 // reasoning token 数
    },
    "total_tokens": 150 // 总 token 数
  },
  "metadata": {} // 本次 Response 的业务元数据
}
```


Responses API 最需要理解的是：`"output": [...]`，它不是简单的一段字符串，而是数组，其中每一个元素都可能是不同的类型。

例如：

```text
output
├── message
├── function_call
├── reasoning
├── web_search_call
├── file_search_call
└── ...
```

也就是说，Responses API 的核心设计是：

```text
input -> 模型 -> output[]
```

而不是 Chat Completions 的：

```text
messages -> 模型 -> choices[]
```

`output` 的长度和顺序取决于模型的实际响应，不应该简单假设 `output[0]` 永远是 assistant message；SDK 提供 `output_text` 来方便获取最终文本。


假设我们定义一个天气函数：

```jsonc
{
  "model": "你的实际模型ID", // 使用哪个模型
  "input": "北京今天的天气怎么样？", // 用户问题
  "tools": [
    {
      "type": "function", // 自定义函数工具
      "name": "get_weather", // 函数名称
      "description": "根据城市查询当前天气", // 函数用途说明
      "parameters": { // 函数参数 JSON Schema
        "type": "object", // 参数整体为对象
        "properties": {
          "city": {
            "type": "string", // city 为字符串
            "description": "城市名称，例如北京" // 参数说明
          }
        },
        "required": ["city"], // city 必填
        "additionalProperties": false // 不允许额外参数
      },
      "strict": true // 严格按照 JSON Schema 调用
    }
  ],
  "tool_choice": "auto", // 由模型自己决定是否调用工具
  "parallel_tool_calls": true // 允许并行调用多个工具
}
```

Responses API 的 function tool 和 Chat Completions 有一个明显区别：

### Chat Completions

```jsonc
{
  "type": "function",
  "function": {
    "name": "get_weather",
    "description": "查询天气",
    "parameters": {}
  }
}
```

### Responses

```jsonc
{
  "type": "function",
  "name": "get_weather",
  "description": "查询天气",
  "parameters": {}
}
```

也就是 Responses API 把 `function` 这一层去掉了，函数定义直接放在 tool 对象上。官方当前 Responses function tool schema 就是这种结构。

调用函数时的 Response

```jsonc
{
  "id": "resp_123456", // 本次 Response ID
  "object": "response", // 对象类型
  "created_at": 1760000000, // 创建时间
  "status": "completed", // 当前 Response 状态
  "error": null, // 错误信息，正常为 null
  "incomplete_details": null, // 未完成原因
  "model": "你的实际模型ID", // 使用的模型
  "output": [
    {
      "type": "function_call", // 当前 output item 是函数调用
      "id": "fc_123", // Function Call 的唯一 ID
      "call_id": "call_123", // 函数调用关联 ID
      "name": "get_weather", // 要执行的函数名称
      "arguments": "{\"city\":\"北京\"}", // 模型生成的函数参数，JSON 字符串
      "status": "completed" // 当前函数调用状态
    }
  ],
  "parallel_tool_calls": true, // 是否允许并行工具调用
  "previous_response_id": null, // 上一轮 Response ID
  "reasoning": {
    "effort": null, // reasoning 强度
    "summary": null // reasoning summary
  },
  "store": true, // 是否保存 Response
  "temperature": 1.0, // 实际使用的 temperature
  "text": {
    "format": {
      "type": "text" // 文本输出格式
    }
  },
  "tool_choice": "auto", // 工具选择模式
  "tools": [
    {
      "type": "function", // 工具类型
      "name": "get_weather", // 工具名称
      "description": "根据城市查询当前天气", // 工具说明
      "parameters": { // 工具参数 Schema
        "type": "object",
        "properties": {
          "city": {
            "type": "string"
          }
        },
        "required": ["city"]
      },
      "strict": true // 严格参数匹配
    }
  ],

  "top_p": 1.0, // 实际使用的 top_p
  "truncation": "disabled", // 上下文截断策略
  "usage": {
    "input_tokens": 100, // 输入 token
    "input_tokens_details": {
      "cached_tokens": 0, // 命中缓存的 token
      "cache_write_tokens": 0 // 写入缓存的 token
    },
    "output_tokens": 20, // 输出 token
    "output_tokens_details": {
      "reasoning_tokens": 0 // reasoning token
    },
    "total_tokens": 120 // 总 token
  },
  "metadata": {} // 元数据
}
```

官方当前 Responses function calling 返回项使用 `type = function_call`，包含 `call_id`、`name`、`arguments` 和 `status` 等字段。

模型返回：

```json
{
  "name": "get_weather",
  "arguments": "{\"city\":\"北京\"}"
}
```

你的程序负责真正执行函数：

```python
weather = get_weather(city="北京")
```

例如得到：

```json
{
  "city": "北京",
  "weather": "晴",
  "temperature": 25
}
```

得到函数结果以后，需要把 `function_call_output` 作为下一轮输入：

```jsonc
{
  "model": "你的实际模型ID", // 使用的模型

  "previous_response_id": "resp_123456", // 指向刚才产生 function_call 的 Response

  "input": [
    {
      "type": "function_call_output", // 表示这是函数执行结果

      "call_id": "call_123", // 必须对应之前 function_call 的 call_id

      "output": "{\"city\":\"北京\",\"weather\":\"晴\",\"temperature\":25}" // 你的程序实际执行函数后的结果
    }
  ]
}
```

**多轮对话**

Responses API 的一个重要能力是多轮回复，假设请求：

```jsonc
{
  "model": "你的实际模型ID", // 使用的模型
  "input": "我叫 Tom" // 第一轮输入
}
```

返回：

```text
resp_abc123
```

第二轮：

```jsonc
{
  "model": "你的实际模型ID", // 使用的模型
  "previous_response_id": "resp_abc123", // 指定上一轮 Response
  "input": "我叫什么？" // 新的问题
}
```

这样就可以继续之前的 Response 上下文。官方明确将 `previous_response_id` 定义为用于创建 multi-turn conversations，并说明它不能和 `conversation` 同时使用。


**reasoning**

Responses API 对 reasoning 有专门的配置：

```jsonc
{
  "reasoning": {
    "effort": "high", // 控制 reasoning 强度
    "summary": "auto" // 是否生成 reasoning summary
  }
}
```

例如：

```text
effort = none
→ 不额外进行 reasoning

effort = low
→ 较少推理

effort = medium
→ 中等推理

effort = high
→ 更强推理
```

具体可用的值取决于模型。

并且 usage 中可以进一步看到：

```jsonc
{
  "output_tokens_details": {
    "reasoning_tokens": 832 // reasoning 使用的 token 数
  }
}
```

官方当前 Responses schema 明确包含 `reasoning` 配置，以及 `usage.output_tokens_details.reasoning_tokens`。

---

# 十三、`text`：普通文本和结构化输出

Responses API 不使用 Chat Completions 的：

```text
response_format
```

作为主要文本输出配置，而是使用：

```jsonc
{
  "text": {
    "format": {
      "type": "text" // 普通文本
    },
    "verbosity": "medium" // 控制输出详细程度
  }
}
```

如果要求 JSON Schema，可以配置：

```jsonc
{
  "text": {
    "format": {
      "type": "json_schema", // 使用 JSON Schema 约束输出
      "name": "person", // Schema 名称
      "strict": true, // 是否严格遵守 Schema
      "schema": {
        "type": "object",
        "properties": {
          "name": {
            "type": "string"
          },
          "age": {
            "type": "integer"
          }
        },
        "required": [
          "name",
          "age"
        ],
        "additionalProperties": false
      }
    }
  }
}
```

所以两套 API 的结构是：

```text
Chat Completions
    ↓
response_format

Responses
    ↓
text.format
```

**事件流响应**

请求：

```jsonc
{
  "model": "你的实际模型ID", // 使用的模型
  "input": "写一个故事", // 用户输入
  "stream": true, // 开启流式
  "stream_options": {
    "include_obfuscation": false // 流式 obfuscation 配置
  }
}
```

Responses API 的流式模型不是简单的：

```text
一个 chunk
一个 chunk
一个 chunk
```

而是**事件流**。

例如：

```text
response.created
response.in_progress
response.output_item.added
response.content_part.added
response.output_text.delta
response.output_text.done
response.content_part.done
response.output_item.done
response.completed
```

官方当前 Responses streaming API 就是通过这些 typed events 描述 Response 的生成生命周期。


### chat completions 与 responses 流式响应的区别

Chat Completions 的流式响应主要是“不断返回 message 的 delta”；Responses API 的流式响应是“不断返回有类型的事件（event）”。

Chat Completions 返回的不是一个完整 JSON，而是一串 SSE 数据：

```jsonc
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"role":"assistant"},"finish_reason":null}]}
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":"Elasticsearch"},"finish_reason":null}]}
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" 是一个"},"finish_reason":null}]}
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","choices":[{"index":0,"delta":{"content":" 分布式搜索引擎。"},"finish_reason":null}]}
data: {"id":"chatcmpl-123","object":"chat.completion.chunk","choices":[{"index":0,"delta":{},"finish_reason":"stop"}]}
data: [DONE]
```

Responses API 流式响应:

```jsonc
event: response.created
data: {...}

event: response.in_progress
data: {...}

event: response.output_item.added
data: {...}

event: response.content_part.added
data: {...}

event: response.output_text.delta
data: {"delta":"Elasticsearch"}

event: response.output_text.delta
data: {"delta":" 是一个"}

event: response.output_text.delta
data: {"delta":" 分布式搜索引擎。"}

event: response.output_text.done
data: {...}

event: response.content_part.done
data: {...}

event: response.output_item.done
data: {...}

event: response.completed
data: {...}
```


普通请求：

```http
POST /v1/responses HTTP/1.1
Host: api.openai.com
Authorization: Bearer sk-xxx
Content-Type: application/json

{
  "model": "xxx",
  "input": "介绍一下 Elasticsearch"
}
```

流式请求：

```http
POST /v1/responses HTTP/1.1
Host: api.openai.com
Authorization: Bearer sk-xxx
Content-Type: application/json

{
  "model": "xxx",
  "input": "介绍一下 Elasticsearch",
  "stream": true
}
```

```
HTTP/1.1 200 OK
Content-Type: text/event-stream
Cache-Control: no-cache
Connection: keep-alive
```
