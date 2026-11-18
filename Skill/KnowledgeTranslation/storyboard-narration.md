---
name: storyboard-narration
description: |
  根據 storyboard-design 輸出的旁白文稿，優先以 Gemini TTS（gemini-3.8-flash-lite-tts）
  生成具表演力的繁體中文旁白音檔（自動依文稿語氣選用合適女聲、自動標註 style／pacing／
  pauses／emphasis／vocal bursts），若 Gemini TTS 無法使用則自動改用 Edge-TTS 備援；
  將旁白合成進對應的 storyboard-video 分鏡影片（片頭0.3秒後開始），最後以 ffmpeg 串接
  所有已配音分鏡影片為完整影片，分鏡之間加入0.3秒轉場，輸出 yuv420p MP4。

  當使用者提到「旁白配音」、「加旁白」、「TTS」、「Gemini TTS」、「Edge-TTS」、「配音合成」、
  「串接影片」、「生成完整影片」、「把分鏡影片串起來」，或分鏡影片生成後要求加入旁白並輸出
  完整影片時，務必使用此技能。
---

# 分鏡旁白配音與串接技能（Storyboard Narration Skill）

## 任務概述

從對話中讀取 storyboard-design 的旁白文稿，優先以 **Gemini TTS**
（`gemini-3.8-flash-lite-tts`）生成繁體中文旁白音檔——根據文稿語氣自動選用合適女聲、
自動標註語氣（style）、節奏（pacing）、停頓（pauses）、重音（emphasis）與人聲音效
（vocal bursts / non-speech sounds）；若 Gemini TTS 無法使用（未安裝 SDK、未設定 API
Key、呼叫失敗等），自動改用 **Edge-TTS** 備援生成。音檔完成後合成進各分鏡影片
（storyboard_video_XX.mp4），再依序串接為完整影片，分鏡之間加入 0.3 秒轉場特效，
最終輸出 pixel format 為 yuv420p 的 MP4。

---

## 輸出目錄規則（所有 storyboard 技能共用）

本技能產生的**所有檔案**（`storyboard_narration.py`、`narration_XX.wav`/`.mp3`、
`storyboard_narrated_XX.mp4`、`storyboard_final.mp4`…）以及讀取的輸入分鏡影片
（`storyboard_video_XX.mp4`），都位於與 storyboard-design 相同的**共用資料夾**內。

- 基底名稱為 `storyboard`；若已存在則以連號尾數區分：`storyboard001`、`storyboard002`、…
- 本技能屬於**下游技能**，沿用**最新**的既有資料夾（`create_new=False`）。
- 作法：將 `storyboard_narration.py` 寫入該資料夾，腳本開頭 `os.chdir()`
  至自身所在目錄，之後所有相對路徑（音檔、影片）都會落在該資料夾內。

```python
import re
from pathlib import Path

def resolve_storyboard_dir(create_new=False):
    base = Path("storyboard")
    existing = []
    if base.is_dir():
        existing.append((0, base))
    for p in sorted(Path(".").glob("storyboard[0-9][0-9][0-9]")):
        if p.is_dir():
            existing.append((int(p.name[len("storyboard"):]), p))
    existing.sort(key=lambda t: t[0])
    if create_new:
        target = base if not existing else Path(f"storyboard{existing[-1][0] + 1:03d}")
        target.mkdir(parents=True, exist_ok=True)
        return target
    if existing:
        return existing[-1][1]
    base.mkdir(parents=True, exist_ok=True)
    return base
```

---

## 語音對照表（Gemini TTS 女聲）

Gemini TTS 不會自行判斷語氣選聲，**由本技能依旁白文稿的語氣與用途，從下表挑選最合適的
女聲**，填入 Step 2 的 `voice` 欄位：

| 語音 | 音高 | 特色 | 建議用途 |
|---|---|---|---|
| Achernar | 高 | 柔和、平靜 | 安靜或個人敘事 |
| Aoede | 中 | 輕鬆、隨性 | 生活、旅遊 vlog |
| Autonoe | 中 | 明亮、熱情 | 促銷、產品功能更新 |
| Callirrhoe | 中 | 隨和、休閒 | 友善導覽、休閒 vlog |
| Despina | 中 | 圓潤、流暢 | 行銷廣告、專業 e-learning |
| Erinome | 中 | 清晰、咬字精準 | 教學、操作說明 |
| Gacrux | 中 | 成熟、權威 | 正式公告、嚴肅簡報 |
| Kore | 中 | 堅定、直接 | 新聞播報、官方訊息 |
| Laomedeia | 高 | 活潑、有活力 | 銷售、社群行銷 |
| Leda | 高 | 年輕、有朝氣 | 流行文化、遊戲內容 |
| Sulafat | 中 | 溫暖、親切 | 新手引導、歡迎訊息 |
| Vindemiatrix | 中 | 溫和、平靜 | 兒童內容、冥想 |
| Zephyr | 高 | 明亮、正向 | 展示影片、產品發表 |

