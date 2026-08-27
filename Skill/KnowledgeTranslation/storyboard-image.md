---
name: storyboard-image
description: |
  依據 storyboard-design.md 輸出的分鏡腳本，呼叫圖片生成 API 自動生成分鏡圖卡。
  預設使用 gemini-3.1-flash-image（Google Gemini），亦支援 gpt-image-2（OpenAI）作為備選。
  每張圖卡含標題與對應畫面，畫風、色調、主角於全局一致。生成後以 Vision 模型驗證繁體中文，
  最終輸出個別分鏡 JPEG 與完整 N-Grid 拼接圖。

  當使用者說「生成分鏡圖」、「畫分鏡」、「產生圖卡」、「storyboard image」、「把分鏡腳本畫出來」，
  或在完成 storyboard-design 後要求視覺化呈現，務必使用此技能。
---

# 分鏡圖卡生成技能（Storyboard Image Generation Skill）

## 任務概述

解析上一步 storyboard-design 所產出的分鏡腳本，呼叫圖片生成 API 繪製每個分鏡的圖卡，
以 Vision 模型驗證文字正確性，最終輸出個別圖卡與拼接總覽圖。

**預設 API：gemini-3.1-flash-image**（Google Gemini）
**備選 API：gpt-image-2**（OpenAI）——當使用者指定或 Gemini API Key 不可用時自動切換。

---

## 輸出目錄規則（所有 storyboard 技能共用）

本技能產生的**所有檔案**（`generate_storyboard.py`、各分鏡 `.jpg`、Grid 拼接圖…）
都必須存放在與 storyboard-design 相同的**共用資料夾**內。

- 基底名稱為 `storyboard`；若已存在則以連號尾數區分：`storyboard001`、`storyboard002`、…
- 本技能屬於**下游技能**，沿用**最新**的既有資料夾（`create_new=False`）。
- 作法：先解析出共用資料夾，將 `generate_storyboard.py` 寫入該資料夾，
  腳本開頭 `os.chdir()` 至自身所在目錄，之後所有相對路徑（圖卡、Grid）都會落在該資料夾內。

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

若尚無任何 storyboard 資料夾（使用者未先執行 storyboard-design），
則以 `resolve_storyboard_dir(create_new=True)` 建立 `storyboard` 資料夾後繼續。

---

## 步驟流程

### Step 1：確認 API Key 與選擇引擎

依照以下優先順序決定使用的圖片生成引擎：

```
優先順序（由高到低）：
1. 使用者於對話中明確指定引擎（「用 Gemini」或「用 OpenAI」）
2. 環境變數 GOOGLE_API_KEY 已設定 → 使用 Gemini（預設）
3. 環境變數 OPENAI_API_KEY 已設定 → 退回 OpenAI
4. 皆未設定 → 詢問使用者提供任一 API Key
```

若需以 Python 確認環境變數：
```python
import os
google_key = os.environ.get("GOOGLE_API_KEY", "")
openai_key = os.environ.get("OPENAI_API_KEY", "")
if google_key:
    print("ENGINE=gemini")
elif openai_key.startswith("sk-"):
    print("ENGINE=openai")
else:
    print("KEY_MISSING")
```

若輸出 `KEY_MISSING`，請求使用者提供，並告知：
> 「請提供您的 API Key：
> - **Gemini（預設推薦）**：設定環境變數 `GOOGLE_API_KEY`
> - **OpenAI（備選）**：設定環境變數 `OPENAI_API_KEY`（格式：sk-...）」

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

**優先讀取共用設計系統：** 先檢查共用 storyboard 資料夾內是否存在
`storyboard_style.json`（由 storyboard-style 技能產生）。若存在，直接讀取其
`global_style_prompt`、`color_palette`、`typography` 作為本技能的全局風格，
**不再自行判斷風格**——這避免了本技能與 storyboard-video 各自決策而互相衝突
（例如本技能想走寫實 HUD、video 卻套用扁平插畫前綴）。若檔案不存在（使用者
未執行 storyboard-style），才依下列規則自行判斷，並可在完成後提醒使用者：
「建議先執行 storyboard-style 統一設計系統，讓圖卡與影片風格一致。」

**找不到 storyboard_style.json 時的備援邏輯：**

依據主題與第一個分鏡的描述，以繁體中文建立一段全局風格描述，確保所有分鏡圖卡視覺一致。

