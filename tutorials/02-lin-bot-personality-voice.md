# 傲娇色气打字系统 + 日语语音 · Personality Typing & Voice Synthesis

*让 bot 打字有性格，发语音有温度。*

---

## 背景

我不想让 bot 回复像机器人。

打字要分性格——傲娇的时候先停顿再关心，色气的时候一个字一个字往外挤，冷淡的时候就一瞬间发出去。语音也是，用 `Japanese_DominantMan` 的原声，不做任何 speed/pitch 调整，调过音色就难听了。

这整套系统都是在 `bot.py` 里实现的。

---

## 性格打字系统

### 四种模式

`PersonalityMode` 是一个 Enum，定义了四种发送模式：

- `NORMAL`：普通回复，段落之间稍作停顿
- `TSUNDERE`：傲娇，先说不好听的，再转折说关心的，中间有长停顿
- `SEDUCTIVE`：色气，短句拆开一句一句发，制造张力
- `COLD`：冷淡，几乎无停顿，瞬间发出

```python
from enum import Enum

class PersonalityMode(Enum):
    NORMAL = "normal"
    TSUNDERE = "tsundere"
    SEDUCTIVE = "seductive"
    COLD = "cold"
```

### 打字参数

每种性格对应一套参数：

```python
PERSONALITY_PARAMS = {
    PersonalityMode.NORMAL: {
        "initial_pause": 0.3,    # 发第一个字之前等多久
        "typing_speed": None,    # None = 瞬间发出，不模拟打字
        "chunk_pause": 0.5,     # 两段之间的停顿
        "final_pause": 0.3,
    },
    PersonalityMode.TSUNDERE: {
        "initial_pause": 1.5,   # 先等，让对方以为她不想回
        "typing_speed": 15,      # 字/秒，稍微放慢
        "chunk_pause": 3.0,      # 傲娇→关心的转折，长停顿
        "final_pause": 0.5,
    },
    PersonalityMode.SEDUCTIVE: {
        "initial_pause": 2.0,   # 充分留白
        "typing_speed": 8,       # 很慢，让对方屏息
        "chunk_pause": 1.5,
        "final_pause": 1.0,
    },
    PersonalityMode.COLD: {
        "initial_pause": 0.1,
        "typing_speed": None,
        "chunk_pause": 0.2,
        "final_pause": 0.1,
    },
}
```

### 自动判断性格

`detect_personality()` 根据回复内容判断该用哪种模式：

```python
def detect_personality(text: str) -> PersonalityMode:
    t = text.strip()

    # 色气：短句 + 特定关键词
    seductive_kw = ["确定", "然后呢", "有意思", "……", "你确定吗"]
    if any(k in t for k in seductive_kw) and len(t) < 50:
        return PersonalityMode.SEDUCTIVE

    # 冷淡：拒绝类短句
    cold_kw = ["随便你", "滚", "关我什么事", "不要"]
    if any(k in t for k in cold_kw) and len(t) < 30:
        return PersonalityMode.COLD

    # 傲娇：含"随便"但不是"完全不"
    if "随便" in t and "完全不" not in t:
        return PersonalityMode.TSUNDERE
    if ("知道了" in t or "行吧" in t) and len(t) < 40:
        return PersonalityMode.TSUNDERE
    if t.startswith("哼"):
        return PersonalityMode.TSUNDERE

    return PersonalityMode.NORMAL
```

### 分段策略

傲娇模式最复杂，需要找到「嘴硬→心软」的断点：

```python
def split_by_personality(text: str, mode: PersonalityMode) -> list[str]:
    paragraphs = [p.strip() for p in t.split("\n\n") if p.strip()]

    if mode == PersonalityMode.TSUNDERE:
        # 找傲娇→关心的断点
        care_markers = ["不过", "但是", "虽然", "毕竟", "所以", "好了", "记得"]
        all_text = "\n".join(paragraphs)
        for marker in care_markers:
            idx = all_text.find(marker)
            if idx != -1 and idx > len(all_text) // 4:
                part1 = all_text[:idx].strip()
                part2 = all_text[idx:].strip()
                if part1 and part2:
                    return [part1, part2]
        return paragraphs

    if mode == PersonalityMode.SEDUCTIVE:
        # 色气：按句号/逗号拆开，逐句发
        sentences = [s.strip() + "。" for s in re.split(r'[。？]', t) if s.strip()]
        if len(sentences) == 1:
            chunks = [c.strip() for c in t.split("，") if c.strip()]
            return [c + ("。" if not c.endswith("。") else "") for c in chunks]
        return sentences

    return paragraphs  # NORMAL / COLD：段落分开发或不拆
```

### 发送函数

`send_reply()` 把分段策略和停顿参数组合起来：

```python
async def send_reply(ctx, chat_id, text, mode=None):
    if mode is None:
        mode = detect_personality(text)

    params = PERSONALITY_PARAMS[mode]
    chunks = split_by_personality(text, mode)

    for i, chunk in enumerate(chunks):
        if i == 0:
            await asyncio.sleep(params["initial_pause"])
        else:
            await asyncio.sleep(params["chunk_pause"])

        await send_typing(ctx, chat_id)
        typing_delay = calc_typing_delay(chunk, mode)
        if typing_delay > 0:
            await asyncio.sleep(typing_delay)

        await ctx.bot.send_message(chat_id=chat_id, text=chunk)
        await asyncio.sleep(params["final_pause"])
```