未特別符合上述情境時，預設使用 `Despina`（gemini-3.8-flash-lite-tts 預設人聲；圓潤、流暢，適合多數旁白）。

---

## 步驟流程

### Step 1：確認環境（Gemini TTS 優先，Edge-TTS 備援）

```python
import subprocess, sys, os, importlib, importlib.metadata

def ensure_package(pip_spec, import_name, min_version=None):
    """Install/upgrade pip_spec unless import_name is already present at
    >= min_version. A stale version that still imports fine (e.g. an old
    google-genai without the `interactions` annotation schema) must trigger
    an upgrade, not be silently accepted -- that's what produced the
    confusing 'UNKNOWN' annotation-type error during testing."""
    def ver_tuple(v):
        return tuple(int(x) for x in v.split(".")[:3] if x.isdigit())

    try:
        installed = importlib.metadata.version(import_name.split(".")[0])
    except importlib.metadata.PackageNotFoundError:
        installed = None

    needs_install = installed is None or (
        min_version and ver_tuple(installed) < ver_tuple(min_version)
    )
    if needs_install:
        subprocess.check_call([sys.executable, "-m", "pip", "install", "-U", pip_spec])
        importlib.invalidate_caches()
    return importlib.import_module(import_name)

TTS_ENGINE = "gemini"
GEMINI_MODEL = "gemini-3.8-flash-lite-tts"
gemini_client = None

try:
    ensure_package("google-genai>=2.25.0", "google.genai", min_version="2.25.0")
    from google import genai
    from google.genai import types
    if not os.environ.get("GEMINI_API_KEY"):
        raise RuntimeError("環境變數 GEMINI_API_KEY 未設定")
    # TTS 模型目前僅在 v1beta 提供；SDK 預設走 v1 會回傳 404，且此預設行為會被使用者
    # 環境中是否設有 GOOGLE_GENAI_API_VERSION 悄悄左右，故在程式碼中明確指定 api_version，
    # 不依賴環境變數。
    gemini_client = genai.Client(http_options=types.HttpOptions(api_version="v1beta"))
    # 以極短文字做一次 smoke test，確認模型可正常呼叫
    _test = gemini_client.interactions.create(
        model=GEMINI_MODEL,
        input=[{"type": "user_input", "content": [{
            "type": "text", "text": "測試",
            "annotations": [{"type": "speech_metadata", "style": "neutral, normal pace"}],
        }]}],
        response_format={"type": "audio"},
        generation_config={"speech_config": [{"voice": "Despina"}]},
    )
    if not getattr(_test, "output_audio", None):
        raise RuntimeError("Gemini TTS 回應中沒有音訊資料")
    print(f"[OK] Gemini TTS ready ({GEMINI_MODEL})")
except Exception as e:
    print(f"[INFO] Gemini TTS unavailable ({e}); falling back to Edge-TTS")
    TTS_ENGINE = "edge"

if TTS_ENGINE == "edge":
    try:
        import edge_tts
    except ImportError:
        subprocess.check_call([sys.executable, "-m", "pip", "install", "edge-tts"])
        import edge_tts

# 確認 ffmpeg
result = subprocess.run(["ffmpeg", "-version"], capture_output=True)
if result.returncode != 0:
    print("ERROR: ffmpeg not found. Please install ffmpeg and add it to PATH.")
    sys.exit(1)
print(f"[OK] Environment ready (TTS_ENGINE={TTS_ENGINE})")
```

**備援判斷時機**：僅在 Step 1 做一次整體判斷（smoke test），之後**全部分鏡統一使用同一
引擎**，不要中途切換，以避免不同分鏡音色不一致。若 smoke test 通過但正式生成途中仍失敗
（例如額度用盡），將 `TTS_ENGINE` 切為 `"edge"` 並對**所有**分鏡重新以 Edge-TTS 生成，
而不是只重試失敗的那一段。

