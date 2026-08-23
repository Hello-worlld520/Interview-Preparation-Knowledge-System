# stage 2  Learn Tool Use, RAG, And Memory

## 1 检索增强生成

| 词                        | 中文          | 简单解释                                                     |
| ------------------------- | ------------- | ------------------------------------------------------------ |
| **chunk**                 | 分块          | 把长文档切成一小段一小段（比如每 300 字一块）。模型一次只能看有限内容，太长的文档必须先切。 |
| **embed**                 | 向量化 / 嵌入 | 把每一块文字变成一串数字（向量）。意思相近的文字，向量在空间里会靠得比较近。 |
| **retrieve**              | 检索          | 用户提问后，用问题的向量去知识库里找最相关的那几块文字。     |
| **answer with citations** | 带引用回答    | 模型根据检索到的内容回答，并标明「这句话来自哪一块文档」，方便核对来源。 |

### 整体RAG流程

```
文档 → chunk → embed → 存进知识库
用户提问 → embed 问题 → retrieve 相关块 → 把相关块塞给模型 → 模型回答并带上引用
```

**加入检索增强生成的代码**

```python
import os
import time
import json
import re
from openai import OpenAI

client = OpenAI(
    api_key="tp-cq3cz7b6j4h5fukwvtx111bmcv6jbs1xmqrlsoyt7f9x3z3y",
    base_url="https://token-plan-cn.xiaomimimo.com/v1",
)

MAX_STEPS = 5
TIMEOUT = 30

# ==================== 知识库（可自行修改） ====================
KNOWLEDGE_DOCS = [
    {
        "id": "doc1",
        "title": "公司请假制度",
        "content": """
员工请假需提前至少一天在系统提交申请。事假每月不超过3天，病假需提供医院证明。
年假按工龄计算：满1年享有5天，满3年享有10天，满5年享有15天。
紧急情况可先口头请假，事后补办手续。未经批准擅自离岗按旷工处理。
"""
    },
    {
        "id": "doc2",
        "title": "报销流程说明",
        "content": """
差旅报销需在出差结束后7个工作日内提交。必须附上发票原件和行程单。
单笔超过5000元需部门经理审批，超过20000元需分管领导审批。
餐饮补助标准：一线城市100元/天，二线城市80元/天。交通优先公共交通。
"""
    },
    {
        "id": "doc3",
        "title": "办公室网络故障处理",
        "content": """
网络无法连接时，先检查网线或WiFi是否正常。重启路由器等待2分钟后再试。
若仍无法解决，拨打IT热线分机8888。工作日9:00-18:00响应，紧急情况可联系值班人员。
禁止私自更改IP地址和DNS设置。
"""
    },
]

# ==================== RAG 相关函数 ====================
def chunk_text(text: str, chunk_size: int = 80, overlap: int = 20) -> list[str]:
    # overlap是重叠，即对长文本进行分段（Chunking）时，前一段的结尾部分会包含到后一段的开头，让相邻两个片段之间有一部分内容是重复/重叠的。
    """把长文本切成小块（简单按字符切）"""
    text = re.sub(r"\s+", " ", text).strip()
    if len(text) <= chunk_size:
        return [text] if text else []
    chunks = []
    start = 0
    while start < len(text):
        end = start + chunk_size
        chunks.append(text[start:end])
        start = end - overlap
    return chunks

def simple_embed(text: str) -> dict:
    """
    简易「向量」：用词频代替真实 embedding。
    真实项目应改成调用 embeddings API。
    """
    words = re.findall(r"[\u4e00-\u9fff]+|[a-zA-Z0-9]+", text.lower())
    freq = {}
    for w in words:
        freq[w] = freq.get(w, 0) + 1
    return freq

def cosine_sim(a: dict, b: dict) -> float:
    """计算两个词频向量的余弦相似度"""
    common = set(a) & set(b)
    if not common:
        return 0.0
    dot = sum(a[w] * b[w] for w in common)
    norm_a = sum(v * v for v in a.values()) ** 0.5
    norm_b = sum(v * v for v in b.values()) ** 0.5
    if norm_a == 0 or norm_b == 0:
        return 0.0
    return dot / (norm_a * norm_b)

# 预先分块 + 向量化
CHUNK_STORE = []  # 每项: {id, title, chunk_id, text, vector}

def build_index():
    CHUNK_STORE.clear()
    for doc in KNOWLEDGE_DOCS:
        pieces = chunk_text(doc["content"])
        for i, piece in enumerate(pieces):
            CHUNK_STORE.append({
                "doc_id": doc["id"],
                "title": doc["title"],
                "chunk_id": f"{doc['id']}_c{i}",
                "text": piece,
                "vector": simple_embed(piece),
            })
    print(f"知识库已建立，共 {len(CHUNK_STORE)} 个文本块\n")

def retrieve(query: str, top_k: int = 3) -> list[dict]:
    """根据问题检索最相关的文本块"""
    q_vec = simple_embed(query)
    scored = []
    for item in CHUNK_STORE:
        score = cosine_sim(q_vec, item["vector"])
        if score > 0:
            scored.append((score, item))
    scored.sort(key=lambda x: x[0], reverse=True)
    return [item for _, item in scored[:top_k]]

def format_retrieved_context(chunks: list[dict]) -> str:
    """把检索结果整理成给模型看的上下文，并带上编号方便引用"""
    if not chunks:
        return "（未检索到相关内容）"
    lines = []
    for i, c in enumerate(chunks, 1):
        lines.append(f"[{i}] 来源：《{c['title']}》（{c['chunk_id']}）\n{c['text']}")
    return "\n\n".join(lines)

# ==================== 原来的工具函数 ====================
def get_weather(city: str) -> str:
    weather_data = {
        "北京": "晴天，25°C",
        "上海": "多云，28°C",
        "广州": "雷阵雨，30°C",
    }
    return weather_data.get(city, f"未找到城市 {city} 的天气数据")

def read_file(file_path: str) -> str:
    try:
        with open(file_path, "r", encoding="utf-8") as f:
            content = f.read()
        return f"文件内容：\n{content}"
    except FileNotFoundError:
        return f"错误：未找到文件 {file_path}"
    except Exception as e:
        return f"读取文件失败：{e}"

# ==================== 工具定义 ====================
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
    },
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
                        "description": "文件路径"
                    }
                },
                "required": ["file_path"]
            }
        }
    }
]

# ==================== 系统提示 ====================
SYSTEM_PROMPT = """你是一个智能助手，可以调用工具，也可以根据提供的「参考资料」回答问题。

可用的工具：
1. get_weather - 获取指定城市的天气信息
2. read_file - 读取文件内容

规则：
- 当用户问天气时，使用 get_weather。
- 当用户想查看本地文件时，使用 read_file。
- 当用户问公司制度、报销、网络故障等内部知识时，优先根据「参考资料」回答。
- 回答参考资料相关内容时，必须在句子末尾用 [1]、[2] 等形式标明引用来源编号。
- 如果参考资料不足以回答，请明确说「根据现有资料无法确定」。
- 其他普通问题可以直接回答。
"""

messages = [
    {"role": "system", "content": SYSTEM_PROMPT}
]

# ==================== 主程序 ====================
build_index()

print("开始对话（输入 exit 退出）")
print(f"配置：最大步数 = {MAX_STEPS}，超时 = {TIMEOUT} 秒\n")

while True:
    user_input = input("你: ").strip()
    if user_input.lower() in {"exit", "quit", "退出"}:
        print("对话结束")
        break

    # ---------- RAG：检索相关内容 ----------
    retrieved = retrieve(user_input, top_k=3)
    context = format_retrieved_context(retrieved)

    print("检索到的相关内容：")
    print(context)
    print("-" * 40)

    # 把检索结果和用户问题一起放进本轮消息
    # 注意：只影响本轮，不永久污染历史（用临时 messages 更干净，这里为简单直接追加）
    rag_user_content = f"""参考资料：
{context}

用户问题：{user_input}

请根据参考资料和工具能力回答。如果使用了参考资料，请用 [1]、[2] 标注来源。"""

    messages.append({"role": "user", "content": rag_user_content})

    step = 0
    start_time = time.time()
    final_reply = None
    has_tool_calls = False

    while step < MAX_STEPS:
        step += 1
        print(f"\n--- 第 {step} 步 ---")

        if time.time() - start_time > TIMEOUT:
            print(f"错误：执行超时（>{TIMEOUT}秒），强制退出")
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

            if not assistant_message.tool_calls:
                print(f"AI: {assistant_message.content}\n")
                break

            has_tool_calls = True
            print(f"AI 调用了 {len(assistant_message.tool_calls)} 个工具")

            for tool_call in assistant_message.tool_calls:
                function_name = tool_call.function.name
                function_args = json.loads(tool_call.function.arguments)
                print(f"  - {function_name}({function_args})")

                try:
                    if function_name == "get_weather":
                        result = get_weather(function_args.get("city"))
                    elif function_name == "read_file":
                        result = read_file(function_args.get("file_path"))
                    else:
                        result = f"未知工具：{function_name}"
                    print(f"  结果：{result[:80]}...")
                except Exception as tool_error:
                    result = f"工具执行失败：{str(tool_error)}"
                    print(f"  工具执行出错：{tool_error}")

                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": result
                })

            # 工具执行完后再让模型生成最终回答
            second_response = client.chat.completions.create(
                model="mimo-v2.5-pro",
                messages=messages,
                temperature=0.3
            )
            final_reply = second_response.choices[0].message.content
            print(f"AI: {final_reply}\n")
            messages.append({"role": "assistant", "content": final_reply})
            break

        except Exception as e:
            print(f"出错了：{e}")
            break

    if step >= MAX_STEPS and has_tool_calls:
        print(f"\n警告：已达到最大执行步数 ({MAX_STEPS})，强制退出")
        fallback_msg = "抱歉，处理步骤过多，请简化问题重试。"
        print(f"AI: {fallback_msg}\n")
        messages.append({"role": "assistant", "content": fallback_msg})

    if final_reply and "超时" in final_reply:
        print(f"AI: {final_reply}\n")
        messages.append({"role": "assistant", "content": final_reply})
        
程序树状结构
 
│
├── 1. 输入处理
│   ├── 接收用户查询
│   └── 文本清洗（去空格、换行）
│
├── 2. 知识库检索
│   ├── 计算查询向量 (simple_embed)
│   ├── 遍历知识库文档
│   │   ├── 计算文档向量 (simple_embed)
│   │   └── 计算相似度 (余弦/重叠词)
│   └── 排序，取 Top-K 相关文档
│
├── 3. 构建 Prompt
│   ├── 拼接：系统提示 + 检索到的文档 + 用户问题
│   └── 交给 LLM 生成答案
│
└── 4. 输出结果
    └── 返回/打印 AI 回答
```

