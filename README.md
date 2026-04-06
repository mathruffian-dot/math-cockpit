# 📐 國中數學教學駕駛艙

互動式黑板粉筆風格教學平台 — 由 NotebookLM + Claude Code 自動生成

## 線上預覽

- **首頁**：https://mathruffian-dot.github.io/math-cockpit/
- **3-1 三角形與多邊形的內角與外角**：https://mathruffian-dot.github.io/math-cockpit/3-1-triangles-polygons/

## 功能特色

- 黑板粉筆風格互動教學平台
- 每個教學重點含：簡報翻頁器、Canvas 互動視覺化、形成性評量、資訊圖表
- 浮動工具面板：抽籤器（可調座號範圍）、計時器
- 獨立畫筆工具（5 色粉筆）
- 教師 / 學生模式切換
- 響應式設計，支援平板操作
- YouTube 影片嵌入

## 倉庫結構

```
math-cockpit/
├── index.html                     # 首頁（單元導覽）
├── 3-1-triangles-polygons/        # 教學駕駛艙
│   ├── index.html
│   └── assets/
├── skills/                        # Claude Code Skills
│   ├── README.md                  # 安裝說明
│   ├── teaching-cockpit/SKILL.md  # 駕駛艙生成 skill
│   └── lesson-prep/SKILL.md       # 備課素材生成 skill
└── .github/workflows/deploy.yml   # 自動部署
```

## Claude Code Skills

本倉庫包含兩個可分享的 Claude Code Skills，詳見 [skills/README.md](skills/README.md)。

## 授權

MIT License
