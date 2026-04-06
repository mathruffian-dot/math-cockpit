---
name: teaching-cockpit
description: 教學駕駛艙完整生成技能。從備課 PDF 出發，自動上傳 NotebookLM 生成素材（簡報、影片、資訊圖表），下載並優化圖片，生成黑板粉筆風格的互動式教學駕駛艙 HTML，最後部署到 GitHub Pages。觸發情境：「做教學駕駛艙」、「產生駕駛艙」、「做互動教學平台」、「teaching cockpit」、「把教材做成駕駛艙」。
---

# 教學駕駛艙完整生成 (Teaching Cockpit Workflow)

## 用途

從教師提供的備課 PDF 出發，自動完成以下全流程：
1. Claude 讀取教材 + NotebookLM 生成簡報/影片/資訊圖表
2. 下載素材並優化圖片（PNG → 1920×1080 JPG）
3. 生成黑板粉筆風格的互動式教學駕駛艙（單一 HTML）
4. 部署到 GitHub Pages

## 觸發情境

當使用者說：
- 「做教學駕駛艙」
- 「把這份教材做成駕駛艙」
- 「產生互動教學平台」
- 「teaching cockpit」
- 「做一個像黑板風格的教學網頁」

## 輸入參數

使用者需提供：
- **備課 PDF 路徑**（必要）
- **單元名稱**（必要，例：「3-1 三角形與多邊形的內角與外角」）
- **YouTube 影片網址**（選填，可之後再提供）
- **GitHub 倉庫名稱**（選填，預設使用 `math-cockpit`）
- **輸出資料夾名稱**（選填，例：`3-1-triangles-polygons`）

## 執行流程（7 個階段）

### 階段 1：Claude 讀取教材 + NotebookLM 上傳

這是最關鍵的階段，Claude 必須深入理解教材內容。

**1a. Claude 讀取教材 PDF**
```python
# 使用 PyMuPDF 提取文字（因為 Read 工具可能無法讀取中文 PDF）
import fitz, sys, io
sys.stdout = io.TextIOWrapper(sys.stdout.buffer, encoding='utf-8')
doc = fitz.open(r'<PDF路徑>')
for i in range(doc.page_count):
    page = doc[i]
    text = page.get_text()
    if text.strip():
        print(f'\n=== Page {i+1} ===')
        print(text[:1500])
doc.close()
```

從教材中提取：
- 學習內容標準（如 S-8-1, S-8-2）
- 所有定理、公式、性質
- 例題與解法
- 教師備註與常見錯誤
- 生活情境與素養題

**1b. NotebookLM 上傳**
1. `mcp__notebooklm-mcp__notebook_create`（標題用單元名稱）
2. `mcp__notebooklm-mcp__source_add`（source_type=file, wait=true, wait_timeout=300）
3. `mcp__notebooklm-mcp__notebook_describe`（取得摘要）

**1c. 歸納教學重點**
結合 Claude 理解 + NotebookLM 摘要，歸納出 **5-7 個核心教學重點**。每個重點需包含：
- 重點名稱（簡短，如「三角形內角和」）
- 重點代號（如 sec1, sec2...）
- 對應的定理/公式
- 關鍵例題

> ⚠️ 若 NotebookLM 認證過期，先執行 `nlm login`（可能遇到 cp950 編碼錯誤可忽略），再 `refresh_auth`。

### 階段 2：批次生成 NotebookLM 素材

**平行啟動**（一次發出所有呼叫），每個都需 `confirm: true`、`language: "zh-TW"`：

| 素材 | 設定 |
|------|------|
| 授課簡報 | `slide_deck`, `detailed_deck`, focus: 粉筆黑板風格 20+ 頁，列出所有重點 |
| 學生自學影片 | `video`, `explainer`, `whiteboard` |
| 資訊圖表 × N | `infographic`, `sketch_note`, `landscape`, `detailed`，每個重點一張 |

**focus_prompt 範本**：

簡報：
```
製作國中數學「【單元名稱】」授課簡報，至少 20 頁以上，使用粉筆黑板風格 (chalkboard style)：深黑/墨綠色背板、手寫粉筆字體、白色與彩色粉筆線條、手繪圖形。內容須完整涵蓋以下重點：(1)【重點1】 (2)【重點2】...。每頁包含清楚的標題、定義、圖示與例題。
```

資訊圖表（每個重點）：
```
粉筆黑板風格資訊圖表 (chalkboard style：深色背景、手寫粉筆字、手繪圖形)。主題：【該重點】。呈現定義、圖示、性質、例題，16:9 橫向詳細版。
```

影片：
```
製作供學生自學使用的影片概覽，完整講解「【單元名稱】」的所有重點：(列出重點)。內容循序漸進，搭配圖形與例題說明，適合學生在家自學複習。
```

### 階段 3：下載與圖片優化

