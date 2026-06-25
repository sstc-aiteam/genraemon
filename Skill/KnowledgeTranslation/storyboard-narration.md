---
name: storyboard-narration
description: |
  根據 storyboard-design 輸出的旁白文稿，以 Edge-TTS 繁體中文人聲生成各分鏡旁白音檔，
  將旁白合成進對應的 storyboard-video 分鏡影片（片頭0.3秒後開始），最後以 ffmpeg 串接
  所有已配音分鏡影片為完整影片，分鏡之間加入0.3秒轉場，輸出 yuv420p MP4。

  當使用者提到「旁白配音」、「加旁白」、「TTS」、「Edge-TTS」、「配音合成」、「串接影片」、
  「生成完整影片」、「把分鏡影片串起來」，或分鏡影片生成後要求加入旁白並輸出完整影片時，
  務必使用此技能。
---

# 分鏡旁白配音與串接技能（Storyboard Narration Skill）

## 任務概述

從對話中讀取 storyboard-design 的旁白文稿，以 Edge-TTS 生成繁體中文旁白音檔，
合成進各分鏡影片（storyboard_video_XX.mp4），再依序串接為完整影片，
分鏡之間加入 0.3 秒轉場特效，最終輸出 pixel format 為 yuv420p 的 MP4。

---

## 步驟流程

### Step 1：確認環境

```python
import subprocess, sys, os

# 確認 edge-tts
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
print("[OK] Environment ready")
```

### Step 2：解析旁白文稿

從對話中讀取最近一份 storyboard-design 輸出，格式為 `## 分鏡 N｜標題`，
擷取每個分鏡的 **🎙️ 旁白** 欄位文字。

```python
# 手動或程式化填入，保留繁體中文原文
narrations = [
    {"index": 1, "title": "...", "text": "旁白原文..."},
    {"index": 2, "title": "...", "text": "旁白原文..."},
    # ...
]
```

### Step 3：以 Edge-TTS 生成旁白音檔

**預設人聲：** `zh-TW-HsiaoChenNeural`（繁體中文女聲）
可替換選項：`zh-TW-YunJheNeural`（男聲）、`zh-TW-HsiaoYuNeural`（女聲）

```python
import asyncio
import edge_tts

VOICE = "zh-TW-HsiaoChenNeural"

async def generate_tts(text, output_path):
    communicate = edge_tts.Communicate(text, VOICE)
    await communicate.save(output_path)

for scene in narrations:
    audio_path = f"narration_{scene['index']:02d}.mp3"
    asyncio.run(generate_tts(scene["text"], audio_path))
    print(f"[OK] TTS generated -> {audio_path}")
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
    audio_path = f"narration_{idx:02d}.mp3"
    adjusted_path = f"narration_{idx:02d}_adjusted.mp3"

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
合成後只保留 Edge-TTS 旁白（移除原片聲音）。
輸出加上 `-pix_fmt yuv420p`。

```python
for scene in narrations:
    idx = scene["index"]
    video_path  = f"storyboard_video_{idx:02d}.mp4"
    audio_path  = f"narration_{idx:02d}_adjusted.mp3"
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

    # Build xfade filter chain for video
    # Build concat filter for audio
    filter_parts = []
    current_offset = 0.0

    # Video xfade chain
    prev_label = "[0:v]"
    for i in range(1, n):
        current_offset += durations[i - 1] - TRANSITION_DUR
        out_label = f"[v{i}]" if i < n - 1 else "[vout]"
        filter_parts.append(
            f"{prev_label}[{i}:v]xfade=transition=fade:duration={TRANSITION_DUR}:offset={current_offset:.6f}{out_label}"
        )
        prev_label = out_label

    # Audio concat (simple concat, no crossfade needed since audio is TTS only)
    audio_inputs = "".join(f"[{i}:a]" for i in range(n))
    filter_parts.append(f"{audio_inputs}concat=n={n}:v=0:a=1[aout]")

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

儲存為 `storyboard_narration.py`，填入旁白後執行：

