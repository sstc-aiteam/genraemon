# KnowledgeTranslation／分鏡技能組 Release Notes

涵蓋範圍：`Skill/KnowledgeTranslation/` 下的 6 個 Claude skill，組成一條「文件 → 分鏡腳本 → 設計系統 → 圖卡 → 影片 → 旁白 → 配樂」的自動化流水線。

## 目前狀態（截至 2026-09-17）

| 順序 | Skill | 角色 | 狀態 |
|---|---|---|---|
| 1 | `storyboard-design` | 進入點：讀取上傳文件，產出 `storyboard_script.md`，並建立共用輸出資料夾 | 穩定 |
| 2 | `storyboard-style` | 選用但建議：分析分鏡腳本，決定唯一一份美術風格／色票／標題排版，輸出 `storyboard_style.json` | 穩定 |
| 3 | `storyboard-image` | 呼叫 Gemini（預設）或 OpenAI（備選）生成分鏡圖卡，Vision 驗證繁體中文 | 穩定 |
| 4 | `storyboard-video` | **雙引擎**：優先呼叫地端 MinMax H3（ComfyUI Reference-to-Video），單鏡失敗才 fallback 至 xAI `grok-imagine-video` | 本次大幅擴充 |
| 5 | `storyboard-narration` | Edge-TTS 生成旁白、合成進分鏡影片、xfade 轉場串接為完整影片 | 穩定 |
| 6 | `storyboard-bgm` | 呼叫 Gemini Lyria 3 生成純器樂 BGM，混入旁白影片輸出最終成品 | 穩定 |

流程順序：`storyboard-design` → （建議）`storyboard-style` → `storyboard-image` → `storyboard-video` → `storyboard-narration` → `storyboard-bgm`。除 design 外皆為下游技能，共用同一個 `storyboard`／`storyboard001`／… 輸出資料夾。

---

## 2026-09-17 — `storyboard-video` 新增地端 MinMax H3 雙引擎（較上次發布的變動）

本次僅 `storyboard-video.md` 變動（542 行新增、90 行刪除），其餘 5 個技能與 2026-08-27 版本相同。核心變化：影片生成從「單一呼叫 xAI Grok」改為「地端優先、雲端備援」的雙引擎策略。

**新增引擎一：MinMax H3（地端 ComfyUI，優先）**
- 透過內網 ComfyUI 伺服器（環境變數 `MINIMAX_SERVER_ADDRESS`，格式 `IP:Port`）的 Reference-to-Video（R2V）Lightning-LoRA turbo 工作流程生成影片，不需 API Key。
- 內嵌完整的上傳／送出／WebSocket 輪詢／下載流程（`minimax_upload_image` / `minimax_queue_prompt` / `minimax_wait_for_completion` / `minimax_download_outputs`），workflow JSON（`video_minimax_h3_r2v_turbo.json` 對應版本）直接內嵌於腳本中，不依賴外部檔案。
- 每個分鏡僅 wiring 對應的 1 張 `storyboard_panel_XX.jpg` 作為參考圖（`ref_image_0`），其餘未使用的參考圖插槽會在送出前移除 wiring，避免指向不存在的 placeholder 檔案而執行失敗。

**Grok 降級為引擎二（備援）**
- 切換粒度為**單一分鏡**：`generate_via_minimax_h3()` 失敗（連線失敗、上傳失敗、執行錯誤、逾時等任何例外）時，該鏡才自動改呼叫 `generate_via_grok()`；其餘分鏡仍優先嘗試 MinMax H3。兩個引擎皆需要時才判定該分鏡失敗。
- `MINIMAX_SERVER_ADDRESS`／`XAI_API_KEY` 均非強制擋關：任一缺失只警告並停用對應引擎，讓另一引擎單獨運作；兩者皆不可用時才在個別分鏡執行時各自回報失敗，不會中止整支腳本。

**新增內嵌「H3 Prompt Writing skill · Full-Reference Mode」六段式提示詞規則**
- MinMax H3 的提示詞不再使用 Grok 的簡單串接文字，必須依 `subject_definitions` / `summary` / `retention_analysis` / `detailed_description` / `overall_soundscape` / `non_diegetic_music` 六段式格式逐鏡改寫（新增 Step 5b），並在對話中原樣輸出每鏡改寫結果供使用者確認後才繼續。
- 改寫結果同時寫入新輸出檔 `storyboard_video_prompts.md`，留存供事後複查。
- Grok 與 MinMax H3 使用完全不同格式的提示詞，兩者不得混用或互相取代。

**其他變動**
- `RESOLUTION`（480p/720p）同時驅動 Grok 的 `resolution` 欄位與 MinMax H3 `ResolutionSelector` 節點的 `megapixels`。
- 執行結果摘要新增 `engine` 欄位，回報每個分鏡實際成功的引擎（`minimax_h3` 或 `grok`），若有分鏡因地端引擎失敗而改用 Grok，會額外提醒使用者畫面風格可能與地端產出略有差異。
- `storyboard_style.json` 讀取邏輯不變，MinMax H3 與 Grok 兩者的 `STYLE_PREFIX` 仍共用同一份設計系統。

---

## 2026-08-27 — 共用輸出資料夾 + 統一設計系統

這是目前最大的一次修訂，同時觸及全部 6 個檔案。

