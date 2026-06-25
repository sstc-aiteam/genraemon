---
name: storyboard-image
description: |
  依據 storyboard-design.md 輸出的分鏡腳本，呼叫 gpt-image-2 API 自動生成分鏡圖卡。
  每張圖卡含標題與對應畫面，畫風、色調、主角於全局一致。生成後以 GPT-4o Vision 驗證繁體中文，
  最終輸出個別分鏡 JPEG 與完整 N-Grid 拼接圖。

  當使用者說「生成分鏡圖」、「畫分鏡」、「產生圖卡」、「storyboard image」、「把分鏡腳本畫出來」，
  或在完成 storyboard-design 後要求視覺化呈現，務必使用此技能。
---

# 分鏡圖卡生成技能（Storyboard Image Generation Skill）

## 任務概述

解析上一步 storyboard-design 所產出的分鏡腳本，呼叫 **gpt-image-2** 繪製每個分鏡的圖卡，
以 GPT-4o Vision 驗證文字正確性，最終輸出個別圖卡與拼接總覽圖。

---

## 步驟流程

### Step 1：確認 OpenAI API Key

執行以下檢查，任一條件符合即可繼續，否則停止並提示使用者：

```
優先順序（由高到低）：
1. 對話中使用者是否已明確提供 API Key（格式：sk-...）
2. 環境變數 OPENAI_API_KEY 是否已設定（執行 Python 確認）
3. 詢問使用者提供 API Key
```

若需以 Python 確認環境變數：
```python
import os
key = os.environ.get("OPENAI_API_KEY", "")
print("KEY_FOUND" if key.startswith("sk-") else "KEY_MISSING")
```

若輸出 `KEY_MISSING`，請求使用者提供，並告知：
> 「請提供您的 OpenAI API Key（格式：sk-...），或在環境變數 OPENAI_API_KEY 中設定。」

---

### Step 2：解析分鏡腳本

從對話中找到 storyboard-design 輸出的分鏡腳本，提取以下欄位：

- **主題**（`**主題：**` 後的文字）
- **分鏡總數** N
- 每個分鏡的：
  - 編號與標題（`## 分鏡 N｜標題`）
  - 畫面構圖（`**🎨 畫面構圖**` 後的段落）
  - 重點動作（`**🎬 重點動作**` 後的段落）

若對話中找不到分鏡腳本，提示使用者：
> 「請先執行分鏡腳本設計（storyboard-design），再生成圖卡。」

---

### Step 3：建立全局 Style Prompt

依據主題與第一個分鏡的描述，以繁體中文建立一段全局風格描述，確保所有分鏡圖卡視覺一致。

**Style Prompt 結構（以英文撰寫，傳入 API）：**
```
[ART_STYLE] flat illustration, clean lines, 2D vector style, minimal shading
[COLOR_PALETTE] consistent warm/cool palette (derived from topic — education: blue/orange; business: navy/gold; story: earthy tones)
[CHARACTER] same protagonist design across all frames if applicable
[TEXT_LANGUAGE] all visible text labels must be written in Traditional Chinese characters (繁體中文), never Simplified Chinese
[COMPOSITION] image-dominant layout, minimal text overlay, clear visual hierarchy
[QUALITY] high detail, HD resolution, sharp edges
```

在生成每個分鏡的個別 Prompt 時，**前綴加入全局 Style Prompt**，再接續該分鏡的畫面構圖與重點動作描述（翻譯為英文）。

---

### Step 4：產出 Python 生成腳本

生成以下 Python 腳本（儲存至當前工作目錄，命名為 `generate_storyboard.py`），並執行之：