```python
import asyncio, json, os, shutil, subprocess, sys

sys.stdout.reconfigure(encoding="utf-8")

try:
    import edge_tts
except ImportError:
    subprocess.check_call([sys.executable, "-m", "pip", "install", "edge-tts"])
    import edge_tts

VOICE = "zh-TW-HsiaoChenNeural"
TRANSITION_DUR = 0.3

# -------------------------------------------------------------------
# Fill in narrations from storyboard-design output
# -------------------------------------------------------------------
narrations = [
    # {"index": 1, "title": "...", "text": "旁白文字..."},
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


async def generate_tts(text, output_path):
    communicate = edge_tts.Communicate(text, VOICE)
    await communicate.save(output_path)


# Step 1: Generate TTS audio
print("[START] Generating TTS audio...")
for scene in narrations:
    audio_path = f"narration_{scene['index']:02d}.mp3"
    asyncio.run(generate_tts(scene["text"], audio_path))
    print(f"  [OK] {audio_path}")

# Step 2: Adjust speed if needed
print("[START] Checking audio duration vs video duration...")
for scene in narrations:
    idx = scene["index"]
    video_path   = f"storyboard_video_{idx:02d}.mp4"
    audio_path   = f"narration_{idx:02d}.mp3"
    adjusted_path = f"narration_{idx:02d}_adjusted.mp3"

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

# Step 3: Merge audio into each video
print("[START] Merging narration into videos...")
for scene in narrations:
    idx = scene["index"]
    video_path   = f"storyboard_video_{idx:02d}.mp4"
    audio_path   = f"narration_{idx:02d}_adjusted.mp3"
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

# Step 4: Concatenate with xfade transitions
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
    current_offset = 0.0
    prev_label = "[0:v]"
    for i in range(1, n):
        current_offset += durations[i - 1] - TRANSITION_DUR
        out_label = f"[v{i}]" if i < n - 1 else "[vout]"
        filter_parts.append(
            f"{prev_label}[{i}:v]xfade=transition=fade:duration={TRANSITION_DUR}"
            f":offset={current_offset:.6f}{out_label}"
        )
        prev_label = out_label

    audio_inputs = "".join(f"[{i}:a]" for i in range(n))
    filter_parts.append(f"{audio_inputs}concat=n={n}:v=0:a=1[aout]")

    cmd = [
        "ffmpeg", "-y", *input_args,
        "-filter_complex", "; ".join(filter_parts),
        "-map", "[vout]", "-map", "[aout]",
        "-c:v", "libx264", "-pix_fmt", "yuv420p",
        "-c:a", "aac",
        final_output
    ]
    subprocess.run(cmd, check=True)

# Step 5: Verify output
verify = json.loads(subprocess.check_output([
    "ffprobe", "-v", "quiet", "-show_streams", "-select_streams", "v:0",
    "-print_format", "json", final_output
]))
pix_fmt  = verify["streams"][0].get("pix_fmt", "unknown")
total_dur = get_duration(final_output)

print(f"\n[DONE] {final_output} | duration={total_dur:.2f}s | pix_fmt={pix_fmt}")
print(f"[DONE] Narrated panels: storyboard_narrated_01.mp4 ~ storyboard_narrated_{n:02d}.mp4")
```

---

## 輸出檔案

| 檔案 | 說明 |
|---|---|
| `narration_01.mp3` … `narration_N.mp3` | Edge-TTS 原始旁白音檔 |
| `narration_01_adjusted.mp3` … | 速度調整後的旁白音檔（若有調整） |
| `storyboard_narrated_01.mp4` … | 已配音的各分鏡影片 |
| `storyboard_final.mp4` | 串接完成的完整影片（yuv420p） |
| `storyboard_narration.py` | 保留的程式碼，不自動刪除 |

---

## 輸出原則

1. **無 Emoji 日誌**：所有 print 與程式碼註解一律使用 `[OK]`、`[FAIL]`、`[INFO]`、`[DONE]`、`[START]` 等純文字標籤
2. **旁白完整**：每段旁白必須在分鏡影片結束前播完；不足時提高速度，絕不截斷
3. **片頭靜音**：每段分鏡旁白從 0.3 秒後才開始，不覆蓋片頭
4. **僅保留 TTS**：合成後的分鏡影片只含 Edge-TTS 旁白，移除原片聲音
5. **轉場 0.3 秒**：串接時每段之間加入 0.3 秒 fade 轉場
6. **yuv420p**：最終輸出必須通過 ffprobe 驗證 pix_fmt=yuv420p
7. **保留程式碼**：`storyboard_narration.py` 執行後不刪除

---

## 完成後

回報生成結果，並詢問使用者：
> 「旁白與影片是否符合需求？若需要更換人聲（男/女）、調整旁白文字、重新生成特定分鏡，或修改轉場效果，請告訴我。」
