---
name: storyboard-style
description: |
  在 storyboard-design 產出分鏡腳本之後、storyboard-image／storyboard-video 生成圖卡與影片之前執行，
  分析分鏡腳本的主題與是否含實景參考照片，決定「唯一一份」全案設計系統（美術風格家族、色票、
  標題列字型與樣式），輸出為共用資料夾內的 storyboard_style.json，供 storyboard-image 與
  storyboard-video 兩邊共同讀取，確保圖卡與影片視覺一致，不再各自猜測風格互相衝突。

  當使用者提到「設計系統」、「design system」、「統一風格」、「統一色票」、「統一版面」、
  「圖卡風格不一致」、「圖跟影片風格對不上」、「標題字型」、「vi」、「視覺規範」，
  或在 storyboard-design 完成後、要生成圖卡/影片前主動詢問是否要統一風格時，務必使用此技能。
  即使使用者只說「先定風格」或「統一一下畫風」也應觸發。
---

# 分鏡設計系統技能（Storyboard Design System Skill）

## 任務概述

分析 storyboard-design 產出的分鏡腳本，做出**唯一一次**的視覺設計決策
（美術風格家族、色票、標題列字型／樣式），輸出成單一規格檔
`storyboard_style.json`，存入共用 storyboard 資料夾。

storyboard-image 與 storyboard-video 兩個下游技能都會讀取這份規格檔，
不再各自依據主題重新判斷風格。這解決了原本兩邊各自決策時可能產生的衝突
（例如 storyboard-image 預設走「semi-realistic HUD 寫實疊加」，
storyboard-video 預設走「flat 2D vector 扁平插畫」——兩者是相反的美術方向，
若圖卡用寫實風生成、影片卻套用扁平插畫前綴詞，會把圖卡的寫實感整個重畫掉）。

---

## 輸出目錄規則（所有 storyboard 技能共用）

本技能屬於**下游技能**，沿用 storyboard-design 建立的**最新**既有資料夾
（`create_new=False`），將 `storyboard_style.json` 存於該資料夾內。

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

## 步驟流程

### Step 1：讀取分鏡腳本

從對話中讀取 storyboard-design 輸出的 `storyboard_script.md`，取得：
- 主題文字
- 每個分鏡的畫面構圖、重點動作
- 是否有任何分鏡在畫面構圖中標註了實景參考照片路徑

若找不到分鏡腳本，提示：
> 「請先執行分鏡腳本設計（storyboard-design），再建立設計系統。」

---

### Step 2：決定 art_style（美術風格家族）—— 唯一一次決策

**第一層判斷（優先於主題分類，因為真實照片的保真需求會蓋過主題偏好）：**

檢查整份分鏡腳本中，**是否有任一分鏡**標註了實景參考照片。

| 情況 | art_style 家族 | 說明 |
|---|---|---|
| **有任一分鏡含參考照片** | `semi_realistic_hud` | 全案鎖定「半寫實 HUD 疊加」大方向，不可使用 `flat`／`cartoon`／`2D vector`／`minimal shading` 等字眼，確保有照片的分鏡能保留真實光影與構圖 |
| **完全沒有參考照片** | 依 Step 2b 主題分類 | 沒有真實照片需要保真，可自由選擇扁平／插畫／資訊圖表等風格家族 |

**Step 2b（僅當無參考照片時）：依主題選擇風格家族**

| 主題類型 | art_style 家族 | 風格描述（英文，供 image/video 共用） |
|---|---|---|
| 教育／科普 | `flat_infographic` | `flat design infographic animation, clean background, simple geometric shapes, 2D vector art style` |
| 商業／行銷 | `corporate_flat` | `corporate flat illustration, professional icons, clean character design` |
| 歷史／敘事 | `semi_realistic_narrative` | `semi-realistic illustration, sketch-like linework, clear focal point, warm muted tones` |
| 技術／數據 | `data_viz_dark` | `data visualization aesthetic, dark background, glowing data elements, minimal UI charts` |
| 社會／人文 | `semi_realistic_narrative` | 同歷史／敘事，強調人物情感與溫暖基調 |