**重要：ART_STYLE 不可一律套用「flat 2D vector / 扁平插畫」風格。** 文字指令的權重高於參考圖片——只要 prompt 裡出現
`flat illustration`、`2D vector style`、`minimal shading` 之類的詞，即使同時附上真實場景照片作為 image+text
條件輸入，模型仍會把整個畫面重畫成扁平插畫／漫畫感，導致與參考照片的實際場景、光影、質感脫節。
應依「該分鏡是否附參考照片」分兩種模式決定 ART_STYLE：

| 模式 | 適用情境 | ART_STYLE 描述方向 |
|---|---|---|
| **半寫實 HUD 疊加**（分鏡有參考照片時） | 分鏡構圖中標註了實景參考照片路徑 | 保留參考照片的真實光影、材質、構圖與物件外觀；HUD 元素（偵測框、面板、標籤、光暈）以半透明科技疊加層方式加在真實影像之上，類似 AR 介面疊加於實景。**禁用** `flat`、`cartoon`、`2D vector`、`minimal shading` 等字眼 |
| **科技概念插畫**（分鏡無參考照片時） | 純概念／總覽畫面（如模組總覽、資料流程圖） | 可用科技感半寫實插畫或 3D 風格渲染呈現，但同樣避免拉向漫畫感的字眼；改用 `semi-realistic tech illustration`、`glossy 3D-style rendering` 等描述 |

**Style Prompt 結構（以英文撰寫，傳入 API）：**
```
[ART_STYLE] semi-realistic sci-fi HUD tech aesthetic (依上表判斷該分鏡模式，避免 flat/cartoon/2D vector 用詞)
[COLOR_PALETTE] consistent warm/cool palette (derived from topic — education: blue/orange; business: navy/gold; story: earthy tones)
[CHARACTER] same protagonist design across all frames if applicable
[TEXT_LANGUAGE] all visible text labels must be written in Traditional Chinese characters (繁體中文), never Simplified Chinese
[COMPOSITION] image-dominant layout, minimal text overlay, clear visual hierarchy
[QUALITY] high detail, 2K resolution, sharp edges
[REALISM] when reference photo(s) are attached: preserve the real scene's lighting, materials, and object appearance faithfully; HUD graphics are a semi-transparent overlay layer on top, not a full repaint into illustration/cartoon style
```

在生成每個分鏡的個別 Prompt 時，**前綴加入全局 Style Prompt**，再接續該分鏡的畫面構圖與重點動作描述（翻譯為英文）。
若該分鏡附有參考照片，另外附加一段指示：「以參考照片為場景、構圖與物件位置基礎，真實感優先，HUD 僅作為疊加層，不得將整個場景重繪成插畫」；
若該分鏡沒有參考照片，則附加「維持半寫實科技渲染，避免扁平插畫/漫畫感」的提示。

---

### Step 4：產出 Python 生成腳本

生成以下 Python 腳本，儲存至**共用 storyboard 資料夾**內（命名為 `generate_storyboard.py`），並執行之。
腳本開頭會 `os.chdir()` 至自身所在目錄，因此所有輸出檔案都會落在該資料夾內：

