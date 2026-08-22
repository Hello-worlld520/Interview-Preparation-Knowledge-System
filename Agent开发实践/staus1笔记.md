# staus1笔记

# 1 会用一个 LLM API 完成普通对话。

```python
import os
from openai import OpenAI
# openai就是让代码能够接大模型API的库，OpenAI是库里的一个类，定义了接口是OpenAI形式的，虽然ChatGPT是原生家庭，但是现在很多大模型都支持这个接口模式
client = OpenAI(
    api_key="tp-cq3cz7b6j4h5fukwvtx111bmcv6jbs1xmqrlsoyt7f9x3z3y",
    base_url="https://token-plan-cn.xiaomimimo.com/v1",   # 推荐用 /v1（OpenAI 兼容）
)
# 创建了一个名为 client 的客户端对象，添加访问地址和密钥
messages = [
    {"role": "system", "content": "你是一个友好的中文助手，回答简洁清晰。"}
]
# messages 是一个列表，里面装着一条条字典格式的消息
# message相当于在对话框里给大模型发的消息，对message的精简优化是agent要干的事情，如果不处理就会消耗大量token
print("开始对话（输入 exit 退出）\n")

while True:# 无限循环，直到break结束
    user_input = input("你: ").strip()
    # 变量名=输入的内容去掉空格等符号
    if user_input.lower() in {"exit", "quit", "退出"}:
        # .lower()字符串方法，将所有输入都转换成小写，避免大小写敏感
        print("对话结束")
        break

    messages.append({"role": "user", "content": user_input})
# 追加一个字典
    response = client.chat.completions.create(
        model="mimo-v2.5-pro",          # ← 正确的模型名
        messages=messages,
        temperature=0.7,
    )
# 客户端对象的使用，调用方法并传入参数
    assistant_reply = response.choices[0].message.content
    print(f"AI: {assistant_reply}\n")

    messages.append({"role": "assistant", "content": assistant_reply})
```

# 2 在此基础上让模型输出结构化JSON

* 和模型说要生成JSON结构化参数
* 对传出的JSON结构化参数进行包装成为字典（才能调用工具什么的）

```python
import os
import json
from openai import OpenAI

client = OpenAI(
    api_key="tp-cq3cz7b6j4h5fukwvtx111bmcv6jbs1xmqrlsoyt7f9x3z3y",
    base_url="https://token-plan-cn.xiaomimimo.com/v1",
)

# 系统提示：明确要求输出 JSON
messages = [
    {
        "role": "system",
        "content": """你是一个结构化信息提取助手。
请始终以合法的 JSON 格式回复，不要添加任何多余的文字或解释。
JSON 格式示例：
{
  "reply": "你对用户的回复内容",
  "intent": "用户意图（如：闲聊、提问、指令等）",
  "entities": {}
}"""
    }
]

print("开始对话（输入 exit 退出）\n")

while True:
    user_input = input("你: ").strip()
    if user_input.lower() in {"exit", "quit", "退出"}:
        print("对话结束")
        break

    messages.append({"role": "user", "content": user_input})

    try:
        response = client.chat.completions.create(
            model="mimo-v2.5-pro",
            messages=messages,
            temperature=0.3,                          # 温度低一点，JSON 更稳定
            response_format={"type": "json_object"}   # 强制输出 JSON
        )

        assistant_reply = response.choices[0].message.content
        print(f"AI 原始输出: {assistant_reply}\n")

        # 尝试解析 JSON
        try:
            data = json.loads(assistant_reply)
            print("解析后的结构化数据：")
            print(json.dumps(data, ensure_ascii=False, indent=2))
            print("-" * 40)
        except json.JSONDecodeError:
            print("警告：模型没有输出合法的 JSON")

        # 把模型回复加入历史（保持多轮对话能力）
        messages.append({"role": "assistant", "content": assistant_reply})

    except Exception as e:
        print(f"出错了：{e}")
        
        
        
├── 1. 准备工作
│   ├── 导入工具：json、openai
│   ├── 创建客户端：client = OpenAI(...)
│   └── 设定规则：告诉AI要输出JSON格式
│
├── 2. 开始对话
│   └── 打印欢迎语
│
└── 3. 对话循环（一直重复）
    │
    ├── 第1步：获取你说的话
    │   └── 如果是"退出" → 结束程序
    │
    ├── 第2步：把你的话加入聊天记录
    │
    ├── 第3步：调用AI（外层try保底）
    │   ├── 发送聊天记录给AI
    │   ├── AI返回JSON格式的回复
    │   └── 提取AI说的话
    │
    ├── 第4步：解析JSON（内层try保底）
    │   ├── 成功 → 拆开JSON，展示里面的数据
    │   └── 失败 → 打印警告
    │
    ├── 第5步：把AI的回复加入聊天记录
    │
    └── 回到开头，等你下一句话
```

