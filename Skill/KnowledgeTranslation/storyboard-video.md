---
name: storyboard-video
description: |
  根據 storyboard-design 輸出的分鏡腳本文字稿，以及 storyboard-image 輸出的分鏡圖卡，
  呼叫 xAI Imagine Video Generation API，為每個分鏡生成一段短影片並儲存至工作目錄。
  每段影片預設 6 秒，畫面文字以繁體中文為主，禁用簡體字。

  當使用者提到「生成影片」、「分鏡影片」、「storyboard video」、「幫我做影片」、
  「把分鏡做成影片」、「影片生成」，或在分鏡腳本／圖卡輸出後要求影片化時，務必使用此技能。
  即使使用者只說「做成影片」、「生成影片」也應觸發。
---

# 分鏡影片生成技能（Storyboard Video Generation Skill）

## 任務概述

以 storyboard-design 的分鏡文字稿（畫面構圖、重點動作）與 storyboard-image 的圖卡
作為輸入，呼叫 xAI Imagine Video Generation API，逐鏡生成影片並儲存。

---

## 輸出目錄規則（所有 storyboard 技能共用）

本技能產生的**所有檔案**（`storyboard_video.py`、各分鏡 `storyboard_video_XX.mp4`…）
以及讀取的輸入圖卡（`storyboard_panel_XX.jpg`），都位於與 storyboard-design 相同的**共用資料夾**內。

- 基底名稱為 `storyboard`；若已存在則以連號尾數區分：`storyboard001`、`storyboard002`、…
- 本技能屬於**下游技能**，沿用**最新**的既有資料夾（`create_new=False`）。
- 作法：先解析出共用資料夾，將 `storyboard_video.py` 寫入該資料夾，
  腳本開頭 `os.chdir()` 至自身所在目錄，之後所有相對路徑（圖卡、影片）都會落在該資料夾內。

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

## xAI Video API 正確行為（實測確認）

| 項目 | 正確值 |
|---|---|
| 送出端點 | `POST https://api.x.ai/v1/videos/generations` |
| 送出回傳 | `{"request_id": "..."}` （注意是 `request_id`，不是 `id`） |
| 輪詢端點 | `GET https://api.x.ai/v1/videos/{request_id}` |
| 輪詢進行中 | HTTP 202 `{"status":"pending","progress":N}` |
| 輪詢完成 | HTTP 200 `{"status":"done","video":{"url":"..."},"progress":100}` |
| 解析度 | 使用者可選 `480p`（預設）或 `720p`；若 400 錯誤自動升試下一階 |
| 圖片欄位格式 | 頂層獨立欄位 `"image": {"url": "data:image/jpeg;base64,..."}` ；`prompt` 保持純文字字串 |
| 必填 model | `"model": "grok-imagine-video"` |
| 輪詢 headers | 僅需 `Authorization`，不需 `Content-Type` |
| 終止狀態 | `"done"` 或 `"expired"`（非 `"failed"`/`"cancelled"`） |

---

## 步驟流程

### Step 1：確認環境

```python
import os
if not os.environ.get("XAI_API_KEY"):
    print("ERROR: XAI_API_KEY is not set. Please configure the environment variable.")
    exit(1)
```

### Step 2：詢問生成品質

在開始生成前，詢問使用者偏好的影片解析度：

> 「請選擇影片生成品質：
> - **480p**（預設，生成速度較快，適合快速預覽）
> - **720p**（較高畫質，適合正式輸出）
>
> 若未指定，預設使用 480p。」

將使用者的選擇儲存為 `RESOLUTION`（值為 `"480p"` 或 `"720p"`），後續程式碼使用此變數。

### Step 3：解析輸入

從對話中讀取最近一份分鏡腳本，格式為 `## 分鏡 N｜標題`，擷取每個分鏡的：
- 標題
- 🎨 畫面構圖
- 🎬 重點動作

同時掃描工作目錄尋找對應圖卡（`storyboard_panel_01.jpg`、`storyboard_panel_02.jpg`…）。
若圖卡存在，以 image-to-video 模式提交；若不存在或圖片上傳失敗，以純文字 prompt 模式備援。

### Step 4：建立共通風格描述（Style Prefix）