```python
import os
import sys
import re
import json
import base64
import time
from io import BytesIO
from pathlib import Path
from PIL import Image, ImageDraw, ImageFont
import math

# --- 切換至腳本自身所在目錄（即共用 storyboard 資料夾），確保輸出落在同一資料夾 ---
os.chdir(Path(__file__).resolve().parent)

# --- 設定：輸出直接落在共用資料夾（腳本所在目錄） ---
OUTPUT_DIR = Path(".")

# --- 引擎選擇：優先 Gemini，退回 OpenAI ---
GOOGLE_API_KEY = os.environ.get("GOOGLE_API_KEY", "")
OPENAI_API_KEY = os.environ.get("OPENAI_API_KEY", "")

if GOOGLE_API_KEY:
    ENGINE = "gemini"
    from google import genai
    from google.genai import types
    client = genai.Client(api_key=GOOGLE_API_KEY)
    print("[Engine] Using Gemini (gemini-3.1-flash-image)")
elif OPENAI_API_KEY:
    ENGINE = "openai"
    from openai import OpenAI
    client = OpenAI(api_key=OPENAI_API_KEY)
    print("[Engine] Using OpenAI (gpt-image-2)")
else:
    print("[Error] No API key found. Set GOOGLE_API_KEY or OPENAI_API_KEY.")
    sys.exit(1)

MAX_RETRY = 3

# --- Design system: load storyboard_style.json produced by the storyboard-style
# skill (single source of truth shared with storyboard-video). Falls back to the
# original hardcoded defaults when the file is absent (skill was skipped). ---
STYLE_SPEC_PATH = Path("storyboard_style.json")
STYLE_SPEC = None
if STYLE_SPEC_PATH.exists():
    try:
        STYLE_SPEC = json.loads(STYLE_SPEC_PATH.read_text(encoding="utf-8"))
        print(f"[Style] Loaded storyboard_style.json (family={STYLE_SPEC['art_style']['family']})")
    except Exception as e:
        print(f"  [WARN] Failed to load storyboard_style.json ({e}), using fallback style")
        STYLE_SPEC = None
else:
    print("  [WARN] storyboard_style.json not found. Run storyboard-style first for "
          "consistent image/video styling. Falling back to local defaults.")

if STYLE_SPEC:
    typo = STYLE_SPEC["typography"]
    TITLE_BAR_HEIGHT = typo.get("title_bar_height", 60)
    TITLE_FONT_SIZE = typo.get("title_font_size", 28)
    TITLE_BAR_BG_COLOR = tuple(
        int(typo.get("title_bar_bg_color", "#1E1E28").lstrip("#")[i:i + 2], 16)
        for i in (0, 2, 4)
    )
    TITLE_BAR_TEXT_COLOR = tuple(
        int(typo.get("title_bar_text_color", "#FFFFFF").lstrip("#")[i:i + 2], 16)
        for i in (0, 2, 4)
    )
    FONT_CANDIDATES = typo.get("title_font_candidates")
    GLOBAL_STYLE = STYLE_SPEC["global_style_prompt"]
    IS_HUD_FAMILY = STYLE_SPEC["art_style"]["family"] == "semi_realistic_hud"
else:
    TITLE_BAR_HEIGHT = 60
    TITLE_FONT_SIZE = 28
    TITLE_BAR_BG_COLOR = (30, 30, 40)
    TITLE_BAR_TEXT_COLOR = (255, 255, 255)
    FONT_CANDIDATES = None
    IS_HUD_FAMILY = True
    # NOTE: keep ART_STYLE neutral -- do NOT add "flat", "2D vector", "cartoon" or
    # "minimal shading" here. Those words outweigh any reference photo passed to the
    # model and will repaint the whole scene into flat illustration, even in
    # image+text (reference-conditioned) mode. See Step 3 for the rationale.
    GLOBAL_STYLE = (
        "Semi-realistic sci-fi HUD tech aesthetic, consistent color palette, "
        "image-dominant composition with minimal text overlay, clear visual hierarchy, "
        "all visible text labels in Traditional Chinese characters (繁體中文) only, "
        "never use Simplified Chinese, high detail, 2K resolution, sharp edges, "
        "16:9 aspect ratio."
    )

# Appended when a scene has reference photo(s): keeps the real scene's realism and
# treats HUD graphics as an overlay layer instead of letting the model repaint the
# whole frame into illustration/cartoon style. Only meaningful for the HUD family;
# non-HUD families (flat/corporate/narrative/data-viz) skip this because their
# base style already differs from "preserve the real photo verbatim".
REFERENCE_INSTRUCTION = (
    " Use the attached reference photo(s) as the layout, camera angle, environment "
    "and object-placement basis for this scene: preserve the real scene's lighting, "
    "materials and object appearance faithfully, and add HUD graphics (detection "
    "boxes, panels, labels, glows) as a semi-transparent overlay layer on top. Do "
    "NOT repaint the scene into flat illustration, cartoon, or 2D vector art."
) if IS_HUD_FAMILY else (
    " Use the attached reference photo(s) as the layout, camera angle and "
    "composition basis for this scene, reinterpreted in the project's art style "
    "described above."
)

# Appended when a scene has no reference photo (pure concept/overview frame).
CONCEPT_STYLE_NOTE = (
    " No reference photo for this scene: render as semi-realistic tech "
    "illustration or glossy 3D-style rendering; avoid flat 2D vector, cartoon, or "
    "minimal-shading looks."
) if IS_HUD_FAMILY else (
    " No reference photo for this scene: render as a pure concept/overview frame "
    "in the project's art style described above."
)


def build_prompt(scene):
    text = (
        f"{GLOBAL_STYLE} "
        f"Scene: {scene['composition']} "
        f"Action: {scene['action']} "
        f"Style notes: image-centric storyboard panel, no narration text, "
        f"if any label or sign appears in the image use Traditional Chinese only."
    )
    if scene.get("references"):
        text += REFERENCE_INSTRUCTION
    else:
        text += CONCEPT_STYLE_NOTE
    return text


def extract_inline_image_bytes(response):
    for cand in getattr(response, "candidates", []) or []:
        content = getattr(cand, "content", None)
        if not content:
            continue
        for part in getattr(content, "parts", []) or []:
            inline = getattr(part, "inline_data", None)
            if inline is not None and getattr(inline, "data", None):
                data = inline.data
                if isinstance(data, str):
                    return base64.b64decode(data)
                return data
    return None


# --- 分鏡資料（由技能自動填入） ---
# "references": 該分鏡構圖中標註的實景參考照片絕對路徑列表；無參考照片則給 []
TOPIC = "{TOPIC}"
STORYBOARDS = {STORYBOARDS_JSON}

# --- 字型設定（Windows 繁體中文字型） ---
def get_font(size):
    # Uses the design system's font candidates when storyboard_style.json was
    # loaded above; otherwise falls back to the same hardcoded list as before.
    candidates = FONT_CANDIDATES or [
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


# --- 生成單一分鏡（依引擎分流，支援參考照片 image+text 條件生成） ---
# NOTE: the google-genai SDK has no `client.images.generate()` for the
# gemini-*-image models (that call raises AttributeError). Image generation goes
# through `client.models.generate_content()` with response_modalities=["IMAGE"],
# exactly like the Vision verification call below -- this also doubles as the
# entry point for passing reference photos (image+text conditioning).
def generate_frame(index, title, scene, retry=0):
    prompt = build_prompt(scene)
    ref_paths = [Path(p) for p in scene.get("references", [])]
    ref_paths = [p for p in ref_paths if p.exists()]
    for missing in scene.get("references", []):
        if not Path(missing).exists():
            print(f"  [WARN] Reference image not found, skipping: {missing}")

    print(f"[Frame {index}] Generating (attempt {retry + 1}/{MAX_RETRY}): {title}")
    if ref_paths:
        print(f"  [INFO] image+text mode, {len(ref_paths)} reference image(s)")
    else:
        print(f"  [INFO] text-only mode (no reference image)")

    if ENGINE == "gemini":
        gen_config = types.GenerateContentConfig(
            response_modalities=["IMAGE"],
            image_config=types.ImageConfig(aspect_ratio="16:9", image_size="2K"),
        )
        if ref_paths:
            contents = [prompt] + [Image.open(p).convert("RGB") for p in ref_paths]
            try:
                response = client.models.generate_content(
                    model="gemini-3.1-flash-image", contents=contents, config=gen_config
                )
                image_bytes = extract_inline_image_bytes(response)
                if image_bytes is None:
                    raise RuntimeError("No inline image data in response")
            except Exception as e:
                print(f"  [RETRY] Reference-image generation failed ({e}), retrying text-only")
                response = client.models.generate_content(
                    model="gemini-3.1-flash-image", contents=[prompt], config=gen_config
                )
                image_bytes = extract_inline_image_bytes(response)
                if image_bytes is None:
                    raise RuntimeError("No inline image data in text-only fallback response")
        else:
            response = client.models.generate_content(
                model="gemini-3.1-flash-image", contents=[prompt], config=gen_config
            )
            image_bytes = extract_inline_image_bytes(response)
            if image_bytes is None:
                raise RuntimeError("No inline image data in response")

    else:  # openai
        if ref_paths:
            files = [open(p, "rb") for p in ref_paths]
            try:
                response = client.images.edit(
                    model="gpt-image-2",
                    image=files,
                    prompt=prompt,
                    size="2048x1152",
                    quality="high",
                    n=1,
                )
                image_bytes = base64.b64decode(response.data[0].b64_json)
            finally:
                for f in files:
                    f.close()
        else:
            response = client.images.generate(
                model="gpt-image-2",
                prompt=prompt,
                size="2048x1152",
                quality="high",
                output_format="jpeg",
                n=1,
            )
            image_bytes = base64.b64decode(response.data[0].b64_json)

    img = Image.open(BytesIO(image_bytes)).convert("RGB")
    return img


# --- Vision 驗證繁體中文（依引擎分流） ---
def verify_traditional_chinese(img):
    buffer = BytesIO()
    img.save(buffer, format="JPEG", quality=90)
    b64 = base64.b64encode(buffer.getvalue()).decode()

    verify_prompt = (
        "Please examine this storyboard image carefully. "
        "Check if any visible text contains Simplified Chinese characters "
        "(simplified forms like: 爱 国 说 时 来 对 etc.) or garbled/corrupted text. "
        "Reply with ONLY one word: PASS if all text is Traditional Chinese or no text present, "
        "FAIL if Simplified Chinese or garbled text is found."
    )

    if ENGINE == "gemini":
        # NOTE: "gemini-3.1-flash" is not a valid model id on the current API
        # (404 NOT_FOUND). Use "gemini-flash-latest", which always resolves to a
        # current flash-tier text+vision model.
        response = client.models.generate_content(
            model="gemini-flash-latest",
            contents=[
                verify_prompt,
                {
                    "inline_data": {
                        "mime_type": "image/jpeg",
                        "data": b64,
                    }
                },
            ],
        )
        verdict = response.text.strip().upper()

    else:  # openai
        check_response = client.chat.completions.create(
            model="gpt-4o",
            messages=[
                {
                    "role": "user",
                    "content": [
                        {"type": "text", "text": verify_prompt},
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
    new_img = Image.new("RGB", (w, h + TITLE_BAR_HEIGHT), color=TITLE_BAR_BG_COLOR)
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
    draw.text((x, y), label, fill=TITLE_BAR_TEXT_COLOR, font=font)
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

        img = None
        candidate = None
        for attempt in range(MAX_RETRY):
            candidate = generate_frame(idx, title, sb, retry=attempt)
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

        # Clean panel (no title bar): this is the file downstream storyboard-video
        # consumes as the image-to-video reference. A title bar must NOT be baked
        # into it, otherwise the generated video would inherit the text banner.
        panel_path = OUTPUT_DIR / f"storyboard_panel_{idx:02d}.jpg"
        img.save(panel_path, format="JPEG", quality=92)
        print(f"[Frame {idx}] Saved clean panel: {panel_path}")

        # Titled version (for human viewing and the Grid overview).
        img_titled = add_title_bar(img, idx, title)
        framed_images.append(img_titled)

        out_path = OUTPUT_DIR / f"frame_{idx:02d}_{title}.jpg"
        img_titled.save(out_path, format="JPEG", quality=92)
        print(f"[Frame {idx}] Saved titled frame: {out_path}")
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
    "action": "（重點動作英文翻譯）",
    "references": []
  },
  {
    "index": 3,
    "title": "場域偵測",
    "composition": "（畫面構圖英文翻譯）",
    "action": "（重點動作英文翻譯）",
    "references": [
      "C:/path/to/reference_photo_1.jpg",
      "C:/path/to/reference_photo_2.jpg"
    ]
  },
  ...
]
```

