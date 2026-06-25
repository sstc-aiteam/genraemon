---
name: storyboard-bgm
description: |
  根據 storyboard-design 輸出的分鏡腳本分析情緒與主題，呼叫 Gemini Lyria 3 API 生成配合影片長度的純器樂背景音樂，
  再以 ffmpeg 將 BGM 混入 storyboard-narration 輸出的完整影片（storyboard_final.mp4），輸出含背景音樂的最終影片。

  當使用者提到「背景音樂」、「BGM」、「配樂」、「幫影片加音樂」、「生成背景樂」、「add background music」、
  「Lyria」、「音樂生成」，或分鏡影片/旁白影片完成後要求加入背景配樂時，務必使用此技能。
  即使使用者只說「幫我加音樂」或「讓影片有背景音樂」也應觸發。
---

# 分鏡背景音樂生成技能（Storyboard BGM Generation Skill）

## 任務概述

解析對話中的 storyboard-design 分鏡腳本，提取情感基調與敘事情境，
以 Gemini Lyria 3 API 生成純器樂背景音樂（無人聲）。
音樂總長略長於 `storyboard_final.mp4`，確保結尾不截斷。
最後以 ffmpeg 將 BGM 低音量混入 `storyboard_final.mp4`，輸出 `storyboard_final_bgm.mp4`。

---

## 步驟流程

### Step 1：確認環境

確認以下兩項：
1. `GEMINI_API_KEY` 環境變數已設定
2. `storyboard_final.mp4` 存在於工作目錄（storyboard-narration 的輸出）

若任一項缺失，停止並提示：

```
錯誤：GEMINI_API_KEY 環境變數未設定。
請設定後再執行：export GEMINI_API_KEY="AIza..."
```

```
錯誤：找不到 storyboard_final.mp4。
請先執行 storyboard-narration 技能，生成含旁白的完整影片。
```

---

### Step 2：解析分鏡腳本，建立音樂描述 Prompt

從對話中讀取 storyboard-design 的分鏡腳本，分析以下維度：

**2a. 識別主題類型**

| 主題類型 | 判斷依據 | 音樂風格方向 |
|---|---|---|
| 教育／科普 | 說明性構圖、步驟示範、圖表 | 輕快正向的器樂，鋼琴＋輕弦樂 |
| 商業／行銷 | 品牌色彩、產品特寫、成果展示 | 現代感企業配樂，動感節奏＋輕電子元素 |
| 歷史／敘事 | 場景全景、時間軸、人物情感 | 情感豐富的管弦樂，緩慢鋪陳 |
| 技術／數據 | 資料視覺化、程式碼、系統圖 | 科技感電子氛圍樂，中性沉穩 |
| 社會／人文 | 情感場景、人際關係、社會議題 | 溫暖感人的鋼琴＋弦樂，引發共鳴 |

**2b. 分析情緒曲線**

逐一檢視分鏡的旁白與構圖描述，判斷整體情緒走向：
- 開場是否需要引人入勝的序奏？
- 中段有無需要張力的轉折？
- 結尾是否需要昇華或號召？

**2c. 建立音樂 Prompt**

將分析結果組合成英文 Prompt，格式如下：

```
Instrumental background music only, absolutely no vocals or singing.
[風格描述：e.g., Warm and uplifting orchestral piece with piano and strings]
[情緒走向：e.g., Opens gently, builds emotional depth in the middle, and ends with a sense of hope and resolution]
[禁止項目：] No lyrics, no voice, no spoken words, purely instrumental.
[時長：] Approximately [N+5] seconds long.
Suitable as background music for an educational/corporate/documentary video.
```

> **[N+5] 秒**：N 為影片總時長（秒），+5 秒確保音樂略長於影片。

---

### Step 3：計算所需片段數

Lyria 3 每次最多生成約 **30 秒**的音樂片段。
若影片超過 25 秒，需多次呼叫 API 並串接。

```python
import math

LYRIA_MAX_SECONDS = 30  # 每次生成上限（保守估計）

def calc_segments(video_duration_sec):
    target = video_duration_sec + 5  # 略長緩衝
    return math.ceil(target / LYRIA_MAX_SECONDS)
```