**優先讀取共用設計系統：** 先檢查共用 storyboard 資料夾內是否存在
`storyboard_style.json`（由 storyboard-style 技能產生）。若存在，直接使用其
`global_style_prompt` 作為 STYLE_PREFIX，**不再依主題自行判斷**——這確保影片與
storyboard-image 產出的圖卡套用完全相同的美術風格與色票，不會出現圖卡走寫實
HUD、影片卻套用 flat 2D vector 前綴的衝突。若檔案不存在（使用者未執行
storyboard-style），才依下列主題分類表自行判斷，並可在完成後提醒使用者：
「建議先執行 storyboard-style 統一設計系統，讓圖卡與影片風格一致。」

**找不到 storyboard_style.json 時的備援邏輯：** 依分鏡主題選擇對應的 style
prefix，確保所有影片畫風一致：

**教育／科學類（預設）：**
```
flat design infographic animation, clean white background, simple geometric shapes,
unified blue-green color palette, minimal labels in Traditional Chinese,
storyboard panel aesthetic, non-realistic style, 2D vector art animation
```

**商業／行銷類：**
```
corporate flat illustration animation, warm tones (orange, teal, white),
professional icons, clean character design, storyboard panel aesthetic
```

**歷史／敘事類：**
```
semi-realistic illustration animation, warm muted tones, sketch-like linework,
clear focal point, storyboard panel aesthetic
```

**技術／數據類：**
```
data visualization animation, dark blue background, glowing data elements,
minimal UI charts, storyboard panel aesthetic
```

### Step 5：組合每個分鏡的 Video Prompt

```
[Style prefix], [畫面構圖原文], [重點動作原文], [固定規則]
```

固定規則（每個分鏡都加入）：
```
any on-screen text must use Traditional Chinese characters only, no Simplified Chinese,
16:9 aspect ratio, 6 seconds, smooth animation, clear visual hierarchy
```

> 畫面構圖與重點動作保留分鏡腳本的繁體中文原文，不翻譯為英文。

### Step 6：生成 Python 程式碼並儲存為 storyboard_video.py

**程式碼生成規則：**
- 所有 `print()` 日誌不得含有 Emoji，使用純文字標籤：`[OK]`、`[FAIL]`、`[ERROR]`、`[DONE]`、`[START]`、`[SUBMITTED]`、`[STATUS]`、`[INFO]`、`[DEBUG]`、`[RETRY]`、`[TIMEOUT]`
- 所有程式碼註解不得含有 Emoji
- 生成完成後，程式碼檔案 `storyboard_video.py` 不得刪除

**完整程式碼範本：**