### Step 2：解析旁白文稿、選擇語音與風格標註

從對話中讀取最近一份 storyboard-design 輸出，格式為 `## 分鏡 N｜標題`，
擷取每個分鏡的 **🎙️ 旁白** 欄位文字，並為每段旁白補上：

- `voice`：依「語音對照表」，依本段情境/語氣挑選最合適的女聲（找不到明確對應時用預設人聲 `Despina`）。
- `style`：一句英文描述，涵蓋整段的**語氣、語速（pacing）、音量**等持續性特質
  （例如 `"warm, friendly, slightly slow pace, mid volume"`）。瞬間的聲音事件不要寫在
  這裡，寫進 `gemini_text` 的行內標籤。
- `gemini_text`：在乾淨逐字稿基礎上加註，**只給 Gemini TTS 使用**：
  - 停頓（pauses）：在語意停頓處插入 `<short pause>` 或 `<long pause>`（小寫、空格，
    無底線）。
  - 重音（emphasis）：將需要強調的詞改成**全大寫**。
  - 人聲音效（vocal bursts / non-speech sounds）：視語境適度插入人類聲音標籤，例如
    `<sigh>`、`<laugh>`、`<breath>`、`<gasp>`、`<chuckle>`，避免使用非人聲音效標籤；
    中文文稿中的標籤本身仍用英文書寫。
  - `text`：維持**乾淨逐字稿**（不含任何標籤或大寫強調），作為記錄與 Edge-TTS 備援用。

```python
# 手動或程式化填入，保留繁體中文原文
narrations = [
    {
        "index": 1,
        "title": "...",
        "text": "旁白原文（乾淨逐字稿，供記錄與 Edge-TTS 備援）...",
        "gemini_text": "旁白原文，視語氣插入 <short pause>／<long pause>／<sigh> 等標籤，"
                        "並將需強調的詞改為全大寫...",
        "style": "warm, friendly, normal pace, mid volume",
        "voice": "Despina",
    },
    # ...
]
```

### Step 3：生成語音（Gemini TTS 主要／Edge-TTS 備援）

Gemini TTS 輸出為 WAV（24kHz、單聲道、16-bit PCM），存為 `.wav`；
Edge-TTS 輸出為 `.mp3`。依 `TTS_ENGINE` 決定副檔名與呼叫方式，兩者皆走同一份流程。

```python
import asyncio, base64

def audio_ext():
    return "wav" if TTS_ENGINE == "gemini" else "mp3"

def generate_tts_gemini(text, style, voice, output_path):
    interaction = gemini_client.interactions.create(
        model=GEMINI_MODEL,
        input=[{
            "type": "user_input",
            "content": [{
                "type": "text",
                "text": text,
                "annotations": [{"type": "speech_metadata", "style": style}],
            }],
        }],
        response_format={"type": "audio"},
        generation_config={"speech_config": [{"voice": voice}]},
    )
    audio_bytes = base64.b64decode(interaction.output_audio.data)
    with open(output_path, "wb") as f:
        f.write(audio_bytes)

async def generate_tts_edge(text, output_path, voice="zh-TW-HsiaoChenNeural"):
    communicate = edge_tts.Communicate(text, voice)
    await communicate.save(output_path)

for scene in narrations:
    audio_path = f"narration_{scene['index']:02d}.{audio_ext()}"
    try:
        if TTS_ENGINE == "gemini":
            generate_tts_gemini(scene["gemini_text"], scene["style"], scene["voice"], audio_path)
        else:
            asyncio.run(generate_tts_edge(scene["text"], audio_path))
        print(f"[OK] TTS generated -> {audio_path}")
    except Exception as e:
        if TTS_ENGINE == "gemini":
            print(f"[WARN] Gemini TTS failed ({e}); switching ALL scenes to Edge-TTS")
            TTS_ENGINE = "edge"
            # 重新以 Edge-TTS 生成本段及之前已生成的分鏡，確保整部影片音色一致
            for s in narrations:
                p = f"narration_{s['index']:02d}.{audio_ext()}"
                asyncio.run(generate_tts_edge(s["text"], p))
                print(f"  [OK] (fallback) {p}")
            break
        raise
```

### Step 4：取得音訊與影片時長

```python
import json

def get_duration(path):
    cmd = [
        "ffprobe", "-v", "quiet", "-print_format", "json",
        "-show_streams", path
    ]
    out = subprocess.check_output(cmd)
    data = json.loads(out)
    for stream in data["streams"]:
        if "duration" in stream:
            return float(stream["duration"])
    return 0.0
```