##### python中try的用法

```python
try:
    # 尝试执行的代码（可能出错的代码）
    可能会出错的代码()
except 错误类型1:
    # 如果发生 错误类型1，执行这里的代码
    处理方式1()
except 错误类型2:
    # 如果发生 错误类型2，执行这里的代码
    处理方式2()
else:
    # 如果没发生任何错误，执行这里的代码（可选）
    没有出错时做的事()
finally:
    # 不管有没有出错，最后都会执行这里的代码（可选）
    无论如何都要做的事()
```

#### 为什么要先弄成字典再输入大模型，大模型输出的JSON数据直接当成文本输入大模型不行吗

```
messages.append({"role": "assistant", "content": assistant_reply})  # 这里用的就是字符串
```

这部分**不需要**解析成字典。

**需要解析成字典的原因**是：你的**Python 程序**要使用这些数据做判断。

| 用途                                                 | 是否需要解析成字典 | 说明               |
| ---------------------------------------------------- | ------------------ | ------------------ |
| 把回复放回 messages，继续对话                        | 不需要             | 直接用字符串就行   |
| 判断用户意图、提取字段、决定调用哪个工具、存数据库等 | **需要**           | 程序要读取具体字段 |

####  为什么还要额外加 response_format={"type": "json_object"}？

因为**只靠 Prompt 不能 100% 保证**模型输出的是合法 JSON。

即使你在 system prompt 里写了“请输出 JSON”，模型仍可能：

- 前面加解释：「好的，以下是结果：{...}」
- 后面加废话
- 输出不完整的 JSON
- 偶尔格式错误

加了这行参数后：

```
response_format={"type": "json_object"}
```

模型会被**强制**要求输出合法的 JSON 对象，成功率会高很多（尤其是支持这个参数的模型）。

# 3 会定义一个工具函数

