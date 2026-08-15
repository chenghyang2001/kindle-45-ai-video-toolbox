# 剪映 TTS 字幕逐字對齊（不用語音辨識）

**日期**：2026-08-15
**環境**：家用機 Yama-Desktop / 剪映（CapCut）桌面版台灣版 / Windows 10
**驗證**：同一流程連跑兩份不同文字，兩次都做到「字幕串接 == 原文，逐字相同」

---

## 一句話結論

**不要用「識別字幕」處理自己 TTS 出來的語音。**
原文明明在手上，識別字幕卻繞去聽聲音重猜一遍，繁簡、同音字、英文全部會壞掉。
正解是**自己產 SRT 匯入**：文字 100% 由你指定，只用音檔的**停頓**決定時間點。

---

## 一、問題：為什麼「識別字幕」一定對不齊

### 實測錯誤（第一次，55 字含 Claude / Anthropic）

| 原文 | 識別結果 | 錯誤類型 |
| --- | --- | --- |
| Claude | `clause` | 英文聽錯 |
| Anthropic | （整段消失） | 英文漏掉 |
| 程式設計 | `城市设计` | **同音字**（chéngshì shèjì 完全同音） |
| 擅長寫作 | `擅长写作` | 簡體 |
| 遇到／說明／確認 | `遇到／说明／确认` | 簡體 |

斷句也錯：`擅长写作题` / `译与城市设` / `计也能阅读文件` —— 從「翻譯」「設計」的**字中間**切開。
55 字被切成 10 段，沒有一段對得上句子邊界。

### 根本原因

「識別字幕」是**語音辨識（ASR）**，它不看你的文字，只聽 TTS 唸出來的聲音重新轉一次：

```
你的文字  →（TTS）→  聲音  →（ASR）→  猜出來的文字
         ↑ 資訊在這一步就丟了
```

聲音裡**不存在**這些資訊，所以 ASR 再準也救不回來：

- 沒有「繁體/簡體」→ 輸出簡體
- `程式` 和 `城市` 發音完全相同 → 只能猜
- `Claude` 聽起來像 clause → 就變 clause

這是**方法走錯方向**，不是辨識率高低的問題。

### 為什麼不用「文稿匹配」

文稿匹配（把文字稿對齊語音時間軸）正是為此設計的功能，但**這台剪映台灣版沒有**。
`字幕` 分頁只有四項：自動字幕 / 範本 / 自動歌詞 / **新增字幕（匯入 SRT、LRC、ASS）**。

所以走 **新增字幕 → 匯入 SRT**。這反而更好：時間點與文字都完全由我們決定。

---

## 二、完整流程

### 步驟 1：文字進剪映

`文字` → `新增文字` → 滑鼠移到「預設文字」上，右下角浮出 **＋** → 點它
→ 右側編輯框全選（Ctrl+A）後貼上你的文字

### 步驟 2：產生語音

右側 `文字轉語音` 分頁 → 搜尋框輸入音色名（例：`解說小帥`）→ 點選音色
→ 面板**最底部**的 `產生語音`

### 步驟 3：產 SRT（本文重點，見第三節）

```bash
# 音檔位置（剪映把 TTS 存在專案資料夾底下）
%USERPROFILE%\AppData\Local\CapCut\User Data\Projects\com.lveditor.draft\<專案>\textReading\*.wav

# 偵測停頓
ffmpeg -nostdin -i "<音檔>" -af "silencedetect=noise=-35dB:d=0.15" -f null - 2>&1 | grep silence_
```

再用第三節的腳本把停頓對應到句子，輸出 SRT。

### 步驟 4：匯入

`字幕` → `新增字幕` → `匯入檔案` → 選 SRT（進素材庫，還沒上時間軸）
→ **按 Home 把播放頭歸零** → 滑鼠移到 SRT 卡片上 → 點 **＋**

### 收尾

步驟 1 那段長文字只是 TTS 的輸入，留著會整段疊在畫面上 → **刪掉**。
存檔後從 `draft_content.json` 驗證（見第五節）。

---

## 三、DP 對齊邏輯（核心）

### 問題

靜音偵測切出的語音段數，**通常多於句子數**。因為 TTS 不只在句號停，
**頓號（、）逗號（，）也會停**。

實測：

| 次數 | 句子數 | 語音段數 | 多出來的那段 |
| --- | --- | --- | --- |
| 第一次 | 5 | 6 | 第 **2** 句的頓號（「寫作、」） |
| 第二次 | 5 | 6 | 第 **4** 句的頓號（「顏色、」） |

**多出來的位置每次都不一樣**，所以不能寫死。靠人工指認 = 換一份文字就要重看一次。

### 解法