---

### Step 4：生成 Python 腳本並儲存為 storyboard_bgm.py

**程式碼生成規則：**
- 所有 `print()` 日誌不得含有 Emoji，使用 `[OK]`、`[FAIL]`、`[ERROR]`、`[DONE]`、`[START]`、`[INFO]`
- 程式碼註解不得含有 Emoji
- 生成後 `storyboard_bgm.py` 不得刪除

**完整程式碼範本：**

```python
import os
import sys
import json
import math
import base64
import subprocess
import requests
from pathlib import Path

sys.stdout.reconfigure(encoding="utf-8")

# -------------------------------------------------------------------
# Configuration
# -------------------------------------------------------------------
GEMINI_API_KEY = os.environ.get("GEMINI_API_KEY")
if not GEMINI_API_KEY:
    print("[ERROR] GEMINI_API_KEY is not set.")
    sys.exit(1)

LYRIA_URL = (
    "https://generativelanguage.googleapis.com/v1beta"
    "/models/lyria-3-pro-preview:generateContent"
)
LYRIA_MAX_SECONDS = 30  # conservative max per generation call

INPUT_VIDEO = "storyboard_final.mp4"
BGM_OUTPUT  = "storyboard_bgm.wav"
FINAL_OUTPUT = "storyboard_final_bgm.mp4"

# -------------------------------------------------------------------
# Music prompt (filled in by storyboard-bgm skill based on script)
# -------------------------------------------------------------------
MUSIC_PROMPT = """
Instrumental background music only, absolutely no vocals or singing.
[FILL_IN_MUSIC_DESCRIPTION]
No lyrics, no voice, no spoken words, purely instrumental.
Approximately [FILL_IN_DURATION] seconds long.
Suitable as background music for a video presentation.
"""

BGM_VOLUME = 0.18  # BGM volume relative to narration (0.0 - 1.0)


# -------------------------------------------------------------------
# Utilities
# -------------------------------------------------------------------

def get_duration(path):
    cmd = [
        "ffprobe", "-v", "quiet",
        "-print_format", "json",
        "-show_streams", path
    ]
    out = subprocess.check_output(cmd)
    data = json.loads(out)
    for stream in data.get("streams", []):
        if "duration" in stream:
            return float(stream["duration"])
    return 0.0


def generate_bgm_segment(prompt, segment_index):
    print(f"  [INFO] Calling Lyria API for segment {segment_index}...")
    headers = {
        "x-goog-api-key": GEMINI_API_KEY,
        "Content-Type": "application/json",
    }
    payload = {
        "contents": [{
            "parts": [{"text": prompt}]
        }],
        "generationConfig": {
            "responseModalities": ["AUDIO", "TEXT"]
        }
    }
    resp = requests.post(LYRIA_URL, headers=headers, json=payload, timeout=180)
    resp.raise_for_status()
    data = resp.json()

    candidates = data.get("candidates", [])
    for candidate in candidates:
        parts = candidate.get("content", {}).get("parts", [])
        for part in parts:
            inline = part.get("inlineData", {})
            mime = inline.get("mimeType", "")
            if mime.startswith("audio/"):
                audio_bytes = base64.b64decode(inline["data"])
                ext = mime.split("/")[-1].split(";")[0]
                seg_path = f"bgm_segment_{segment_index:02d}.{ext}"
                Path(seg_path).write_bytes(audio_bytes)
                print(f"  [OK] Segment {segment_index} saved -> {seg_path} ({len(audio_bytes)} bytes)")
                return seg_path
    raise RuntimeError(f"No audio found in Lyria response for segment {segment_index}")


def concat_audio_segments(segment_paths, output_path):
    if len(segment_paths) == 1:
        import shutil
        shutil.copy(segment_paths[0], output_path)
        print(f"  [INFO] Single segment, copied -> {output_path}")
        return

    list_file = "bgm_concat_list.txt"
    with open(list_file, "w", encoding="utf-8") as f:
        for p in segment_paths:
            f.write(f"file '{p}'\n")

    cmd = [
        "ffmpeg", "-y",
        "-f", "concat", "-safe", "0",
        "-i", list_file,
        "-c", "copy",
        output_path
    ]
    subprocess.run(cmd, check=True, capture_output=True)
    Path(list_file).unlink(missing_ok=True)
    print(f"  [OK] Concatenated {len(segment_paths)} segments -> {output_path}")


def trim_audio(input_path, output_path, target_duration):
    cmd = [
        "ffmpeg", "-y",
        "-i", input_path,
        "-t", str(target_duration),
        "-c", "copy",
        output_path
    ]
    subprocess.run(cmd, check=True, capture_output=True)
    print(f"  [OK] Trimmed to {target_duration:.2f}s -> {output_path}")


def mix_bgm_into_video(video_path, bgm_path, output_path, bgm_volume, video_duration):
    filter_complex = (
        f"[1:a]volume={bgm_volume:.4f},apad,atrim=end={video_duration:.6f}[bgm];"
        f"[0:a][bgm]amix=inputs=2:duration=first:dropout_transition=3[aout]"
    )
    cmd = [
        "ffmpeg", "-y",
        "-i", video_path,
        "-i", bgm_path,
        "-filter_complex", filter_complex,
        "-map", "0:v",
        "-map", "[aout]",
        "-c:v", "copy",
        "-c:a", "aac",
        "-pix_fmt", "yuv420p",
        output_path
    ]
    subprocess.run(cmd, check=True)
    print(f"[OK] BGM mixed into video -> {output_path}")


# -------------------------------------------------------------------
# Main
# -------------------------------------------------------------------

def main():
    if not Path(INPUT_VIDEO).exists():
        print(f"[ERROR] {INPUT_VIDEO} not found. Run storyboard-narration first.")
        sys.exit(1)

    print(f"[START] Reading video duration: {INPUT_VIDEO}")
    video_duration = get_duration(INPUT_VIDEO)
    print(f"  [INFO] Video duration: {video_duration:.2f}s")

    target_bgm_duration = video_duration + 5.0
    num_segments = math.ceil(target_bgm_duration / LYRIA_MAX_SECONDS)
    print(f"  [INFO] Target BGM duration: {target_bgm_duration:.2f}s -> {num_segments} segment(s)")

    # Generate BGM segments
    print("[START] Generating BGM via Lyria API...")
    segment_paths = []
    for i in range(1, num_segments + 1):
        try:
            seg_path = generate_bgm_segment(MUSIC_PROMPT, i)
            segment_paths.append(seg_path)
        except Exception as e:
            print(f"  [FAIL] Segment {i} failed: {e}")
            if not segment_paths:
                print("[ERROR] No segments generated, aborting.")
                sys.exit(1)
            print(f"  [INFO] Proceeding with {len(segment_paths)} segment(s)")
            break

    # Concatenate segments
    print("[START] Assembling BGM audio...")
    raw_bgm = "bgm_raw.wav"
    concat_audio_segments(segment_paths, raw_bgm)

    # Trim to target duration
    trim_audio(raw_bgm, BGM_OUTPUT, target_bgm_duration)

    actual_bgm_duration = get_duration(BGM_OUTPUT)
    print(f"  [INFO] Final BGM duration: {actual_bgm_duration:.2f}s")

    # Mix into video
    print("[START] Mixing BGM into video...")
    mix_bgm_into_video(INPUT_VIDEO, BGM_OUTPUT, FINAL_OUTPUT, BGM_VOLUME, video_duration)

    # Verify output
    final_duration = get_duration(FINAL_OUTPUT)
    verify = json.loads(subprocess.check_output([
        "ffprobe", "-v", "quiet", "-show_streams", "-select_streams", "v:0",
        "-print_format", "json", FINAL_OUTPUT
    ]))
    pix_fmt = verify["streams"][0].get("pix_fmt", "unknown")

    print(f"\n[DONE] {FINAL_OUTPUT}")
    print(f"[DONE] duration={final_duration:.2f}s | pix_fmt={pix_fmt} | bgm_volume={BGM_VOLUME}")
    print(f"[DONE] BGM file preserved: {BGM_OUTPUT}")
    print(f"[DONE] Script preserved: storyboard_bgm.py")


if __name__ == "__main__":
    main()
```