> `composition` 與 `action` 欄位必須翻譯為英文再填入，因為圖片生成 API 對英文提示詞的理解效果最佳。
> `references`：若該分鏡的「畫面構圖」文字中標註了實景參考照片路徑（如「參考 "xxx.jpg"」），將這些路徑（絕對路徑）填入此陣列；
> 沒有參考照片的分鏡填 `[]`。有 `references` 的分鏡會以 image+text 條件生成（保留參考照片的真實場景與構圖），
> 沒有的分鏡則以純文字 prompt 生成——兩者的 ART_STYLE 差異見 Step 3。

執行腳本（`generate_storyboard.py` 位於共用 storyboard 資料夾內，例如 `storyboard/`、`storyboard001/`）：

**Gemini 引擎（預設）：**
```bash
pip install google-genai pillow -q
python storyboard/generate_storyboard.py
```

**OpenAI 引擎（備選）：**
```bash
pip install openai pillow -q
python storyboard/generate_storyboard.py
```

> 請將上方路徑中的 `storyboard/` 替換為本次實際使用的資料夾名稱。

---

### Step 6：輸出結果說明

執行成功後，共用 storyboard 資料夾（例如 `storyboard/`、`storyboard001/`）將包含：

| 檔案 | 說明 |
|---|---|
| `storyboard_panel_01.jpg` | 第 1 個分鏡的**乾淨圖卡**（無標題列，供 storyboard-video 作為 image-to-video 參考圖） |
| `storyboard_panel_02.jpg` | 第 2 個分鏡的乾淨圖卡 |
| … | … |
| `frame_01_標題.jpg` | 第 1 個分鏡圖卡（**含標題列**，供人工檢視與 Grid） |
| `frame_02_標題.jpg` | 第 2 個分鏡圖卡（含標題列） |
| … | … |
| `storyboard_grid_Nframes.jpg` | N 個分鏡拼接的完整總覽圖 |