```python
import os
import sys
import re
import json
import base64
import time
from io import BytesIO
from pathlib import Path
from openai import OpenAI
from PIL import Image, ImageDraw, ImageFont
import math

# --- 設定 ---
OUTPUT_DIR = Path("storyboard_output")
OUTPUT_DIR.mkdir(exist_ok=True)

API_KEY = os.environ.get("OPENAI_API_KEY", "")
client = OpenAI(api_key=API_KEY)

IMAGE_SIZE = "1536x1024"
IMAGE_QUALITY = "high"
IMAGE_FORMAT = "jpeg"
TITLE_BAR_HEIGHT = 60
TITLE_FONT_SIZE = 28
MAX_RETRY = 3

GLOBAL_STYLE = (
    "flat illustration, clean 2D vector style, minimal shading, "
    "consistent color palette, image-dominant composition with minimal text overlay, "
    "all visible text labels in Traditional Chinese characters (繁體中文) only, "
    "never use Simplified Chinese, high detail, HD resolution, sharp edges."
)

# --- 分鏡資料（由技能自動填入） ---
TOPIC = "{TOPIC}"
STORYBOARDS = {STORYBOARDS_JSON}

# --- 字型設定（Windows 繁體中文字型） ---
def get_font(size):
    candidates = [
        "C:/Windows/Fonts/msjh.ttc",
        "C:/Windows/Fonts/mingliu.ttc",
        "C:/Windows/Fonts/kaiu.ttf",
        "/usr/share/fonts/truetype/noto/NotoSansCJK-Regular.ttc",
        "/System/Library/Fonts/PingFang.ttc",
    ]
    for path in candidates:
        if os.path.exists(path):
            try:
                return ImageFont.truetype(path, size)
            except Exception:
                continue
    return ImageFont.load_default()


# --- 生成單一分鏡 ---
def generate_frame(index, title, composition, action, retry=0):
    prompt = (
        f"{GLOBAL_STYLE} "
        f"Scene: {composition} "
        f"Action: {action} "
        f"Style notes: image-centric storyboard panel, no narration text, "
        f"if any label or sign appears in the image use Traditional Chinese only."
    )

    print(f"[Frame {index}] Generating (attempt {retry + 1}/{MAX_RETRY}): {title}")

    response = client.images.generate(
        model="gpt-image-2",
        prompt=prompt,
        size=IMAGE_SIZE,
        quality=IMAGE_QUALITY,
        output_format=IMAGE_FORMAT,
        n=1,
    )

    image_bytes = base64.b64decode(response.data[0].b64_json)
    img = Image.open(BytesIO(image_bytes)).convert("RGB")
    return img


# --- GPT-4o Vision 驗證繁體中文 ---
def verify_traditional_chinese(img):
    buffer = BytesIO()
    img.save(buffer, format="JPEG", quality=90)
    b64 = base64.b64encode(buffer.getvalue()).decode()

    check_response = client.chat.completions.create(
        model="gpt-4o",
        messages=[
            {
                "role": "user",
                "content": [
                    {
                        "type": "text",
                        "text": (
                            "Please examine this storyboard image carefully. "
                            "Check if any visible text contains Simplified Chinese characters "
                            "(simplified forms like: 爱 国 说 时 来 对 etc.) or garbled/corrupted text. "
                            "Reply with ONLY one word: PASS if all text is Traditional Chinese or no text present, "
                            "FAIL if Simplified Chinese or garbled text is found."
                        ),
                    },
                    {
                        "type": "image_url",
                        "image_url": {"url": f"data:image/jpeg;base64,{b64}"},
                    },
                ],
            }
        ],
        max_tokens=10,
    )

    verdict = check_response.choices[0].message.content.strip().upper()
    return verdict == "PASS"


# --- 加入標題列 ---
def add_title_bar(img, frame_number, title):
    w, h = img.size
    new_img = Image.new("RGB", (w, h + TITLE_BAR_HEIGHT), color=(30, 30, 40))
    new_img.paste(img, (0, TITLE_BAR_HEIGHT))

    draw = ImageDraw.Draw(new_img)
    font = get_font(TITLE_FONT_SIZE)
    label = f"分鏡 {frame_number}｜{title}"

    try:
        bbox = draw.textbbox((0, 0), label, font=font)
        text_w = bbox[2] - bbox[0]
        text_h = bbox[3] - bbox[1]
    except AttributeError:
        text_w, text_h = draw.textsize(label, font=font)

    x = (w - text_w) // 2
    y = (TITLE_BAR_HEIGHT - text_h) // 2
    draw.text((x, y), label, fill=(255, 255, 255), font=font)
    return new_img


# --- 拼接 N-Grid ---
def build_grid(images, cols=3):
    rows = math.ceil(len(images) / cols)
    cell_w, cell_h = images[0].size
    grid = Image.new("RGB", (cell_w * cols, cell_h * rows), color=(20, 20, 30))

    for i, img in enumerate(images):
        row = i // cols
        col = i % cols
        grid.paste(img, (col * cell_w, row * cell_h))

    return grid


# --- 主流程 ---
def main():
    framed_images = []
    total = len(STORYBOARDS)

    for sb in STORYBOARDS:
        idx = sb["index"]
        title = sb["title"]
        composition = sb["composition"]
        action = sb["action"]

        img = None
        for attempt in range(MAX_RETRY):
            candidate = generate_frame(idx, title, composition, action, retry=attempt)
            print(f"[Frame {idx}] Verifying Traditional Chinese text...")
            if verify_traditional_chinese(candidate):
                print(f"[Frame {idx}] Verification PASSED.")
                img = candidate
                break
            else:
                print(f"[Frame {idx}] Verification FAILED (attempt {attempt + 1}). Retrying...")

        if img is None:
            print(f"[Frame {idx}] Max retries reached. Using last generated image.")
            img = candidate

        img_titled = add_title_bar(img, idx, title)
        framed_images.append(img_titled)

        out_path = OUTPUT_DIR / f"frame_{idx:02d}_{title}.jpg"
        img_titled.save(out_path, format="JPEG", quality=92)
        print(f"[Frame {idx}] Saved: {out_path}")
        time.sleep(1)

    # 拼接 Grid
    cols = 3
    grid_img = build_grid(framed_images, cols=cols)
    grid_path = OUTPUT_DIR / f"storyboard_grid_{total}frames.jpg"
    grid_img.save(grid_path, format="JPEG", quality=92)
    print(f"[Grid] Saved: {grid_path}")
    print(f"[Done] All {total} frames generated. Output directory: {OUTPUT_DIR.resolve()}")


if __name__ == "__main__":
    main()
```

