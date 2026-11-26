---
name: vs-window-screenshot
description: 在 Windows 上自动化截取 VS2010/VS2022 等桌面程序窗口与经典控制台画面，绕开 Windows Terminal 委派、抢焦点、启动画面、GBK 编码等陷阱。适用于任何需要"程序运行后出图"的交付场景；文末附 C 语言作业（VS2010）的编译/截图/打包全流程。
---

# VS/桌面程序窗口自动化截图（Windows）

> 本文含 C 语言作业（VS2010）的完整流程（第六节），修正了一条被实测推翻的断言（第二节）。

## 何时用

需要在 Windows 上**程序化地**得到"某个窗口在某一刻的正确画面"时用本技能：

- 交作业：`.c` 源码 + IDE 代码截图 + 运行结果截图，打包提交（**第七节完整用例**）
- 做文档：给操作手册、教学材料配程序界面图
- 做回归：UI 变更前后对比、给 bug 报告配现场图
- 做演示：批量生成一系列步骤图

**核心难点不在写代码，在截图。** 编译、运行都是机械活；真正耗时间的是"窗口没出来 / 截到桌面 / 截成空屏 / 中文全乱"。

---

## 铁律（血泪条款，按顺序执行）

1. **先看现状，再动手。** 列桌面、读已有的产出文件夹，确认命名风格（`例1.9.png` 点号？`练习2-7.png` 横杠？）和图片类型。**照抄现有风格，不自创。**

2. **不动用户的文件夹结构与命名。** 需要放新内容时，**先问**：并入原文件夹，还是新建独立文件夹。
   **改名、删除、覆盖成品之前先问，并留下可执行的复原方法。**

3. **破坏性操作一律走回收站**，并明确回复「已移入回收站，可恢复」。
   恢复要用 Shell.Application **遍历 verbs 按名字匹配**，不能索引：
   ```powershell
   foreach ($x in $it.Verbs()) { if ("$($x.Name)" -replace '&','' -match '还原|Restore') { $x.DoIt(); break } }
   ```
   （`$verb[0].DoIt()` 会抛 `DISP_E_NOTACOLLECTION`。）

4. **每张截图必须当场读一遍**（`read_image`）。批量跑完再统一看 = 连废 3~4 轮。

5. **不要用 `Stop-Process conhost`**：会把托管自身会话的控制台一起杀掉，之后所有命令报
   `0xE9`「管道的另一端上无任何进程」。只关自己启动的那个窗口（`PostMessage WM_CLOSE`）。

6. **不要删 `.exe`**：清临时文件时别 `Remove-Item *.exe`，否则后续「不是内部或外部命令」，截出来是空屏。

7. **VS2010 启动方式是坑**：`Stop-Process VCExpress` 会杀掉用户已经打开的 IDE；而 `VCExpress.exe <文件>` 在
   本机会弹「未知错误」对话框（窗口类 `#32770`）。**先看 `MainWindowHandle` 的类名是不是 IDE 类，
   是对话框就别拍照**（见第一节「等待」）。

8. **能手工点就别写一堆胶水**，但编译/运行批量跑更快——**难的是截图，不是编译**。
   用户说「在软件里编」就是让你少造轮子：能用 IDE 图形操作就别写一堆命令行胶水；
   编译本身仍用 `cl` 批量，跟他解释清楚「难的是截图不是编译」。

9. **放新内容的目录先问**：原文件夹保持原样，新作业默认放 `学号姓名_练习X.Y-...` 的独立文件夹，
   别把新文件混进他既有命名体系（**改名/移动前先问，并留下复原方法**）。

---

## 截图总原则：三条路，按可靠性排序

| 方法 | 可靠性 | 适用 | 坑 |
|---|---|---|---|
| `PrintWindow(h, hdc, flags)` | **高** | GUI 窗口；**经典控制台也可用（实测）** | 对 Win11 委派给 Terminal 的控制台失败 |
| `CopyFromScreen` | 中 | 需要窗口在前台可见；`PrintWindow` 失败时用它 | 会被遮挡、抢焦点 |
| `BitBlt` | 中 | 备选 | 同样依赖可见性 |

**`PrintWindow` 的最大好处是不依赖前台、不抢焦点、不怕被遮挡**——这是它比截全屏强的根本原因。

