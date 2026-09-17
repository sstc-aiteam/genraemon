---
name: storyboard-video
description: |
  根據 storyboard-design 輸出的分鏡腳本文字稿，以及 storyboard-image 輸出的分鏡圖卡，
  以雙引擎策略為每個分鏡生成一段短影片並儲存至工作目錄：優先呼叫地端 MinMax H3
  （ComfyUI Reference-to-Video 工作流程），失敗才 fallback 呼叫 xAI Grok Imagine Video API。
  傳入 MinMax H3 的提示詞一律以本技能內嵌的 H3 Prompt Writing skill／Full-Reference Mode
  六段式格式改寫（subject_definitions／summary／retention_analysis／detailed_description／
  overall_soundscape／non_diegetic_music），不使用外部連結檔案。
  每段影片預設 6 秒，畫面文字以繁體中文為主，禁用簡體字。

  當使用者提到「生成影片」、「分鏡影片」、「storyboard video」、「幫我做影片」、
  「把分鏡做成影片」、「影片生成」，或在分鏡腳本／圖卡輸出後要求影片化時，務必使用此技能。
  即使使用者只說「做成影片」、「生成影片」也應觸發。
---

# 分鏡影片生成技能（Storyboard Video Generation Skill）

## 任務概述

以 storyboard-design 的分鏡文字稿（畫面構圖、重點動作）與 storyboard-image 的圖卡
作為輸入，採**雙引擎策略**逐鏡生成影片並儲存：

1. **優先**：呼叫地端 MinMax H3（透過內網 ComfyUI 伺服器的 Reference-to-Video 工作流程）。
2. **備援**：該分鏡的 MinMax H3 呼叫失敗（伺服器無法連線、上傳失敗、執行錯誤、逾時等
   任何例外）時，自動改呼叫 xAI Grok Imagine Video API 生成同一分鏡。
3. 切換以**單一分鏡為單位**——某一鏡 MinMax H3 失敗不影響其他鏡，其他鏡仍優先嘗試 MinMax H3。

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

## 引擎一：MinMax H3（地端 ComfyUI）正確行為

| 項目 | 正確值 |
|---|---|
| ComfyUI 伺服器 | 讀取環境變數 `MINIMAX_SERVER_ADDRESS`（格式 `IP:Port`，例如 `192.168.x.x:8188`）；未設定時向使用者索取，不寫死於程式碼中 |
| 使用工作流程 | Reference-to-Video (R2V) Lightning-LoRA turbo 版，節點結構對應 `video_minimax_h3_r2v_turbo.json` |
| 上傳參考圖 | `POST http://{server}/upload/image`（multipart，欄位名 `image`），回傳 `{"name": "..."}` 供 LoadImage 節點使用 |
| 送出端點 | `POST http://{server}/prompt`，body 為 `{"prompt": <workflow graph>, "client_id": ...}`，回傳 `{"prompt_id": "..."}` |
| 進度／完成 | 以 WebSocket `ws://{server}/ws?clientId=...` 監聽訊息：`type="executing", data.node=None` 代表完成；`type="execution_error"` 代表失敗 |
| 讀取輸出 | `GET http://{server}/history/{prompt_id}`，逐一 output 找出含 `filename` 的項目，再以 `GET http://{server}/view?filename=...&subfolder=...&type=...` 下載 |
| 關鍵節點 ID | Prompt=138、R2V=136、Resolution=115、Duration=132、Seed=129、Steps(Full)=143、Steps(LoRA)=144、LoRA開關=146、SaveVideo=92、參考圖 LoadImage=137/139/148/149/150（依序對應 ref_image_0~4） |
| 參考圖上限 | 最多 9 張；**本技能每鏡僅使用對應的 1 張 `storyboard_panel_XX.jpg`**（wire 到 ref_image_0 = 節點 137，其餘 ref_image_1~4 的 wiring 直接刪除，避免指向不存在的 placeholder 檔案而執行失敗） |
| 判定失敗 | 連線失敗、上傳失敗、`POST /prompt` 非 200、WebSocket 收到 `execution_error`、或 `/history` 最終狀態為 `error`，一律視為該分鏡 MinMax H3 失敗 -> fallback Grok |

## 引擎二：xAI Grok Imagine Video API 正確行為（實測確認）

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

## MinMax H3 呼叫方式（對齊 generate_minmax_h3_r2v_video.py）

本技能第 6 步 Python 範本中的 MinMax H3 呼叫邏輯（`minimax_upload_image` / `minimax_queue_prompt` /
`minimax_wait_for_completion` / `minimax_download_outputs`），其端點、node ID、上傳／送出／輪詢／
下載流程，均對齊 `generate_minmax_h3_r2v_video.py` 的作法：

- 上傳一律用 `POST /upload/image`（ComfyUI 把它當通用「存入 input/」端點使用，圖片走這個路由）。
- 送出前先刪除 workflow 中所有 `ref_images.ref_image_N` wiring，僅重新接上本次實際提供的參考圖，
  避免未提供的插槽仍指向不存在的 placeholder 檔名而導致 `required_input_missing`／執行失敗。
- 送出用 `POST /prompt`（body 含 `prompt` 與 `client_id`），以 WebSocket `ws://{server}/ws?clientId=...`
  監聽 `executing`（`data.node=None` 代表完成）與 `execution_error`。
- 完成後以 `GET /history/{prompt_id}` 交叉確認 `status.status_str` 不是 `"error"`，
  再逐一掃描 outputs 找出含 `filename` 的項目，以 `GET /view` 下載。
- 節點 ID 常數（`MM_NODE_PROMPT=138`、`MM_NODE_R2V=136`、`MM_NODE_RESOLUTION=115`、
  `MM_NODE_DURATION=132`、`MM_NODE_SEED=129`、`MM_NODE_SAVE_VIDEO=92`、
  `MM_REF_IMAGE_NODE_0=137`）與 `video_minimax_h3_r2v_turbo.json` 匯出檔一致，
  若地端重新匯出過 workflow，需同步更新這些常數。