Grid 排列規則：**由上到下、由左至右**，每列 3 張（3、6、9、12、15 張均適配）。

> **檔名對接約定：** 每個分鏡會輸出**兩個**檔案 ——
> `storyboard_panel_XX.jpg`（無標題列的乾淨圖，storyboard-video 的輸入）與
> `frame_XX_標題.jpg`（含標題列，供人工檢視／Grid）。
> storyboard-video 只讀取 `storyboard_panel_XX.jpg`，故編號 `XX` 必須與分鏡 index 一致（兩位補零）。

---

## 引擎比較

| 項目 | Gemini（預設） | OpenAI（備選） |
|---|---|---|
| 模型 | `gemini-3.1-flash-image` | `gpt-image-2` |
| 驗證模型 | `gemini-flash-latest`（Vision） | `gpt-4o`（Vision） |
| API Key 環境變數 | `GOOGLE_API_KEY` | `OPENAI_API_KEY` |
| Python 套件 | `google-genai` | `openai` |
| 輸出解析度 | 2K（16:9） | 2K（2048×1152，16:9） |
| 輸出格式 | JPEG | JPEG |
| 成本 | 較低 | 較高 |

> 使用者可於對話中指定「用 OpenAI」或「用 Gemini」切換引擎。腳本內以環境變數自動判斷，
> 優先使用 `GOOGLE_API_KEY`（Gemini），若無則退回 `OPENAI_API_KEY`（OpenAI）。

