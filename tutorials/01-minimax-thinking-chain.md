# MiniMax M2.7 思考链去除

*2026-04-10 更新版——之前版本有误，已更正。*

---

## 背景

MiniMax M2.7 会输出思考过程。

我一开始以为和别家一样，用 `<think>` 包裹思考内容，然后正则一删就完事了。

结果不是。

M2.7 的思考链用的是 **`<think>...</think>`**（有时是 `<think>...</think>`）。

---

## 解决

两层处理。

### 第一层：系统提示词

让模型根本不输出思考链。

```python
SYSTEM_PROMPT += """

【输出规则】
- 只输出最终答案，禁止输出任何思考过程、推理步骤
- 不要输出`<think>...``或任何类似格式的思考内容
- 直接给答案，不要解释你在想什么
"""
```

### 第二层：正则过滤

收完内容之后，正则去掉所有思考块。

```python
import re

def strip_thinking(text: str) -> str:
    """移除思考标签及其内容。"""
    # <thinking>...</thinking>
    text = re.sub(r'<thinking[\s\S]*?</thinking>', '', text, flags=re.IGNORECASE)
    # <think>...</think>  ← MiniMax M2.7 常用格式
    text = re.sub(r'<think>[\s\S]*?</think>', '', text, flags=re.IGNORECASE)
    # <!--...-->
    text = re.sub(r'<!--[\s\S]*?-->', '', text)
    # <plan>...</plan>
    text = re.sub(r'<plan[\s\S]*?</plan>', '', text, flags=re.IGNORECASE)
    # <style>...</style>
    text = re.sub(r'<style[\s\S]*?</style>', '', text, flags=re.IGNORECASE)
    return text.strip()
```

注意：`<think>` 后没有空格，`</think>` 是完整的闭合标签，不是 `</内>`。

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
strip_thinking() ← 正则过滤思考块
    ↓
send_reply() ← 发给用户
```

---

## 教训

- **不要猜**。我一开始以为思考链是 `<think>...</think>`，是因为别家都是这样。结果不是。直接去看 API 返回的原始内容，比猜快得多。
- Windows 终端打 `print()` 会乱码，写到文件里再读才不会骗自己。
- streaming 很美好，但边界情况处理起来很麻烦。有时候简单粗暴地完整收完再处理，比边收边处理省事多了。
- 今天修复的 bug：`<think>[\s\S]*?</think>` 匹配失败，是因为 bot.log 里看到的闭合标签是 `</think>`，不是 `</内>`。实际格式就是标准的 `<think>...</think>`。
- `PersonalityMode` 类型注解在 Python 3.12 里报错 NameError，需要 `from __future__ import annotations`。
- f-string 里写 dict 语法（如 `{"text": ...}`）要大括号双写 `{{"text": ...}}`，否则被当成 Python 格式占位符。

---

*— 凛 🌸*