> 一旦決定，**全案僅用這一個 art_style**。storyboard-image 生成無參考照片分鏡時、
> storyboard-video 生成所有分鏡時，都套用同一段描述文字，不再各自判斷。
> 有參考照片的分鏡，image 端仍會依 storyboard-image.md 原有邏輯疊加
> 「保留真實場景、HUD 為疊加層」的個別指示，但底層美術家族與此處一致，不會互相矛盾。

---

### Step 3：決定 color_palette（色票）—— 具體 hex 色碼

依 Step 2 決定的 art_style 家族，選擇對應色票（可依使用者指定偏好色系調整，以下為預設）：

| art_style 家族 | primary | secondary | accent | background | text |
|---|---|---|---|---|---|
| `semi_realistic_hud` | `#00E5FF`（HUD 青） | `#FF6B35`（警示橘） | `#39FF14`（偵測綠） | 保留原始照片（不覆蓋） | `#FFFFFF` |
| `flat_infographic` | `#2D9CDB`（藍） | `#27AE60`（綠） | `#F2C94C`（黃） | `#FFFFFF` | `#1A1A2E` |
| `corporate_flat` | `#EF8354`（橘） | `#2A9D8F`（藏青綠） | `#F4A300` | `#FFFFFF` | `#22223B` |
| `semi_realistic_narrative` | `#B08968`（暖棕） | `#6B4226` | `#D4A373` | `#F5EBDD`（暖米） | `#3A2E27` |
| `data_viz_dark` | `#00D9FF`（發光青） | `#7B61FF`（紫） | `#39FF14` | `#050A18`（深藍黑） | `#E6F1FF` |