```python
import os
import json
import time
import base64
import requests
from pathlib import Path

# Switch to the script's own directory (the shared storyboard folder) so all
# input images and output videos live in the same folder.
os.chdir(Path(__file__).resolve().parent)

XAI_API_KEY = os.environ.get("XAI_API_KEY")
if not XAI_API_KEY:
    print("ERROR: XAI_API_KEY is not set.")
    exit(1)

SUBMIT_URL = "https://api.x.ai/v1/videos/generations"
POLL_BASE  = "https://api.x.ai/v1/videos"
HEADERS = {
    "Authorization": f"Bearer {XAI_API_KEY}",
    "Content-Type": "application/json"
}

# Resolution chosen by user: "480p" (default) or "720p"
RESOLUTION = "480p"
RESOLUTION_FALLBACK = {"480p": "720p", "720p": "1080p"}

# --- Design system: load storyboard_style.json produced by the storyboard-style
# skill (single source of truth shared with storyboard-image). Falls back to the
# original hardcoded default when the file is absent (skill was skipped). ---
STYLE_SPEC_PATH = Path("storyboard_style.json")
if STYLE_SPEC_PATH.exists():
    try:
        _style_spec = json.loads(STYLE_SPEC_PATH.read_text(encoding="utf-8"))
        STYLE_PREFIX = _style_spec["global_style_prompt"]
        print(f"[Style] Loaded storyboard_style.json (family={_style_spec['art_style']['family']})")
    except Exception as e:
        print(f"  [WARN] Failed to load storyboard_style.json ({e}), using fallback style")
        STYLE_PREFIX = None
else:
    print("  [WARN] storyboard_style.json not found. Run storyboard-style first for "
          "consistent image/video styling. Falling back to local default.")
    STYLE_PREFIX = None

if not STYLE_PREFIX:
    STYLE_PREFIX = (
        "flat design infographic animation, clean white background, simple geometric shapes, "
        "unified blue-green color palette, minimal labels in Traditional Chinese, "
        "storyboard panel aesthetic, non-realistic style, 2D vector art animation"
    )

TEXT_RULE = (
    "any on-screen text must use Traditional Chinese characters only, no Simplified Chinese, "
    "16:9 aspect ratio, 6 seconds, smooth animation, clear visual hierarchy"
)

# -------------------------------------------------------------------
# storyboard_scenes is populated from storyboard-design output.
# Each entry must contain: index, title, composition, motion.
# -------------------------------------------------------------------
storyboard_scenes = [
    # populated at generation time from parsed storyboard script
]


def build_text_prompt(scene):
    return f"{STYLE_PREFIX}, {scene['composition']}, {scene['motion']}, {TEXT_RULE}"


def load_image_data_uri(path):
    with open(path, "rb") as f:
        b64 = base64.b64encode(f.read()).decode("utf-8")
    return f"data:image/jpeg;base64,{b64}"


POLL_HEADERS = {"Authorization": f"Bearer {XAI_API_KEY}"}


def submit_generation(scene, panel_path=None):
    text_prompt = build_text_prompt(scene)

    payload = {
        "model": "grok-imagine-video",
        "prompt": text_prompt,
        "duration": 6,
        "resolution": RESOLUTION,
    }

    # Attach reference image as top-level "image" field (image-to-video mode)
    if panel_path and os.path.exists(panel_path):
        payload["image"] = {"url": load_image_data_uri(panel_path)}
        print(f"  [INFO] image-to-video mode, reference: {panel_path}")
    else:
        print(f"  [INFO] text-only mode (no reference image found)")

    try:
        resp = requests.post(SUBMIT_URL, headers=HEADERS, json=payload, timeout=60)

        # Image rejected: fall back to text-only
        if resp.status_code == 422 and "image" in payload:
            print(f"  [RETRY] Image rejected (422), retrying text-only")
            payload.pop("image")
            resp = requests.post(SUBMIT_URL, headers=HEADERS, json=payload, timeout=60)

        # Resolution not supported: fall back to next tier
        if resp.status_code == 400 and RESOLUTION in RESOLUTION_FALLBACK:
            fallback_res = RESOLUTION_FALLBACK[RESOLUTION]
            print(f"  [RETRY] Resolution {RESOLUTION} rejected (400), retrying {fallback_res}")
            payload["resolution"] = fallback_res
            resp = requests.post(SUBMIT_URL, headers=HEADERS, json=payload, timeout=60)

        resp.raise_for_status()
        return resp.json()

    except requests.exceptions.HTTPError as e:
        body = e.response.text[:300] if e.response is not None else "no response body"
        raise RuntimeError(f"Submit failed: HTTP {e.response.status_code}: {body}") from e


def poll_status(request_id, max_wait=300, interval=5):
    url = f"{POLL_BASE}/{request_id}"
    elapsed = 0
    while elapsed < max_wait:
        resp = requests.get(url, headers=POLL_HEADERS, timeout=30)
        data = resp.json()
        status = data.get("status", "unknown")
        progress = data.get("progress", 0)
        print(f"  [STATUS] id={request_id} status={status} progress={progress}% elapsed={elapsed}s")

        if status == "done":
            return data
        if status == "expired":
            print(f"  [FAIL] Request expired")
            return None

        time.sleep(interval)
        elapsed += interval

    print(f"  [TIMEOUT] Exceeded {max_wait}s for id={request_id}")
    return None


def extract_video_url(data):
    video = data.get("video")
    if isinstance(video, dict):
        return video.get("url")
    for candidate in [
        data.get("video_url"),
        data.get("url"),
        data.get("download_url"),
    ]:
        if candidate:
            return candidate
    return None


def download_video(video_url, output_path):
    resp = requests.get(video_url, stream=True, timeout=120)
    resp.raise_for_status()
    with open(output_path, "wb") as f:
        for chunk in resp.iter_content(chunk_size=8192):
            f.write(chunk)


results = []

for scene in storyboard_scenes:
    idx = scene["index"]
    panel_path = f"storyboard_panel_{idx:02d}.jpg"
    output_path = f"storyboard_video_{idx:02d}.mp4"

    print(f"[START] Scene {idx}: {scene['title']}")
    text_prompt = build_text_prompt(scene)
    print(f"  [PROMPT] {text_prompt[:120]}...")

    try:
        gen_data = submit_generation(scene, panel_path)
        request_id = gen_data.get("request_id") or gen_data.get("id")

        if not request_id:
            print(f"  [FAIL] No request_id in response")
            print(f"  [DEBUG] Response: {gen_data}")
            results.append({"index": idx, "title": scene["title"], "status": "failed", "path": None})
            continue

        print(f"  [SUBMITTED] request_id={request_id}")
        final_data = poll_status(request_id)

        if not final_data:
            results.append({"index": idx, "title": scene["title"], "status": "failed", "path": None})
            continue

        video_url = extract_video_url(final_data)
        if not video_url:
            print(f"  [FAIL] No video URL found in completed response")
            print(f"  [DEBUG] Response keys: {list(final_data.keys())}")
            results.append({"index": idx, "title": scene["title"], "status": "failed", "path": None})
            continue

        download_video(video_url, output_path)
        print(f"  [OK] Saved -> {output_path}")
        results.append({"index": idx, "title": scene["title"], "status": "ok", "path": output_path})

    except requests.exceptions.HTTPError as e:
        body = e.response.text[:300] if e.response is not None else "no response body"
        print(f"  [ERROR] HTTP {e.response.status_code if e.response is not None else 'N/A'}: {body}")
        results.append({"index": idx, "title": scene["title"], "status": "error", "path": None})
    except Exception as e:
        print(f"  [ERROR] {type(e).__name__}: {e}")
        results.append({"index": idx, "title": scene["title"], "status": "error", "path": None})

    time.sleep(2)

print("\n[DONE] Summary:")
for r in results:
    path_label = r["path"] if r["path"] else "N/A"
    print(f"  Scene {r['index']} ({r['title']}): [{r['status'].upper()}] {path_label}")
```