```python
import os
import json
from openai import OpenAI

client = OpenAI(
    api_key="tp-cq3cz7b6j4h5fukwvtx111bmcv6jbs1xmqrlsoyt7f9x3z3y",
    base_url="https://token-plan-cn.xiaomimimo.com/v1",
)
#定义客户端对象
# -------------------- 第1步：定义工具函数 --------------------
def get_weather(city: str) -> str:
    """获取指定城市的天气（模拟实现）"""
    # 这里可以替换为真实的天气 API 调用
    weather_data = {
        "北京": "晴天，25°C",
        "上海": "多云，28°C",
        "广州": "雷阵雨，30°C",
    }
    return weather_data.get(city, f"未找到城市 {city} 的天气数据")

# -------------------- 第2步：向 AI 描述工具 --------------------
tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，如：北京、上海、广州"
                    }
                },
                "required": ["city"]
            }
        }
    }
]

# 系统提示（可以保持不变）
messages = [
    {
        "role": "system",
        "content": """你是一个智能助手，可以调用工具来获取信息。
当用户询问天气时，请使用 get_weather 工具。
如果用户问其他问题，你可以直接回答。"""
    }
]

print("开始对话（输入 exit 退出）\n")

while True:
    user_input = input("你: ").strip()
    if user_input.lower() in {"exit", "quit", "退出"}:
        print("对话结束")
        break

    messages.append({"role": "user", "content": user_input})

    try:
        # -------------------- 第3步：调用 AI 并传入工具定义 --------------------
        response = client.chat.completions.create(
            model="mimo-v2.5-pro",
            messages=messages,
            tools=tools,                    # ⬅️ 传入工具定义
            tool_choice="auto",              # 让 AI 决定是否调用
            temperature=0.3
        )

        assistant_message = response.choices[0].message
        messages.append(assistant_message)   # 先把 AI 的回复加入历史

        # -------------------- 第4步：检查 AI 是否想调用工具 --------------------
        if assistant_message.tool_calls:
            for tool_call in assistant_message.tool_calls:
                function_name = tool_call.function.name
                function_args = json.loads(tool_call.function.arguments)

                # -------------------- 第5步：执行对应的工具函数 --------------------
                if function_name == "get_weather":
                    city = function_args.get("city")
                    result = get_weather(city)   # 调用真正的工具函数

                    # 把工具执行结果返回给 AI
                    messages.append({
                        "role": "tool",
                        "tool_call_id": tool_call.id,
                        "content": result
                    })

            # 把工具执行结果再发给 AI，让 AI 组织最终回复
            second_response = client.chat.completions.create(
                model="mimo-v2.5-pro",
                messages=messages,
                temperature=0.3
            )
            final_reply = second_response.choices[0].message.content
            print(f"AI: {final_reply}\n")
            messages.append({"role": "assistant", "content": final_reply})
        else:
            # 没有调用工具，直接输出回复
            print(f"AI: {assistant_message.content}\n")
            messages.append({"role": "assistant", "content": assistant_message.content})

    except Exception as e:
        print(f"出错了：{e}")



程序
├── 初始化
│   ├── 导入库 (os, json, openai)
│   └── 创建客户端
│
├── 工具系统
│   ├── get_weather() 函数
│   └── tools 描述
│
├── 会话配置
│   └── system messages
│
└── 主循环
    ├── 输入处理
    ├── 第一次AI请求
    ├── 判断
    │   ├── 有工具调用 → 执行 → 第二次AI请求 → 输出
    │   └── 无工具调用 → 直接输出
    └── 异常处理
```

AI是否想调用工具，**完全取决于响应消息中是否包含 `tool_calls` 字段**。

#### 真实的天气API调用如下

```py
import os
import requests  # 需要先安装：pip install requests

def get_weather(city: str) -> str:
    """使用 OpenWeatherMap API 获取真实天气"""
    # 1. 从环境变量安全获取密钥 (不要硬编码在代码里!)
    api_key = os.environ.get("OPENWEATHER_API_KEY")
    if not api_key:
        return "错误：未配置 OpenWeatherMap API 密钥"

    # 2. 构建 API 请求 URL
    # units=metric 表示使用摄氏温度单位
    url = f"http://api.openweathermap.org/data/2.5/weather?q={city}&appid={api_key}&units=metric&lang=zh_cn"

    try:
        # 3. 发送网络请求
        response = requests.get(url, timeout=10)
        response.raise_for_status()  # 如果状态码不是200，会抛出异常

        # 4. 解析返回的JSON数据
        data = response.json()

        # 5. 提取我们需要的核心信息
        city_name = data['name']
        temp = data['main']['temp']
        condition = data['weather'][0]['description']

        # 6. 格式化成友好的字符串返回
        return f"{city_name}天气：{condition}，{temp}°C"

    except requests.exceptions.RequestException as e:
        # 处理网络请求错误
        return f"获取天气信息失败：{e}"
    except KeyError as e:py
        # 处理数据解析错误（比如城市不存在）
        return f"未找到城市 {city} 的天气数据，请检查城市名是否正确。"
```

# 4 再添加一个函数