### Step 5：速度調整（旁白過長時）

每段旁白可用時間 = 分鏡影片時長 - 0.3 秒（片頭空白）。
若旁白音檔時長超過可用時間，提高播放速度使其剛好收尾。

atempo 最大值為 2.0，需要更快時串接多個 atempo filter：
- 例：2.5× = `atempo=2.0,atempo=1.25`

```python
def build_atempo_chain(speed):
    """Build atempo filter chain; each atempo max=2.0, min=0.5."""
    filters = []
    while speed > 2.0:
        filters.append("atempo=2.0")
        speed /= 2.0
    filters.append(f"atempo={speed:.6f}")
    return ",".join(filters)

# 計算所需速度並調整
for scene in narrations:
    idx = scene["index"]
    video_path = f"storyboard_video_{idx:02d}.mp4"
    audio_path = f"narration_{idx:02d}.{audio_ext()}"
    adjusted_path = f"narration_{idx:02d}_adjusted.{audio_ext()}"

    video_dur = get_duration(video_path)
    audio_dur = get_duration(audio_path)
    available = video_dur - 0.3  # 片頭0.3秒不配音

    if audio_dur > available and available > 0:
        speed = audio_dur / available
        atempo = build_atempo_chain(speed)
        print(f"  [INFO] Scene {idx}: audio {audio_dur:.2f}s > available {available:.2f}s, speed x{speed:.2f}")
        subprocess.run([
            "ffmpeg", "-y", "-i", audio_path,
            "-filter:a", atempo,
            adjusted_path
        ], check=True)
    else:
        # 速度不需調整，直接複製
        import shutil
        shutil.copy(audio_path, adjusted_path)
        print(f"  [INFO] Scene {idx}: audio {audio_dur:.2f}s fits within {available:.2f}s, no speed adjustment")
```

### Step 6：合成旁白至分鏡影片

每段分鏡影片在 **片頭 0.3 秒後** 才開始旁白，旁白之前為靜音。
合成後只保留 TTS 旁白（移除原片聲音）。
輸出加上 `-pix_fmt yuv420p`。

```python
for scene in narrations:
    idx = scene["index"]
    video_path  = f"storyboard_video_{idx:02d}.mp4"
    audio_path  = f"narration_{idx:02d}_adjusted.{audio_ext()}"
    output_path = f"storyboard_narrated_{idx:02d}.mp4"

    video_dur = get_duration(video_path)
    audio_dur = get_duration(audio_path)

    # 旁白從0.3秒開始，結尾補靜音至影片長度
    # adelay=300: 延遲300ms; apad: 補長至影片結尾; atrim確保不超過影片長度
    filter_audio = (
        f"adelay=300|300,apad,atrim=end={video_dur:.6f}"
    )

    cmd = [
        "ffmpeg", "-y",
        "-i", video_path,
        "-i", audio_path,
        "-filter_complex", f"[1:a]{filter_audio}[aout]",
        "-map", "0:v",
        "-map", "[aout]",
        "-c:v", "libx264",
        "-pix_fmt", "yuv420p",
        "-c:a", "aac",
        "-shortest",
        output_path
    ]
    subprocess.run(cmd, check=True)
    print(f"[OK] Narrated video -> {output_path}")
```

### Step 7：串接所有已配音分鏡影片（xfade 轉場）

分鏡之間加入 0.3 秒 fade 轉場。
xfade offset = 前段影片累計時長 - 0.3 秒。
最終輸出 pixel format 為 yuv420p。

**重要：音訊絕不可用 `concat` 直接串接。**
`xfade` 讓每次轉場重疊 0.3 秒，最終影片總長 = Σ影片時長 - (n-1)×0.3 秒；
但 `concat` 對音訊是首尾相接、完全不重疊，音訊總長 = Σ影片時長，比影片多出 (n-1)×0.3 秒，
且誤差每經過一次轉場就多累加 0.3 秒（分鏡 5 = 4 次轉場 → 累積 1.2 秒），導致後段影片與旁白對不齊。
正確做法是音訊也使用與 `xfade` **相同的 offset** 做 `adelay` 延遲後以 `amix` 混音（而非 `concat`），
讓每段音訊在最終時間軸上的起始點與對應分鏡影片的起始點完全一致。