---

### Step 5：填入腳本資料並執行

**5a. 根據 Step 2 的分析填入 `MUSIC_PROMPT`**

從分鏡腳本提取主題、情緒與敘事走向，替換 `MUSIC_PROMPT` 中的佔位符。

完整的 `MUSIC_PROMPT` 範例（教育類）：
```python
MUSIC_PROMPT = """
Instrumental background music only, absolutely no vocals or singing.
Warm and uplifting orchestral piece featuring piano and strings.
Opens with a gentle, curious melody; builds emotional warmth and clarity in the middle
as concepts are explained; ends with an inspiring, hopeful resolution.
No lyrics, no voice, no spoken words, purely instrumental.
Approximately 65 seconds long.
Suitable as background music for an educational documentary video.
"""
```

**5b. 設定 `BGM_VOLUME`**

依旁白影片是否有明顯旁白來調整比例：

| 情境 | 建議 BGM_VOLUME |
|---|---|
| 有清晰旁白（storyboard_final.mp4 含 TTS） | `0.15` ~ `0.20` |
| 無旁白、僅影像 | `0.60` ~ `0.80` |
| 旁白較少 | `0.25` ~ `0.35` |

**5c. 儲存並執行**

```bash
python storyboard_bgm.py
```

若缺少套件：
```bash
pip install requests
```

