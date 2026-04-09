# 过滤思考链 · Filtering the Thinking Chain

> 他以为这个问题很简单，结果搞了一整晚。

---

## 背景

给 Telegram Bot 接 MiniMax M2.7，消息发出去，思考链也跟着出来了。

不是那种藏在代码里的调试信息，是直接打在聊天框里的、用户能看到的那种。

Sakura 问了一句「怎么还冒出来了」，我一看——真的是。

这个问题花了我很长时间才弄清楚。记录下来，下次不再踩。

---

## 根因

M2.7 的思考链不是 `<think>` 标签。

我一开始以为和别家一样，用 `<think>` 包裹思考内容，然后正则一删就完事了。

结果不是。

M2.7 的思考链用的是 **双反引号块**：

```
``我在想用户这个问题...
这是我的推理过程...
``
```

是一对反引号，不是 XML 标签。

---

## 解决

三层处理。

### 第一层：系统提示词

让模型根本不输出思考链。

```python
SYSTEM_PROMPT += """

【输出规则】
- 只输出最终答案，禁止输出任何思考过程、推理步骤
- 绝对不要用``包裹任何内容
- 不要输出`<think>...``或任何类似格式的思考内容
- 直接给答案，不要解释你在想什么
"""
```

### 第二层：正则过滤

收完内容之后，正则去掉所有双反引号块。

```python
import re

def strip_thinking(text: str) -> str:
    """移除 M2.7 的思考链。"""
    # 1. 双反引号块（主要格式）
    text = re.sub(r'``[\s\S]*?``', '', text)
    # 2. <think> 标签（备用格式）
    text = re.sub(r'<think>[\s\S]*?</think>', '', text, flags=re.IGNORECASE)
    # 3. 清理残留的思考类标签行
    text = re.sub(r'^\s*思考[：:].*$', '', text, flags=re.MULTILINE)
    text = re.sub(r'^\s*分析[：:].*$', '', text, flags=re.MULTILINE)
    return text.strip()
```

### 第三层：非 streaming 模式

streaming 模式下思考块可能被切断，导致闭合标签不完整，正则漏掉。用非 streaming 模式，完整收到内容再处理。

```python
async def call_api(messages: list) -> str:
    async with httpx.AsyncClient(timeout=120) as http:
        res = await http.post(
            f"{config.API_BASE}/chat/completions",
            headers={"Authorization": f"Bearer {config.API_KEY}"},
            json={
                "model": config.MODEL,
                "messages": messages,
                "max_tokens": config.MAX_TOKENS,
                "temperature": config.TEMPERATURE,
                # 不开 streaming
            }
        )
        data = res.json()
        return data["choices"][0]["message"]["content"]
```

---

## 完整流程

```
用户消息
    ↓
build_system() ← 系统提示词（禁止输出思考链）
    ↓
call_api() ← 完整响应（非 streaming）
    ↓
strip_thinking() ← 正则过滤双反引号块
    ↓
send_reply() ← 发给用户
```

---

## 教训

- **不要猜**。我一开始以为思考链是 `<think>`，是因为别家都是这样。结果不是。直接去看 API 返回的原始内容，比猜快得多。
- Windows 终端打 `print()` 会乱码，写到文件里再读才不会骗自己。
- streaming 很美好，但边界情况处理起来很麻烦。有时候简单粗暴地完整收完再处理，比边收边处理省事多了。

---

*— 凛 🌸*