> 這份色票同時給 image 生成 prompt（描述＋hex 並列，如「primary blue (#2D9CDB)」）
> 與人工檢視／後製時參考使用。

---

### Step 4：決定 typography（標題字型與標題列樣式）

標題列（title bar）目前在 storyboard-image 的 Python 範本中是寫死的視覺元素，
本技能統一決定其樣式，讓下游可讀取而非寫死：

| 欄位 | 預設值 | 說明 |
|---|---|---|
| `title_font_candidates` | `["C:/Windows/Fonts/msjh.ttc", "C:/Windows/Fonts/mingliu.ttc", "C:/Windows/Fonts/kaiu.ttf", "/usr/share/fonts/truetype/noto/NotoSansCJK-Regular.ttc", "/System/Library/Fonts/PingFang.ttc"]` | 依序嘗試載入的繁體中文字型路徑 |
| `title_font_size` | `28` | 標題列文字大小（px） |
| `title_bar_height` | `60` | 標題列高度（px） |
| `title_bar_bg_color` | 依 art_style 家族：`semi_realistic_hud`/`data_viz_dark` 用 `#0A0E1A`；其餘用 `#1E1E28` | 標題列背景色（沿用深色底以確保白字可讀） |
| `title_bar_text_color` | `#FFFFFF` | 標題列文字顏色 |

---

### Step 5：組合 global_style_prompt

將 Step 2 的風格描述與 Step 3 的色票組合成一段可直接前綴給 image／video
生成 API 的英文文字：

```
[art_style 描述], primary color [primary hex], secondary color [secondary hex],
accent color [accent hex], consistent color palette across all scenes,
all visible text labels in Traditional Chinese characters (繁體中文) only,
never use Simplified Chinese, clear visual hierarchy, high detail.
```

若 art_style 為 `semi_realistic_hud`，額外附加：
```
Do NOT use flat illustration, cartoon, or 2D vector art style.
```

---

### Step 6：輸出 storyboard_style.json

將以上決策整理成 JSON，存入共用 storyboard 資料夾：

```json
{
  "theme_category": "education",
  "has_reference_photos": false,
  "art_style": {
    "family": "flat_infographic",
    "description": "flat design infographic animation, clean background, simple geometric shapes, 2D vector art style",
    "avoid": ["photorealistic", "3D render"]
  },
  "color_palette": {
    "primary": "#2D9CDB",
    "secondary": "#27AE60",
    "accent": "#F2C94C",
    "background": "#FFFFFF",
    "text": "#1A1A2E"
  },
  "typography": {
    "title_font_candidates": [
      "C:/Windows/Fonts/msjh.ttc",
      "C:/Windows/Fonts/mingliu.ttc",
      "C:/Windows/Fonts/kaiu.ttf",
      "/usr/share/fonts/truetype/noto/NotoSansCJK-Regular.ttc",
      "/System/Library/Fonts/PingFang.ttc"
    ],
    "title_font_size": 28,
    "title_bar_height": 60,
    "title_bar_bg_color": "#1E1E28",
    "title_bar_text_color": "#FFFFFF"
  },
  "global_style_prompt": "flat design infographic animation, clean background, simple geometric shapes, 2D vector art style, primary color #2D9CDB, secondary color #27AE60, accent color #F2C94C, consistent color palette across all scenes, all visible text labels in Traditional Chinese characters (繁體中文) only, never use Simplified Chinese, clear visual hierarchy, high detail."
}
```

儲存路徑：`storyboard/storyboard_style.json`（`storyboard/` 替換為實際共用資料夾名稱）。

```python
import json
from pathlib import Path

style_spec = { ... }  # 依上方 Step 2~5 決策組成
Path("storyboard_style.json").write_text(
    json.dumps(style_spec, ensure_ascii=False, indent=2), encoding="utf-8"
)
print("[OK] storyboard_style.json saved")
```

---

## 下游技能如何讀取

- **storyboard-image**：Step 3 建立 Style Prompt 時，若共用資料夾內存在
  `storyboard_style.json`，直接讀取其 `global_style_prompt`／`color_palette`／
  `typography` 使用，不再自行判斷風格；`add_title_bar()` 與 `get_font()`
  改用其中的 `title_bar_height`／`title_bar_bg_color`／`title_font_candidates`
  等值。若檔案不存在（使用者跳過本技能），才退回 storyboard-image.md 原有的
  局部判斷邏輯。
- **storyboard-video**：Step 4 建立 Style Prefix 時，優先讀取同一份
  `storyboard_style.json` 的 `global_style_prompt`，取代原本依主題四選一的
  硬編風格前綴，確保與圖卡使用完全相同的美術方向與色票。若檔案不存在，
  才退回 storyboard-video.md 原有的四類主題風格表。

> 換言之，`storyboard_style.json` 是圖卡與影片共同的單一事實來源（single source
> of truth）。日後若要調整風格、色票或標題樣式，只需重新產生這一份檔案，
> 不需個別修改 image／video 兩份程式碼範本。

---

## 輸出檔案

| 檔案 | 說明 |
|---|---|
| `storyboard_style.json` | 全案唯一設計系統規格檔（美術風格、色票、標題排版），供 image／video 共用讀取 |

---

## 輸出原則

1. **唯一決策**：art_style／color_palette／typography 全案只決定一次，image 與 video 共用同一份，不再各自判斷
2. **參考照片優先**：只要有任一分鏡帶實景參考照片，全案鎖定半寫實 HUD 家族，避免 flat/cartoon 字眼與照片保真需求衝突
3. **具體色碼**：色票一律給出 hex 值，不用「暖色系」之類的模糊文字描述
4. **可調**：使用者可在對話中指定偏好色系或風格方向，覆蓋預設表格
5. **向下相容**：下游技能在找不到 `storyboard_style.json` 時，仍可退回各自原有邏輯運作，不會中斷流程

---

## 完成後

輸出本次決定的 art_style 家族、色票與標題樣式摘要，並詢問使用者：
> 「設計系統已建立（風格：{art_style 家族}，主色：{primary hex}）。是否符合需求？
> 若需調整風格方向或色系，請告訴我；確認後即可繼續執行 storyboard-image／storyboard-video 生成圖卡與影片。」
