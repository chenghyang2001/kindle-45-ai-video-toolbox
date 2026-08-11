# Session 1 — 2026-08-11

## 完成事項

- **研讀第 1 章前言逐字稿**：讀取 `kindle-45-transcripts/01 前言與目錄.txt`（NotebookLM 語音摘要對談稿，VPS faster-whisper 批次轉錄產物），確認內容為書籍第四版前言與目錄的概念導覽對談，涵蓋「搜尋時代→生成時代」「AI 二廚」「關鍵幀」「遮罩」「自動字幕」「AI Relight」等主題，並向使用者摘要重點。
- **CapCut Live Demo 三 Module 全套示範**：使用者已開啟 `capcut.exe`，先試 `youtube-ai演練` skill 被使用者喊停，改派正確的 `capcut-chapter-livedemo` skill 執行：
  - 依逐字稿點名的三個真實 CapCut 功能設計課程大綱：關鍵幀動畫／遮罩合成／自動字幕（不強求逐字稿本身是逐步教學，而是把概念導覽對談轉譯成可操作 Module）
  - **Phase 2 環境準備**：EnumWindows 抓到 CapCut 主視窗（多螢幕環境，座標 `(1912,-117,3848,939)`），發現專案是預設日期式命名 `0811 (1)`，依 skill 鐵律右鍵重新命名為 `前言概念-關鍵幀遮罩字幕`
  - **Module 1 關鍵幀**：貼圖庫加入星星貼圖 → 在右側屬性面板「轉換」區塊為 位置／縮放 各自的獨立菱形圖示分別在 t=0（X=-300, 縮放50%）與 t≈2s（X=300, 縮放150%）建立關鍵幀，驗證畫面平滑插值動畫
  - **Module 2 遮罩**：媒體庫資料庫加入一段人物影片，找到藏在「遮罩轉場」子分頁（非獨立頂層分類）的遮罩功能，套用圓形遮罩 + 羽化 30 呈現柔邊局部顯示效果
  - **Module 3 自動字幕**：素材是純文字逐字稿沒有現成人聲，改用 Phase 2「次選」策略——把文字圖層內容改成一句話 → 用「文字轉語音」（解說小廚聲線）生成配音 → 對新音軌跑「自動字幕」，成功產出兩段與語音波形對齊的字幕
  - 每個 Module 完成後用 `AskUserQuestion` 互動檢查點，使用者三次都選「繼續下一個 Module」
  - Phase 4 收尾：寫入 `doc/demo-recordings/preface-toc-2026-08-11/notes.md`（含 3 個 Module 的操作步驟、關鍵設定值、關鍵結果截圖、講解逐字稿、踩坑記錄），順手清掉 Git Bash 崩潰殘留的 `bash.exe.stackdump` 並補進 `.gitignore`，commit + push
- **答覆關鍵幀操作追問**：使用者問「關鍵幀菱形控制在哪裡加、有沒有快捷鍵」，依本次實測結果回覆位置在右側屬性面板（非時間軸窗格本身，但建好後時間軸上會顯示菱形標記），並誠實告知沒有實測過鍵盤快捷鍵、不確定是否存在，主動提議可以去 CapCut 設定裡查快捷鍵表。

## 關鍵技術筆記（CapCut 桌面自動化，本次新發現）

1. **數值輸入框不能用 `keybd_event` 模擬鍵盤打數字**：即使是純數字/負號也可能被目前鍵盤/IME 狀態吃成亂碼中文字（實測 `-300` 打出「爾马」）。正解跟 CJK 文字輸入一致：`Set-Clipboard` 放進剪貼簿 → `Ctrl+A` 全選 → `Ctrl+V` 貼上 → `Tab` 確認（不按 Enter，Enter 會還原舊值）。
2. **CapCut 桌面版「遮罩」功能藏在影片/圖片屬性面板的「遮罩轉場」子分頁**（基礎／移除背景／**遮罩轉場**／潤飾），不是獨立頂層分類，命名容易誤導。
3. **關鍵幀控制是逐屬性獨立的**：縮放／位置／旋轉各自右側都有專屬的 `‹ ◇ ›` 控制，不是單一全域按鈕；要做「移動+縮放」同動動畫需要兩個屬性都各自點一次菱形。
4. **兩種素材庫的「加入時間軸」互動深度不同**：貼圖/文字/特效庫是「先選再加」兩段式（單擊只預覽，再單擊一次才冒出「+」）；媒體庫（資料庫）的影片/照片素材是單擊直接顯示「+」。
5. **自動字幕辨識輸出可能是簡體字**，即使來源文字圖層與 TTS 語音都是繁體中文/標準中文發音——語音辨識模型的輸出字集跟輸入文字字集是分開的兩件事。
6. **「文字轉語音」是文字圖層專屬的頂層分頁**（跟「文字」「插入動畫」「追蹤」同排），只有選取文字類型片段時才會出現。

## 產出檔案

| 檔案 | 說明 |
| --- | --- |
| `doc/demo-recordings/preface-toc-2026-08-11/notes.md` | 3 個 Module 完整複習筆記 |
| `doc/demo-recordings/preface-toc-2026-08-11/screenshots/module1-keyframe-kf2.png` | 關鍵幀動畫關鍵結果截圖 |
| `doc/demo-recordings/preface-toc-2026-08-11/screenshots/module2-mask-circle-feather.png` | 遮罩合成關鍵結果截圖 |
| `doc/demo-recordings/preface-toc-2026-08-11/screenshots/module3-autocaption-result.png` | 自動字幕關鍵結果截圖 |
| `.gitignore` | 新增排除 `*.stackdump` / `core` / `*.core` / `*.dmp` |
| CapCut 專案：`前言概念-關鍵幀遮罩字幕` | 本機 CapCut 專案檔（非版控，本機 `%LOCALAPPDATA%\CapCut\User Data\Projects\`） |

Git commit：`35e3249`（新增《前言與目錄》CapCut live demo 複習筆記）

## HANDOFF（下次 session 優先處理）

### 立即行動

- [ ] 若要繼續 Kindle 45 書籍的其他章節 live demo，可比照本次 SOP（`capcut-chapter-livedemo` skill）直接套用到第 2 章以後的逐字稿
- [ ] 若使用者想確認 CapCut 是否有「新增關鍵幀」的鍵盤快捷鍵，可開 CapCut「選單→設定→快捷鍵」查詢並回報（本次未查證）
- [ ] 全書 15 章語音摘要逐字稿已於上個 session 批次轉錄完成，尚未逐章做 live demo，可依使用者需求排程後續章節

### 進行中（需接續）

- 無未完成的進行中工作，本 session 三個 Module 與筆記均已完整收尾並 push。

### 注意事項

- 專案有兩個容易混淆的資料夾：`kindle-45-AI影片工具箱`（存放原始素材/逐字稿，非 git 專案）與 `kindle-45-ai-video-toolbox`（本 git repo，工作目錄）；兩者路徑不同、不是同一個目錄，下次 session 若要找逐字稿要去前者，寫檔案/commit 要在後者。
- 本機 CapCut 是多螢幕環境，視窗座標會飄移，操作前務必重新 `EnumWindows` 取得實際 rect，不可沿用先前記住的座標（`capcut-chapter-livedemo` skill 的 `desktop-automation-lessons.md` 已有完整踩坑清單）。
- 本 session 是這個專案第一次寫 session summary，`~/.claude/projects/C--Users-B00332-workspace-kindle-45-ai-video-toolbox/memory/` 目錄尚未建立，Memory Keeper 需要新建。