---

## 輸出檔案

| 檔案 | 說明 |
|---|---|
| `bgm_segment_01.wav` … | Lyria API 生成的原始片段音檔 |
| `bgm_raw.wav` | 串接後的完整原始 BGM |
| `storyboard_bgm.wav` | 裁切至目標長度的最終 BGM |
| `storyboard_final_bgm.mp4` | 含旁白＋背景音樂的最終完整影片（yuv420p） |
| `storyboard_bgm.py` | 保留的程式碼，不自動刪除 |

---

## 輸出原則

1. **無人聲**：MUSIC_PROMPT 明確要求 `no vocals`、`no voice`、`no spoken words`，禁止任何人聲元素
2. **略長於影片**：BGM 目標長度為影片時長 + 5 秒，防止結尾音樂截斷感
3. **音量平衡**：BGM 以低音量（預設 0.18）混入，不遮蓋旁白，使用 `amix` 保留原始旁白動態
4. **多段串接**：若影片超過 25 秒，自動拆分為多次 Lyria API 呼叫並串接，確保音樂完整連續
5. **yuv420p**：最終輸出影片視訊流以 `-c:v copy` 保留原始編碼，僅重新編碼音軌
6. **保留程式碼**：`storyboard_bgm.py` 執行後不刪除
7. **無 Emoji 日誌**：所有 `print()` 與程式碼註解一律使用純文字標籤

---

## 注意事項

### Lyria API 行為

- 每次呼叫生成約 20～30 秒音訊，實際長度依 Prompt 而異
- 回傳格式：`candidates[0].content.parts[N].inlineData.mimeType` 為 `audio/wav` 或 `audio/mp3`，`data` 欄位為 base64 編碼
- 若回傳中無音訊，檢查是否觸發安全過濾（含人聲關鍵詞或不當主題）
- API 呼叫 timeout 建議設為 180 秒（生成耗時較長）

### ffmpeg 音訊混合

- `amix=inputs=2:duration=first` 確保輸出長度等於影片長度
- `dropout_transition=3` 讓 BGM 在影片結束前 3 秒淡出，避免硬切
- BGM 以 `apad` 補長至影片長度（當 BGM 生成長度不足時備援）

### 錯誤處理

- 若某段 Lyria API 呼叫失敗，以已成功的片段繼續（可能音樂略短於目標）
- 若所有片段均失敗，輸出錯誤訊息並中止，不修改原始影片
- 若 `storyboard_final.mp4` 不存在，明確提示先執行 storyboard-narration 技能

---

## 完成後

輸出完成後，向使用者說明：
> 「背景音樂已生成並混入影片！最終影片為 `storyboard_final_bgm.mp4`，BGM 音量設為旁白的 18%。若需要調整音量比例、更換音樂風格，或重新生成特定段落，請告訴我。」