把「N 段語音切成 M 個**連續**群組」當成最佳化問題，用動態規劃求解：

**目標**：每個群組的「時長佔比」要盡量接近對應句子的「字數佔比」。

```
cost = Σ ( 群組時長/總時長 − 句子字數/總字數 )²
```

**為什麼字數比例可靠**：同一個 TTS 音色語速穩定。
實測各段落都落在 **6.5 字/秒**附近（句尾那段略慢，約 5.2），
變異小到足以讓比例匹配唯一收斂。

關鍵在於——**我們不需要知道它「說了什麼」，只需要知道它「說到哪裡」**。
這就是能繞開語音辨識的原因。

### 完整程式碼

```python
"""剪映 TTS 語音 → 逐字對齊的 SRT

用法：改 TXT 與 WAV 兩個變數後執行。
不做語音辨識，只用停頓位置 + 字數比例對齊。
"""
import re
import subprocess
import sys

TXT = '好的字幕能讓影片更容易理解。字體不宜太小。每行最好控制在十五字以內。顏色、亮度都要與背景形成對比。適時斷句能減輕觀看負擔。'
WAV = r'<剪映專案>\textReading\xxxx.wav'
OUT = 'subtitle.srt'
TAIL_PAD = 0.26          # 最後一句往後延的秒數（讓字幕不要跟語音同時消失）


def detect_speech(wav: str) -> list[tuple[float, float]]:
    """用 ffmpeg silencedetect 反推語音段 [(start, end), ...]"""
    try:
        r = subprocess.run(
            ['ffmpeg', '-nostdin', '-i', wav,
             '-af', 'silencedetect=noise=-35dB:d=0.15', '-f', 'null', '-'],
            capture_output=True, encoding='utf-8', errors='replace')
    except FileNotFoundError:
        print('錯誤：找不到 ffmpeg', file=sys.stderr)
        sys.exit(1)
    log = r.stderr or ''
    starts = [float(x) for x in re.findall(r'silence_start:\s*([\d.]+)', log)]
    ends = [float(x) for x in re.findall(r'silence_end:\s*([\d.]+)', log)]
    if not starts or not ends:
        print('錯誤：偵測不到靜音，音檔可能全程有底噪', file=sys.stderr)
        sys.exit(1)
    sil = sorted(zip(starts, ends))
    # 語音段 = 相鄰兩段靜音之間
    return [(sil[i][1], sil[i + 1][0]) for i in range(len(sil) - 1)]


def align(segs, sents):
    """DP：把 N 段語音切成 M 個連續群組，最小化『時長佔比 vs 字數佔比』平方誤差。

    回傳每個句子的 (start, end)。
    """
    n, m = len(segs), len(sents)
    if n < m:
        print(f'錯誤：語音段({n}) 少於句子數({m})，'
              '可能是句子太短沒停頓，或 silencedetect 門檻太嚴', file=sys.stderr)
        sys.exit(1)
    dur = [b - a for a, b in segs]
    total_d, total_c = sum(dur), sum(len(s) for s in sents)
    INF = float('inf')
    dp = [[INF] * (m + 1) for _ in range(n + 1)]
    bk = [[None] * (m + 1) for _ in range(n + 1)]
    dp[0][0] = 0.0
    for i in range(1, n + 1):
        for j in range(1, m + 1):
            for k in range(j - 1, i):            # 第 j 組 = segs[k..i-1]
                if dp[k][j - 1] == INF:
                    continue
                d = sum(dur[k:i]) / total_d
                c = len(sents[j - 1]) / total_c
                cost = dp[k][j - 1] + (d - c) ** 2
                if cost < dp[i][j]:
                    dp[i][j] = cost
                    bk[i][j] = k
    groups, i, j = [], n, m
    while j > 0:
        k = bk[i][j]
        groups.append((k, i))
        i, j = k, j - 1
    groups.reverse()
    return [(segs[a][0], segs[b - 1][1]) for a, b in groups]


def ts(t: float) -> str:
    """秒 → SRT 時間碼 HH:MM:SS,mmm"""
    m, s = divmod(t, 60)
    h, m = divmod(int(m), 60)
    return f'{h:02d}:{int(m):02d}:{int(s):02d},{int(round((s - int(s)) * 1000)):03d}'


def main():
    sents = [s + '。' for s in TXT.split('。') if s]
    segs = detect_speech(WAV)
    print(f'語音段 {len(segs)} / 句子 {len(sents)}')
    bounds = align(segs, sents)

    # 字幕首尾相接：每句延到下一句開始，最後一句加 TAIL_PAD
    starts = [b[0] for b in bounds]
    lines = []
    for i, ((a, b), txt) in enumerate(zip(bounds, sents), 1):
        end = starts[i] if i < len(starts) else b + TAIL_PAD
        lines += [str(i), f'{ts(a)} --> {ts(end)}', txt, '']

    with open(OUT, 'w', encoding='utf-8', newline='\r\n') as f:
        f.write('\n'.join(lines))

    # 驗證：串接後必須與原文逐字相同
    assert ''.join(sents) == TXT, '句子切分後與原文不符'
    print(f'已寫出 {OUT}，共 {len(sents)} 句，驗證通過')


if __name__ == '__main__':
    main()
```