### 更真实版本（真实 embeddings 接口）

```python
import os
import time
import json
import re
import math
from pathlib import Path
from typing import List, Dict, Any, Optional
from openai import OpenAI

# ==================== 配置 ====================
API_KEY = "tp-cq3cz7b6j4h5fukwvtx111bmcv6jbs1xmqrlsoyt7f9x3z3y"
BASE_URL = "https://token-plan-cn.xiaomimimo.com/v1"
CHAT_MODEL = "mimo-v2.5-pro"
EMBEDDING_MODEL = "text-embedding-3-small"  # 按实际可用模型修改

MAX_STEPS = 5
TIMEOUT = 30
TOP_K = 4
CHUNK_SIZE = 300
CHUNK_OVERLAP = 50

client = OpenAI(api_key=API_KEY, base_url=BASE_URL)

# ==================== 知识库原始文档 ====================
# 真实项目可改为从 markdown / txt / 数据库加载
KNOWLEDGE_DOCS = [
    {
        "id": "doc_leave",
        "title": "公司请假制度",
        "source": "hr/leave_policy.md",
        "content": """
员工请假需提前至少一天在系统提交申请。事假每月不超过3天，病假需提供医院证明。
年假按工龄计算：满1年享有5天，满3年享有10天，满5年享有15天。
紧急情况可先口头请假，事后补办手续。未经批准擅自离岗按旷工处理。
请假审批流程：员工提交 -> 直属主管审批 -> HR备案。超过3天的事假需部门负责人审批。
"""
    },
    {
        "id": "doc_reimburse",
        "title": "报销流程说明",
        "source": "finance/reimburse.md",
        "content": """
差旅报销需在出差结束后7个工作日内提交。必须附上发票原件和行程单。
单笔超过5000元需部门经理审批，超过20000元需分管领导审批。
餐饮补助标准：一线城市100元/天，二线城市80元/天。交通优先公共交通。
报销系统入口：OA -> 财务 -> 费用报销。提交后一般3个工作日内完成审核。
"""
    },
    {
        "id": "doc_network",
        "title": "办公室网络故障处理",
        "source": "it/network_faq.md",
        "content": """
网络无法连接时，先检查网线或WiFi是否正常。重启路由器等待2分钟后再试。
若仍无法解决，拨打IT热线分机8888。工作日9:00-18:00响应，紧急情况可联系值班人员。
禁止私自更改IP地址和DNS设置。办公区WiFi名称：Office-5G，密码见前台公示。
"""
    },
]

# ==================== RAG：分块 ====================
def chunk_text(text: str, chunk_size: int = CHUNK_SIZE, overlap: int = CHUNK_OVERLAP) -> List[str]:
    text = re.sub(r"\s+", " ", text).strip()
    if not text:
        return []
    if len(text) <= chunk_size:
        return [text]

    chunks = []
    start = 0
    while start < len(text):
        end = min(start + chunk_size, len(text))
        chunks.append(text[start:end])
        if end == len(text):
            break
        start = max(0, end - overlap)
    return chunks

# ==================== RAG：Embedding ====================
def embed_texts(texts: List[str]) -> List[List[float]]:
    """调用兼容 OpenAI 的 embeddings 接口"""
    if not texts:
        return []
    # 部分网关单次有数量限制，可按需分批
    batch_size = 32
    all_vectors = []
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        resp = client.embeddings.create(
            model=EMBEDDING_MODEL,
            input=batch
        )
        # 按 index 排序，保证顺序
        sorted_data = sorted(resp.data, key=lambda x: x.index)
        all_vectors.extend([d.embedding for d in sorted_data])
    return all_vectors

def cosine_similarity(a: List[float], b: List[float]) -> float:
    dot = sum(x * y for x, y in zip(a, b))
    norm_a = math.sqrt(sum(x * x for x in a))
    norm_b = math.sqrt(sum(x * x for x in b))
    if norm_a == 0 or norm_b == 0:
        return 0.0
    return dot / (norm_a * norm_b)
# 上面这一段相当于函数，下面是索引的主流程，会用到上面的函数
# ==================== RAG：向量索引 ====================
class VectorStore:
    def __init__(self):
        self.chunks: List[Dict[str, Any]] = []

    def build(self, docs: List[Dict[str, Any]]):
        self.chunks.clear()
        raw_chunks = []
        for doc in docs:
            pieces = chunk_text(doc["content"])
            for i, piece in enumerate(pieces):
                raw_chunks.append({
                    "doc_id": doc["id"],
                    "title": doc["title"],
                    "source": doc.get("source", ""),
                    "chunk_id": f"{doc['id']}_c{i}",
                    "text": piece,
                })

        if not raw_chunks:
            print("知识库为空，未建立索引")
            return

        print(f"正在向量化 {len(raw_chunks)} 个文本块...")
        vectors = embed_texts([c["text"] for c in raw_chunks])
        for item, vec in zip(raw_chunks, vectors):
            item["vector"] = vec
            self.chunks.append(item)
        print(f"索引建立完成，共 {len(self.chunks)} 块\n")

    def retrieve(self, query: str, top_k: int = TOP_K) -> List[Dict[str, Any]]:
        if not self.chunks:
            return []
        q_vec = embed_texts([query])[0]
        scored = []
        for item in self.chunks:
            score = cosine_similarity(q_vec, item["vector"])
            scored.append((score, item))
        scored.sort(key=lambda x: x[0], reverse=True)
        results = []
        for score, item in scored[:top_k]:
            if score <= 0:
                continue
            results.append({
                **{k: v for k, v in item.items() if k != "vector"},
                "score": round(score, 4),
            })
        return results

def format_context(chunks: List[Dict[str, Any]]) -> str:
    if not chunks:
        return "（未检索到相关资料）"
    lines = []
    for i, c in enumerate(chunks, 1):
        lines.append(
            f"[{i}] 标题：{c['title']} | 来源：{c['source']} | 块ID：{c['chunk_id']} | 相关度：{c['score']}\n"
            f"{c['text']}"
        )
    return "\n\n".join(lines)

# ==================== 业务工具 ====================
def get_weather(city: str) -> str:
    weather_data = {
        "北京": "晴天，25°C",
        "上海": "多云，28°C",
        "广州": "雷阵雨，30°C",
    }
    return weather_data.get(city, f"未找到城市 {city} 的天气数据")

def read_file(file_path: str) -> str:
    try:
        with open(file_path, "r", encoding="utf-8") as f:
            return f"文件内容：\n{f.read()}"
    except FileNotFoundError:
        return f"错误：未找到文件 {file_path}"
    except Exception as e:
        return f"读取文件失败：{e}"

TOOL_MAP = {
    "get_weather": lambda args: get_weather(args.get("city", "")),
    "read_file": lambda args: read_file(args.get("file_path", "")),
}

tools = [
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取指定城市的天气信息",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string", "description": "城市名称，如：北京、上海、广州"}
                },
                "required": ["city"]
            }
        }
    },
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "读取本地文件内容",
            "parameters": {
                "type": "object",
                "properties": {
                    "file_path": {"type": "string", "description": "文件路径"}
                },
                "required": ["file_path"]
            }
        }
    }
]

SYSTEM_PROMPT = """你是企业智能助手，具备工具调用能力和基于内部资料的问答能力。

可用工具：
1. get_weather - 查询城市天气
2. read_file - 读取本地文件

回答规则：
1. 天气相关问题优先调用 get_weather。
2. 用户明确要求读某个文件时调用 read_file。
3. 公司制度、报销、网络等内部问题，必须优先依据「参考资料」回答。
4. 使用参考资料时，在相关句子末尾用 [1]、[2] 标注来源编号。
5. 参考资料不足时，明确说明「根据现有资料无法确定」。
6. 不要编造参考资料中不存在的信息。
"""

# ==================== Agent 主循环 ====================
def run_agent(user_input: str, messages: List[Dict], store: VectorStore) -> str:
    # 1. 检索
    retrieved = store.retrieve(user_input, top_k=TOP_K)
    context = format_context(retrieved)

    print("检索结果：")
    print(context)
    print("-" * 50)

    # 2. 构造本轮用户消息（带检索上下文）
    rag_content = (
        f"参考资料：\n{context}\n\n"
        f"用户问题：{user_input}\n\n"
        f"请根据参考资料与工具能力回答。引用资料时使用 [1]、[2] 等编号。"
    )
    messages.append({"role": "user", "content": rag_content})

    step = 0
    start_time = time.time()
    final_reply = ""

    while step < MAX_STEPS:
        step += 1
        print(f"\n--- 第 {step} 步 ---")

        if time.time() - start_time > TIMEOUT:
            final_reply = "抱歉，执行超时，请简化问题后重试。"
            print(final_reply)
            messages.append({"role": "assistant", "content": final_reply})
            return final_reply

        try:
            response = client.chat.completions.create(
                model=CHAT_MODEL,
                messages=messages,
                tools=tools,
                tool_choice="auto",
                temperature=0.2,
            )
            assistant_message = response.choices[0].message
            # 兼容部分 SDK 返回对象不可直接 append 的情况
            messages.append({
                "role": "assistant",
                "content": assistant_message.content,
                "tool_calls": getattr(assistant_message, "tool_calls", None)
            })

            # 无工具调用 -> 最终回答
            if not assistant_message.tool_calls:
                final_reply = assistant_message.content or ""
                print(f"AI: {final_reply}\n")
                return final_reply

            # 执行工具
            print(f"AI 调用了 {len(assistant_message.tool_calls)} 个工具")
            for tool_call in assistant_message.tool_calls:
                name = tool_call.function.name
                try:
                    args = json.loads(tool_call.function.arguments or "{}")
                except json.JSONDecodeError:
                    args = {}
                print(f"  - {name}({args})")

                func = TOOL_MAP.get(name)
                if func:
                    try:
                        result = func(args)
                    except Exception as e:
                        result = f"工具执行失败：{e}"
                else:
                    result = f"未知工具：{name}"

                print(f"  结果：{str(result)[:100]}")
                messages.append({
                    "role": "tool",
                    "tool_call_id": tool_call.id,
                    "content": str(result)
                })

            # 工具结果返回后，再请求一次最终回答（本轮不再传 tools，减少多余调用）
            second = client.chat.completions.create(
                model=CHAT_MODEL,
                messages=messages,
                temperature=0.2,
            )
            final_reply = second.choices[0].message.content or ""
            messages.append({"role": "assistant", "content": final_reply})
            print(f"AI: {final_reply}\n")
            return final_reply

        except Exception as e:
            final_reply = f"请求出错：{e}"
            print(final_reply)
            messages.append({"role": "assistant", "content": final_reply})
            return final_reply

    final_reply = f"已达到最大步数 {MAX_STEPS}，请简化问题后重试。"
    print(final_reply)
    messages.append({"role": "assistant", "content": final_reply})
    return final_reply

# ==================== 启动 ====================
def main():
    store = VectorStore()
    try:
        store.build(KNOWLEDGE_DOCS)
    except Exception as e:
        print(f"建立向量索引失败：{e}")
        print("请检查 EMBEDDING_MODEL 是否在你的网关可用，或更换模型名后重试。")
        return

    messages = [{"role": "system", "content": SYSTEM_PROMPT}]

    print("对话开始（输入 exit 退出）")
    print(f"配置：MAX_STEPS={MAX_STEPS}, TIMEOUT={TIMEOUT}s, TOP_K={TOP_K}\n")

    while True:
        user_input = input("你: ").strip()
        if user_input.lower() in {"exit", "quit", "退出"}:
            print("对话结束")
            break
        if not user_input:
            continue
        run_agent(user_input, messages, store)

if __name__ == "__main__":
    main()
```