**新增 `storyboard-style` skill**
- 在 design 產出腳本之後、image／video 生成之前執行，針對整案只做一次美術風格、色票（hex）、標題字型／樣式決策，寫入共用資料夾內的 `storyboard_style.json`。
- 解決了先前 `storyboard-image`（預設偏好半寫實 HUD 疊加）與 `storyboard-video`（預設偏好 flat 2D vector 扁平插畫）兩邊各自判斷、方向互相衝突的問題。
- `storyboard-image`／`storyboard-video` 皆改為優先讀取此檔案；若使用者略過此技能，兩者各自退回原本的局部判斷邏輯，不中斷流程。

**全線改用共用輸出資料夾**
- `storyboard-design`／`storyboard-image`／`storyboard-video`／`storyboard-narration`／`storyboard-bgm` 全部加入同一套 `resolve_storyboard_dir()`：`storyboard-design` 以 `create_new=True` 建立新資料夾（`storyboard`、`storyboard001`…），其餘技能以 `create_new=False` 沿用最新的既有資料夾。
- 每個技能產生的腳本開頭改為 `os.chdir()` 至自身所在目錄，確保所有中間檔與輸出檔（腳本、圖卡、影片、音檔）都落在同一個專案資料夾內，不再散落在工作目錄各處。

**`storyboard-image` 大幅擴充**（本次修訂中改動最多的檔案）
- 新增參考照片支援：分鏡可標註實景參考照片路徑，以 image+text 條件生成（保留真實場景光影與構圖），並明訂「文字提示詞權重高於參考圖片」的限制——prompt 中一旦出現 `flat`/`cartoon`/`2D vector` 等字眼，即使附了真實照片，模型仍會整個重畫成插畫，因此依「該分鏡是否附照片」分兩種 ART_STYLE 模式處理。
- 新增雙引擎支援：預設 `gemini-3.1-flash-image`，`GOOGLE_API_KEY` 不可用時自動退回 `gpt-image-2`（OpenAI）。
- 輸出檔案拆分為兩種：`storyboard_panel_XX.jpg`（無標題列的乾淨圖，供 `storyboard-video` 作為 image-to-video 輸入）與 `frame_XX_標題.jpg`（含標題列，供人工檢視／Grid 總覽）。

**`storyboard-video`／`storyboard-bgm`／`storyboard-design`／`storyboard-narration`**：主要是接上 `storyboard_style.json` 讀取邏輯與共用資料夾寫入路徑的調整，行為細節無重大變動。

---

## 2026-07-02 — `storyboard-narration` 音訊同步修正

- 修正分鏡影片以 `xfade` 轉場串接時，旁白音軌不可再用 `concat` 直接首尾相接的問題：`xfade` 會讓相鄰片段重疊 0.3 秒（總長縮短 (n-1)×0.3 秒），但 `concat` 對音訊不重疊，導致每經過一次轉場音訊就比畫面多 0.3 秒、隨分鏡數增加而累積偏移（例如 5 個分鏡 = 4 次轉場 → 累積 1.2 秒），旁白最終會跟畫面對不上。
- 改為音訊也採用與 `xfade` 相同的 offset，先 `adelay` 延遲再以 `amix` 混音，取代 `concat`，確保每段旁白起始點與對應分鏡影片起始點在時間軸上完全一致。

---

## 2026-06-25 — 初始版本

- 新增 `storyboard-design`：分鏡腳本設計進入點，依文件長度建議 3/6/9/12/15 個分鏡（須為 3 的倍數）。
- 同日新增下游四技能：`storyboard-image`（分鏡圖卡生成）、`storyboard-video`（xAI 分鏡短片生成）、`storyboard-narration`（Edge-TTS 旁白＋串接）、`storyboard-bgm`（Lyria BGM 生成＋混音）。

---

## 依賴與已知限制

| 項目 | 說明 |
|---|---|
| API Key / 位址 | `GOOGLE_API_KEY`（或 `OPENAI_API_KEY` 備選，image）、`MINIMAX_SERVER_ADDRESS`（video 引擎一，地端 ComfyUI，`IP:Port`，非 API Key）、`XAI_API_KEY`（video 引擎二 Grok 備援）、`GEMINI_API_KEY`（bgm，供 Lyria 3 使用） |
| 系統工具 | `ffmpeg`／`ffprobe` 需存在於 PATH（narration、bgm 皆會檢查並在缺失時中止）；地端 ComfyUI 伺服器需已部署 MinMax H3 相關模型（video 引擎一，缺失時自動停用並全數改走 Grok） |
| Python 套件 | `google-genai` 或 `openai`（image）、`edge-tts`（narration，未安裝會自動 `pip install`）、`websocket-client`（video，供 MinMax H3 輪詢用，缺少時自動停用該引擎）、`requests` |
| 字型 | 標題列預設嘗試載入 Windows 繁體中文字型路徑（`msjh.ttc` 等），找不到則退回 PIL 預設字型 |
| 語言限制 | 全流程強制輸出繁體中文，明確禁用簡體字；圖卡以 Vision 模型驗證，最多重試 3 次 |
| 影片時長 | `storyboard-video` 每段固定 6 秒；`storyboard-bgm` 的 Lyria 3 單次生成約 20～30 秒，超過需自動分段串接 |
| 向下相容 | `storyboard-style` 為選用技能，未執行時 image／video 會退回各自原有的局部風格判斷邏輯，不會中斷流程 |
