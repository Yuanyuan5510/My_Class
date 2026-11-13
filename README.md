# My class
- 一个暂未开源的班级管理系统

这里存有编译版本与介绍

---

### 1.班级抽签 · Guess One

教室管理系统（当前核心：随机抽取学生）。**Electron(WebView) + Python 后端 + Vite/React 前端** 三层架构。

### 架构

```
┌─────────────────────────────────────────────────────────┐
│ Electron 外壳（WebView 宿主）                              │
│  · 主窗口 / 右缘页签窗口 / 托盘 / 开机自启 / UAC 提权        │
├──────────────────────────┬──────────────────────────────┤
│ Python 后端（独立进程）     │ 前端（Vite + React + TS 构建） │
│  · http.server + REST API │  · 抽取舞台 / 学生管理          │
│  · SQLite 单文件库 (WAL)   │  · JSON 批量导入 / 记录 / 设置  │
│  · Win32: 提权/自启/资源管理│  · 仅 35KB 业务代码，零 UI 依赖 │
└──────────────────────────┴──────────────────────────────┘
```

- **通信**：前端经 `fetch` 调用 `http://127.0.0.1:<随机端口>/api/*`，本机回环 + 一次性 Token（Cookie 注入）双重限制。
- **数据目录**：`%APPDATA%\Wang\MyClass\Guess\one`（即 `C:\Users\<你>\AppData\Roaming\Wang\MyClass\Guess\one`）
  - `myclass.db` — SQLite（WAL 模式）
  - `logs\app.log`、`imports\`、`exports\`、`runtime.json`
  
### 更多
- 制作中