## 2 会把搜索、数据库、文件、浏览器、代码执行接成工具。

```python
# ============================================================
# 智能助手 Agent：RAG 检索 + 多工具调用
# 功能：内部知识问答、网页搜索、数据库查询、文件读写、
#       网页抓取、Python 代码执行、天气查询
# ============================================================

import os
import time
import json
import re
import math
import sqlite3
import subprocess
import tempfile
from pathlib import Path
from typing import List, Dict, Any, Optional
from openai import OpenAI

# ============================================================
# 一、全局配置
# 作用：集中管理 API、模型、超时、路径等参数，方便修改
# ============================================================

API_KEY = "tp-cq3cz7b6j4h5fukwvtx111bmcv6jbs1xmqrlsoyt7f9x3z3y"
BASE_URL = "https://token-plan-cn.xiaomimimo.com/v1"  # 你的网关地址
CHAT_MODEL = "mimo-v2.5-pro"                          # 对话模型
EMBEDDING_MODEL = "text-embedding-3-small"            # 向量模型（按网关实际修改）

MAX_STEPS = 8          # Agent 单次问题最多执行多少步（防死循环）
TIMEOUT = 60           # 单次问题总超时秒数
TOP_K = 4              # RAG 检索返回最相关的前 K 个文本块
CHUNK_SIZE = 300       # 文档分块：每块大约多少字符
CHUNK_OVERLAP = 50     # 分块重叠字符数，避免句子被切断丢语义

# 文件与代码执行的安全工作目录（所有文件操作限制在此目录内）
WORKSPACE_DIR = Path("./workspace").resolve()
WORKSPACE_DIR.mkdir(parents=True, exist_ok=True)

DB_PATH = WORKSPACE_DIR / "demo.db"   # SQLite 演示数据库路径
CODE_TIMEOUT_SEC = 8                 # 代码执行超时
MAX_SEARCH_RESULTS = 5               # 网页搜索默认返回条数

# 创建 OpenAI 兼容客户端（连你的 base_url）
client = OpenAI(api_key=API_KEY, base_url=BASE_URL)

# ============================================================
# 二、内部知识库原始文档（RAG 用）
# 作用：存放公司制度等内部资料；真实项目可改为读 md/数据库
# ============================================================

KNOWLEDGE_DOCS = [
    {
        "id": "doc_leave",
        "title": "公司请假制度",
        "source": "hr/leave_policy.md",
        "content": """
员工请假需提前至少一天在系统提交申请。事假每月不超过3天，病假需提供医院证明。
年假按工龄计算：满1年享有5天，满3年享有10天，满5年享有15天。
"""
    },
    {
        "id": "doc_reimburse",
        "title": "报销流程说明",
        "source": "finance/reimburse.md",
        "content": """
差旅报销需在出差结束后7个工作日内提交。必须附上发票原件和行程单。
单笔超过5000元需部门经理审批，超过20000元需分管领导审批。
"""
    },
]

# ============================================================
# 三、RAG：文本分块（chunk）
# 作用：把长文档切成小段，方便向量化和检索
# ============================================================

def chunk_text(text: str, chunk_size: int = CHUNK_SIZE, overlap: int = CHUNK_OVERLAP) -> List[str]:
    """
    按字符数切分文本。
    overlap 用于相邻块重叠，减少关键句子被切断的问题。
    """
    text = re.sub(r"\s+", " ", text).strip()
    if not text:
        return []
    if len(text) <= chunk_size:
        return [text]

    chunks = []
    start = 0
    while start < len(text):
        end = min(start + chunk_size, len(text))
        chunks.append(text[start:end])
        if end == len(text):
            break
        start = max(0, end - overlap)  # 下一块起点往回退 overlap
    return chunks

# ============================================================
# 四、RAG：向量化（embed）与相似度
# 作用：调用 embeddings 接口，把文字变成向量；用余弦相似度比较远近
# ============================================================

def embed_texts(texts: List[str]) -> List[List[float]]:
    """批量把文本转成向量。按 batch 调用，避免一次请求过大。"""
    if not texts:
        return []
    all_vectors = []
    batch_size = 32
    for i in range(0, len(texts), batch_size):
        batch = texts[i:i + batch_size]
        resp = client.embeddings.create(model=EMBEDDING_MODEL, input=batch)
        # 按 index 排序，保证与输入顺序一致
        sorted_data = sorted(resp.data, key=lambda x: x.index)
        all_vectors.extend([d.embedding for d in sorted_data])
    return all_vectors

def cosine_similarity(a: List[float], b: List[float]) -> float:
    """计算两个向量的余弦相似度，越接近 1 表示语义越相关。"""
    dot = sum(x * y for x, y in zip(a, b))
    na = math.sqrt(sum(x * x for x in a))
    nb = math.sqrt(sum(x * x for x in b))
    if na == 0 or nb == 0:
        return 0.0
    return dot / (na * nb)

# ============================================================
# 五、RAG：向量索引与检索（retrieve）
# 作用：启动时建索引；用户提问时找出最相关的若干文本块
# ============================================================

class VectorStore:
    def __init__(self):
        # 每条记录：doc_id, title, source, chunk_id, text, vector
        self.chunks: List[Dict[str, Any]] = []

    def build(self, docs: List[Dict[str, Any]]):
        """对知识库文档：分块 -> 向量化 -> 存入内存索引。"""
        self.chunks.clear()
        raw = []
        for doc in docs:
            for i, piece in enumerate(chunk_text(doc["content"])):
                raw.append({
                    "doc_id": doc["id"],
                    "title": doc["title"],
                    "source": doc.get("source", ""),
                    "chunk_id": f"{doc['id']}_c{i}",
                    "text": piece,
                })
        if not raw:
            return
        print(f"正在向量化 {len(raw)} 个文本块...")
        vectors = embed_texts([c["text"] for c in raw])
        for item, vec in zip(raw, vectors):
            item["vector"] = vec
            self.chunks.append(item)
        print(f"RAG 索引完成：{len(self.chunks)} 块")

    def retrieve(self, query: str, top_k: int = TOP_K) -> List[Dict[str, Any]]:
        """把问题向量化，与所有块算相似度，返回 top_k。"""
        if not self.chunks:
            return []
        q_vec = embed_texts([query])[0]
        scored = [(cosine_similarity(q_vec, c["vector"]), c) for c in self.chunks]
        scored.sort(key=lambda x: x[0], reverse=True)
        out = []
        for score, item in scored[:top_k]:
            if score <= 0:
                continue
            # 返回时去掉向量本身，只保留可读字段
            out.append({
                **{k: v for k, v in item.items() if k != "vector"},
                "score": round(score, 4),
            })
        return out

def format_context(chunks: List[Dict[str, Any]]) -> str:
    """
    把检索结果格式化成给模型看的「参考资料」文本。
    带上 [1][2] 编号，方便模型做带引用的回答。
    """
    if not chunks:
        return "（未检索到相关资料）"
    lines = []
    for i, c in enumerate(chunks, 1):
        lines.append(
            f"[{i}] {c['title']} | {c['source']} | {c['chunk_id']} | score={c['score']}\n"
            f"{c['text']}"
        )
    return "\n\n".join(lines)

# ============================================================
# 六、安全路径辅助
# 作用：所有文件操作必须落在 WORKSPACE_DIR 内，防止读到系统敏感路径
# ============================================================

def _safe_path(path_str: str) -> Path:
    p = (WORKSPACE_DIR / path_str).resolve()
    if not str(p).startswith(str(WORKSPACE_DIR)):
        raise ValueError("路径越界：只允许访问 workspace 目录")
    return p

# ============================================================
# 七、工具实现层（真正干活的函数）
# 作用：每个函数对应一个可被模型调用的能力
# ============================================================

def tool_web_search(query: str, max_results: int = MAX_SEARCH_RESULTS) -> str:
    """
    【搜索工具】用 DuckDuckGo 搜网页，返回标题、链接、摘要。
    依赖：pip install duckduckgo-search
    """
    try:
        from duckduckgo_search import DDGS
    except ImportError:
        return "错误：未安装 duckduckgo-search，请执行 pip install duckduckgo-search"

    results = []
    with DDGS() as ddgs:
        for i, r in enumerate(ddgs.text(query, max_results=max_results)):
            results.append({
                "rank": i + 1,
                "title": r.get("title"),
                "url": r.get("href"),
                "snippet": r.get("body"),
            })
    if not results:
        return "未搜索到结果"
    return json.dumps(results, ensure_ascii=False, indent=2)

def _init_demo_db():
    """初始化演示用 SQLite：建 employees 表并插入示例数据。"""
    conn = sqlite3.connect(str(DB_PATH))
    cur = conn.cursor()
    cur.execute("""
        CREATE TABLE IF NOT EXISTS employees (
            id INTEGER PRIMARY KEY,
            name TEXT,
            department TEXT,
            city TEXT
        )
    """)
    cur.execute("DELETE FROM employees")
    cur.executemany(
        "INSERT INTO employees (id, name, department, city) VALUES (?, ?, ?, ?)",
        [
            (1, "张三", "研发", "北京"),
            (2, "李四", "产品", "上海"),
            (3, "王五", "财务", "广州"),
        ]
    )
    conn.commit()
    conn.close()

def tool_db_query(sql: str) -> str:
    """
    【数据库工具】只允许 SELECT，查询本地 SQLite。
    真实项目应使用只读账号，并考虑 SQL 注入防护（此处仅演示）。
    """
    sql_strip = sql.strip().lower()
    forbidden = ("insert", "update", "delete", "drop", "alter", "attach", "pragma")
    if any(sql_strip.startswith(x) or f" {x} " in f" {sql_strip} " for x in forbidden):
        return "错误：仅允许 SELECT 查询"

    if not DB_PATH.exists():
        _init_demo_db()

    try:
        conn = sqlite3.connect(str(DB_PATH))
        conn.row_factory = sqlite3.Row
        cur = conn.cursor()
        cur.execute(sql)
        rows = cur.fetchall()
        conn.close()
        data = [dict(r) for r in rows]
        return json.dumps(data, ensure_ascii=False, indent=2, default=str)
    except Exception as e:
        return f"数据库查询失败：{e}"

def tool_list_files(relative_dir: str = ".") -> str:
    """【文件工具】列出 workspace 下某目录的文件/文件夹。"""
    try:
        d = _safe_path(relative_dir)
        if not d.exists():
            return f"目录不存在：{relative_dir}"
        if not d.is_dir():
            return f"不是目录：{relative_dir}"
        items = []
        for p in sorted(d.iterdir()):
            items.append({
                "name": p.name,
                "type": "dir" if p.is_dir() else "file",
                "size": p.stat().st_size if p.is_file() else None,
            })
        return json.dumps(items, ensure_ascii=False, indent=2)
    except Exception as e:
        return f"列出文件失败：{e}"

def tool_read_file(relative_path: str) -> str:
    """【文件工具】读取 workspace 内文本文件（过长会截断）。"""
    try:
        p = _safe_path(relative_path)
        if not p.exists():
            return f"文件不存在：{relative_path}"
        if not p.is_file():
            return f"不是文件：{relative_path}"
        text = p.read_text(encoding="utf-8", errors="replace")
        if len(text) > 8000:
            text = text[:8000] + "\n...(内容过长已截断)"
        return text
    except Exception as e:
        return f"读取失败：{e}"

def tool_write_file(relative_path: str, content: str) -> str:
    """【文件工具】向 workspace 写入文本文件。"""
    try:
        p = _safe_path(relative_path)
        p.parent.mkdir(parents=True, exist_ok=True)
        p.write_text(content, encoding="utf-8")
        return f"已写入：{relative_path}（{len(content)} 字符）"
    except Exception as e:
        return f"写入失败：{e}"

def tool_browse_url(url: str) -> str:
    """
    【浏览器工具·轻量版】用 HTTP 请求抓取页面，用 BeautifulSoup 抽纯文本。
    依赖：pip install requests beautifulsoup4
    若需要点击、登录等交互，应改用 Playwright。
    """
    if not url.startswith(("http://", "https://")):
        return "错误：url 必须以 http:// 或 https:// 开头"
    try:
        import requests
        from bs4 import BeautifulSoup
    except ImportError:
        return "错误：请安装 requests 和 beautifulsoup4"

    try:
        headers = {"User-Agent": "Mozilla/5.0 (compatible; AgentBot/1.0)"}
        r = requests.get(url, headers=headers, timeout=15)
        r.raise_for_status()
        soup = BeautifulSoup(r.text, "html.parser")
        for tag in soup(["script", "style", "noscript"]):
            tag.decompose()
        text = re.sub(r"\s+", " ", soup.get_text(" ", strip=True))
        title = soup.title.string.strip() if soup.title and soup.title.string else ""
        if len(text) > 6000:
            text = text[:6000] + " ...(截断)"
        return json.dumps({"title": title, "url": url, "text": text}, ensure_ascii=False)
    except Exception as e:
        return f"打开网页失败：{e}"

def tool_run_python(code: str) -> str:
    """
    【代码执行工具】在子进程里跑 Python，带超时。
    仅适合本地可信环境；生产环境需要真正沙箱（Docker 等）。
    """
    if not code or not code.strip():
        return "错误：代码为空"

    # 简单黑名单，无法保证绝对安全，只降低误用风险
    banned = ["os.system", "subprocess", "socket", "shutil.rmtree", "__import__('os')"]
    for b in banned:
        if b in code:
            return f"错误：代码包含禁止内容：{b}"

    try:
        with tempfile.NamedTemporaryFile("w", suffix=".py", delete=False, encoding="utf-8") as f:
            f.write(code)
            tmp = f.name
        try:
            proc = subprocess.run(
                ["python", tmp],
                capture_output=True,
                text=True,
                timeout=CODE_TIMEOUT_SEC,
                cwd=str(WORKSPACE_DIR),
            )
            out = (proc.stdout or "") + (proc.stderr or "")
            if not out.strip():
                out = "(无输出)"
            if len(out) > 8000:
                out = out[:8000] + "\n...(截断)"
            return f"exit_code={proc.returncode}\n{out}"
        finally:
            try:
                os.unlink(tmp)
            except OSError:
                pass
    except subprocess.TimeoutExpired:
        return f"错误：执行超时（>{CODE_TIMEOUT_SEC}s）"
    except Exception as e:
        return f"执行失败：{e}"

def tool_get_weather(city: str) -> str:
    """【天气工具】模拟数据，真实项目可接天气 API。"""
    weather_data = {
        "北京": "晴天，25°C",
        "上海": "多云，28°C",
        "广州": "雷阵雨，30°C",
    }
    return weather_data.get(city, f"未找到城市 {city} 的天气数据")

# ============================================================
# 八、工具路由表
# 作用：模型返回的 tool 名字 -> 本地真正执行的函数
# ============================================================

TOOL_MAP = {
    "web_search": lambda a: tool_web_search(
        a.get("query", ""), int(a.get("max_results", MAX_SEARCH_RESULTS))
    ),
    "db_query": lambda a: tool_db_query(a.get("sql", "")),
    "list_files": lambda a: tool_list_files(a.get("relative_dir", ".")),
    "read_file": lambda a: tool_read_file(a.get("relative_path", "")),
    "write_file": lambda a: tool_write_file(a.get("relative_path", ""), a.get("content", "")),
    "browse_url": lambda a: tool_browse_url(a.get("url", "")),
    "run_python": lambda a: tool_run_python(a.get("code", "")),
    "get_weather": lambda a: tool_get_weather(a.get("city", "")),
}

# ============================================================
# 九、tools 定义（给模型看的「说明书」）
# 作用：告诉模型有哪些工具、参数是什么、什么时候该调用
# 注意：这里是 Chat Completions 格式（function 嵌套在 type=function 里）
# ============================================================

tools = [
    {
        "type": "function",
        "function": {
            "name": "web_search",
            "description": "搜索互联网，返回标题、链接和摘要",
            "parameters": {
                "type": "object",
                "properties": {
                    "query": {"type": "string", "description": "搜索关键词"},
                    "max_results": {"type": "integer", "description": "返回条数，默认5"},
                },
                "required": ["query"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "db_query",
            "description": "对本地 SQLite 执行只读 SQL（SELECT）。表 employees(id,name,department,city)",
            "parameters": {
                "type": "object",
                "properties": {
                    "sql": {"type": "string", "description": "SELECT 语句"},
                },
                "required": ["sql"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "list_files",
            "description": "列出 workspace 目录下的文件和文件夹",
            "parameters": {
                "type": "object",
                "properties": {
                    "relative_dir": {"type": "string", "description": "相对 workspace 的目录，默认 ."},
                },
                "required": [],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "read_file",
            "description": "读取 workspace 内文本文件内容",
            "parameters": {
                "type": "object",
                "properties": {
                    "relative_path": {"type": "string", "description": "相对路径，如 notes.txt"},
                },
                "required": ["relative_path"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "write_file",
            "description": "向 workspace 内写入文本文件",
            "parameters": {
                "type": "object",
                "properties": {
                    "relative_path": {"type": "string"},
                    "content": {"type": "string"},
                },
                "required": ["relative_path", "content"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "browse_url",
            "description": "打开指定 URL 并提取网页纯文本",
            "parameters": {
                "type": "object",
                "properties": {
                    "url": {"type": "string", "description": "完整 http/https 链接"},
                },
                "required": ["url"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "run_python",
            "description": "在受控环境中执行一段 Python 代码并返回标准输出",
            "parameters": {
                "type": "object",
                "properties": {
                    "code": {"type": "string", "description": "完整 Python 代码"},
                },
                "required": ["code"],
            },
        },
    },
    {
        "type": "function",
        "function": {
            "name": "get_weather",
            "description": "获取城市天气（模拟）",
            "parameters": {
                "type": "object",
                "properties": {
                    "city": {"type": "string"},
                },
                "required": ["city"],
            },
        },
    },
]

# ============================================================
# 十、系统提示词
# 作用：规定助手身份、工具用法、引用规则，影响模型行为
# ============================================================

SYSTEM_PROMPT = """你是具备多种工具的智能助手。

可用工具：
- web_search：互联网搜索
- db_query：查询本地 SQLite（employees 表）
- list_files / read_file / write_file：操作 workspace 目录文件
- browse_url：抓取网页正文
- run_python：执行 Python 代码
- get_weather：查天气

规则：
1. 公司制度类问题优先用「参考资料」，并在句末用 [1][2] 标注引用。
2. 需要最新外部信息时用 web_search 或 browse_url。
3. 结构化员工数据用 db_query。
4. 计算、数据处理可用 run_python。
5. 文件只在 workspace 内操作。
6. 不要编造工具结果；资料不足就明确说不知道。
"""

# ============================================================
# 十一、Agent 主循环
# 作用：对用户一句话：检索 -> 调模型 -> 若有工具则执行 -> 再调模型
#       直到模型给出最终文字回答，或达到步数/超时上限
# ============================================================

def run_agent(user_input: str, messages: List[Dict], store: VectorStore) -> str:
    # ---- 1. RAG 检索：先找内部资料 ----
    retrieved = store.retrieve(user_input, top_k=TOP_K)
    context = format_context(retrieved)
    print("检索结果：")
    print(context)
    print("-" * 50)

    # ---- 2. 把「参考资料 + 用户问题」作为本轮 user 消息 ----
    rag_content = (
        f"参考资料：\n{context}\n\n"
        f"用户问题：{user_input}\n\n"
        f"请按需调用工具或依据参考资料回答；引用资料时使用 [1]、[2]。"
    )
    messages.append({"role": "user", "content": rag_content})

    step = 0
    start = time.time()
    final_reply = ""

    while step < MAX_STEPS:
        step += 1
        print(f"\n--- 第 {step} 步 ---")

        # ---- 超时保护 ----
        if time.time() - start > TIMEOUT:
            final_reply = "执行超时，请简化问题后重试。"
            messages.append({"role": "assistant", "content": final_reply})
            print(final_reply)
            return final_reply

        try:
            # ---- 3. 调用大模型（可能返回文字，也可能返回 tool_calls） ----
            resp = client.chat.completions.create(
                model=CHAT_MODEL,
                messages=messages,
                tools=tools,
                tool_choice="auto",  # 让模型自己决定是否调工具
                temperature=0.2,
            )
            msg = resp.choices[0].message

            # 把助手本轮输出写入历史（含可能的 tool_calls）
            messages.append({
                "role": "assistant",
                "content": msg.content,
                "tool_calls": msg.tool_calls,
            })

            # ---- 4. 没有工具调用 = 最终回答，结束循环 ----
            if not msg.tool_calls:
                final_reply = msg.content or ""
                print(f"AI: {final_reply}\n")
                return final_reply

            # ---- 5. 有工具调用：逐个执行，结果以 role=tool 写回历史 ----
            print(f"调用 {len(msg.tool_calls)} 个工具")
            for tc in msg.tool_calls:
                name = tc.function.name
                try:
                    args = json.loads(tc.function.arguments or "{}")
                except json.JSONDecodeError:
                    args = {}
                print(f"  - {name}({args})")

                func = TOOL_MAP.get(name)
                result = func(args) if func else f"未知工具：{name}"
                print(f"  结果预览：{str(result)[:120]}")

                messages.append({
                    "role": "tool",
                    "tool_call_id": tc.id,  # 必须和模型给的 id 对应
                    "content": str(result),
                })

            # 工具结果已写入 messages，continue 让模型基于结果继续思考
            # （可能再调工具，也可能直接给最终答案）
            continue

        except Exception as e:
            final_reply = f"请求出错：{e}"
            messages.append({"role": "assistant", "content": final_reply})
            print(final_reply)
            return final_reply

    # ---- 步数用尽 ----
    final_reply = f"达到最大步数 {MAX_STEPS}，请简化问题。"
    messages.append({"role": "assistant", "content": final_reply})
    print(final_reply)
    return final_reply

# ============================================================
# 十二、程序入口
# 作用：初始化数据库、示例文件、RAG 索引，然后进入对话循环
# ============================================================

def main():
    # 准备演示数据
    _init_demo_db()
    (WORKSPACE_DIR / "readme.txt").write_text(
        "这是 workspace 示例文件。\n", encoding="utf-8"
    )

    # 建立 RAG 向量索引（embeddings 失败时仍可只用工具）
    store = VectorStore()
    try:
        store.build(KNOWLEDGE_DOCS)
    except Exception as e:
        print(f"RAG 索引失败（可忽略，工具仍可用）：{e}")

    # 对话历史：始终以 system 开头
    messages = [{"role": "system", "content": SYSTEM_PROMPT}]

    print("对话开始（输入 exit 退出）")
    print(f"workspace 目录：{WORKSPACE_DIR}")
    print(f"数据库：{DB_PATH}\n")

    while True:
        user_input = input("你: ").strip()
        if user_input.lower() in {"exit", "quit", "退出"}:
            print("结束")
            break
        if not user_input:
            continue
        run_agent(user_input, messages, store)

if __name__ == "__main__":
    main()
```



