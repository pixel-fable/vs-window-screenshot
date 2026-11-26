# vs-window-screenshot

> 在 Windows 上程序化地截出「某个窗口此刻的正确画面」。

一个面向 AI 编程助手（Claude Code / DSH）的技能包：把 VS2010/VS2022 等桌面程序窗口、以及经典控制台窗口，自动化截成可交付的图片——绕开 Windows Terminal 委派、抢焦点、启动画面、GBK 编码这些隐蔽陷阱。

> **v2（2026-11-26）**：修正了一处被实测推翻的断言（见下「更正」），并并入原 `c-homework-vs2010` 的完整 C 作业流程。

---

## 解决什么问题

**核心难点不在写代码，在截图。**

编译是机械活，运行是机械活。真正耗时间的是这四件事：

- 窗口没出来就截了（截到启动画面）
- 截到的是桌面或其他窗口，不是目标窗口
- 截到空屏
- 中文全变乱码

这个技能把踩过的坑全部固化成了可判定的规则。

---

## ⚠️ 更正：PrintWindow 与经典控制台

**旧版说**：「`PrintWindow` 对控制台**返回失败**，必须用 `CopyFromScreen`」。

**实测**（Windows 11 + PowerShell 5.1）：在 `conhost.exe` 起的经典 `ConsoleWindowClass` 上，
`PrintWindow(h, dc, 0)` **返回 TRUE，画面完整**（992×517，文字清晰）。

真正失败的是**被 Windows Terminal 委派**的控制台（`CASCADIA_HOSTING_WINDOW_CLASS`）。

| 场景 | PrintWindow |
|---|---|
| 经典 `ConsoleWindowClass`（conhost 起的） | ✅ 可用，实测通过 |
| Win11 Terminal 委派的控制台 | ❌ 失败 |

**结论**：能用 conhost 起窗就用 `PrintWindow`——不抢焦点、不闪屏；`CopyFromScreen` 只作 fallback。

---

## 三条截图路线的选择依据

| 方法 | 可靠性 | 适用 | 已知限制 |
|---|---|---|---|
| `PrintWindow(h, hdc, flags)` | **最高** | 绝大多数 GUI 程序；**经典控制台也可用** | 对 Terminal 委派的控制台失败 |
| `CopyFromScreen` | 中 | 需要前台像素时；`PrintWindow` 失败时 | 会被遮挡、抢焦点 |
| `BitBlt` | 中 | 备选 | 同样依赖可见性 |

**`PrintWindow` 的根本优势**：不依赖前台、不抢焦点、不怕被别的窗口遮挡。这是它比截全屏强的地方。

---

## 最关键的几条（完整版见 `SKILL.md`）

**1. VS2010 和 VS2022 的 `PrintWindow` flags 不一样**

| 版本 | `nFlags` |
|---|---|
| VS2010 | `0` |
| VS2022 | `2` |

写 `0` 抓 VS2022 会得到**黑图或白图**——这是最容易被误判成「程序坏了」的坑。

**2. 等待窗口就绪要「三段」，不是两段**

只等 `MainWindowHandle` 是错的。IDE 启动瞬间句柄就有了，但画面还是启动图。
**更糟的是句柄可能来自一个错误对话框**——实测 `VCExpress.exe <路径>` 会弹「未知错误」，
窗口类 `#32770`，尺寸 241×163。

所以第 2 段必须核对类名：

```
类名含 "#32770" → 启动失败，别拍照，报错退出
```

**3. 控制台要用句柄差集定位**

前先快照已有控制台句柄，只认「快照里没有的那个新句柄」。按类名查找会抓到历次残留窗口，截出来是旧画面。

**4. Windows 11 的控制台要主动起 conhost**

Win11 默认把控制台委派给 Windows Terminal（`CASCADIA_HOSTING_WINDOW_CLASS`），`PrintWindow` 抓不到、标题不稳定。
解法是用 `conhost.exe` 主动起一个经典 `ConsoleWindowClass` 窗口。

**5. 不要 `Stop-Process conhost`**