- 每個分鏡使用獨立亂數 seed（見 Step 6 範本的 `MM_NODE_SEED` 用法），避免所有分鏡因共用同一顆
  預設 seed 而畫面雷同。

---

## MinMax H3 提示詞撰寫規則（H3 Prompt Writing skill · Full-Reference Mode，已完整內嵌）

**本節是 `h3-prompt-writing` skill 的 Full-Reference Mode（`references/ref-en.txt`）規則，
已完整內嵌於本技能，執行時不需另外讀取或連結該 skill 的檔案。**

傳入 MinMax H3（workflow 節點 138 `value`）的提示詞，**一律不得使用 Step 5a 的簡單串接文字**，
必須依下列六段式格式改寫，寫成 `storyboard_scenes[i]["minimax_prompt"]` 的內嵌字串。
Grok（Step 5a）則繼續使用簡單串接文字，兩者提示詞內容不同、各自獨立。

### 適用範圍（本技能的簡化條件）

由於本技能的 MinMax H3 workflow 每鏡僅 wiring 1 張參考圖（`ref_image_0` = 對應的
`storyboard_panel_XX.jpg`），且不傳送參考影片／音訊，因此每鏡的六段式提示詞只會用到：

- `<Picture 1>`：固定指向該分鏡的 `storyboard_panel_XX.jpg`。
- `<Subject N>`：從畫面構圖／重點動作中萃取的關鍵人物、物件、場景（可有多個）。
- **不使用** `<Video N>` 與 `<Audio N>`（本技能不傳參考影片／音訊）。

### 六段式結構與順序

| Section | 內容 |
|---|---|
| `subject_definitions` | 定義 `<Subject N>`，每個一行，說明其在 `<Picture 1>` 中的外觀特徵 |
| `summary` | 一段英文摘要，開頭為方括號任務類型前綴 |
| `retention_analysis` | 逐一說明每個 `<Subject N>`／`<Picture 1>` 的保留關係 |
| `detailed_description` | 主體：風格開場句 + `[Shot 1]` 逐鏡描述構圖、動作、鏡頭運動、聲音 |
| `overall_soundscape` | 環境音／音效總覽 |
| `non_diegetic_music` | 僅觀眾聽得到的配樂描述，無配樂則寫 `N/A` |

### 撰寫細則

1. **語言**：六段落內文一律英文；畫面上出現的文字（含對白）維持原語言——本技能固定為繁體中文，
   禁用簡體字。
2. **`subject_definitions`**：每個 `<Subject N>` 一行，格式如
   `<Subject 1> is [描述], as shown in <Picture 1>.`——描述取自分鏡的畫面構圖／重點動作原文語意，
   翻譯並具體化為外觀特徵（勿用「cinematic」「beautiful」等抽象詞）。
3. **`summary`**：以方括號任務類型開頭。本技能的圖卡多半作為該分鏡影片的第一幀／關鍵幀，
   預設用 `[keyframe completion]`；若圖卡僅作構圖／風格參考而非具體幀，改用
   `[reference generation]`。摘要只能使用已定義的 `<Subject N>` / `<Picture 1>` 標籤，
   不得新增未定義標籤。
4. **`retention_analysis`**：逐一列出 `<Subject N>` 與 `<Picture 1>` 的保留關係，
   本技能情境下通常為 `fully_preserved`（圖卡特徵完整延續到影片中），格式如：
   `<Subject 1> (appears in [Shot 1]): fully_preserved - ...`
   `<Picture 1> ([Shot 1] first frame): fully_preserved - ...`
5. **`detailed_description`**：
   - 開場先以一到兩句英文建立風格（帶入 Step 4 的 `STYLE_PREFIX` 語意），再接 `[Shot 1]`。
   - `[Shot 1]` 不加時間戳（本技能每鏡固定單一鏡頭、6 秒，不切鏡）。
   - 依序描述：構圖與畫面中人物/物件位置（引用 `<Subject N>`）、環境與光線、動作與狀態變化
     （對應分鏡的「重點動作」原文語意）、鏡頭運動（如有）、當下聲音。
   - 畫面上出現的文字一律標註原始繁體中文，例如 `on-screen Traditional Chinese text "..." appears ...`。
   - 因單鏡僅 6 秒，字數依比例縮減，約 80–150 個英文字即可，仍須涵蓋構圖、主體、動作、鏡頭、聲音。
6. **`overall_soundscape`**：一句話總結該分鏡的環境音／音效。
7. **`non_diegetic_music`**：若分鏡無配樂需求，寫 `N/A`；否則簡述樂器、節奏、情緒發展。
8. **固定規則**：與 Step 5 的 `TEXT_RULE` 一致——16:9、6 秒、畫面文字僅限繁體中文。

### 範例（單圖卡、單鏡頭 6 秒）

```text
subject_definitions:
<Subject 1> is the orange instant-noodle cup with a printed brand wordmark and photography of
noodles and toppings, as shown in <Picture 1>.
<Subject 2> is the stainless steel kettle pouring water, as shown in <Picture 1>.

summary:
[keyframe completion] The target video begins from <Picture 1>, showing boiling water being
poured into <Subject 1> as steam rises, establishing the opening moment of the scene.

retention_analysis:
<Subject 1> (appears in [Shot 1]): fully_preserved - the cup's packaging, wordmark, and printed
photography remain identical to <Picture 1>.
<Subject 2> (appears in [Shot 1]): fully_preserved - the kettle's angle and pour position are
retained.
<Picture 1> ([Shot 1] first frame): fully_preserved - the framing, lighting, and object placement
are retained exactly.

detailed_description:
The target video is a bright, appetizing live-action commercial style with warm natural lighting
and a softly blurred kitchen backdrop.
[Shot 1] The shot begins from <Picture 1>: <Subject 2>, the stainless steel kettle, tilts in from
the upper left and pours a steady stream of boiling water into <Subject 1>, the instant-noodle
cup resting on a white countertop. Dense steam rises and curls upward as the dry noodles and
toppings sit visible inside the cup. On-screen Traditional Chinese text "酸香開胃" fades in near
the top of the frame. The camera pushes in with small amplitude at slow speed toward the rising
steam.

overall_soundscape:
Water rushes and bubbles as it pours into the cup, with a soft hiss of rising steam.

non_diegetic_music:
N/A
```

