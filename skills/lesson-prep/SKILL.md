---
name: lesson-prep
description: 國中/高中備課教材自動化生成技能。當使用者提及「製作備課教材」、「備課」、「備課教材」、「產生教學素材」、「把這份教材做成備課包」或類似請求,並提供課本 PDF / Word / 備課資料時,請一定要使用此技能。會自動上傳至 NotebookLM 並產出:粉筆黑板風格授課簡報 (20+ 頁)、學生自學影片、音訊總覽、各重點資訊圖表 (16:9 詳細)、Bloom 三層次差異化教材 (記憶理解/應用/分析)、三種難度測驗各 10 題並匯出試算表,最後將所有內容下載至本地。
---

# 備課教材自動化生成 (Lesson Prep Workflow)

## 用途

針對教師提供的備課 PDF/Word 檔案,自動透過 NotebookLM 產生完整的備課教材包,涵蓋簡報、影片、音訊、資訊圖表、差異化教材與測驗。

## 觸發情境

當使用者說:
- 「幫我製作備課教材」
- 「把這份教材做成備課包」
- 「備課:3-1 三角形與多邊形」
- 「產生教學素材」
- 或上傳一份教材檔案並要求製作教學材料

## 預設產出項目 (共 15 項作品)

1. **授課簡報** × 1 — 粉筆黑板風格 (chalkboard)、至少 20 頁、格式 detailed_deck
2. **影片概覽** × 1 — explainer / whiteboard 風格,供學生自學
3. **音訊總覽** × 1 — deep_dive 深度討論
4. **資訊圖表** × N — 每個課程重點一張,粉筆黑板 sketch_note 風格、16:9 橫向、detailed
5. **差異化教材報告** × 3 — Bloom 三層次:A 記憶與理解、B 應用、C 分析
6. **測驗** × 3 — easy/medium/hard 各 10 題,下載為 CSV 試算表

## 執行流程 (Workflow)

### 步驟 1:分析教材並規劃重點

1. 使用 `Glob` / `Read` 定位使用者提供的教材檔案
2. 若未建立筆記本,呼叫 `mcp__notebooklm-mcp__notebook_create` 建立新筆記本 (標題用教材主題)
3. 呼叫 `mcp__notebooklm-mcp__source_add` 上傳檔案 (source_type=file, wait=true, wait_timeout=300)
4. 呼叫 `mcp__notebooklm-mcp__notebook_describe` 取得摘要
5. 根據摘要與教材主題,歸納出 **5-7 個核心重點**,作為資訊圖表主題

### 步驟 2:平行啟動所有作品生成

所有 `studio_create` 呼叫都是非阻塞的,請用單一訊息同時發出多個呼叫以節省時間。每次呼叫都需 `confirm: true`、`language: "zh-TW"`。

**(a) 授課簡報**
```
studio_create(
  artifact_type="slide_deck",
  slide_format="detailed_deck",
  slide_length="default",
  focus_prompt="製作國中/高中【主題】授課簡報,至少 20 頁以上,使用粉筆黑板風格 (chalkboard style):深黑/墨綠色背板、手寫粉筆字體、白色與彩色粉筆線條、手繪圖形。內容須完整涵蓋:(列出所有重點)。每頁包含清楚的標題、定義、圖示與例題。"
)
```

**(b) 影片概覽**
```
studio_create(
  artifact_type="video",
  video_format="explainer",
  visual_style="whiteboard",
  focus_prompt="製作供學生自學使用的影片概覽,完整講解【主題】的所有重點(列出重點)。內容循序漸進,搭配圖形與例題說明,適合學生在家自學複習。"
)
```

**(c) 音訊總覽**
```
studio_create(
  artifact_type="audio",
  audio_format="deep_dive",
  audio_length="default",
  focus_prompt="深入討論【主題】課程重點,包含(列出重點)。"
)
```

