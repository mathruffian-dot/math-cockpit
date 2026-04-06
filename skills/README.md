# Claude Code Skills — 教學駕駛艙工具包

這個資料夾包含兩個 Claude Code Skills，用於自動化生成國中數學互動式教學駕駛艙。

## Skills 清單

| Skill | 說明 | 觸發語句 |
|-------|------|---------|
| **teaching-cockpit** | 完整教學駕駛艙生成（PDF → HTML → GitHub Pages） | 「做教學駕駛艙」、「把教材做成駕駛艙」 |
| **lesson-prep** | NotebookLM 備課素材生成（簡報、影片、圖表、測驗） | 「備課」、「製作備課教材」 |

## 安裝方式

### 方法 1：複製到 Claude Code skills 資料夾

```bash
# Windows
copy skills\teaching-cockpit\SKILL.md %USERPROFILE%\.claude\skills\teaching-cockpit\SKILL.md
copy skills\lesson-prep\SKILL.md %USERPROFILE%\.claude\skills\lesson-prep\SKILL.md

# macOS / Linux
cp skills/teaching-cockpit/SKILL.md ~/.claude/skills/teaching-cockpit/SKILL.md
cp skills/lesson-prep/SKILL.md ~/.claude/skills/lesson-prep/SKILL.md
```

### 方法 2：用 Claude Code 安裝

在 Claude Code 中輸入：
```
幫我安裝 https://github.com/mathruffian-dot/math-cockpit 的 skills
```

## 前置需求

### Python 套件
```bash
pip install pymupdf pillow python-docx
```

### CLI 工具
- [GitHub CLI (gh)](https://cli.github.com/) — 用於倉庫管理和 Pages 部署
- [NotebookLM CLI (nlm)](https://github.com/nicholasgasior/notebooklm-cli) — 用於 NotebookLM API

### MCP Servers
- `notebooklm-mcp` — NotebookLM 整合

## 工作流程概覽

```
備課 PDF
    │
    ▼
┌─────────────┐     ┌──────────────────┐
│ lesson-prep │ ──▶ │ NotebookLM 素材   │
│  (選用)     │     │ 簡報/影片/圖表    │
└─────────────┘     └──────┬───────────┘
                           │
                           ▼
                ┌─────────────────────┐
                │ teaching-cockpit    │
                │ 完整駕駛艙生成      │
                │                     │
                │ 1. 讀取教材         │
                │ 2. 生成 NB 素材     │
                │ 3. 下載+優化圖片    │
                │ 4. 生成互動 HTML    │
                │ 5. GitHub 限制檢查  │
                │ 6. 部署 Pages       │
                └─────────┬───────────┘
                          │
                          ▼
                ┌─────────────────────┐
                │ GitHub Pages 網站   │
                │ 互動式教學駕駛艙    │
                └─────────────────────┘
```

## 產出範例

- [教學駕駛艙首頁](https://mathruffian-dot.github.io/math-cockpit/)
- [3-1 三角形與多邊形的內角與外角](https://mathruffian-dot.github.io/math-cockpit/3-1-triangles-polygons/)