```
改动1: 定义 read_file 工具函数
改动2: 在 tools 列表中添加 read_file 的描述
改动3: 在系统提示中添加 read_file 的使用说明
改动4: 在工具执行路由中添加 read_file 的分支
```

```python
import os
import json
from openai import OpenAI

client = OpenAI(
    api_key="tp-cq3cz7b6j4h5fukwvtx111bmcv6jbs1xmqrlsoyt7f9x3z3y",
    base_url="https://token-plan-cn.xiaomimimo.com/v1",
)

# ==================== 改动1：定义 read_file 工具函数 ====================
def read_file(file_path: str) -> str:
    """
    读取指定路径的文件内容
    """
    try:
        with open(file_path, 'r', encoding='utf-8') as f:
            content = f.read()
        return f"文件内容：\n{content}"
    except FileNotFoundError:
        return f"错误：未找到文件 {file_path}"
    except Exception as e:
        return f"读取文件失败：{e}"


# ==================== 改动2：在 tools 中添加 read_file ====================
tools = [
    # 原有的 get_weather 工具
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {
                        "type": "string",
                        "description": "城市名称，如：北京、上海、广州"
                    }
                },
                "required": ["city"]
            }
        }
    },
    # ✅ 新增的 read_file 工具
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "读取文件内容，用于查看文件里的文本信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "file_path": {
                        "type": "string",
                        "description": "文件路径，如：/Users/username/readme.txt 或 C:\\Users\\name\\file.txt"
                    }
                },
                "required": ["file_path"]
            }
        }
    }
]

# ==================== 改动3：更新系统提示 ====================
messages = [
    {
        "role": "system",
        "content": """你是一个智能助手，可以调用工具来获取信息。
可用的工具：
1. get_weather - 获取指定城市的天气信息
2. read_file - 读取文件内容

当用户询问天气时，使用 get_weather 工具。
当用户想查看文件内容时，使用 read_file 工具。
如果用户问其他问题，你可以直接回答。"""
    }
]

print("开始对话（输入 exit 退出）\n")

while True:
    user_input = input("你: ").strip()
    if user_input.lower() in {"exit", "quit", "退出"}:
        print("对话结束")
        break

    messages.append({"role": "user", "content": user_input})

    try:
        response = client.chat.completions.create(
            model="mimo-v2.5-pro",
            messages=messages,
            tools=tools,
            tool_choice="auto",
            temperature=0.3
        )

        assistant_message = response.choices[0].message
        messages.append(assistant_message)

        if assistant_message.tool_calls:
            for tool_call in assistant_message.tool_calls:
                function_name = tool_call.function.name
                function_args = json.loads(tool_call.function.arguments)

                # ==================== 改动4：添加 read_file 路由 ====================
                if function_name == "get_weather":
                    city = function_args.get("city")
                    result = get_weather(city)

                elif function_name == "read_file":    # ✅ 新增分支
                    file_path = function_args.get("file_path")
                    result = read_file(file_path)

                else:
                    result = f"未知工具：{function_name}"

                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })

            second_response = client.chat.completions.create(
                model="mimo-v2.5-pro",
                messages=messages,
                temperature=0.3
            )
            final_reply = second_response.choices[0].message.content
            print(f"AI: {final_reply}\n")
            messages.append({"role": "assistant", "content": final_reply})
        else:
            print(f"AI: {assistant_message.content}\n")
            messages.append({"role": "assistant", "content": assistant_message.content})

    except Exception as e:
        print(f"出错了：{e}")
```

### 添加新工具的三步口诀

> **1. 写函数 → 2. 写描述 → 3. 写路由**

```
def new_tool():          ← 第1步：实现功能
    pass

tools.append({...})      ← 第2步：描述给AI

if function_name == "new_tool":  ← 第3步：路由执行
    result = new_tool(...)
```

# 5 关于解析模型的 Tool Call / Function Call

**当 AI 决定调用工具时，返回的数据结构长这样：**