**(d) 資訊圖表 × N (每個重點一張)**
對每個重點呼叫一次:
```
studio_create(
  artifact_type="infographic",
  orientation="landscape",
  detail_level="detailed",
  infographic_style="sketch_note",
  focus_prompt="粉筆黑板風格資訊圖表 (chalkboard style:深色背景、手寫粉筆字、手繪圖形)。主題:【該重點】。呈現定義、圖示、性質、例題,16:9 橫向詳細版。"
)
```

**(e) 差異化教材報告 × 3 (Bloom 三層次)**
```
# A - 記憶與理解
studio_create(
  artifact_type="report",
  report_format="Create Your Own",
  custom_prompt="請根據 Bloom 認知層次的「記憶 (Remember) 與理解 (Understand)」層次,為【主題】設計差異化教材。內容需包含:(1)課程重點定義條列 (2)基本定理與公式清單 (3)白話文說明與簡單圖示描述 (4)基礎辨識與回想練習(填空、配對、是非題)。適合需要鞏固基礎的學生使用。請用繁體中文撰寫。"
)

# B - 應用
studio_create(
  artifact_type="report",
  report_format="Create Your Own",
  custom_prompt="請根據 Bloom 認知層次的「應用 (Apply)」層次,為【主題】設計差異化教材。內容需包含:(1)應用題型分類 (2)每類 2-3 個範例題與完整解題步驟 (3)關鍵技巧提示 (4)實際生活情境應用 (5)中等難度練習題。適合已具備基礎、需要練習應用的學生。請用繁體中文撰寫。"
)

# C - 分析
studio_create(
  artifact_type="report",
  report_format="Create Your Own",
  custom_prompt="請根據 Bloom 認知層次的「分析 (Analyze)」層次,為【主題】設計差異化教材。內容需包含:(1)證明題與推理題 (2)進階題目與關鍵技巧分析 (3)複合題型分析 (4)邏輯推理與反例題 (5)開放性探究題。適合學力較佳、需要挑戰高層次思考的學生。請用繁體中文撰寫。"
)
```

**(f) 測驗 × 3 (10 題每份)**
```
# Easy - 記憶理解
studio_create(artifact_type="quiz", question_count=10, difficulty="easy",
  focus_prompt="記憶與理解層次測驗:【主題】,10 題簡易測驗,著重定義辨識、公式記憶、基本性質理解。")

# Medium - 應用
studio_create(artifact_type="quiz", question_count=10, difficulty="medium",
  focus_prompt="應用層次測驗:【主題】,10 題中等難度,著重公式應用與計算,要求套用定理解題。")

# Hard - 分析
studio_create(artifact_type="quiz", question_count=10, difficulty="hard",
  focus_prompt="分析層次測驗:【主題】,10 題高難度,著重推理、證明、複合題分析,要求綜合運用多個定理。")
```

### 步驟 3:輪詢作品生成狀態

1. 呼叫 `studio_status` 檢查進度
2. 作品依速度依序完成 (通常順序:infographic → report → quiz → audio → video → slide_deck)
3. 可使用 `Bash sleep 60` 等待,每 1-2 分鐘檢查一次
4. 所有 15 個作品 `status: "completed"` 時進入下一步

### 步驟 4:建立 downloads/ 資料夾並下載所有內容

```bash
mkdir -p "<project_dir>/downloads"
```

下載每個作品:

| 類型 | 工具呼叫 | 輸出檔名 |
|------|---------|---------|
| 6+ 資訊圖表 | `download_artifact(artifact_type="infographic")` | `01_XX.png` ~ `06_XX.png` |
| 3 報告 | `download_artifact(artifact_type="report")` | `報告_A_記憶理解.md` 等 |
| 3 測驗 | `download_artifact(artifact_type="quiz", output_format="json")` | `quiz_easy_記憶理解.json` 等 |
| 影片 | `download_artifact(artifact_type="video")` | `影片概覽_學生自學.mp4` |
| 音訊 | `download_artifact(artifact_type="audio")` | `音訊總覽.mp3` |
| 簡報 | `download_artifact(artifact_type="slide_deck", slide_deck_format="pdf")` 與 `pptx` | `授課簡報_粉筆黑板風格.pdf` / `.pptx` |

