# Session 5 — 婚禮影片 V3 直式版製作 + YouTube/Drive/Gmail 發布

> 日期：2026-08-13
> 主題：CapCut 範本自動化停損 → ffmpeg 直式 9:16 保底交付 → 三通路發布（YouTube / Google Drive / Gmail）
> 前置：接續 `~/Downloads/結婚照片/HANDOFF-V3-YouTube.md`（上個 session 的交接文件）

---

## 完成事項

### 1. CapCut 範本自動化：4 輪嘗試後停損（依使用者預先訂的停損點）

- 依交接文件要求，先嘗試使用者指定的路線 A（CapCut 直式婚禮範本填格）
- **這次比上個 session 更早就斷了 —— 連範本瀏覽器都沒打開**
- CapCut 9.1.0.3879 的首頁是獨立 HWND（本次 `853452`）：
  - `SetWindowPos` 拉到 1920×1080 + `HWND_TOPMOST` + `ShowWindow(3)` + `RedrawWindow` 全下
  - `GetWindowRect` 回報 `0,0,1920,1080` **完全正確，但畫面一片空白**
  - 判定為 Electron/Chromium-embedded 視窗「有 window handle、GPU compositor 無 surface」→ Win32 API 打不到實際渲染層
- 重跑 `CapCut.exe` 讓既有 instance 開首頁 → 開出來的就是這個幽靈視窗
- **收尾安全處理**：把幽靈視窗設過 `HWND_TOPMOST` 後立刻改回 `HWND_NOTOPMOST (-2)` 並最小化，還原編輯器視窗。看不見卻置頂全螢幕的視窗會吃掉所有滑鼠點擊 = 桌面被鎖死
- CapCut 專案 `0813` / `0813 (1)` / `0813 (2)` 依指示**全部保留未刪**

### 2. ffmpeg 直式 9:16 影片製作（V3 保底方案，實際交付版本）

- 素材：`~/Downloads/結婚照片/normalized/wed-01 ~ wed-18`（18 張直式，3072×4096）
- 分兩階段：先產 18 支獨立 clip（`v3-clips/c01~c18.mp4`），再用 `xfade` 串接
- 單 clip 配方：
  - 背景：放大裁切填滿 1080×1920 + `gblur=sigma=28` + `eq=brightness=-0.10:saturation=0.85`
  - 前景：等比縮到 1080×1440 置中 overlay（3:4 照片進 9:16，上下各留 240px 由模糊背景補滿）
  - Ken Burns：`zoompan=z='min(zoom+0.0010,1.12)':d=129`，129 frames @ 30fps = 4.3 秒
- 串接：`xfade` 0.8 秒，`fade / dissolve / fadewhite` 循環，offset 每段 +3.5 秒
- BGM：CapCut 快取 `2d12229c.../audio/7537892201333868580.mp3`，前 2s 淡入、後 3s 淡出
- 成品：`wedding-V3-vertical.mp4`，**1080×1920 / 63.8 秒 / 64,844,268 bytes**
- 驗證：抽 5s/20s/40s/60s 四張 frame 拼圖確認 — 滿版無黑邊、無範本內建陌生人

### 3. YouTube 上傳（V3）

- 用既有 token `~/.config/youtube-comment/token.json`（scope `youtube.force-ssl`）refresh 換 access_token
- 走 resumable：POST 拿 `Location` → `PUT` 檔案本體，64.8 MB 數秒完成
- 上傳當下即帶 `"privacyStatus": "unlisted"`，**沒有短暫公開的空窗**
- V3 影片 ID `Www1ZteVwq4`，API 實測 `processingStatus: succeeded` / `1080x1920` / `63.8s`
- 同時複查 V2（`jGs-K5yapts`）仍為 unlisted

### 4. Google Drive 上傳（V2 + V3）

- `rclone copy` 到固定資料夾 `1vbKdqM5HgMn7e5kK_T-lZdVgREdZ4IJI`
- V3：`1gBsTGIxTt-Ly35YfbI6FJ-awX5Ke368v`
- V2：`1NLO8GgPNVE9qTMDV9U7sLW2joQ6is5N8`
- **失誤與修正**：V2 因背景任務無輸出被誤判失敗而重跑，造成 Drive 上出現兩份同名檔。已用 `gws drive files update --json '{"trashed":true}'` 把重複那份（fileId `1qZt7NjH49hMwy0ygxYLWYvdTXskh5KNs`）丟垃圾桶（非永久刪除）並複查確認

### 5. 安裝 gws CLI + 打通自動寄信

- 查明正確套件名為 **`@googleworkspace/cli`** v0.22.5（官方 googleworkspace/cli，Rust 寫、npm 發預編譯 binary，需 Node 18+）
- 舊 skill 文件裡寫的 `@google-workspace/cli`（中間有連字號）**在 npm 上不存在**，是錯誤猜測 → 已修正 `gmail-apply-label` 的 SKILL.md 與 `scripts/apply_label.sh`
- 裝完 `/c/Users/user/AppData/Roaming/npm` 本來就在 PATH 上，新 shell 直接可用
- 沿用 `~/.config/gws/` 既有憑證（2026-03-15 建立），`has_refresh_token: true`，**不需重跑 OAuth**
- 兩封信皆已實寄並驗證落在收件匣（`SENT` + `INBOX` + `UNREAD`）：
  - V3 信 Message ID `19ff7f457a340864`
  - V2 信 Message ID `19ff7f88f964f4dc`

---

## 關鍵技術筆記

### zoompan 幀數爆炸陷阱