**3a. 輪詢等待**
- 每 60-90 秒檢查 `studio_status`
- 資訊圖表最快（~2 分鐘），先下載完成的
- 影片和簡報最慢（~10-15 分鐘）
- 狀態為 `unknown` 但有 URL 時不要下載，等 `completed`

**3b. 下載到本地**
```
mkdir -p <project_dir>/assets
```

| 素材 | 下載到 |
|------|--------|
| 簡報 PDF | `<project_dir>/assets/授課簡報.pdf` |
| 資訊圖表 | `<project_dir>/assets/01_重點名.png` ~ `0N_重點名.png` |
| 影片 | `<project_dir>/影片概覽_學生自學.mp4` |

**3c. 簡報 PDF → 單張圖片**
```python
import fitz
doc = fitz.open(r'<PDF路徑>')
for i, page in enumerate(doc):
    mat = fitz.Matrix(300/72, 300/72)  # 300 DPI
    pix = page.get_pixmap(matrix=mat)
    pix.save(f'assets/slide_{i+1:02d}.png')
doc.close()
```

**3d. 關鍵：PNG → 1920×1080 JPG**
```python
from PIL import Image
import os
for f in sorted(os.listdir('assets')):
    if f.endswith('.png'):
        img = Image.open(f'assets/{f}').convert('RGB')
        img = img.resize((1920, 1080), Image.LANCZOS)
        img.save(f'assets/{f.replace(".png", ".jpg")}', 'JPEG', quality=85)
```

> ⚠️ 這步驟是必要的！原始 PNG 解析度太高（~5734×3200），會導致瀏覽器渲染困難。

### 階段 4：生成教學駕駛艙 HTML

生成一個完整的單一 HTML 檔案，參考 `C:\Users\user\.claude\skills\teaching-cockpit\TEMPLATE_REFERENCE.md` 的架構。

**核心架構**：
```
HTML 結構：
├── <head>: CSS 全部內嵌（黑板粉筆風格 + 響應式 3 段 media query）
├── <body>:
│   ├── Header（木框風格，sticky）
│   ├── Mode Bar（教師/學生模式，密碼 math2026）
│   ├── 學習目標 chips
│   ├── 主題導覽 pills（錨點跳轉）
│   ├── 開場簡報翻頁器
│   ├── 每個重點 × N：
│   │   ├── 簡報翻頁器（手動 ‹ › 切換，display:none/block）
│   │   ├── Canvas 互動視覺化
│   │   ├── 形成性評量 2-3 題（記憶/理解層次）
│   │   ├── 資訊圖表（點擊全螢幕）
│   │   └── [教師模式] 教學提示
│   ├── 總結 Cheat Sheet
│   ├── 學生自學影片（video, preload=none）
│   ├── YouTube 教學影片（iframe 或佔位圖）
│   └── [教師模式] 常見錯誤、NotebookLM 連結
├── 浮動工具面板（右側）：抽籤器（可調座號）+ 計時器
├── 獨立畫筆按鈕（左下角）+ 左側垂直工具列
└── <script>: JS 全部內嵌（Canvas 互動、翻頁、工具、resize handler）
```

**設計規範**：
- 風格：深墨綠背景 `#1a2a1a`、白色粉筆文字 `#e8e4d8`、黃 `#ffd54f`、粉紅 `#ff8a80`、淺藍 `#81d4fa`
- 木框 Header：`#8d6e4a` 漸層
- 所有圖片用 `loading="lazy"` + `.jpg` 格式
- Video 使用 `preload="none"`
- 3 段 media query：平板(1024px)、手機(768px)、觸控友善(44px 最小觸控區)
- Canvas resize handler：`window.addEventListener('resize', resizeCanvases)`

**互動視覺化設計原則**：
每個重點的 Canvas 互動要根據教材內容客製化。常見類型：
- 拖曳頂點觀察角度變化
- 滑桿調整參數即時計算
- 切換按鈕比較不同概念
- 動畫展示幾何性質

**形成性評量設計原則**：
- 每個重點 2-3 題
- 以 Bloom「記憶」和「理解」層次為主
- 記憶題：直接回想定義、公式、性質
- 理解題：簡單代入計算、判斷對錯
- 4 選 1 選擇題，附解題回饋
- 題目數值不重複教材範例