---

## 步驟流程

### Step 1：確認環境

MinMax H3 為地端服務，不需 API Key，但需要環境變數 `MINIMAX_SERVER_ADDRESS`（格式 `IP:Port`，
例如 `192.168.x.x:8188`）指向使用者的地端 ComfyUI 伺服器；Grok 為備援，需要 `XAI_API_KEY`。
三者皆非強制擋關——`MINIMAX_SERVER_ADDRESS` 未設定或連線異常時只警告並停用 MinMax H3，
仍讓 Grok 單獨運作；反之 `XAI_API_KEY` 缺失也只警告，讓 MinMax H3 單獨運作。只有兩個引擎都
不可用時，個別分鏡才會在執行時分別回報失敗。

**若對話中尚未取得 `MINIMAX_SERVER_ADDRESS`，必須先詢問使用者**：
> 「請提供您地端 ComfyUI 伺服器的位址（格式 IP:Port，例如 192.168.x.x:8188），
> 我會將它設定為環境變數 `MINIMAX_SERVER_ADDRESS`。若您沒有地端伺服器，可以略過，
> 屆時所有分鏡將改用 Grok 引擎生成。」

若使用者選擇略過，`MINIMAX_SERVER_ADDRESS` 留空即可，MinMax H3 引擎會整批自動停用，
所有分鏡直接使用 Grok，不影響流程繼續執行。

```python
import os

MINIMAX_SERVER_ADDRESS = os.environ.get("MINIMAX_SERVER_ADDRESS", "")
if not MINIMAX_SERVER_ADDRESS:
    print("[WARN] MINIMAX_SERVER_ADDRESS is not set. MinMax H3 (local) engine disabled "
          "for this run; all scenes will go straight to Grok.")

XAI_API_KEY = os.environ.get("XAI_API_KEY")
if not XAI_API_KEY:
    print("[WARN] XAI_API_KEY is not set. MinMax H3 (local) will still be attempted, "
          "but the Grok fallback will be unavailable if MinMax H3 fails.")

try:
    import websocket  # websocket-client
except ImportError:
    print("[WARN] websocket-client not installed (pip install websocket-client). "
          "MinMax H3 engine disabled for this run; all scenes will go straight to Grok.")
```

若三者皆不可用（無 `MINIMAX_SERVER_ADDRESS`、無 `XAI_API_KEY`、或無 `websocket-client`／
地端伺服器不可連線），仍繼續執行，讓每個分鏡在真正嘗試時各自回報 `[FAIL]`，而非直接中止
整支腳本。

### Step 2：詢問生成品質

在開始生成前，詢問使用者偏好的影片解析度：

> 「請選擇影片生成品質：
> - **480p**（預設，生成速度較快，適合快速預覽）
> - **720p**（較高畫質，適合正式輸出）
>
> 若未指定，預設使用 480p。」

將使用者的選擇儲存為 `RESOLUTION`（值為 `"480p"` 或 `"720p"`），後續程式碼使用此變數。
此變數同時驅動兩個引擎：Grok 的 `resolution` 欄位，以及 MinMax H3
`ResolutionSelector` 節點的 `megapixels`（見 Step 6 的 `RESOLUTION_MEGAPIXELS` 對照表）。

### Step 3：解析輸入

從對話中讀取最近一份分鏡腳本，格式為 `## 分鏡 N｜標題`，擷取每個分鏡的：
- 標題
- 🎨 畫面構圖
- 🎬 重點動作

同時掃描工作目錄尋找對應圖卡（`storyboard_panel_01.jpg`、`storyboard_panel_02.jpg`…）。
若圖卡存在，MinMax H3 與 Grok 皆以 image-to-video／reference-to-video 模式提交；
若不存在，MinMax H3 直接略過改走 Grok，Grok 則以純文字 prompt 模式備援。

### Step 4：建立共通風格描述（Style Prefix）

**優先讀取共用設計系統：** 先檢查共用 storyboard 資料夾內是否存在
`storyboard_style.json`（由 storyboard-style 技能產生）。若存在，直接使用其
`global_style_prompt` 作為 STYLE_PREFIX，**不再依主題自行判斷**——這確保兩個引擎生成的
影片與 storyboard-image 產出的圖卡套用完全相同的美術風格與色票。若檔案不存在
（使用者未執行 storyboard-style），才依下列主題分類表自行判斷，並可在完成後提醒使用者：
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

### Step 5a：組合 Grok 用的 Video Prompt（簡單串接）

```
[Style prefix], [畫面構圖原文], [重點動作原文], [固定規則]
```

固定規則（每個分鏡都加入）：
```
any on-screen text must use Traditional Chinese characters only, no Simplified Chinese,
16:9 aspect ratio, 6 seconds, smooth animation, clear visual hierarchy
```

> 畫面構圖與重點動作保留分鏡腳本的繁體中文原文，不翻譯為英文。
> 此簡單串接 prompt **僅供 Grok 使用**（填入 `prompt` 欄位），對應到 Python 範本中的
> `build_text_prompt(scene)`。

### Step 5b：為每個分鏡撰寫 MinMax H3 用的 Ref2VA 提示詞（Full-Reference Mode）

**MinMax H3 不使用 Step 5a 的簡單串接文字。** 依照上方「MinMax H3 提示詞撰寫規則」內嵌章節，
為每個有對應 `storyboard_panel_XX.jpg` 的分鏡，逐鏡撰寫六段式（`subject_definitions` /
`summary` / `retention_analysis` / `detailed_description` / `overall_soundscape` /
`non_diegetic_music`）Ref2VA 提示詞：