### Step 7：填入場景資料並執行

從對話中解析分鏡腳本，填入 `storyboard_scenes` 陣列，並依使用者選擇設定 `RESOLUTION`（`"480p"` 或 `"720p"`，未指定則保持預設 `"480p"`）後，將完整程式碼儲存為**共用 storyboard 資料夾**內的 `storyboard_video.py`，再執行：

```bash
python storyboard/storyboard_video.py
```

> 請將 `storyboard/` 替換為本次實際使用的資料夾名稱（例如 `storyboard001/`）。

---

## 輸出檔案

| 檔案 | 說明 |
|---|---|
| `storyboard_video_01.mp4` | 分鏡 1 影片 |
| `storyboard_video_02.mp4` | 分鏡 2 影片 |
| … | … |
| `storyboard_video_N.mp4` | 分鏡 N 影片 |
| `storyboard_video.py` | 保留的程式碼，不自動刪除 |

---

## 輸出原則

1. **無 Emoji 日誌與註解**：Python 程式碼中的所有 `print()` 輸出與註解，一律不得含有 Emoji
2. **繁體中文強制**：prompt 中明確要求畫面文字使用繁體中文，禁用簡體字
3. **圖卡優先**：有 `storyboard_panel_XX.jpg` 時，以 `"image": {"url": "data:image/jpeg;base64,..."}` 頂層欄位送出；422 錯誤時自動退為純文字備援
4. **必填 model**：每次請求都帶 `"model": "grok-imagine-video"`
5. **保留程式碼**：`storyboard_video.py` 執行後不刪除
6. **逐鏡儲存**：每段影片獨立為 MP4 檔案，方便後製
7. **共用資料夾**：所有輸入圖卡與輸出影片、程式碼均存放於共用 storyboard 資料夾（`storyboard`／`storyboardNNN`）內

---

## 完成後

回報生成結果，並詢問使用者：
> 「影片是否符合需求？若需要調整影片長度、重新生成特定分鏡，或修改畫風描述，請告訴我。」

若生成時發現共用資料夾內沒有 `storyboard_style.json`（本技能是以局部主題判斷生成 STYLE_PREFIX），
額外提醒使用者：
> 「這次影片是用本技能內建的主題風格判斷生成的。若圖卡與影片風格看起來不一致，
> 建議先執行 storyboard-style 技能建立共用設計系統，重新生成後圖卡與影片會套用同一套風格。」