**注意:影片/音訊有時需等作品狀態真正為 `completed` 才能下載,unknown 狀態可能 500 錯誤。遇到下載失敗時等 30-60 秒再重試。**

### 步驟 5:將測驗 JSON 轉為 CSV 試算表

使用以下 Python 腳本 (寫入 `convert_quiz_to_csv.py` 後執行):

```python
import json, csv
from pathlib import Path

quiz_files = [
    ("quiz_easy_記憶理解.json", "quiz_easy_記憶理解.csv", "記憶與理解"),
    ("quiz_medium_應用.json", "quiz_medium_應用.csv", "應用"),
    ("quiz_hard_分析.json", "quiz_hard_分析.csv", "分析"),
]
base = Path("<downloads path>")
all_rows = [["難度層次","題號","題目","選項A","選項B","選項C","選項D","正確答案","解析","提示"]]
for jn, cn, lv in quiz_files:
    data = json.loads((base/jn).read_text(encoding="utf-8"))
    with open(base/cn, "w", encoding="utf-8-sig", newline="") as f:
        w = csv.writer(f)
        w.writerow(["題號","題目","選項A","選項B","選項C","選項D","正確答案","解析","提示"])
        for idx, q in enumerate(data.get("questions", []), 1):
            opts = q.get("answerOptions", [])
            while len(opts) < 4: opts.append({"text":"","isCorrect":False,"rationale":""})
            letters = ["A","B","C","D"]
            correct = ""; rat = ""
            for i, o in enumerate(opts[:4]):
                if o.get("isCorrect"): correct = letters[i]; rat = o.get("rationale","")
            row = [idx, q.get("question",""), opts[0]["text"], opts[1]["text"], opts[2]["text"], opts[3]["text"], correct, rat, q.get("hint","")]
            w.writerow(row)
            all_rows.append([lv] + row)
with open(base/"quiz_合併_三種難度.csv", "w", encoding="utf-8-sig", newline="") as f:
    csv.writer(f).writerows(all_rows)
```

### 步驟 6:將 MD 報告轉為 Word (.docx)

若 `python-docx` 未安裝,先 `pip install python-docx`。
使用自訂 Markdown→DOCX 轉換腳本 (支援 #標題、**粗體**、*斜體*、`code`、$LaTeX$、表格、引用、列表),將:
- `報告_A_記憶理解.md` → `報告_A_記憶理解.docx`
- `報告_B_應用.md` → `報告_B_應用.docx`
- `報告_C_分析.md` → `報告_C_分析.docx`

字型使用 `Microsoft JhengHei` (繁體中文)。

### 步驟 7:提供最終清單摘要

用繁體中文回報所有產出檔案清單,以分類呈現 (簡報/影片/音訊/資訊圖表/差異化教材/測驗),並附上 NotebookLM 筆記本連結。

## 可調整選項

使用者可能指定不同:
- **風格**:chalkboard / bento_grid / professional / anime / watercolor 等 (infographic_style)
- **影片視覺**:whiteboard / classic / anime / cinematic (visual_style)
- **音訊格式**:deep_dive / brief / critique / debate (audio_format)
- **簡報頁數**:若使用者指定頁數,在 focus_prompt 明確要求
- **測驗題數**:question_count 可自訂

若使用者未指定,一律套用預設 (粉筆黑板風格、20+頁簡報、10題測驗)。

## 注意事項

- 若 NotebookLM 認證過期,呼叫 `refresh_auth` 或提示使用者執行 `nlm login`
- 作品 title 會由 NotebookLM 自動生成中文標題,不用擔心初始顯示英文主題
- 上傳 PDF 時務必 `wait=true` 等待處理完成,否則後續生成會缺資料
- 影片/音訊生成最久,可能需要 5-10 分鐘,先下載其他完成項再回頭處理
- 所有檔案下載至專案目錄下的 `downloads/` 子資料夾