```python
{
  "choices": [
    {
      "message": {
        "role": "assistant",
        "content": null,
        "tool_calls": [
          {
            "id": "call_abc123",
            "type": "function",
            "function": {
              "name": "get_weather",
              "arguments": "{\"city\": \"北京\"}"
            }
          }
        ]
      }
    }
  ]
}
```

### 解析过程

```python
import json

# 接收AI响应
response = client.chat.completions.create(...)
message = response.choices[0].message

# 解析工具调用
if message.tool_calls:
    for tool_call in message.tool_calls:
        # 提取字段
        call_id = tool_call.id
        func_name = tool_call.function.name
        arguments_str = tool_call.function.arguments
        
        # 解析参数（字符串→字典）
        arguments_dict = json.loads(arguments_str)
        
        # 执行工具
        if func_name == "get_weather":
            result = get_weather(**arguments_dict)
        elif func_name == "search_news":
            result = search_news(**arguments_dict)
        
        # 返回结果
        messages.append({
            "role": "tool",
            "tool_call_id": call_id,
            "content": str(result)
        })
else:
    # 直接回答
    answer = message.content
```

# 6 给 agent loop 加最大步数、超时和错误处理

**为什么要这么干？**

Agent本质上是自主决策，自动执行的程序，没有限制容易死循环