- `<Picture 1>` 固定對應該分鏡的 `storyboard_panel_XX.jpg`。
- 從該分鏡的畫面構圖／重點動作原文中萃取 `<Subject N>`（人物、物件、場景），轉譯為具體英文外觀描述。
- `detailed_description` 開場句帶入 Step 4 的 `STYLE_PREFIX` 語意，`[Shot 1]` 的動作描述須對應
  該分鏡的重點動作原文語意，畫面文字一律標註繁體中文原文。
- 寫完後的完整六段式文字，整段作為 `storyboard_scenes[i]["minimax_prompt"]` 的字串值
  （Step 6 範本中 `generate_via_minimax_h3()` 會直接把它填入 workflow 節點 138 的 `value`）。
- 若該分鏡沒有對應的 `storyboard_panel_XX.jpg`（無圖卡），則不需撰寫 `minimax_prompt`
  （`generate_via_minimax_h3()` 會因找不到圖卡而直接跳過改走 Grok）。

**輸出改寫結果以便確認：** 每完成一個分鏡的 Full-Reference Mode 改寫，必須在對話中原樣輸出該分鏡
完整的六段式 `minimax_prompt`（不可省略或截斷），格式如下，讓使用者在進入 Step 6／7 產生
`storyboard_video.py` 前能逐鏡確認改寫內容是否正確：

> 【分鏡 N｜標題】MinMax H3 提示詞（H3 Prompt Writing · Full-Reference Mode 改寫）：
> ```text
> subject_definitions:
> ...
> summary:
> ...
> retention_analysis:
> ...
> detailed_description:
> ...
> overall_soundscape:
> ...
> non_diegetic_music:
> ...
> ```

全部分鏡皆輸出完成後，才繼續 Step 6。若使用者要求修改，改寫後需重新輸出該分鏡的完整結果再繼續。

### Step 6：生成 Python 程式碼並儲存為 storyboard_video.py

**程式碼生成規則：**
- 所有 `print()` 日誌不得含有 Emoji，使用純文字標籤：`[OK]`、`[FAIL]`、`[ERROR]`、`[DONE]`、`[START]`、`[SUBMITTED]`、`[STATUS]`、`[INFO]`、`[DEBUG]`、`[RETRY]`、`[TIMEOUT]`
- 所有程式碼註解不得含有 Emoji
- 生成完成後，程式碼檔案 `storyboard_video.py` 不得刪除
- 每個分鏡的執行邏輯必須是：**先嘗試 `generate_via_minimax_h3()`，例外或回傳失敗時
  才印出 `[RETRY]` 訊息並改呼叫 `generate_via_grok()`**。兩者都失敗才記為該分鏡 `failed`。
- 每個 `storyboard_scenes` 項目若有對應圖卡，**必須**依 Step 5b 規則填入 `minimax_prompt`
  （H3 Prompt Writing skill · Full-Reference Mode 六段式文字）；`generate_via_minimax_h3()`
  一律使用 `scene["minimax_prompt"]` 填入節點 138，**不得**使用 `build_text_prompt(scene)`
  （後者僅供 Grok 使用）

**完整程式碼範本：**