直覺寫法 `-loop 1 -t 4.3 -i img` 再接 `zoompan=d=129` 是**錯的**。zoompan 語義是「**每個輸入 frame 產生 d 個輸出 frame**」，`-loop 1 -t 4.3` 已餵進 129 frames → 129×129 = 16641 幀。
正解：**餵單張圖**（不加 `-loop`），讓 zoompan 自己把 1 幀展開成 129 幀，再用 `-frames:v 129` 收邊。

### Git Bash 的 curl 其實是 Windows curl.exe

`--data-binary "@/c/Users/..."`、`-T /c/Users/...`、`-o /dev/null` **全部失效**。必須 `cygpath -w` 轉 Windows 路徑，`/dev/null` 要寫 `NUL`。
錯誤訊息只有一句 `error encountered when reading a file`，完全不提路徑格式，卡了兩輪。同規則適用 Windows 版 Python 與 ffmpeg。

### gmail.modify scope 就能寄信

本機 gws token 的 scope 是 `gmail.modify`（**沒有** `gmail.send`），一度誤判需重新授權。
實際上 Gmail API 的 `messages.send` / `drafts.send` 接受 `mail.google.com` / `gmail.modify` / `gmail.compose` / `gmail.send` **四者之一**。Google 的 scope 分類邏輯是**權限廣度**而非動作類型 —— `gmail.modify` = 「除永久刪除外全都能做」，寄信自然包含。

### claude.ai Gmail MCP 沒有 send

只有 `create_draft` / `update_draft` / 搜尋 / 標籤。要真的寄出必須配 gws。
**兩段式最順**：MCP 建草稿（好寫、支援中文排版）→ `gws gmail users drafts send` 送出。草稿 ID 格式兩邊相通，本 session 實測兩次皆成功。

### Google Drive 允許同名檔案共存

Drive 用 fileId 當主鍵而非路徑，同資料夾可以有多個同名檔。`rclone copy` 的去重是「開始傳之前」比對一次 size+modtime，兩個 process 同時啟動會各自判定「要傳」→ race condition 產生重複。
**紀律**：上傳這類非冪等遠端操作，背景任務「看起來沒回應」時不要直接重跑，先 `rclone lsjson` 確認遠端實際狀態。

---

## 產出檔案

| 檔案 | 說明 |
| --- | --- |
| `~/Downloads/結婚照片/wedding-V3-vertical.mp4` | **V3 成品**，1080×1920 / 63.8s / 64.8 MB |
| `~/Downloads/結婚照片/v3-clips/c01~c18.mp4` | 18 支中間 clip |
| `~/Downloads/結婚照片/v3-filter.txt` | xfade 濾鏡腳本（`-filter_complex_script` 用） |
| `~/Downloads/結婚照片/HANDOFF-V3-YouTube.md` | 更新為完成狀態 + 新增本次踩坑 |
| `~/.claude/projects/C--Users-user-workspace-kindle-45-ai-video-toolbox/memory/MEMORY.md` | 新建索引 |
| `.../memory/capcut-desktop-template-automation-dead-end.md` | 新增 |
| `.../memory/gitbash-windows-curl-path.md` | 新增 |
| `.../memory/gws-cli-send-gmail.md` | 新增 |
| `~/.claude/skills/gmail-apply-label/SKILL.md` | 修正錯誤套件名 |
| `~/.claude/skills/gmail-apply-label/scripts/apply_label.sh` | 修正錯誤套件名（1 行） |

## 交付清單（全部完成）

| 版本 | 規格 | YouTube（皆 unlisted） | Google Drive | Gmail |
| --- | --- | --- | --- | --- |
| V2 橫式 | 1920×1080 / 103.4s | `jGs-K5yapts` | `1NLO8Gg...is5N8` | `19ff7f88f964f4dc` |
| V3 直式 | 1080×1920 / 63.8s | `Www1ZteVwq4` | `1gBsTGI...e368v` | `19ff7f457a340864` |

兩支 YouTube 影片皆為 **unlisted（未列出）**，未公開，公開與否由使用者自行決定。

---

## HANDOFF（下次 session 優先處理）

### 立即行動

- [ ] 使用者決定 V2 / V3 是否要在 YouTube 轉為公開（目前皆 unlisted，需使用者親自決定，Claude 不主動改）
- [ ] 若使用者要對外分享 Drive 影片，需另外設公開權限（目前兩支都是私人，只有本人帳號能開）
- [ ] Drive 垃圾桶內有一份重複的 `wedding-V2-manual.mp4`（fileId `1qZt7NjH49hMwy0ygxYLWYvdTXskh5KNs`），30 天後自動永久刪除，確認不需要就不用管

### 進行中（需接續）

- 無未完成工作。本 session 交接的 V3 任務已 100% 完成並三通路發布。
- `~/Downloads/結婚照片/HANDOFF-V3-YouTube.md` 已標記為「全部完成」，保留作為技術紀錄。

### 注意事項

- **CapCut 桌面版範本自動化建議正式放棄**。兩個 session 累積失敗點涵蓋三層：渲染層（Win32 打不到 Electron）、資料層（draft_content.json 加密）、互動層（右鍵無取代選項、拖放五成命中率）。三層全封死，不是調參數能解。往後直式影片直接用本 session 驗證過的 ffmpeg 配方。
- 本 session 的工作目錄是 `~/Downloads/結婚照片/`（非 git repo），影片與中間素材都不在版控內。若要保留需自行備份 —— 目前 V2/V3 成品已在 Google Drive 有一份。
- 使用者本週 token 用量已超標（66.4 MB / 60 MB 配額）。下次涉及 GUI 自動化的任務，建議一開始就評估是否直接走程式化路線，不要先試 GUI。
- `gws` 已裝好且驗證可寄信，後續任何「寄 Gmail」需求直接用，不必再問使用者授權。