```python
n = len(narrations)
narrated_videos = [f"storyboard_narrated_{i+1:02d}.mp4" for i in range(n)]
durations = [get_duration(v) for v in narrated_videos]

TRANSITION_DUR = 0.3
final_output = "storyboard_final.mp4"

if n == 1:
    import shutil
    shutil.copy(narrated_videos[0], final_output)
    print(f"[DONE] Single video, copied -> {final_output}")
else:
    # Build ffmpeg input args
    input_args = []
    for v in narrated_videos:
        input_args += ["-i", v]

    filter_parts = []

    # Video xfade chain; also record each clip's start offset on the
    # final timeline so audio can be aligned to the same points.
    video_offsets = [0.0]
    current_offset = 0.0
    prev_label = "[0:v]"
    for i in range(1, n):
        current_offset += durations[i - 1] - TRANSITION_DUR
        video_offsets.append(current_offset)
        out_label = f"[v{i}]" if i < n - 1 else "[vout]"
        filter_parts.append(
            f"{prev_label}[{i}:v]xfade=transition=fade:duration={TRANSITION_DUR}:offset={current_offset:.6f}{out_label}"
        )
        prev_label = out_label

    # Audio: delay each track to its clip's start offset (same offsets as
    # xfade above), then mix -- NOT concat. concat ignores the 0.3s overlap
    # that xfade removes from the video, so it would drift out of sync by
    # TRANSITION_DUR at every transition (cumulative across scenes).
    for i in range(n):
        delay_ms = round(video_offsets[i] * 1000)
        filter_parts.append(f"[{i}:a]adelay={delay_ms}|{delay_ms}[a{i}]")
    audio_labels = "".join(f"[a{i}]" for i in range(n))
    filter_parts.append(f"{audio_labels}amix=inputs={n}:duration=longest:normalize=0[aout]")

    filter_complex = "; ".join(filter_parts)

    cmd = [
        "ffmpeg", "-y",
        *input_args,
        "-filter_complex", filter_complex,
        "-map", "[vout]",
        "-map", "[aout]",
        "-c:v", "libx264",
        "-pix_fmt", "yuv420p",
        "-c:a", "aac",
        final_output
    ]
    subprocess.run(cmd, check=True)
    print(f"[DONE] Final video saved -> {final_output}")

# Verify pixel format
verify = subprocess.check_output([
    "ffprobe", "-v", "quiet", "-show_streams", "-select_streams", "v:0",
    "-print_format", "json", final_output
])
vinfo = json.loads(verify)
pix_fmt = vinfo["streams"][0].get("pix_fmt", "unknown")
total_dur = get_duration(final_output)
print(f"[INFO] Output: {final_output} | duration={total_dur:.2f}s | pix_fmt={pix_fmt}")
```

---

## 完整程式碼範本

儲存為**共用 storyboard 資料夾**內的 `storyboard_narration.py`，填入旁白後執行
（例如 `python storyboard/storyboard_narration.py`，`storyboard/` 替換為實際資料夾名稱）：