```python
import copy
import os
import json
import time
import base64
import random
import uuid
import requests
from pathlib import Path

# Switch to the script's own directory (the shared storyboard folder) so all
# input images and output videos live in the same folder.
os.chdir(Path(__file__).resolve().parent)

# -------------------------------------------------------------------
# Shared config
# -------------------------------------------------------------------

# Resolution chosen by user: "480p" (default) or "720p"
RESOLUTION = "480p"

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


def build_text_prompt(scene):
    return f"{STYLE_PREFIX}, {scene['composition']}, {scene['motion']}, {TEXT_RULE}"


def load_image_data_uri(path):
    with open(path, "rb") as f:
        b64 = base64.b64encode(f.read()).decode("utf-8")
    return f"data:image/jpeg;base64,{b64}"


# -------------------------------------------------------------------
# storyboard_scenes is populated from storyboard-design output.
# Each entry must contain: index, title, composition, motion (used by
# build_text_prompt() for the Grok fallback), and -- when a matching
# storyboard_panel_XX.jpg exists -- minimax_prompt: the full six-section
# Ref2VA prompt (subject_definitions / summary / retention_analysis /
# detailed_description / overall_soundscape / non_diegetic_music) written
# per the embedded H3 Prompt Writing skill (Full-Reference Mode) rules.
# minimax_prompt is used as-is for workflow node 138 and is NOT derived
# from build_text_prompt() -- the two engines use different prompt formats.
# -------------------------------------------------------------------
storyboard_scenes = [
    # populated at generation time from parsed storyboard script
]


# =====================================================================
# Engine 1 (primary): MinMax H3 via local/on-prem ComfyUI (R2V, turbo)
# =====================================================================

# Read from environment -- ask the user for their on-prem ComfyUI server
# address (IP:Port) during Step 1/7 and set this env var rather than
# hardcoding a specific network's address here. Empty string disables the
# MinMax H3 engine for this run (see generate_via_minimax_h3 below).
MINIMAX_SERVER_ADDRESS = os.environ.get("MINIMAX_SERVER_ADDRESS", "")

# Node IDs from video_minimax_h3_r2v_turbo.json -- keep in sync if the
# exported workflow is regenerated.
MM_NODE_PROMPT = "138"       # PrimitiveStringMultiline -> inputs.value
MM_NODE_R2V = "136"          # MiniMaxH3ReferenceToVideo
MM_NODE_RESOLUTION = "115"   # ResolutionSelector -> inputs.aspect_ratio / megapixels
MM_NODE_DURATION = "132"     # PrimitiveFloat (seconds) -> inputs.value
MM_NODE_SEED = "129"         # RandomNoise -> inputs.noise_seed
MM_NODE_SAVE_VIDEO = "92"    # SaveVideo -> inputs.filename_prefix
MM_REF_IMAGE_NODE_0 = "137"  # LoadImage wired to ref_images.ref_image_0

# Approximate megapixel targets for the ResolutionSelector node, matched to
# the same RESOLUTION choice used by the Grok engine.
RESOLUTION_MEGAPIXELS = {"480p": 0.3, "720p": 0.5}

# Reference-to-Video workflow (video_minimax_h3_r2v_turbo.json), embedded so this
# script has no external file dependency. Node 138's prompt and node 137's image
# are placeholders overwritten per-scene at runtime; nodes 139/148/149/150 are
# unused placeholder ref-image slots removed from the wiring before submission.
MINIMAX_H3_WORKFLOW_JSON = r'''{"92":{"inputs":{"filename_prefix":"video/MiniMax_H3","format":"auto","format.codec":"auto","codec":"auto","video-preview":"","video":["130",0]},"class_type":"SaveVideo","_meta":{"title":"Save Video"}},"115":{"inputs":{"aspect_ratio":"16:9 (Widescreen)","megapixels":1,"multiple":32},"class_type":"ResolutionSelector","_meta":{"title":"Resolution Selector (Size)"}},"119":{"inputs":{"vae_name":"minimax_h3_video_vae_fp16.safetensors"},"class_type":"VAELoader","_meta":{"title":"Load VAE"}},"120":{"inputs":{"vae_name":"minimax_h3_audio_vae_fp32.safetensors"},"class_type":"VAELoader","_meta":{"title":"Load VAE"}},"121":{"inputs":{"samples":["125",0],"vae":["120",0]},"class_type":"VAEDecodeAudio","_meta":{"title":"VAE Decode Audio"}},"122":{"inputs":{"samples":["125",0],"vae":["119",0]},"class_type":"VAEDecode","_meta":{"title":"VAE Decode"}},"123":{"inputs":{"sampler_name":"res_multistep"},"class_type":"KSamplerSelect","_meta":{"title":"KSamplerSelect"}},"124":{"inputs":{"scheduler":"simple","steps":["142",0],"denoise":1,"model":["127",0]},"class_type":"BasicScheduler","_meta":{"title":"BasicScheduler"}},"125":{"inputs":{"noise":["129",0],"guider":["126",0],"sampler":["123",0],"sigmas":["124",0],"latent_image":["136",1]},"class_type":"SamplerCustomAdvanced","_meta":{"title":"SamplerCustomAdvanced"}},"126":{"inputs":{"model":["141",0],"conditioning":["136",0]},"class_type":"BasicGuider","_meta":{"title":"Basic Guider"}},"127":{"inputs":{"unet_name":"minimax_h3_ref2va_pruned_int8_convrot.safetensors","weight_dtype":"default"},"class_type":"UNETLoader","_meta":{"title":"Load Diffusion Model"}},"128":{"inputs":{"clip_name":"qwen3vl_32b_minimax_h3_nvfp4_awq.safetensors","type":"minimax","device":"default"},"class_type":"CLIPLoader","_meta":{"title":"Load CLIP"}},"129":{"inputs":{"noise_seed":821150909694347},"class_type":"RandomNoise","_meta":{"title":"RandomNoise"}},"130":{"inputs":{"fps":24,"bit_depth":8,"color_space":"sRGB","images":["122",0],"audio":["121",0]},"class_type":"CreateVideo","_meta":{"title":"Create Video"}},"131":{"inputs":{"expression":"max(5, round(a * 24)) + (5 - (max(5, round(a * 24)) % 17)) % 17","values.a":["132",0]},"class_type":"ComfyMathExpression","_meta":{"title":"Math Expression"}},"132":{"inputs":{"value":10},"class_type":"PrimitiveFloat","_meta":{"title":"Float (Duration)"}},"136":{"inputs":{"prompt":["138",0],"width":["115",0],"height":["115",1],"length":["131",1],"ref_image_size":"match","clip":["128",0],"vae":["119",0],"audio_vae":["120",0],"ref_images.ref_image_0":["137",0],"ref_images.ref_image_1":["139",0],"ref_images.ref_image_2":["148",0],"ref_images.ref_image_3":["149",0],"ref_images.ref_image_4":["150",0]},"class_type":"MiniMaxH3ReferenceToVideo","_meta":{"title":"MiniMax H3 Reference to Video"}},"137":{"inputs":{"image":"placeholder.jpg"},"class_type":"LoadImage","_meta":{"title":"Load Image"}},"138":{"inputs":{"value":"placeholder prompt - overwritten at runtime by scene['minimax_prompt']"},"class_type":"PrimitiveStringMultiline","_meta":{"title":"Input Text (Prompt)"}},"139":{"inputs":{"image":"placeholder.jpg"},"class_type":"LoadImage","_meta":{"title":"Load Image"}},"141":{"inputs":{"switch":["146",0],"on_false":["127",0],"on_true":["145",0]},"class_type":"ComfySwitchNode","_meta":{"title":"If/Else Switch (model)"}},"142":{"inputs":{"switch":["146",0],"on_false":["143",0],"on_true":["144",0]},"class_type":"ComfySwitchNode","_meta":{"title":"If/Else Switch (Steps)"}},"143":{"inputs":{"value":20},"class_type":"PrimitiveInt","_meta":{"title":"Int (Full)"}},"144":{"inputs":{"value":8},"class_type":"PrimitiveInt","_meta":{"title":"Int (Lightning LoRA)"}},"145":{"inputs":{"lora_name":"minimax_h3_ref2v_turbo_8step_v1.0_768p_comfyui_bf16.safetensors","strength_model":1,"model":["127",0]},"class_type":"LoraLoaderModelOnly","_meta":{"title":"Load LoRA"}},"146":{"inputs":{"value":true},"class_type":"PrimitiveBoolean","_meta":{"title":"Boolean (Enable Lightning LoRA)"}},"148":{"inputs":{"image":"placeholder.jpg"},"class_type":"LoadImage","_meta":{"title":"Load Image"}},"149":{"inputs":{"image":"placeholder.jpg"},"class_type":"LoadImage","_meta":{"title":"Load Image"}},"150":{"inputs":{"image":"placeholder.jpg"},"class_type":"LoadImage","_meta":{"title":"Load Image"}}}'''

MINIMAX_H3_WORKFLOW_TEMPLATE = json.loads(MINIMAX_H3_WORKFLOW_JSON)


class ComfyUIError(RuntimeError):
    """Raised for any ComfyUI-side failure -- triggers fallback to Grok."""


def _minimax_base_url():
    return f"http://{MINIMAX_SERVER_ADDRESS}"


def minimax_upload_image(filepath):
    if not os.path.isfile(filepath):
        raise ComfyUIError(f"Local file not found: {filepath}")
    with open(filepath, "rb") as f:
        try:
            resp = requests.post(
                f"{_minimax_base_url()}/upload/image",
                files={"image": (os.path.basename(filepath), f)},
                timeout=60,
            )
        except requests.exceptions.RequestException as exc:
            raise ComfyUIError(f"Could not reach MinMax H3 server: {exc}") from exc
    if resp.status_code != 200:
        raise ComfyUIError(f"Upload failed (HTTP {resp.status_code}): {resp.text[:300]}")
    return resp.json()["name"]


def minimax_queue_prompt(workflow, client_id):
    try:
        resp = requests.post(
            f"{_minimax_base_url()}/prompt",
            json={"prompt": workflow, "client_id": client_id},
            timeout=30,
        )
    except requests.exceptions.RequestException as exc:
        raise ComfyUIError(f"Could not reach MinMax H3 server: {exc}") from exc
    if resp.status_code != 200:
        raise ComfyUIError(f"MinMax H3 rejected the prompt (HTTP {resp.status_code}): {resp.text[:500]}")
    return resp.json()


def minimax_wait_for_completion(ws, prompt_id, max_wait=300):
    ws.settimeout(max_wait)
    while True:
        out = ws.recv()
        if not isinstance(out, str):
            continue  # binary frames are live preview data
        message = json.loads(out)
        data = message.get("data", {})
        if data.get("prompt_id") != prompt_id:
            continue
        if message.get("type") == "executing" and data.get("node") is None:
            return None  # success
        if message.get("type") == "execution_error":
            return data


def minimax_download_outputs(prompt_id, output_path):
    resp = requests.get(f"{_minimax_base_url()}/history/{prompt_id}", timeout=30)
    resp.raise_for_status()
    history = resp.json()[prompt_id]
    if history.get("status", {}).get("status_str") == "error":
        raise ComfyUIError(f"MinMax H3 prompt {prompt_id} ended in error: {history['status']}")

    for node_output in history["outputs"].values():
        for value in node_output.values():
            if isinstance(value, list) and value and isinstance(value[0], dict) and "filename" in value[0]:
                item = value[0]
                file_resp = requests.get(
                    f"{_minimax_base_url()}/view",
                    params={
                        "filename": item["filename"],
                        "subfolder": item.get("subfolder", ""),
                        "type": item.get("type", "output"),
                    },
                    timeout=120,
                )
                file_resp.raise_for_status()
                with open(output_path, "wb") as f:
                    f.write(file_resp.content)
                return True
    raise ComfyUIError(f"No output file found in MinMax H3 history for prompt {prompt_id}")


def generate_via_minimax_h3(scene, panel_path, output_path):
    """Try the local MinMax H3 engine. Returns True on success, False on any failure."""
    if not MINIMAX_SERVER_ADDRESS:
        print("  [INFO] MinMax H3 skipped (MINIMAX_SERVER_ADDRESS is not set)")
        return False
    if not panel_path or not os.path.exists(panel_path):
        print("  [INFO] MinMax H3 skipped (no reference panel image for this scene)")
        return False

    minimax_prompt = scene.get("minimax_prompt")
    if not minimax_prompt:
        print("  [WARN] Scene has no minimax_prompt (H3 Full-Reference Mode text) -- "
              "falling back to the plain Grok-style prompt for MinMax H3 too")
        minimax_prompt = build_text_prompt(scene)

    workflow = copy.deepcopy(MINIMAX_H3_WORKFLOW_TEMPLATE)
    workflow[MM_NODE_PROMPT]["inputs"]["value"] = minimax_prompt
    workflow[MM_NODE_RESOLUTION]["inputs"]["megapixels"] = RESOLUTION_MEGAPIXELS.get(RESOLUTION, 0.3)
    workflow[MM_NODE_DURATION]["inputs"]["value"] = 6
    workflow[MM_NODE_SEED]["inputs"]["noise_seed"] = random.randint(0, 2**53 - 1)
    workflow[MM_NODE_SAVE_VIDEO]["inputs"]["filename_prefix"] = f"storyboard/scene_{scene['index']:02d}"

    r2v_inputs = workflow[MM_NODE_R2V]["inputs"]
    for key in list(r2v_inputs):
        if key.startswith("ref_images.ref_image_"):
            del r2v_inputs[key]

    try:
        server_filename = minimax_upload_image(panel_path)
        workflow[MM_REF_IMAGE_NODE_0]["inputs"]["image"] = server_filename
        r2v_inputs["ref_images.ref_image_0"] = [MM_REF_IMAGE_NODE_0, 0]

        client_id = str(uuid.uuid4())
        ws = websocket.WebSocket()
        try:
            ws.connect(f"ws://{MINIMAX_SERVER_ADDRESS}/ws?clientId={client_id}", timeout=15)
        except Exception as exc:
            raise ComfyUIError(f"Could not open websocket to MinMax H3 server: {exc}") from exc

        try:
            result = minimax_queue_prompt(workflow, client_id)
            prompt_id = result["prompt_id"]
            print(f"  [SUBMITTED] engine=minimax_h3 prompt_id={prompt_id}")
            error_data = minimax_wait_for_completion(ws, prompt_id)
            if error_data is not None:
                raise ComfyUIError(f"MinMax H3 execution error: {error_data}")
        finally:
            ws.close()

        minimax_download_outputs(prompt_id, output_path)
        print(f"  [OK] engine=minimax_h3 -> {output_path}")
        return True

    except Exception as e:
        print(f"  [RETRY] MinMax H3 failed ({type(e).__name__}: {e}), falling back to Grok")
        return False


# =====================================================================
# Engine 2 (fallback): xAI Grok Imagine Video API
# =====================================================================

GROK_SUBMIT_URL = "https://api.x.ai/v1/videos/generations"
GROK_POLL_BASE = "https://api.x.ai/v1/videos"
GROK_HEADERS = {
    "Authorization": f"Bearer {XAI_API_KEY}",
    "Content-Type": "application/json"
}
GROK_POLL_HEADERS = {"Authorization": f"Bearer {XAI_API_KEY}"}
GROK_RESOLUTION_FALLBACK = {"480p": "720p", "720p": "1080p"}


def grok_submit_generation(scene, panel_path=None):
    text_prompt = build_text_prompt(scene)

    payload = {
        "model": "grok-imagine-video",
        "prompt": text_prompt,
        "duration": 6,
        "resolution": RESOLUTION,
    }

    if panel_path and os.path.exists(panel_path):
        payload["image"] = {"url": load_image_data_uri(panel_path)}
        print(f"  [INFO] engine=grok image-to-video mode, reference: {panel_path}")
    else:
        print(f"  [INFO] engine=grok text-only mode (no reference image found)")

    try:
        resp = requests.post(GROK_SUBMIT_URL, headers=GROK_HEADERS, json=payload, timeout=60)

        if resp.status_code == 422 and "image" in payload:
            print(f"  [RETRY] Image rejected (422), retrying text-only")
            payload.pop("image")
            resp = requests.post(GROK_SUBMIT_URL, headers=GROK_HEADERS, json=payload, timeout=60)

        if resp.status_code == 400 and RESOLUTION in GROK_RESOLUTION_FALLBACK:
            fallback_res = GROK_RESOLUTION_FALLBACK[RESOLUTION]
            print(f"  [RETRY] Resolution {RESOLUTION} rejected (400), retrying {fallback_res}")
            payload["resolution"] = fallback_res
            resp = requests.post(GROK_SUBMIT_URL, headers=GROK_HEADERS, json=payload, timeout=60)

        resp.raise_for_status()
        return resp.json()

    except requests.exceptions.HTTPError as e:
        body = e.response.text[:300] if e.response is not None else "no response body"
        raise RuntimeError(f"Submit failed: HTTP {e.response.status_code}: {body}") from e


def grok_poll_status(request_id, max_wait=300, interval=5):
    url = f"{GROK_POLL_BASE}/{request_id}"
    elapsed = 0
    while elapsed < max_wait:
        resp = requests.get(url, headers=GROK_POLL_HEADERS, timeout=30)
        data = resp.json()
        status = data.get("status", "unknown")
        progress = data.get("progress", 0)
        print(f"  [STATUS] engine=grok id={request_id} status={status} progress={progress}% elapsed={elapsed}s")

        if status == "done":
            return data
        if status == "expired":
            print(f"  [FAIL] Request expired")
            return None

        time.sleep(interval)
        elapsed += interval

    print(f"  [TIMEOUT] Exceeded {max_wait}s for id={request_id}")
    return None


def grok_extract_video_url(data):
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


def grok_download_video(video_url, output_path):
    resp = requests.get(video_url, stream=True, timeout=120)
    resp.raise_for_status()
    with open(output_path, "wb") as f:
        for chunk in resp.iter_content(chunk_size=8192):
            f.write(chunk)


def generate_via_grok(scene, panel_path, output_path):
    """Fallback engine. Returns True on success, False on failure."""
    if not XAI_API_KEY:
        print("  [FAIL] engine=grok unavailable: XAI_API_KEY is not set")
        return False

    try:
        gen_data = grok_submit_generation(scene, panel_path)
        request_id = gen_data.get("request_id") or gen_data.get("id")

        if not request_id:
            print(f"  [FAIL] engine=grok No request_id in response")
            print(f"  [DEBUG] Response: {gen_data}")
            return False

        print(f"  [SUBMITTED] engine=grok request_id={request_id}")
        final_data = grok_poll_status(request_id)
        if not final_data:
            return False

        video_url = grok_extract_video_url(final_data)
        if not video_url:
            print(f"  [FAIL] engine=grok No video URL found in completed response")
            print(f"  [DEBUG] Response keys: {list(final_data.keys())}")
            return False

        grok_download_video(video_url, output_path)
        print(f"  [OK] engine=grok -> {output_path}")
        return True

    except requests.exceptions.HTTPError as e:
        body = e.response.text[:300] if e.response is not None else "no response body"
        print(f"  [ERROR] engine=grok HTTP {e.response.status_code if e.response is not None else 'N/A'}: {body}")
        return False
    except Exception as e:
        print(f"  [ERROR] engine=grok {type(e).__name__}: {e}")
        return False


# =====================================================================
# Main loop: MinMax H3 first, Grok fallback, per scene
# =====================================================================

results = []

for scene in storyboard_scenes:
    idx = scene["index"]
    panel_path = f"storyboard_panel_{idx:02d}.jpg"
    output_path = f"storyboard_video_{idx:02d}.mp4"

    print(f"[START] Scene {idx}: {scene['title']}")
    print(f"  [PROMPT] grok={build_text_prompt(scene)[:100]}...")
    if scene.get("minimax_prompt"):
        print(f"  [PROMPT] minimax_h3={scene['minimax_prompt'][:100]}...")

    ok = generate_via_minimax_h3(scene, panel_path, output_path)
    engine_used = "minimax_h3"

    if not ok:
        ok = generate_via_grok(scene, panel_path, output_path)
        engine_used = "grok"

    if ok:
        results.append({"index": idx, "title": scene["title"], "status": "ok",
                         "path": output_path, "engine": engine_used})
    else:
        results.append({"index": idx, "title": scene["title"], "status": "failed",
                         "path": None, "engine": None})

    time.sleep(2)

print("\n[DONE] Summary:")
for r in results:
    path_label = r["path"] if r["path"] else "N/A"
    engine_label = f" engine={r['engine']}" if r["engine"] else ""
    print(f"  Scene {r['index']} ({r['title']}): [{r['status'].upper()}]{engine_label} {path_label}")
```