> **实测更正（2026-11-26，本机 Win11 + PowerShell 5.1）**：旧版本文三处写着"PrintWindow 对控制台返回失败"，
> 实测在 `conhost.exe` 起的经典 `ConsoleWindowClass` 上 **`PrintWindow(h, dc, 0)` 返回 TRUE，画面完整**
> （992×517，文字清晰）。真正失败的是被 Windows Terminal 委派的控制台。
> → 结论：**能用 conhost 起窗就用 PrintWindow，抢不到焦点、不闪屏**；`CopyFromScreen` 只作 fallback。

---

## 一、抓 GUI 窗口（VS2010 / VS2022）

### 启动与等待

```powershell
Stop-Process -Name devenv -ErrorAction SilentlyContinue      # VS2022
# VCExpress 先看有没有在跑；用户在用时别杀，先问
Start-Process "VCExpress.exe" -ArgumentList "`"$absPath`""
```

等待窗口就绪要**三步**，缺一不可：

```powershell
# 第 1 段：等主窗口句柄出现
while (-not $proc.MainWindowHandle -or $proc.MainWindowHandle -eq 0) { Start-Sleep -Milliseconds 200 }

# 第 2 段：核对主窗口类名 —— 是 #32770（对话框）就说明启动失败，别继续拍
#   实测：VCExpress.exe <路径> 会弹「未知错误」，类名 #32770，尺寸 241x163
#   判定：类名含 "#32770" → 报错退出，或改用 GUI 里 File→Open

# 第 3 段：等启动画面（Splash）类窗口消失，类名通常是 "VSSplash"
```

**只等 `MainWindowHandle` 是错的**——IDE 启动瞬间句柄就有了，但画面还是启动图；
更糟的是句柄可能来自一个错误对话框。

### 抓图

```csharp
[DllImport("user32.dll")]
static extern bool PrintWindow(IntPtr hwnd, IntPtr hdcBlt, uint nFlags);
```

关键参数（**版本差异，务必区分**）：

| 版本 | `nFlags` |
|---|---|
| **VS2010** | **`0`** |
| **VS2022** | **`2`** |

写 `0` 抓 VS2022 会得到**黑图或白图**，这是最容易误判成"没装好"的坑。

### 裁剪

- 4K 满窗比例：`3098 x 1743`，保留标题栏与菜单
- 代码靠左、右侧留白，与历史产出图保持一致
- 代码行多（>19 行）时先 `Ctrl+Home` 回顶，或把高度放到 `1743` 以上，**否则末行 `}` 被切**

---

## 二、抓经典控制台

### 为什么不能直接抓

**Windows 11 默认把控制台委派给 Windows Terminal**（类名 `CASCADIA_HOSTING_WINDOW_CLASS`）：

- `PrintWindow` **抓不到**（返回失败）
- 窗口标题**不稳定**，没法按标题定位

**解法：用 `conhost.exe` 主动起一个经典 `ConsoleWindowClass` 窗口。**

### 起窗口

```powershell
$before = [One4]::Consoles()    # 先快照已有控制台句柄

Start-Process "$env:SystemRoot\System32\conhost.exe" `
  -ArgumentList "cmd.exe /k chcp 936 >nul & $exe" `
  -WorkingDirectory $dir