```python
import asyncio, base64, importlib, importlib.metadata, json, os, shutil, subprocess, sys
from pathlib import Path

sys.stdout.reconfigure(encoding="utf-8")

# Switch to the script's own directory (the shared storyboard folder) so all
# input videos and output audio/videos live in the same folder.
os.chdir(Path(__file__).resolve().parent)

TRANSITION_DUR = 0.3
GEMINI_MODEL = "gemini-3.8-flash-lite-tts"
EDGE_VOICE = "zh-TW-HsiaoChenNeural"


def ensure_package(pip_spec, import_name, min_version=None):
    """Install/upgrade unless already present at >= min_version -- a stale
    version that still imports (e.g. old google-genai missing the
    `interactions` annotation schema) must trigger an upgrade, not be
    silently accepted."""
    def ver_tuple(v):
        return tuple(int(x) for x in v.split(".")[:3] if x.isdigit())

    try:
        installed = importlib.metadata.version(import_name.split(".")[0])
    except importlib.metadata.PackageNotFoundError:
        installed = None

    needs_install = installed is None or (
        min_version and ver_tuple(installed) < ver_tuple(min_version)
    )
    if needs_install:
        subprocess.check_call([sys.executable, "-m", "pip", "install", "-U", pip_spec])
        importlib.invalidate_caches()
    return importlib.import_module(import_name)


# -------------------------------------------------------------------
# Step 1: Determine TTS engine - Gemini TTS first, Edge-TTS fallback
# -------------------------------------------------------------------
TTS_ENGINE = "gemini"
gemini_client = None

try:
    ensure_package("google-genai>=2.25.0", "google.genai", min_version="2.25.0")
    from google import genai
    from google.genai import types

    if not os.environ.get("GEMINI_API_KEY"):
        raise RuntimeError("GEMINI_API_KEY not set")
    # TTS models are v1beta-only right now; the SDK's v1 default 404s, and
    # whether it resolves to v1beta instead depends on the GOOGLE_GENAI_API_VERSION
    # env var being set in the caller's shell -- set it explicitly so this
    # doesn't silently depend on that.
    gemini_client = genai.Client(http_options=types.HttpOptions(api_version="v1beta"))
    _test = gemini_client.interactions.create(
        model=GEMINI_MODEL,
        input=[{"type": "user_input", "content": [{
            "type": "text", "text": "測試",
            "annotations": [{"type": "speech_metadata", "style": "neutral, normal pace"}],
        }]}],
        response_format={"type": "audio"},
        generation_config={"speech_config": [{"voice": "Despina"}]},
    )
    if not getattr(_test, "output_audio", None):
        raise RuntimeError("no audio in Gemini TTS response")
    print(f"[OK] Gemini TTS ready ({GEMINI_MODEL})")
except Exception as e:
    print(f"[INFO] Gemini TTS unavailable ({e}); falling back to Edge-TTS")
    TTS_ENGINE = "edge"

if TTS_ENGINE == "edge":
    try:
        import edge_tts
    except ImportError:
        subprocess.check_call([sys.executable, "-m", "pip", "install", "edge-tts"])
        import edge_tts

result = subprocess.run(["ffmpeg", "-version"], capture_output=True)
if result.returncode != 0:
    print("ERROR: ffmpeg not found. Please install ffmpeg and add it to PATH.")
    sys.exit(1)
print(f"[OK] Environment ready (TTS_ENGINE={TTS_ENGINE})")


# -------------------------------------------------------------------
# Step 2: Fill in narrations from storyboard-design output.
# voice: picked from the Gemini voice table by tone/purpose.
# style: one English sentence, overall tone/pacing/volume (Gemini only).
# gemini_text: text with <short pause>/<long pause>/<sigh>.../CAPS
#   emphasis added (Gemini only).
# text: clean transcript, no tags - used for Edge-TTS fallback.
# -------------------------------------------------------------------
narrations = [
    # {
    #     "index": 1, "title": "...",
    #     "text": "旁白文字...",
    #     "gemini_text": "旁白文字，視語氣插入 <short pause> 等標籤...",
    #     "style": "warm, friendly, normal pace, mid volume",
    #     "voice": "Despina",
    # },
]


def get_duration(path):
    cmd = ["ffprobe", "-v", "quiet", "-print_format", "json", "-show_streams", path]
    out = subprocess.check_output(cmd)
    data = json.loads(out)
    for s in data["streams"]:
        if "duration" in s:
            return float(s["duration"])
    return 0.0


def build_atempo_chain(speed):
    filters = []
    while speed > 2.0:
        filters.append("atempo=2.0")
        speed /= 2.0
    filters.append(f"atempo={speed:.6f}")
    return ",".join(filters)


def audio_ext():
    return "wav" if TTS_ENGINE == "gemini" else "mp3"


def generate_tts_gemini(text, style, voice, output_path):
    interaction = gemini_client.interactions.create(
        model=GEMINI_MODEL,
        input=[{
            "type": "user_input",
            "content": [{
                "type": "text",
                "text": text,
                "annotations": [{"type": "speech_metadata", "style": style}],
            }],
        }],
        response_format={"type": "audio"},
        generation_config={"speech_config": [{"voice": voice}]},
    )
    audio_bytes = base64.b64decode(interaction.output_audio.data)
    with open(output_path, "wb") as f:
        f.write(audio_bytes)


async def generate_tts_edge(text, output_path):
    communicate = edge_tts.Communicate(text, EDGE_VOICE)
    await communicate.save(output_path)


# Step 3: Generate TTS audio (Gemini primary, Edge-TTS fallback)
print("[START] Generating TTS audio...")
for scene in narrations:
    audio_path = f"narration_{scene['index']:02d}.{audio_ext()}"
    try:
        if TTS_ENGINE == "gemini":
            generate_tts_gemini(scene["gemini_text"], scene["style"], scene["voice"], audio_path)
        else:
            asyncio.run(generate_tts_edge(scene["text"], audio_path))
        print(f"  [OK] {audio_path}")
    except Exception as e:
        if TTS_ENGINE != "gemini":
            raise
        print(f"[WARN] Gemini TTS failed ({e}); switching ALL scenes to Edge-TTS")
        TTS_ENGINE = "edge"
        for s in narrations:
            p = f"narration_{s['index']:02d}.{audio_ext()}"
            asyncio.run(generate_tts_edge(s["text"], p))
            print(f"  [OK] (fallback) {p}")
        break

# Step 4: Adjust speed if needed
print("[START] Checking audio duration vs video duration...")
for scene in narrations:
    idx = scene["index"]
    video_path   = f"storyboard_video_{idx:02d}.mp4"
    audio_path   = f"narration_{idx:02d}.{audio_ext()}"
    adjusted_path = f"narration_{idx:02d}_adjusted.{audio_ext()}"

    video_dur = get_duration(video_path)
    audio_dur = get_duration(audio_path)
    available = video_dur - TRANSITION_DUR

    if audio_dur > available and available > 0:
        speed = audio_dur / available
        atempo = build_atempo_chain(speed)
        print(f"  [INFO] Scene {idx}: {audio_dur:.2f}s -> speed x{speed:.2f}")
        subprocess.run(
            ["ffmpeg", "-y", "-i", audio_path, "-filter:a", atempo, adjusted_path],
            check=True, capture_output=True
        )
    else:
        shutil.copy(audio_path, adjusted_path)
        print(f"  [INFO] Scene {idx}: {audio_dur:.2f}s OK (available {available:.2f}s)")

# Step 5: Merge audio into each video
print("[START] Merging narration into videos...")
for scene in narrations:
    idx = scene["index"]
    video_path   = f"storyboard_video_{idx:02d}.mp4"
    audio_path   = f"narration_{idx:02d}_adjusted.{audio_ext()}"
    output_path  = f"storyboard_narrated_{idx:02d}.mp4"
    video_dur = get_duration(video_path)

    filter_audio = f"adelay=300|300,apad,atrim=end={video_dur:.6f}"
    cmd = [
        "ffmpeg", "-y",
        "-i", video_path, "-i", audio_path,
        "-filter_complex", f"[1:a]{filter_audio}[aout]",
        "-map", "0:v", "-map", "[aout]",
        "-c:v", "libx264", "-pix_fmt", "yuv420p",
        "-c:a", "aac", "-shortest",
        output_path
    ]
    subprocess.run(cmd, check=True, capture_output=True)
    print(f"  [OK] {output_path}")

# Step 6: Concatenate with xfade transitions
print("[START] Concatenating videos with xfade transitions...")
n = len(narrations)
narrated_videos = [f"storyboard_narrated_{i+1:02d}.mp4" for i in range(n)]
durations = [get_duration(v) for v in narrated_videos]
final_output = "storyboard_final.mp4"

if n == 1:
    shutil.copy(narrated_videos[0], final_output)
else:
    input_args = []
    for v in narrated_videos:
        input_args += ["-i", v]

    filter_parts = []

    # Video xfade chain; record each clip's start offset on the final
    # timeline (needed to align audio to the same points below).
    video_offsets = [0.0]
    current_offset = 0.0
    prev_label = "[0:v]"
    for i in range(1, n):
        current_offset += durations[i - 1] - TRANSITION_DUR
        video_offsets.append(current_offset)
        out_label = f"[v{i}]" if i < n - 1 else "[vout]"
        filter_parts.append(
            f"{prev_label}[{i}:v]xfade=transition=fade:duration={TRANSITION_DUR}"
            f":offset={current_offset:.6f}{out_label}"
        )
        prev_label = out_label

    # Audio: adelay each track to its clip's start offset, then amix.
    # NOT concat -- xfade removes TRANSITION_DUR of video at every
    # transition, so a plain audio concat drifts later by TRANSITION_DUR
    # per transition (cumulative), pushing narration out of sync with
    # the video the more scenes there are.
    for i in range(n):
        delay_ms = round(video_offsets[i] * 1000)
        filter_parts.append(f"[{i}:a]adelay={delay_ms}|{delay_ms}[a{i}]")
    audio_labels = "".join(f"[a{i}]" for i in range(n))
    filter_parts.append(f"{audio_labels}amix=inputs={n}:duration=longest:normalize=0[aout]")

    cmd = [
        "ffmpeg", "-y", *input_args,
        "-filter_complex", "; ".join(filter_parts),
        "-map", "[vout]", "-map", "[aout]",
        "-c:v", "libx264", "-pix_fmt", "yuv420p",
        "-c:a", "aac",
        final_output
    ]
    subprocess.run(cmd, check=True)

# Step 7: Verify output
verify = json.loads(subprocess.check_output([
    "ffprobe", "-v", "quiet", "-show_streams", "-select_streams", "v:0",
    "-print_format", "json", final_output
]))
pix_fmt  = verify["streams"][0].get("pix_fmt", "unknown")
total_dur = get_duration(final_output)

print(f"\n[DONE] {final_output} | duration={total_dur:.2f}s | pix_fmt={pix_fmt}")
print(f"[DONE] TTS engine used: {TTS_ENGINE}")
print(f"[DONE] Narrated panels: storyboard_narrated_01.mp4 ~ storyboard_narrated_{n:02d}.mp4")
```

