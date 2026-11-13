# 班级抽签 · Guess One

教室管理系统（当前核心：随机抽取学生）。**Electron(WebView) + Python 后端 + Vite/React 前端** 三层架构。

## 架构

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

## 统一数据结构

所有业务表共用同一套列：`id`(uuid) / `created_at` / `updated_at` / `deleted_at`(软删) / `extra`(JSON)。

| 表 | 用途 |
|---|---|
| `classes` | 班级 |
| `students` | 学生（学号/姓名/性别/小组/标签/权重/累计抽中） |
| `draw_sessions` / `draw_items` | 抽取批次与明细（保留姓名快照） |
| `settings` | 键值设置（JSON value） |
| `events` | 审计日志（导入/错误/提权） |

局部唯一索引保证“删除后可重建同名记录”，复合索引保证抽取与搜索走索引扫描。

## 功能

- **随机抽取**：三种模式 —— 均衡优先（抽得少者优先）/ 不放回（一轮不重复，抽完自动开新一轮）/ 纯随机；一次 1~5 人；揭晓动画 + 撒花 + 音效；`Alt+Enter` 快速抽取，`Ctrl+Alt+G` 全局显示/隐藏。
- **JSON 批量导入**：粘贴 / 拖拽文件，支持 `{"class":"...","students":[...]}`、对象数组、纯字符串数组，键名中英文均可；三种策略（合并/仅新增/覆盖），带实时解析预览与结果统计。
- **屏幕边缘页签**：常驻屏幕边缘，悬停展开（带平滑展开/收回动画），一键抽取、显示在池人数与轮次；**可拖动**（按住竖条或展开面板顶栏），松手后自动**磁吸**到光标所在显示器的左/右屏幕边缘，停靠左侧时面板方向自动镜像。
- **托盘常驻**：关闭窗口 = 驻留托盘；托盘菜单可抽取、开关页签（**与页签实际显示状态实时同步**）、打开数据目录、退出。
- **开机自启**：默认开启（登录 Windows 后带 `--hidden` 后台启动），可在设置中关闭。
- **权限自愈**：任何“权限不足 / 无法访问”错误都会返回 `ELEVATION_REQUIRED`，界面弹窗引导 UAC 提权，确认后整个应用以管理员身份自动重启。

## 变更记录

### 1.0.1
- 修复：鼠标悬浮页签时反复展开/收回的抖动 —— 悬停判定从渲染层 DOM 事件（窗口缩放会导致光标反复进出，形成死循环）改为**主进程光标坐标轮询判定**，状态未变化时直接忽略（防抖），实测悬停只展开一次、移开 400ms 后收回一次。
- 动画：展开 easeOutCubic / 收回 easeInOutCubic 分步 `setBounds`（10ms 步进）；收回时页面内容延后 170ms 再切回竖条，与窗口收缩同步“退回去”，不再出现内容先消失的空窗。
- 修复：页签页面 CSP 报错（`Refused to execute inline script`）—— 移除后端向 HTML 注入的内联 token 脚本，统一改为同源 Cookie 鉴权（fetch 默认携带）。
- 版本号收敛为单一来源 `backend/core/version.py`（界面左下角/设置页随版本自动更新）。

### 1.0.0
- 首个发布版本（见上文功能列表）。


## 使用

```bat
scripts\install.bat   rem 首次：建 venv、装依赖、构建前端（加 --check 只做环境自检）
scripts\run.bat       rem 启动应用（加 --check 只校验运行文件是否齐全）
```

### 脚本编码约定（重要）

Windows `cmd.exe` 是按**当前系统代码页**（中文系统为 936/GBK）解析批处理文件字节的，文件里出现 UTF-8 中文必然乱码，甚至被当成命令而报错。本项目因此约定：

- `scripts\*.bat` **只允许纯 ASCII + CRLF 换行（无 BOM）**，只做控制流；所有中文提示交给 Python（`scripts/setup.py`）打印。
- Python 打印前调用 `backend/core/console.py`：
  - 输出到真实控制台 → 跟随 `GetConsoleOutputCP()`（936 就用 `cp936`），因此任何中文系统都不会乱码；
  - 输出到管道/文件（Electron 读握手、重定向日志）→ 统一 UTF-8，与外壳的 `toString("utf-8")` 一致。
- `run.bat` 会主动清空 `ELECTRON_RUN_AS_NODE` / `NODE_OPTIONS`：若这两个变量从外部环境继承进来，Electron 会被降级成纯 Node，报 `app is undefined`。
- 回归守卫：`python tests\encoding_test.py` 会校验 bat 是否纯 ASCII + CRLF + 无 BOM，以及中文提示能否被 cp936 无损表示。

开发：

```bash
npm run build:web          # 构建前端到 frontend/dist
.venv\Scripts\python.exe backend\main.py   # 仅启动后端（浏览器可访问 API）
python tests\api_test.py       # 后端 33 项端到端测试
python tests\encoding_test.py  # 脚本编码回归测试
python scripts\seed_demo.py <数据目录>       # 灌演示数据
```

Electron 主进程读取后端握手 JSON（`{"event":"ready","port":...}`）后创建窗口；后端崩溃自动重启（最多 3 次），父进程退出后端随之退出，不留孤儿进程。

## 打包发布

```bat
scripts\package.bat            # 一键：前端构建 -> PyInstaller 后端 exe -> NSIS 安装包
scripts\package.bat --backend  # 只重建后端 exe
scripts\package.bat --setup    # 只重建安装包
```

产物在 `release\`：`GuessOne-Setup-<版本>.exe`（NSIS 安装器，96MB，可自定义安装目录）与 `win-unpacked\`（免安装目录）。

打包结构：

| 位置 | 内容 |
|---|---|
| `app.asar` | Electron 外壳（main.js / preload.js） |
| `resources\frontend-dist\` | 前端构建产物（Python 后端无法读 asar，故放外部） |
| `resources\backend\guess-backend\` | PyInstaller 打包的 Python 后端（onedir，无需装 Python） |
| `resources\assets\` | 图标资源 |

- 安装后任务栏 / 启动项 / 任务管理器均显示 **“班级抽签 Guess One”**（exe 元数据 ProductName + FileDescription 已写入；开发模式下直跑 electron.exe 时进程名仍显示 Electron，属正常，安装版无此问题）。
- 打包脚本会把所有缓存与临时目录重定向到项目 `.cache\`（不写 C 盘），并通过 npmmirror 镜像下载 Electron / NSIS 工具链。
- 重新打包时 electron-builder 需要先删除旧的 `release\win-unpacked`。若该目录刚被运行过，Windows 可能仍持有 `app.asar` 句柄导致删除失败（`EBUSY` / trash 失败）——此时换一个输出目录 `--config.directories.output=release2`，或重启后再打包即可。