# 轮询：只认 $before 里没有的那个新句柄，且类名 = ConsoleWindowClass
```

**句柄差集定位**是关键：如果按类名找，会抓到之前几轮残留的控制台，截出来是旧画面。

### 抓图前的四件事

1. `SetWindowPos(h, -1, 80, 80, 1300, 700, SWP_SHOWWINDOW)` —— 置顶 + 固定位置
2. 用 `PrintWindow(h, dc, 0)` 抓（**实测可行**）；失败或需要前台像素时再退回 `CopyFromScreen`
3. **用 `CopyFromScreen` 时**要先**点一下窗口内**（`SetCursorPos` + `mouse_event` 左键下/上）拿焦点，
   否则 `SendKeys` 会打到别处；`PrintWindow` 不需要这一步
4. 有输入的程序：`(New-Object -ComObject WScript.Shell).SendKeys("12345{ENTER}")`
   无输入的程序用 `cmd /k` **保窗**（`/c` 会瞬间关窗，截到空屏）

### 裁剪

```
x = 0,  y = 54
两行输出：w = 1095, h = 84
单行：    h = 38
```

**切掉标题栏、命令行（含 `C:\Users\...` 路径）以及后续提示符**，只留程序输出。

顺带好处：切掉路径 = **自动脱敏**（不暴露用户名和目录结构）。
> 注意：用 `PrintWindow` 抓到的图里标题栏显示的是 `conhost.exe` 全路径，同样要裁掉。

---

## 三、编码陷阱（PowerShell 5.1，最隐蔽的一类）

| 现象 | 根因 | 解法 |
|---|---|---|
| 文件名变 `raw_缁冧範...png` | **脚本正文里的中文被按 GBK 误读** | 脚本**正文只用 ASCII**；中文文件名由**父进程**（编码正确的 pwsh）事后重命名 |
| 脚本语法错 | 脚本存成了无 BOM 的 UTF-8 | 存成 **UTF-8 with BOM** |
| 参数错位 | 传空字符串被 PowerShell 吞掉 | 用占位符 `NONE` 代替空值 |
| 中文源码编译后乱码 | MSVC 按 **GBK** 解析无 BOM 文件 | 源码存 **UTF-8 with BOM** |

解码 GBK 输出：

```powershell
[Text.Encoding]::GetEncoding(936).GetString([IO.File]::ReadAllBytes($logPath))
```

---

## 四、环境要求

| 项 | 要求 |
|---|---|
| 系统 | Windows 10 / 11（Win11 的 Terminal 委派是核心坑点） |
| IDE | VS2010 或 VS2022（flags 不同，见上） |
| Shell | PowerShell 5.1 / 7 |
| 编码 | 源码 UTF-8 BOM，控制台 GBK(936) |

**路径注意**：本文里的 `<VS安装盘>\Program Files (x86)\Microsoft Visual Studio 10.0\...` 是示意路径，
换机器要改成自己 VS 的安装位置（`vcvarsall.bat` 用 `Get-ChildItem <盘>\ -Recurse -Filter vcvarsall.bat` 找）。

---

## 五、常见错误对照表

| 现象 | 根因 | 处置 |
|---|---|---|
| 截到桌面而不是目标窗口 | 按类名找窗口，抓到历次残留 | **句柄差集定位** + 先清理残留 |
| 拍到一个「未知错误」对话框 | `VCExpress.exe <路径>` 启动失败，`MainWindowHandle` 指向对话框 | **核对类名 `#32770`，是对话框就别拍** |
| 抓到的是启动画面 | 只等了 `MainWindowHandle` | 再等 Splash 类窗口消失 |
| VS2022 截图全黑 | `PrintWindow` 的 flags 用了 `0` | **VS2022 用 `2`**，VS2010 用 `0` |
| 截图只有第一行提示、无输入 | 控制台丢焦点 | 用 `CopyFromScreen` 时每次截图前**点窗口**拿焦点 |
| 截到空屏 | 程序跑完窗口自动关了 | 用 `cmd /k` 保窗，别用 `/c` |
| 截到旧画面 | 抓到了上一轮残留的控制台 | 句柄差集，只认新句柄 |
| `不是内部或外部命令` | 清临时文件时删了 `.exe` | 重编，别删产物 |
| `0xE9` 管道错误 | 杀了 `conhost` | 只 `WM_CLOSE` 自己那个窗口 |
| 文件名乱码 `raw_缁冧範` | 脚本中文被 GBK 读 | ASCII 脚本 + 父进程改名 |
| 中文源码乱码 | 源码非 UTF-8 BOM | `UTF8Encoding($true)` |
| 日志里源文件名变 `渚?.1.c` | **用 GBK 解了 UTF-8 的 cl 日志** | cl 日志按 **UTF-8** 解；只有控制台输出才是 936 |
| 只出 `.obj` 不出 `.exe` | `/Fe` 用了相对路径 | 绝对路径 + `/link /out:` |

---

## 六、C 语言作业（VS2010）全流程

> C 语言作业（VS2010）的完整流程。**只在交作业时读这一节。**

### 何时用

用户给教材照片（例 X.Y / 同步练习 X-Y），要求在桌面产出可提交的作业包：
`.c` 源码 + IDE 代码截图 +（可选）运行结果截图，最后 `Compress-Archive` 打包。