**翻頁器實作**（非輪播）：
```javascript
// 使用 display:none/block 切換，非 translateX 滑動
function carGo(id, idx) {
  const imgs = document.querySelectorAll(`#${id} .carousel-inner img`);
  imgs.forEach((img, i) => img.classList.toggle('active', i === idx));
}
```

**角度弧繪製**（使用叉積判斷方向）：
```javascript
// 用叉積判斷弧的正確繪製方向
const cross = vA.x * vB.y - vA.y * vB.x;
ctx.arc(cx, cy, r, angleA, angleB, cross < 0);
```

**凹多邊形**（正確定義）：
- 凹多邊形是簡單多邊形（不自交），至少有一個內角 > 180°
- 不是星形！不能用交替半徑畫星形
- 正確做法：正多邊形但將一個頂點向中心推入

### 階段 5：GitHub 限制檢查

部署前必須檢查：

| 限制 | 數值 | 檢查方法 |
|------|------|---------|
| 單檔 < 100 MB | 拒收 | `find . -size +100M` |
| 單檔 < 50 MB | 警告 | `find . -size +50M` |
| 整站 < 1 GB | Pages 上限 | `du -sh .` |

**原則**：靜態檔（JPG、HTML）放 GitHub，影片放 YouTube。

**建立獨立的 GitHub 部署資料夾**（必要步驟！不要直接上傳工作資料夾）：
```bash
# 在專案目錄下建立 github-deploy 資料夾
mkdir -p <project_dir>/github-deploy/assets

# 只複製需要上傳的檔案（JPG + HTML）
cp <project_dir>/index.html <project_dir>/github-deploy/
cp <project_dir>/assets/*.jpg <project_dir>/github-deploy/assets/

# 確認大小符合限制
du -sh <project_dir>/github-deploy/
find <project_dir>/github-deploy/ -size +50M  # 不應有任何結果
```

**不上傳的檔案**（留在本地工作資料夾）：
- `*.png`（原始高解析度圖片）
- `*.mp4`（NotebookLM 影片）
- `*.pdf`（簡報 PDF）
- 其他大型素材

### 階段 6：GitHub 部署

**6a. 若倉庫已存在（如 math-cockpit）**：
```bash
cd <repo_dir>
mkdir -p <unit_folder>/assets
cp <deploy_dir>/index.html <unit_folder>/
cp <deploy_dir>/assets/*.jpg <unit_folder>/assets/
# 更新首頁 index.html 加入新單元連結卡片
git add <unit_folder>/ index.html
git commit -m "feat: 新增【單元名稱】教學駕駛艙"
git push
```

**6b. 若倉庫不存在**：
```bash
gh repo create <repo_name> --public --description "國中數學教學駕駛艙"
# 建立首頁 index.html（單元導覽頁）
# 建立 .github/workflows/deploy.yml
# git init → add → commit → push
# gh api 啟用 Pages (build_type=workflow)
```

**首頁導覽卡片模板**：
```html
<a class="unit-card" href="<folder>/index.html">
  <h2>📐 【單元名稱】</h2>
  <p>【重點摘要】</p>
  <span class="badge">【年級學期】</span>
  <span class="badge">【章節】</span>
</a>
```

**GitHub Actions workflow**：
```yaml
name: Deploy to GitHub Pages
on:
  push:
    branches: [main]
permissions:
  contents: read
  pages: write
  id-token: write
jobs:
  deploy:
    environment:
      name: github-pages
      url: ${{ steps.deployment.outputs.page_url }}
    runs-on: ubuntu-latest
    steps:
      - uses: actions/checkout@v4
      - uses: actions/configure-pages@v5
      - uses: actions/upload-pages-artifact@v3
        with:
          path: '.'
      - id: deployment
        uses: actions/deploy-pages@v4
```

### 階段 7：產出摘要

最終回報：
- NotebookLM 筆記本連結
- GitHub 倉庫連結
- GitHub Pages 網站連結（首頁 + 單元頁）
- 產出檔案清單（本地保留的完整版 + GitHub 部署的精簡版）

## 可調整選項

| 選項 | 預設值 | 說明 |
|------|--------|------|
| 簡報風格 | chalkboard | 可選 bento_grid / professional / anime 等 |
| 資訊圖表風格 | sketch_note | 與簡報統一風格 |
| 影片風格 | whiteboard | 可選 classic / cinematic 等 |
| 簡報頁數 | 20+ | 可自訂 |
| 形成性評量題數 | 每重點 2-3 題 | 記憶/理解層次 |
| 座號範圍 | 1-30 | 可調整 |
| YouTube 影片 | 佔位圖 | 使用者提供網址時才嵌入 |

## 注意事項

1. **PyMuPDF 和 Pillow 必須已安裝**：`pip install pymupdf pillow`
2. **NotebookLM 認證**：過期時 `nlm login` → `refresh_auth`
3. **圖片一定要轉 JPG**：原始 PNG 太大會導致瀏覽器渲染困難
4. **影片不上傳 GitHub**：放 YouTube 不公開，用 iframe 嵌入
5. **Video preload=none**：防止大檔案自動載入卡頓
6. **所有圖片加 loading="lazy"**：避免一次載入太多圖片
7. **HTML 中所有路徑用 .jpg 副檔名**：不要用 .png
8. **Canvas 互動需做觸控支援**：touchstart/touchmove/touchend
9. **外角弧用叉積判斷方向**：避免畫錯弧的方向
10. **凹多邊形不是星形**：要用正確的凹入頂點方式表示