---

## 四、踩坑清單

| 坑 | 症狀 | 解法 |
| --- | --- | --- |
| **SRT 插在播放頭位置** | 整批字幕跑到影片最後面 | 加入前**先按 Home 歸零** |
| 匯入後沒進時間軸 | 只在素材庫看到卡片 | 滑鼠移上去點 **＋**（跟貼紙一樣） |
| 中文輸入法吃掉數值 | 輸入框變方塊亂碼 | 數值欄一律走**剪貼簿**貼上，不要用 `keybd_event` 打字 |
| 語音段數 ≠ 句子數 | 對不齊 | 用 DP 自動合併，不要寫死 |
| 長文字疊在畫面上 | 字幕跟整段原文同時顯示 | TTS 產完就把那段輸入文字刪掉 |
| `。` 結尾切分產生空字串 | 多一個空句子 | `if s` 過濾（程式碼已處理） |

---

## 五、驗證方式

**不要只看畫面**，直接讀專案 JSON 比對：

```python
import json, os
p = os.path.expanduser(
    '~/AppData/Local/CapCut/User Data/Projects/com.lveditor.draft/<專案>/draft_content.json')
d = json.load(open(p, encoding='utf-8'))
texts = {t['id']: json.loads(t['content'])['text'] for t in d['materials']['texts']}
rows = []
for tr in d['tracks']:
    if tr['type'] == 'text':
        for s in sorted(tr['segments'], key=lambda x: x['target_timerange']['start']):
            rows.append(texts.get(s['material_id'], ''))
print('串接 == 原文 :', ''.join(rows) == ORIG)
```

⚠️ 驗證前先在剪映按 **Ctrl+S**，否則讀到的是舊檔。
⚠️ 順便確認每條 `tracks[].attribute == 0`（`2` 代表整條軌道不顯示）。

---

## 六、兩次實測數據

### 第一次：`0815 (4)` — 含英文的文字

原文（55 字）：
> Claude 是 Anthropic 開發的人工智慧助理。它擅長寫作、翻譯與程式設計。也能閱讀文件並整理重點。遇到不確定的事會直接說明。重要內容仍需人工確認。

音檔 13.03s，6 段語音 → 5 句（頓號在第 2 句）

| 段 | 時間 |
| --- | --- |
| 1 | 0.233 – 2.900 |
| 2 | 2.933 – 6.033 |
| 3 | 6.067 – 8.400 |
| 4 | 8.400 – 10.800 |
| 5 | 10.800 – 12.867 |

### 第二次：`0815 (5)` — 純中文

原文（55 字）：
> 好的字幕能讓影片更容易理解。字體不宜太小。每行最好控制在十五字以內。顏色、亮度都要與背景形成對比。適時斷句能減輕觀看負擔。

音檔 12.13s，6 段語音 → 5 句（頓號在第 4 句）

| 段 | 時間 |
| --- | --- |
| 1 | 0.233 – 2.700 |
| 2 | 2.700 – 4.133 |
| 3 | 4.167 – 6.467 |
| 4 | 6.467 – 9.600 |
| 5 | 9.600 – 11.967 |

**兩次都驗證：串接 == 原文（True），中文字數 55。**

---

## 七、環境備註

- 剪映**編輯器**主視窗是 **Qt6**（class `Qt622QWindowIcon`），滑鼠座標自動化有效。
  （既有記錄的「Electron 打不到渲染層」只適用**首頁的範本瀏覽器**，不適用編輯器。）
- 本文的螢幕座標是 3840×1080 雙螢幕、剪映在右螢幕最大化時量的，**換機器必須重量**。
- 搶焦點不要用 `ShowWindow(SW_RESTORE)`，會把最大化視窗還原掉、座標全失效；用 `SW_MAXIMIZE`。
- 截圖用 Python `PIL.ImageGrab.grab(all_screens=True)`，比 PowerShell 組路徑穩。

---

## 相關文件

- `doc/capcut-螢幕錄影標註-handoff-2026-08-14.md` — 螢幕錄影標註（紅框／馬賽克）
- `doc/capcut-render-hang-fix.md` — 匯出卡住的處理
