# vs-window-screenshot

> 在 Windows 上程序化地截出「某个窗口此刻的正确画面」。

一个面向 AI 编程助手（Claude Code / DSH）的技能包：把 VS2010/VS2022 等桌面程序窗口、以及经典控制台窗口，自动化截成可交付的图片——绕开 Windows Terminal 委派、抢焦点、启动画面、GBK 编码这些隐蔽陷阱。

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

## 三条截图路线的选择依据

| 方法 | 可靠性 | 适用 | 已知限制 |
|---|---|---|---|
| `PrintWindow(h, hdc, flags)` | **最高** | 绝大多数 GUI 程序 | 对控制台**返回失败** |
| `CopyFromScreen` | 中 | 控制台必须用它 | 需要窗口在前台可见 |
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

**2. 等待窗口就绪要「两段」**

只等 `MainWindowHandle` 是错的。IDE 启动瞬间句柄就有了，但画面还是启动图。必须再等 Splash 类窗口消失。

**3. 控制台要用句柄差集定位**

`SetWindowPos` 前先快照已有控制台句柄，只认「快照里没有的那个新句柄」。按类名查找会抓到历次残留窗口，截出来是旧画面。

**4. Windows 11 的控制台抓不到**

Win11 默认把控制台委派给 Windows Terminal（`CASCADIA_HOSTING_WINDOW_CLASS`），`PrintWindow` 抓不到、标题不稳定。解法是用 `conhost.exe` 主动起一个经典 `ConsoleWindowClass` 窗口。

**5. 截图前必须点一下窗口拿焦点**

否则 `SendKeys` 会打到别处。

**6. 不要 `Stop-Process conhost`**

会把托管自身会话的控制台一起杀掉，之后所有命令报 `0xE9`。只关自己启动的那个窗口。

**7. 脚本正文只用 ASCII**

PowerShell 5.1 会把脚本里的中文按 GBK 误读，文件名变成 `raw_缁冧範...png`。中文文件名交给编码正确的父进程事后重命名。

---

## 内容

```
.
├── SKILL.md      # 技能主文件：完整流程、代码片段、错误对照表
├── README.md     # 你正在看的这份
└── .gitignore    # 排除截图等产物
```

`SKILL.md` 结构：

| 章节 | 内容 |
|---|---|
| 何时用 | 四类典型场景 |
| 铁律 | 7 条血泪条款 |
| 截图总原则 | 三条路线对照表 |
| 抓 GUI 窗口 | 启动等待、PrintWindow、裁剪 |
| 抓经典控制台 | conhost、句柄差集、焦点、保窗 |
| 编码陷阱 | PowerShell 5.1 / MSVC 的 GBK 问题 |
| 用例：C 作业 | 源码规范、编译、打包、验收清单 |
| 错误对照表 | 11 条：现象 → 根因 → 处置 |

---

## 用例：C 语言作业截图（本技能附带的一个完整样例）

这是把通用方法论落地的完整链路，也是本技能最初诞生的场景：

1. 写 `.c` 源码（UTF-8 with BOM，首行 `_CRT_SECURE_NO_WARNINGS`）
2. 用 VS2010 `cl` 编译，期望 0 警告 0 错误
3. `PrintWindow` 抓 IDE 代码窗口
4. `conhost.exe` + `CopyFromScreen` 抓运行输出
5. `Compress-Archive` 打包

**但它只是用例，不是边界**——任何「程序跑完要出图」的交付都适用同一套方法。

---

## 环境要求

| 项 | 要求 |
|---|---|
| 系统 | Windows 10 / 11（Win11 的 Terminal 委派是核心坑点） |
| IDE | VS2010 或 VS2022（flags 不同） |
| Shell | PowerShell 5.1 / 7 |
| 编码 | 源码 UTF-8 BOM，控制台 GBK(936) |

**注意**：`SKILL.md` 里的路径（`D:\Program Files (x86)\Microsoft Visual Studio 10.0\...`）是示例，换机器需改。

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

## 相关技能

本技能与 `c-homework-vs2010` 的关系：后者是**具体作业流程**（含命名规范、打包、验收），前者是**通用截图方法论**。要交 C 作业时用后者更省事；要解决其他截图问题时用前者。

---

## 许可

个人自用，未附加许可证。如需复用请自行判断。