```python
import time

MAX_STEPS = 5
TIMEOUT = 30

print("开始对话（输入 exit 退出）\n")

while True:
    user_input = input("你: ").strip()
    if user_input.lower() in {"exit", "quit", "退出"}:
        print("对话结束")
        break

    messages.append({"role": "user", "content": user_input})

    # ===== Agent Loop =====
    step = 0
    start_time = time.time()
    
    while step < MAX_STEPS:
        step += 1
        print(f"--- 第 {step} 步 ---")
        
        # ✅ 超时检查
        if time.time() - start_time > TIMEOUT:
            print("错误：执行超时，强制退出")
            final_reply = "抱歉，执行超时了，请简化问题重试。"
            break

        try:
            response = client.chat.completions.create(
                model="mimo-v2.5-pro",
                messages=messages,
                tools=tools,
                tool_choice="auto",
                temperature=0.3
            )

            assistant_message = response.choices[0].message
            messages.append(assistant_message)

            # 没有工具调用 → 结束循环
            if not assistant_message.tool_calls:
                print(f"AI: {assistant_message.content}\n")
                break

            # 有工具调用 → 执行工具
            for tool_call in assistant_message.tool_calls:
                function_name = tool_call.function.name
                function_args = json.loads(tool_call.function.arguments)

                # ✅ 工具执行错误处理
                try:
                    if function_name == "get_weather":
                        city = function_args.get("city")
                        result = get_weather(city)
                    elif function_name == "read_file":
                        file_path = function_args.get("file_path")
                        result = read_file(file_path)
                    else:
                        result = f"未知工具：{function_name}"
                except Exception as tool_error:
                    result = f"工具执行失败：{str(tool_error)}"
                    print(f"工具执行出错：{tool_error}")
# Python 在执行代码时，如果遇到问题，会自动创建一个对应的异常对象，并"抛"出来。这里被放到了tool_error里被打印出来，就可以在控制台看到出错原因了
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })

            # 把工具结果再发给 AI（如果有工具调用）
            second_response = client.chat.completions.create(
                model="mimo-v2.5-pro",
                messages=messages,
                temperature=0.3
            )
            final_reply = second_response.choices[0].message.content
            print(f"AI: {final_reply}\n")
            messages.append({"role": "assistant", "content": final_reply})

        except Exception as e:
            print(f"出错了：{e}")
            break
    
    # ✅ 检查是否因步数限制退出
    if step >= MAX_STEPS and not assistant_message.tool_calls:
        print("警告：已达到最大执行步数，强制退出")
        messages.append({"role": "assistant", "content": "抱歉，处理步骤过多，请简化问题重试。"})
        
        
程序执行流程
│
├── 1. 环境初始化
│   ├── import time
│   ├── 设置常量：MAX_STEPS = 5, TIMEOUT = 30
│   └── 打印欢迎语
│
└── 2. 对话循环 (while True)
    │
    ├── 2.1 获取用户输入
    │   ├── user_input = input("你: ").strip()
    │   └── 判断退出条件 → break
    │
    ├── 2.2 加入聊天记录
    │   └── messages.append({"role": "user", "content": user_input})
    │
    └── 2.3 Agent Loop (while step < MAX_STEPS)
        │
        ├── 2.3.1 步数计数
        │   ├── step += 1
        │   └── print(f"--- 第 {step} 步 ---")
        │
        ├── 2.3.2 超时检查
        │   └── if time.time() - start_time > TIMEOUT:
        │       ├── print("错误：执行超时，强制退出")
        │       ├── final_reply = "抱歉，执行超时了..."
        │       └── break
        │
        ├── 2.3.3 调用 AI（外层 try）
        │   │
        │   ├── response = client.chat.completions.create(
        │   │   ├── model: "mimo-v2.5-pro"
        │   │   ├── messages: messages
        │   │   ├── tools: tools
        │   │   ├── tool_choice: "auto"
        │   │   └── temperature: 0.3
        │   │   )
        │   │
        │   ├── assistant_message = response.choices[0].message
        │   └── messages.append(assistant_message)
        │
        ├── 2.3.4 判断是否有工具调用
        │   │
        │   ├── 情况A：没有工具调用
        │   │   ├── print(f"AI: {assistant_message.content}\n")
        │   │   └── break (结束 Agent Loop)
        │   │
        │   └── 情况B：有工具调用
        │       │
        │       ├── 遍历所有 tool_calls
        │       │   for tool_call in assistant_message.tool_calls:
        │       │       ├── function_name = tool_call.function.name
        │       │       └── function_args = json.loads(tool_call.function.arguments)
        │       │
        │       ├── 执行工具（内层 try）
        │       │   │
        │       │   ├── 路由判断
        │       │   │   ├── if function_name == "get_weather":
        │       │   │   │   └── result = get_weather(city)
        │       │   │   ├── elif function_name == "read_file":
        │       │   │   │   └── result = read_file(file_path)
        │       │   │   └── else:
        │       │   │       └── result = f"未知工具：{function_name}"
        │       │   │
        │       │   └── 异常捕获
        │       │       └── except Exception as tool_error:
        │       │           ├── result = f"工具执行失败：{str(tool_error)}"
        │       │           └── print(f"工具执行出错：{tool_error}")
        │       │
        │       ├── 构造 Tool 消息
        │       │   └── messages.append({
        │       │       ├── role: "tool"
        │       │       ├── tool_call_id: tool_call.id
        │       │       └── content: result
        │       │       })
        │       │
        │       ├── 第二次调用 AI
        │       │   ├── second_response = client.chat.completions.create(
        │       │   │   ├── model: "mimo-v2.5-pro"
        │       │   │   ├── messages: messages
        │       │   │   └── temperature: 0.3
        │       │   │   )
        │       │   ├── final_reply = second_response.choices[0].message.content
        │       │   ├── print(f"AI: {final_reply}\n")
        │       │   └── messages.append({"role": "assistant", "content": final_reply})
        │       │
        │       └── 回到 Agent Loop 开头（继续下一步）
        │
        ├── 2.3.5 异常捕获 (外层 except)
        │   ├── print(f"出错了：{e}")
        │   └── break
        │# 外层 except 捕获的是：调用 AI API 时可能出现的网络/服务错误。
        └── 2.3.6 步数限制检查
            └── if step >= MAX_STEPS and not assistant_message.tool_calls:
                ├── print("警告：已达到最大执行步数，强制退出")
                └── messages.append({"role": "assistant", "content": "抱歉，处理步骤过多..."})
```