### Step 7：填入場景資料並執行

從對話中解析分鏡腳本，填入 `storyboard_scenes` 陣列：每個項目除 `index`／`title`／
`composition`／`motion`（Grok 用）外，若有對應 `storyboard_panel_XX.jpg`，**必須**
依 Step 5b 規則另外填入 `minimax_prompt`（H3 Prompt Writing skill · Full-Reference Mode
六段式文字，逐鏡由你親自撰寫，不可留空或用簡單串接文字代替，且須已在 Step 5b 於對話中
原樣輸出供使用者確認過）。並依使用者選擇設定 `RESOLUTION`（`"480p"` 或 `"720p"`，
未指定則保持預設 `"480p"`）。`MINIMAX_SERVER_ADDRESS` 一律讀取環境變數，若 Step 1 尚未取得
使用者提供的地端 ComfyUI 伺服器位址，執行前務必再次詢問使用者並設定該環境變數
（例如 `export MINIMAX_SERVER_ADDRESS="192.168.x.x:8188"`，或使用者實際的內網位址），
**不得**在 `storyboard_video.py` 中寫死特定位址；使用者若表示沒有地端伺服器，則保持
`MINIMAX_SERVER_ADDRESS` 未設定即可，MinMax H3 引擎會自動停用，全部分鏡改用 Grok。