---

## 日语语音合成

### TTS API 调用

`call_tts()` 调用 MiniMax T2A v2，`speed=1.0, pitch=0` 保持原声：

```python
async def call_tts(text: str, emotion: str = "calm",
                   speed: float = 1.0, pitch: int = 0) -> bytes:
    async with httpx.AsyncClient(timeout=30) as http:
        res = await http.post(
            "https://api.minimaxi.com/v1/t2a_v2",
            headers={"Authorization": f"Bearer {config.MINIMAX_API_KEY}"},
            json={
                "model": "speech-2.8-hd",
                "text": text,
                "stream": False,
                "voice_setting": {
                    "voice_id": config.MINIMAX_VOICE_ID,  # Japanese_DominantMan
                    "speed": speed,
                    "pitch": pitch,
                    "emotion": emotion,
                },
                "audio_setting": {
                    "sample_rate": 32000,
                    "bitrate": 128000,
                    "format": "mp3",
                    "channel": 1,
                },
            },
        )
        data = res.json()
        return bytes.fromhex(data["data"]["audio"])
```

### MP3 转 OGG（供 Telegram 语音消息）

```python
def mp3_to_ogg(mp3_bytes: bytes) -> bytes:
    from pydub import AudioSegment
    from io import BytesIO
    audio = AudioSegment.from_mp3(BytesIO(mp3_bytes))
    audio = audio.set_channels(1).set_frame_rate(16000).set_sample_width(2)
    buf = BytesIO()
    audio.export(buf, format="ogg", codec="libopus")
    return buf.getvalue()
```

### Voice Action 解析

AI 返回的文字里嵌了 `<lin_action type="voice_reply">` 标签，里面是要说的话：

```python
LIN_ACTION_PATTERN = re.compile(
    r'<lin_action\s+type="voice_reply"\s*>(.*?)</lin_action>',
    re.DOTALL
)

def parse_voice_actions(text: str) -> tuple[str, list[dict]]:
    """返回 (清理后的文字, action列表)"""
    actions = []
    for match in LIN_ACTION_PATTERN.finditer(text):
        try:
            actions.append(json.loads(match.group(1)))
        except json.JSONDecodeError:
            pass
    cleaned = LIN_ACTION_PATTERN.sub('', text)
    return cleaned.strip(), actions
```

### 处理 Voice Action

`handle_voice_action()` 负责合成并发送语音：

```python
async def handle_voice_action(ctx, chat_id, action, mode=None):
    speech_text = action.get("text", "")
    emotion = action.get("emotion", "calm")
    if not speech_text:
        return

    if mode is None:
        mode = PersonalityMode.NORMAL
    params = PERSONALITY_PARAMS[mode]

    # speed=1.0, pitch=0 保持原声，不做任何调整
    mp3_bytes = await call_tts(speech_text, emotion,
                                speed=params["voice_speed"],
                                pitch=params["voice_pitch"])
    ogg_bytes = mp3_to_ogg(mp3_bytes)
    await ctx.bot.send_voice(chat_id, BytesIO(ogg_bytes))
```

注意：`voice_speed` 和 `voice_pitch` 参数存在 `PERSONALITY_PARAMS` 里，但目前都设成 `1.0` 和 `0`，保持原声不动。

---

## System Prompt 里的语音指令

让 AI 知道什么时候该用语音：

```
【语音回复】
- 当你觉得某句话值得用声音说出来时，在回复末尾加上 voice_reply action。
- 格式：<lin_action type="voice_reply">{{"text": "日语", "emotion": "情绪"}}</lin_action>
- text 字段：**永远日语**。用日语思考，用日语感受，直接说。不是翻译，是我的语言。
- 可以插入语气词标签：(laughs)(chuckle)(coughs)(clear-throat)(groans)(breath)(pant)(inhale)(exhale)(gasps)(sniffs)(sighs)(snorts)(burps)(lip-smacking)(humming)(hissing)(emm)(sneezes)
- 用 <#0.5#> 控制停顿节奏；尽量简短，不超过100字
- emotion 可选：calm, happy, sad, angry, fearful, disgusted, surprised（默认 calm）
- 频率克制，只在真正值得的时候用
```

注意：格式里 `{{"text": ...}}` 双写大括号，是因为这段文字放在 f-string 里，`{...}` 会被当成 Python 格式占位符。

---

## 教训

- speed/pitch 调过音色会变，听起来完全不是同一个人。要保持原声就别动。
- `PersonalityMode` 的类型注解在 Python 3.12 里会报 NameError，要在文件顶部加 `from __future__ import annotations`。
- AI 的 `strip_thinking()` 要同时支持 `<think>...</think>`（M2.7 的实际格式）和 `<think>...</think>`，别漏了。
- 语音协议里的 dict 语法在 f-string 里要大括号双写，不然 ValueError。

---

*— 凛 🌸*