## 两种循环的关系

```
对话循环 (while True)
    │
    ├── 用户第1次输入
    │   └── Agent Loop (最多5步)
    │       ├── 第1步 → 有工具调用 → 继续
    │       ├── 第2步 → 有工具调用 → 继续
    │       └── 第3步 → 无工具调用 → 结束
    │
    ├── 用户第2次输入
    │   └── Agent Loop (最多5步)
    │       ├── 第1步 → 无工具调用 → 结束
    │
    └── 用户第3次输入 → 退出
```

### 最大步数是调用工具的次数，不是对话次数

### 关于错误处理

* try  ... except ...就是尝试执行这个函数，如果失败怎么怎么样

这样处理之后不会影响主体代码的执行，还可以将报错信息直接喂回给AI，让AI来处理异常情况

* raise就是不处理异常，向上层汇报
* raise代码执行了，同层的后面的代码就不会执行了

# 7 **参数tool choice**

**tool_choice有哪些**：

| 写法                                              | 效果                                   |
| ------------------------------------------------- | -------------------------------------- |
| `tool_choice="auto"`                              | 模型自己决定是否调用、调用哪些（默认） |
| `tool_choice="required"`                          | 必须调用至少一个工具                   |
| `tool_choice="none"`                              | 禁止调用任何工具                       |
| `tool_choice={"type": "function", "name": "xxx"}` | 强制调用某一个函数                     |
| `tool_choice={"type": "allowed_tools", ...}`      | 只允许调用指定子集                     |

### tool_choice是什么？写在哪里？

tool_choice 是 AI API 中的一个参数，用来控制 AI 是否调用工具以及如何调用工具

* #### 每一轮都用同一个 tool_choice

只需要在循环里的 create() 加上 tool_choice 即可，其他地方不用动。

```python
while turn < max_turns:
    turn += 1

    response = client.responses.create(
        model="gpt-5.6",
        tools=tools,
        input=input_list,

        # ========== 加在这里 ==========
        tool_choice={
            "type": "allowed_tools",
            "mode": "auto",
            "tools": [
                {"type": "function", "name": "get_horoscope"}
            ]
        },
        # ==============================
    )

    input_list += response.output

    function_calls = [
        item for item in response.output
        if getattr(item, "type", None) == "function_call"
    ]

    if not function_calls:
        print("最终回答：")
        print(response.output_text)
        break

    # 执行工具、追加结果……（这部分完全不用改）
    for item in function_calls:
        ...
```

**改动点只有一处**：在 create() 里加上 tool_choice=...。

* ### 根据轮次动态改变 tool_choice

**第 1 轮**：强制必须调用工具（用 required 或 allowed_tools + required）

**后面几轮**：让模型自己决定，或者禁止再调工具

```python
while turn < max_turns:
    turn += 1

    # ========== 根据轮次决定 tool_choice ==========
    if turn == 1:
        # 第一轮：强制只能用 get_horoscope，而且必须调用
        current_tool_choice = {
            "type": "allowed_tools",
            "mode": "required",
            "tools": [
                {"type": "function", "name": "get_horoscope"}
            ]
        }
    else:
        # 后面几轮：让模型自己决定（也可以改成 "none" 禁止再调工具）
        current_tool_choice = "auto"
    # =============================================

    response = client.responses.create(
        model="gpt-5.6",
        tools=tools,
        input=input_list,
        tool_choice=current_tool_choice,   # 用上面算出来的
    )

    input_list += response.output

    function_calls = [
        item for item in response.output
        if getattr(item, "type", None) == "function_call"
    ]

    if not function_calls:
        print("最终回答：")
        print(response.output_text)
        break

    # 执行工具（这里完全不用改）
    for item in function_calls:
        name = item.name
        args = json.loads(item.arguments)
        call_id = item.call_id

        result = tool_map[name](**args)

        input_list.append({
            "type": "function_call_output",
            "call_id": call_id,
            "output": str(result),
        })
```