**同時將 Step 5b 已確認的每鏡 `minimax_prompt`，原樣寫入共用 storyboard 資料夾內的
`storyboard_video_prompts.md`**（依分鏡順序列出，格式為 `## 分鏡 N｜標題` 接完整六段式文字），
作為改寫結果的留存紀錄，方便事後複查或重新生成時比對；無 `minimax_prompt` 的分鏡（走純
Grok）在此檔案中標註「（無圖卡，僅 Grok，無 Ref2VA 改寫）」。

將完整程式碼儲存為**共用 storyboard 資料夾**內的 `storyboard_video.py`，再執行：

```bash
python storyboard/storyboard_video.py
```

> 請將 `storyboard/` 替換為本次實際使用的資料夾名稱（例如 `storyboard001/`）。

---

## 輸出檔案

| 檔案 | 說明 |
|---|---|
| `storyboard_video_01.mp4` | 分鏡 1 影片（MinMax H3 或 Grok 生成，依實際成功引擎而定） |
| `storyboard_video_02.mp4` | 分鏡 2 影片 |
| … | … |
| `storyboard_video_N.mp4` | 分鏡 N 影片 |
| `storyboard_video.py` | 保留的程式碼，不自動刪除 |
| `storyboard_video_prompts.md` | 每鏡 MinMax H3 的完整 Ref2VA 改寫結果（Step 5b 已於對話中確認過），留存供複查 |