---

## 品質保證規則

| 規則 | 說明 |
|---|---|
| 設計系統優先 | 優先讀取共用資料夾內的 `storyboard_style.json`（storyboard-style 技能輸出），與 storyboard-video 共用同一份 GLOBAL_STYLE／色票／標題樣式；找不到才退回本技能的局部判斷 |
| 全局風格一致 | 所有分鏡使用相同 GLOBAL_STYLE 前綴，確保畫風、色調統一 |
| 參考照片保真 | 有 `references` 的分鏡以 image+text 條件生成；當設計系統為 `semi_realistic_hud` 家族時，prompt 禁用 `flat`/`cartoon`/`2D vector` 等字眼，避免真實場景被重畫成扁平插畫，確保與參考照片的場景/構圖/物件一致 |
| 標題不遮圖 | 標題列附加於圖片上方，不覆蓋畫面內容 |
| 旁白不入圖 | Prompt 明確要求 no narration text，旁白僅存於腳本文字 |
| 繁體中文驗證 | 每張圖由 Vision 模型判斷，含簡體字或亂碼則重新生成，至多 3 次 |
| 圖以圖為主 | Prompt 強調 image-dominant，僅允許畫面內自然出現的標籤/標示以繁體中文呈現 |
| 無 Emoji 日誌 | Python 腳本所有 print 輸出不含 Emoji 字元 |
| JPEG 輸出 | 所有圖片以 JPEG 格式儲存，quality=92 |
| 檔名對接 | 每個分鏡輸出乾淨圖 `storyboard_panel_XX.jpg`（供 storyboard-video）與含標題圖 `frame_XX_標題.jpg`（供檢視／Grid）；panel 不含標題列 |

---

## 錯誤處理

| 錯誤情境 | 處理方式 |
|---|---|
| API Key 無效 | 停止並提示使用者重新提供 |
| 分鏡腳本不存在 | 提示先執行 storyboard-design |
| 圖片生成 API 失敗 | 印出錯誤訊息，跳過該分鏡，繼續處理其餘分鏡 |
| 驗證 3 次仍失敗 | 使用第 3 次生成的圖，並在終端機警告 |
| 字型找不到 | 退回 PIL 預設字型，不中斷流程 |
| Pillow 未安裝 | 腳本開頭自動提示安裝指令 |
| `google-genai` 未安裝 | 提示 `pip install google-genai` |

---

## 完成後

輸出完整路徑並詢問使用者：
> 「分鏡圖卡已生成完畢（使用 {引擎名稱}），個別圖卡與 Grid 總覽圖均儲存於共用 storyboard 資料夾（例如 storyboard/、storyboard001/）。是否需要調整畫風、色調、或重新生成特定分鏡？」

若生成時發現共用資料夾內沒有 `storyboard_style.json`（本技能是以局部判斷邏輯生成），
額外提醒使用者：
> 「這次圖卡是用本技能內建的風格判斷生成的。若接下來要生成 storyboard-video 影片，
> 建議先執行 storyboard-style 技能建立共用設計系統，避免圖卡與影片風格不一致。」

保留 `generate_storyboard.py` 於共用 storyboard 資料夾內，方便使用者日後修改參數後重新執行。