### 1. 先看现状（不可跳过）

列桌面、读用户已有的作业文件夹（如 `作业截图\`、`<学号><姓名>\` 这类以学号姓名命名的文件夹），
确认命名风格（`例1.9.c` 点号？`练习2-7.c` 横杠？）和配套图片类型。**照抄现有风格，不自创。**

### 2. 源码规范

- **UTF-8 with BOM**（无 BOM 时 MSVC 按 GBK 解析，中文全乱）。
- 首行 `#define _CRT_SECURE_NO_WARNINGS`（压 C4996，VS2010 同样有效）。
- 4 空格缩进，`int main(void) {`，声明的变量放在块首（C89），`return 0;`。
- 教材原题要求「先定义 sex/age/height」之类，就按它的顺序声明。
- 写文件：

  ```powershell
  [IO.File]::WriteAllText($p, ($code -replace "`r`n","`n"), (New-Object System.Text.UTF8Encoding($true)))
  ```

> 以上规范与旧作业文件保持一致——**照抄用户既有文件，不按自己的偏好改**。

### 3. 编译与验证（VS2010 工具链）

```powershell
# 改成你自己的 VS2010 安装路径
$vc10 = "C:\Program Files (x86)\Microsoft Visual Studio 10.0\VC\vcvarsall.bat"
cmd /c "call ""$vc10"" >nul && cd /d ""$dir"" && cl /nologo /W3 ""$f"" /Fe:""$dir\r.exe"" /link /out:""$dir\r.exe"" >log.txt 2>&1"
```

- `/Fe:` 用**绝对路径**，并补 `/link /out:`，否则偶尔只出 `.obj` 不出 `.exe`。
- **日志解码要按 UTF-8，不是 GBK**（实测更正 2026-11-26）：
  源码存成 UTF-8 BOM 后，`cl` 写进日志的文件名也是 **UTF-8 字节**。
  同一个 `例1.1.c`，文件名字节是 `E4 BE 8B`：
  `[Text.Encoding]::UTF8.GetString(bytes)` → `例1.1.c`（对）
  `[Text.Encoding]::GetEncoding(936).GetString(bytes)` → `渚?.1.c`（错）
  ```powershell
  $log = [Text.Encoding]::UTF8.GetString([IO.File]::ReadAllBytes($logPath))
  ```
  → 只有**控制台输出**（程序运行时的中文）才是 GBK 936；**cl 的编译日志是 UTF-8**。两者别混。
- 期望「0 警告 0 错误」，日志里只应出现源文件名。
### 4. 代码截图（VS2010 窗口）

按第一节「抓 GUI 窗口」。裁剪成 `3098x1743`（4K 满窗比例），保留标题栏/菜单，代码靠左、右侧留白 —— 与 `例1.9.png` 一致。
代码行多（>19 行）时先 `Ctrl+Home` 回顶，或把裁剪高度放到 `1743` 以上，**否则末行 `}` 被切**。

### 5. 运行截图（经典控制台）

按第二节「抓经典控制台」。**只留程序输出**：无路径、无标题栏、无残留窗口。
裁剪：`x=0, y=54`；两行输出 `w=1095, h=84`，单行 `h=38`。

### 打包

```
学号姓名_练习X.Y\
  例2.18.c        例2.18.png        例2.18-运行.png
  练习2.9-3.c     练习2.9-3.png     练习2.9-3-运行.png
```
```powershell
Compress-Archive -LiteralPath $folder -DestinationPath "$desk\学号姓名_练习X.Y.zip" -CompressionLevel Optimal
```

打包后核对：**文件夹文件数 == ZIP 条目数**，且解压后是**单层文件夹**。

### 验收清单（缺一不可）

- [ ] 每个 `.c` 在 VS2010 下 **0 警告 0 错误**
- [ ] 每个程序跑出**预期结果**（数值要对，如 5050 / 26 / 105 与 14）
- [ ] 代码截图：**文件名标签页可见 + 完整代码**（末行 `}` 未被切）
- [ ] 运行截图：有输入回显 + 结果，**无路径、无标题栏、无残留窗口**
- [ ] 命名与既有风格一致（点号 / 横杠 / 前缀）
- [ ] 原有文件未被覆盖（必要时先在回收站确认可恢复）
- [ ] ZIP 解压后是单层文件夹