---

## 輸出檔案

| 檔案 | 說明 |
|---|---|
| `narration_01.wav` … `narration_N.wav`（Gemini TTS）或 `.mp3`（Edge-TTS 備援） | TTS 原始旁白音檔 |
| `narration_01_adjusted.wav` / `.mp3` … | 速度調整後的旁白音檔（若有調整） |
| `storyboard_narrated_01.mp4` … | 已配音的各分鏡影片 |
| `storyboard_final.mp4` | 串接完成的完整影片（yuv420p） |
| `storyboard_narration.py` | 保留的程式碼，不自動刪除 |

---

## 輸出原則

1. **無 Emoji 日誌**：所有 print 與程式碼註解一律使用 `[OK]`、`[FAIL]`、`[INFO]`、`[WARN]`、`[DONE]`、`[START]` 等純文字標籤
2. **Gemini TTS 優先、Edge-TTS 備援**：Step 1 以 smoke test 判斷 Gemini TTS（`gemini-3.8-flash-lite-tts`）是否可用（SDK、`GEMINI_API_KEY`、呼叫成功），可用則全程使用；不可用（或中途失敗）則**全部分鏡**統一改用 Edge-TTS（`zh-TW-HsiaoChenNeural`），不混用兩種引擎於同一支影片
3. **自動選聲**：使用 Gemini TTS 時，依「語音對照表」依各分鏡語氣/用途自動挑選合適女聲（`voice`），無明確對應時用預設人聲 `Despina`
4. **自動標註表演細節**：依 Gemini prompting guide，`style` 欄位描述整段語氣/節奏(pacing)/音量等持續性特質；`gemini_text` 以 `<short pause>`／`<long pause>` 標示停頓，以全大寫標示重音(emphasis)，並視語境插入人聲音效標籤（`<sigh>`、`<laugh>`、`<gasp>` 等 vocal bursts / non-speech sounds），避免使用非人聲音效標籤
5. **旁白完整**：每段旁白必須在分鏡影片結束前播完；不足時提高速度，絕不截斷
6. **片頭靜音**：每段分鏡旁白從 0.3 秒後才開始，不覆蓋片頭
7. **僅保留 TTS**：合成後的分鏡影片只含 TTS 旁白，移除原片聲音
8. **轉場 0.3 秒**：串接時每段之間加入 0.3 秒 fade 轉場
9. **音訊不可用 concat 串接**：`xfade` 使影片總長縮短 (n-1)×0.3 秒，音訊必須用與 `xfade` 相同的 offset 做 `adelay` 後 `amix` 混音，禁止用 `concat`（會逐段累積 0.3 秒偏移，導致旁白與畫面不同步）
10. **yuv420p**：最終輸出必須通過 ffprobe 驗證 pix_fmt=yuv420p
11. **保留程式碼**：`storyboard_narration.py` 執行後不刪除
12. **共用資料夾**：所有輸入影片與輸出音檔、影片、程式碼均存放於共用 storyboard 資料夾（`storyboard`／`storyboardNNN`）內

---

## 完成後

回報生成結果（包含實際使用的 TTS 引擎：Gemini TTS 或 Edge-TTS 備援），並詢問使用者：
> 「旁白與影片是否符合需求？若需要更換人聲、調整語氣/節奏標註、重新生成特定分鏡，或修改轉場效果，請告訴我。」