---

### Step 5：填入分鏡資料並執行

在執行前，將腳本中的佔位符替換為實際值：

**替換 `{TOPIC}`**：填入分鏡腳本的主題文字。

**替換 `{STORYBOARDS_JSON}`**：將解析出的分鏡腳本轉為 JSON 陣列，結構如下：
```json
[
  {
    "index": 1,
    "title": "開場引導",
    "composition": "（畫面構圖英文翻譯）",
    "action": "（重點動作英文翻譯）"
  },
  ...
]
```

> `composition` 與 `action` 欄位必須翻譯為英文再填入，因為 gpt-image-2 對英文提示詞的理解效果最佳。

執行腳本：
```bash
pip install openai pillow -q
python generate_storyboard.py
```

---

### Step 6：輸出結果說明

執行成功後，`storyboard_output/` 目錄將包含：

| 檔案 | 說明 |
|---|---|
| `frame_01_標題.jpg` | 第 1 個分鏡圖卡（含標題列） |
| `frame_02_標題.jpg` | 第 2 個分鏡圖卡（含標題列） |
| … | … |
| `storyboard_grid_Nframes.jpg` | N 個分鏡拼接的完整總覽圖 |

Grid 排列規則：**由上到下、由左至右**，每列 3 張（3、6、9、12、15 張均適配）。

---

## 品質保證規則

| 規則 | 說明 |
|---|---|
| 全局風格一致 | 所有分鏡使用相同 GLOBAL_STYLE 前綴，確保畫風、色調統一 |
| 標題不遮圖 | 標題列附加於圖片上方，不覆蓋畫面內容 |
| 旁白不入圖 | Prompt 明確要求 no narration text，旁白僅存於腳本文字 |
| 繁體中文驗證 | 每張圖由 GPT-4o Vision 判斷，含簡體字或亂碼則重新生成，至多 3 次 |
| 圖以圖為主 | Prompt 強調 image-dominant，僅允許畫面內自然出現的標籤/標示以繁體中文呈現 |
| 無 Emoji 日誌 | Python 腳本所有 print 輸出不含 Emoji 字元 |
| JPEG 輸出 | 所有圖片以 JPEG 格式儲存，quality=92，解析度 1536×1024（HD） |

---

## 錯誤處理

| 錯誤情境 | 處理方式 |
|---|---|
| API Key 無效 | 停止並提示使用者重新提供 |
| 分鏡腳本不存在 | 提示先執行 storyboard-design |
| gpt-image-2 API 失敗 | 印出錯誤訊息，跳過該分鏡，繼續處理其餘分鏡 |
| 驗證 3 次仍失敗 | 使用第 3 次生成的圖，並在終端機警告 |
| 字型找不到 | 退回 PIL 預設字型，不中斷流程 |
| Pillow 未安裝 | 腳本開頭自動提示安裝指令 |

---

## 完成後

輸出完整路徑並詢問使用者：
> 「分鏡圖卡已生成完畢，個別圖卡與 Grid 總覽圖均儲存於 storyboard_output/ 目錄。是否需要調整畫風、色調、或重新生成特定分鏡？」

保留 `generate_storyboard.py` 於工作目錄，方便使用者日後修改參數後重新執行。