---

## 輸出原則

1. **雙引擎優先序**：每個分鏡一律先嘗試地端 MinMax H3，失敗才 fallback 呼叫 Grok；兩者都失敗才記為該分鏡失敗
2. **無 Emoji 日誌與註解**：Python 程式碼中的所有 `print()` 輸出與註解，一律不得含有 Emoji
3. **繁體中文強制**：prompt 中明確要求畫面文字使用繁體中文，禁用簡體字
4. **圖卡優先**：有 `storyboard_panel_XX.jpg` 時，MinMax H3 以 `ref_images.ref_image_0` 頂層 wiring、Grok 以 `"image": {"url": "data:image/jpeg;base64,..."}` 欄位送出；Grok 422 錯誤時自動退為純文字備援
5. **雙提示詞格式**：MinMax H3 與 Grok 使用不同格式的提示詞——MinMax H3 一律用 H3 Prompt Writing
   skill（Full-Reference Mode，本文件已內嵌完整規則）六段式 `minimax_prompt`，Grok 一律用簡單串接的
   `build_text_prompt()`；兩者不得混用或互相取代
6. **必填 model**：Grok 每次請求都帶 `"model": "grok-imagine-video"`
7. **保留程式碼**：`storyboard_video.py` 執行後不刪除
8. **逐鏡儲存**：每段影片獨立為 MP4 檔案，方便後製
9. **改寫結果必須輸出確認**：經 H3 Prompt Writing · Full-Reference Mode 改寫的 `minimax_prompt`，
   每鏡完成後都要在對話中原樣完整輸出（見 Step 5b），並寫入 `storyboard_video_prompts.md` 留存，
   不得只放進程式碼裡而不展示給使用者確認
10. **共用資料夾**：所有輸入圖卡與輸出影片、程式碼均存放於共用 storyboard 資料夾（`storyboard`／`storyboardNNN`）內

---

## 完成後

回報生成結果（含每個分鏡實際使用的引擎：`minimax_h3` 或 `grok`），並詢問使用者：
> 「影片是否符合需求？若需要調整影片長度、重新生成特定分鏡，或修改畫風描述，請告訴我。」

若生成時發現共用資料夾內沒有 `storyboard_style.json`（本技能是以局部主題判斷生成 STYLE_PREFIX），
額外提醒使用者：
> 「這次影片是用本技能內建的主題風格判斷生成的。若圖卡與影片風格看起來不一致，
> 建議先執行 storyboard-style 技能建立共用設計系統，重新生成後圖卡與影片會套用同一套風格。」

若某些分鏡最終是由 Grok 產生（MinMax H3 該鏡失敗），額外提醒使用者：
> 「分鏡 N 因地端 MinMax H3 無法完成（詳見執行日誌的 [RETRY] 訊息），已改用 Grok 生成，
> 畫面風格可能與 MinMax H3 產出的分鏡略有差異，如需重試地端引擎請確認 ComfyUI 伺服器狀態後重新執行。」