会把托管自身会话的控制台一起杀掉，之后所有命令报 `0xE9`。只关自己启动的那个窗口。

**6. 脚本正文只用 ASCII**

PowerShell 5.1 会把脚本里的中文按 GBK 误读，文件名变成 `raw_缁冧範...png`。
中文文件名交给编码正确的父进程事后重命名。

**7. `cl` 的编译日志是 UTF-8，不是 GBK**

这一条是 v2 新增的实测发现：

```
同一个「例1.1.c」，日志里写的是文件名字节 E4 BE 8B
  按 UTF-8 解 → 例1.1.c      ✅
  按 GBK(936) 解 → 渚?.1.c   ❌
```

**只有控制台运行时输出是 GBK(936)，`cl` 的编译日志是 UTF-8**，两者不能混用。
旧文档写成"捕获输出后按 GBK 解码"，会把正常日志显示成乱码。

---

## 内容

```
.
├── SKILL.md      # 技能主文件：完整流程、代码片段、错误对照表、C 作业全流程
├── README.md     # 你正在看的这份
└── .gitignore    # 排除截图等产物
```

`SKILL.md` 结构：

| 章节 | 内容 |
|---|---|
| 何时用 | 四类典型场景 |
| 铁律 | 9 条血泪条款 |
| 截图总原则 | 三条路线对照表（含实测更正） |
| 一、抓 GUI 窗口 | 三段等待、PrintWindow、裁剪 |
| 二、抓经典控制台 | conhost、句柄差集、焦点、保窗 |
| 三、编码陷阱 | PowerShell 5.1 / MSVC / cl 日志编码 |
| 四、环境要求 | |
| 五、错误对照表 | 13 条：现象 → 根因 → 处置 |
| 六、C 作业全流程 | 源码规范、编译、截图、打包、验收清单 |

---

## 用例：C 语言作业截图（完整流程见第六节）

这是把通用方法论落地的完整链路：

1. 写 `.c` 源码（UTF-8 with BOM，首行 `_CRT_SECURE_NO_WARNINGS`）
2. 用 VS2010 `cl` 编译，期望 0 警告 0 错误
3. `PrintWindow` 抓 IDE 代码窗口
4. `conhost.exe` + `PrintWindow`/`CopyFromScreen` 抓运行输出
5. `Compress-Archive` 打包

**但它只是用例，不是边界**——任何「程序跑完要出图」的交付都适用同一套方法。

---

## 环境要求

| 项 | 要求 |
|---|---|
| 系统 | Windows 10 / 11（Win11 的 Terminal 委派是核心坑点） |
| IDE | VS2010 或 VS2022（flags 不同） |
| Shell | PowerShell 5.1 / 7 |
| 编码 | 源码 UTF-8 BOM；控制台输出 GBK(936)；cl 日志 UTF-8 |

**注意**：`SKILL.md` 里的 VS 安装路径是示意，换机器需改成自己的安装位置。

---

## 怎么用

### 在 DSH 里

```
%USERPROFILE%\.dsh\skills\vs-window-screenshot\SKILL.md
```

### 在 Claude Code 里

```
%USERPROFILE%\.claude\skills\vs-window-screenshot\SKILL.md
```

### 手动使用

不需要自动加载的话，直接把 `SKILL.md` 内容贴给 AI，说「按这个流程做」也可以。

---

## 与 c-homework-vs2010 的关系

**本仓库已并入原 `c-homework-vs2010` 的全部内容**（第六节）。
单独的 C 作业流程不再单独维护，请以本仓库为唯一来源。

原仓库 [`pixel-fable/c-`](https://github.com/pixel-fable/c-) 保留作历史归档，内容与之等价，
但缺少 v2 的两处更正（PrintWindow 与控台、cl 日志编码）。**新用户请直接用本仓库。**

---

## 相关技能

- `plc-ladder-diagram` —— 画梯形图
- `plc-review` —— 审梯形图
- `corpus-reading` —— 读大批材料
- `adversarial-review` —— 对抗评审

---

## 许可

MIT
