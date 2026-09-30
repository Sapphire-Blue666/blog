# 把一款 macOS 专属的终端字符动画 MV 移植到 Windows

这段时间折腾了一个终端里的字符动画 MV：音频时间轴驱动画面，歌词和动画跟着音乐走，整个演出就是一个「终端里的 MV」。它原本是 macOS 专属的，我把它在 Windows 上跑了起来。

这篇把过程和改动都摊开讲，包括改在哪几个文件、为什么必须那么改，以及中间踩的坑（这部分自认为比结论有意思）。

---

## 一、先搞清楚：它为什么只能在 macOS 上跑

拿到代码先找平台依赖，一共三处，全部是硬依赖：

1. **音频时钟是一个 Swift 编译出来的二进制。** 项目用 `audio-clock` 这个小程序独占音频设备、上报播放头位置，播放器主程序通过管道跟它对话。它基于 macOS 的 AVFoundation，Windows 上既没有 Swift 工具链也没有 AVFoundation，直接卡死在这一步。
2. **终端控制用的是 POSIX 专有接口。** `termios` / `tty` 设置原始输入模式、`select` + `os.read` 做非阻塞读键、`/dev/tty` 接管控制终端——这一整套在 Windows 上不存在。
3. **一个藏得很深的编码问题。** 读歌词数据用的是 `Path.read_text()`，不带编码参数时它按 **locale 编码**解析。macOS 上启动脚本设了 `PYTHONUTF8=1`，把这个坑盖住了；换到中文 Windows（locale 是 GBK）上，读含中文的 JSON 直接抛 `UnicodeDecodeError`。**也就是说，在中文 Windows 上它连启动都做不到**——这一条我一开始完全没预料到。

## 二、总体思路：不动 macOS 那条路

我给自己定的原则是：**新增一层 Windows 适配，而不是大改原来的代码**。

这样做音频时钟的**协议**是稳定的：

```
stdin   play | pause | seek <seconds> | volume <0..1> | quit
stdout  {"time": <s>, "duration": <s>, "playing": <bool>}   每秒 60 行
```

只要 Windows 版把同样的 JSON 行吐到 stdout，`player.py` 就完全不需要知道自己在哪个平台上跑。实际操作下来，`player.py` 真正改动的部分里，**macOS 那条分支是逐字未动的**。

## 三、改了哪几个地方

一共动了 5 个文件：新增 3 个、修改 2 个。

| 文件 | 动作 | 作用 |
| --- | --- | --- |
| `audio-clock.py` | 新增 | Windows 音频时钟。调用 Win32 `winmm` 的 MCI 接口播放 MP3，stdin/stdout 走 JSON 行协议 |
| `winconsole.py` | 新增 | 控制台层。原始输入模式、非阻塞读键、窗口尺寸、可用性探测 |
| `播放MV.bat` | 新增 | 双击启动器 |
| `player.py` | 修改 | 跨平台化，**+74 / −46** |
| `README.md` | 修改 | 补一节 Windows 用法 |

### 3.1 新增 `audio-clock.py`：用 MCI 重写音频时钟

Windows 侧不需要任何第三方库，系统自带的 MCI（Media Control Interface）就能放 MP3。三个设计要点：

**设备拥有时间轴。** 当前位置的唯一真相来自 `mci status <alias> position`，Python 侧不推算、不插值。时钟漂移这种事根本不给它发生的机会。

**线程亲和性。** 读 stdin 的线程只负责往 `queue` 里塞命令，**所有 MCI 调用都在主线程完成**。MCI 的别名状态不是线程安全的，这不是优化，是必须。

**状态宁可保持，也不谎报。** 轮询 `status mode` 时，返回 `playing` / `paused` / `stopped` 之外的值意味着设备正忙，此时沿用上一次的状态，而不是报一个假的 stopped。

### 3.2 新增 `winconsole.py`：把 termios 该做的事补上

这个模块就是「Windows 版的 termios」，提供同样三件事：原始输入模式、非阻塞按键来源、可见窗口尺寸。

几个细节值得说：

- **`enable()`** 清掉 `ENABLE_LINE_INPUT | ENABLE_ECHO_INPUT | ENABLE_QUICK_EDIT_MODE`，置上 `ENABLE_EXTENDED_FLAGS`。**保留 `ENABLE_PROCESSED_INPUT`**——这样 Ctrl+C 仍然会抛 `KeyboardInterrupt`，被主程序的退出逻辑接住。输出侧置上 `ENABLE_VIRTUAL_TERMINAL_PROCESSING`，否则 ANSI 转义序列不起作用，整个画面渲染不出来。
- 清 `QUICK_EDIT_MODE` 是为了**防止鼠标在窗口里点一下就冻结整场演出**（Windows 控制台默认开启快速编辑，点一下会进入选择模式，程序就不再收到按键了）。这个坑不踩一次是想不到的。
- **读键**：先 `GetNumberOfConsoleInputEvents` 问有多少待处理事件，再向 `ReadConsoleInputW` 请求**恰好等于这个数量**的记录数——这样调用不会阻塞。抬起事件、鼠标事件全部过滤掉。
- 按键被翻译成 `player.py` **已经在解析的转义序列**（方向键 → `\x1b[C` / `\x1b[D` 等）。这一步是整个移植的关键：主循环的按键处理因此完全保持平台无关，一行都不用改。

### 3.3 新增 `播放MV.bat`：双击就能放

负责找 Python、切 UTF-8 代码页、把参数透传给 `player.py`、把退出码原样透出来。两个不太起眼但必须的约束：

**整份文件保持纯 ASCII。** `cmd.exe` 是**按当时的代码页逐行重读**批处理文件的，而脚本自己会执行 `chcp 65001`。如果文件里混有 UTF-8 字节，切换代码页的瞬间就可能被重新解释成乱码。全用 ASCII 就不存在这个问题（ASCII 是 UTF-8 的子集），中文提示一律交给 Python 输出。

**`chcp 65001` 是必需的**——否则 Python 输出的 UTF-8 中文在默认的 GBK 代码页里就是乱码。

### 3.4 `player.py`：具体改了哪几处

这是「改了哪几个地方」的正题。虽然按行数只有 +74/−46，但每一处都对应一个具体的平台差异。

**第一处：平台分派与音频后端常量。**

```python
if os.name == 'nt':
    import winconsole
else:
    import select, termios, tty

AUDIO_CLOCK = ([sys.executable, str(ROOT/'audio-clock.py')] if os.name == 'nt'
               else [str(ROOT/'audio-clock')])
```

把「用哪个音频后端」收成一个模块级常量，`Audio.__init__` 只做 `AUDIO_CLOCK + [path]`。原本这里写死了 `str(ROOT/'audio-clock')`，就这一处，改完 macOS 行为完全等价。

（顺带一句：原作者现有代码用的是 `sys.platform`，我上面用了 `os.name`，真要提 PR 应该统一成作者那种风格。）

**第二处：`read_text()` 的编码——这是整个改动里最值钱的一条。**

```python
# 改前
self.lyrics = json.loads((ROOT/'lyrics.json').read_text())
# 改后
self.lyrics = json.loads((ROOT/'lyrics.json').read_text(encoding='utf-8'))
```

原因见第一节第 3 点：`read_text()` 不传编码时用的是 locale 编码，在中文 Windows 上读含中文的 `lyrics.json` 会直接 `UnicodeDecodeError`。三处读取（歌词、频谱、配置）都补上了显式编码。

这条的性质很重要：它是**上游原本就存在的潜在缺陷**，不是「我为 Windows 新加的功能」。它不改变 macOS 上的行为，但它是整个改动里最容易通过 review、最有贡献者含金量的部分——因为它修的是一个真实存在的隐藏 bug，只不过一直被启动脚本的环境变量掩盖着。

**第三处：`isatty()` 在 Windows 上会说谎。**

```python
if os.name == 'nt':
    if not winconsole.usable():
        raise RuntimeError('请在 cmd 或 Windows Terminal 中运行，或双击「播放MV.bat」。')
elif not sys.stdin.isatty():
    raise RuntimeError('请在终端窗口中运行。')
```

原本只有 `if not sys.stdin.isatty(): raise` 这一句守卫。问题在于：**Windows 的 NUL 设备是字符设备**，所以当 stdin 被重定向到 NUL 时，`isatty()` 依然返回 `True`，守卫被绕过，用户最后只能看到一句莫名其妙的「无法读取控制台模式」。

`winconsole.usable()` 改用 `GetConsoleMode` 来判断——控制台模式只对**真正的控制台**存在，在 NUL 上必然失败。这个判断更严格，报错也更早、更清楚。

**第四处：`run()` 内部的平台分支。**

| 位置 | macOS（原样保留） | Windows（新增） |
| --- | --- | --- |
| 进原始模式 | `tty.setcbreak(...)` | `winconsole.enable()` + `sys.stdout.reconfigure(encoding='utf-8', line_buffering=True)` |
| 保存终端状态 | `termios.tcgetattr(...)` | `None`（由 `enable()` 返回的 token 承担） |
| 取终端尺寸 | `os.get_terminal_size(...)` | `winconsole.size() or (100, 36)` |
| 读键 | `select.select` + `os.read` | `winconsole.wait_for_keys` |
| 退出还原 | `termios.tcsetattr(...)` | `winconsole.restore(console)` |

**第五处：把读键抽成模块级函数，外加两处小修。**

```python
def read_keys(timeout):
    """Keys typed within timeout seconds, or ''; paces the frame while waiting."""
    if os.name == 'nt':
        return winconsole.wait_for_keys(timeout)
    if select.select([sys.stdin], [], [], timeout)[0]:
        return os.read(sys.stdin.fileno(), 128).decode('utf-8', errors='ignore')
    return ''
```

抽出来之后，两边共用同一个签名，主循环里只剩下节奏控制，按键处理逻辑一行没动。另外两处小修：

```python
# Windows 上没有 SIGHUP，原写法会直接 AttributeError
old_signals = {getattr(signal, n): signal.signal(getattr(signal, n), quit_signal)
               for n in ('SIGTERM', 'SIGHUP') if hasattr(signal, n)}
```

```python
# 提示语中性化：'macOS 声音输出设备' → '系统声音输出设备'
```

## 四、踩过的坑

### 4.1 MCI 打不开歌曲，原因是一个 141KB 的专辑封面

**现象**：MCI 就是打不开这份 MP3，报一个信息量极低的错误码（ERR277）。

**第一个假设（错的）**：路径里有中文。→ 把文件复制到纯 ASCII 路径，**失败方式一模一样**，假设推翻。

**第二个假设**：文件本身坏了。→ 读文件头，是合法 MP3；本机其他 MP3 都能正常播放。

**定位手法**：写脚本把本机所有 MP3 逐个喂给 MCI，同时打印各自的 ID3 标签长度做**差分对比**：

| 文件 | ID3v2 标签长度 | MCI |
| --- | --- | --- |
| 本机能播的若干 MP3 | 约 1140 B | OK |
| 目标文件 | **141081 B** | ERR277 |

这一份 MP3 内嵌了一张约 141KB 的封面图。**结论：MCI 的 MPEG 驱动在 ID3v2 标签过大时会初始化失败。**

**修法**：按 syncsafe 的 7-bit 格式解析 ID3v2 头部字节 6:10 得到标签长度（`raw[5] & 0x10` 为真时再加 10 字节 footer），把标签之后的部分写到临时文件，从**无标签副本**播放。修完之后时长读数正常，和配置里记录的毫秒值对得上。

这段的价值在于：「怀疑中文路径」是几乎所有人都会先想到的方向，而**用实验把它排除掉**才是重点——两个假设看起来都合理，靠猜是分不出来的。

### 4.2 为什么最后没用 pygame

一度想用 pygame 当音频后端，实测直接否掉：`mixer.music.get_pos()` 返回的是**自 `play()` 起算的毫秒数，会忽略 `start=` 偏移量**，而且在 `set_pos()` 之后返回 **-1**。也就是说它压根不能当设备时钟用——它测的是「我播了多久」，不是「播放头在哪」。MCI 的 `status position` 才是真正的播放头。

### 4.3 MCI 的 seek 会中止播放（AVAudioPlayer 不会）

**现象**：集成测试里出现一段「卡住不动」的窗口——暂停时时钟冻结是对的，但**快进之后时钟也被冻住了**。

**真因**：`player.py` 的左/右快进和 `,`/`.` 歌词跳转**只发 `seek` 不发 `play`**（在 macOS 上这是对的，因为 AVAudioPlayer 的 seek 不会停）。但 Windows 的 MCI 会在 seek 时中止播放，所以快进一次就变成静音了。

**修在音频层，而不是按键层**：

```python
elif verb == 'seek':
    ms = max(0, min(duration_ms - SEEK_MARGIN_MS, round(float(argument) * 1000)))
    mci(f'seek {ALIAS} to {ms}')
    # MCI halts playback on seek where AVAudioPlayer keeps going, so
    # playback has to be re-armed to keep the state at seek time.
    if playing:
        mci(f'play {ALIAS}')
```

选音频层的理由：这样能**精确复刻 AVAudioPlayer 的语义**，包括「暂停中 seek 后仍然保持暂停」这个分支。如果修在按键层就做不到，而且会把平台差异泄漏进 `player.py`——那正是整个移植想避免的。

### 4.4 看起来卡死在 158.786 秒、`quit` 也没反应——其实是管道背压

**现象**：测试驱动里子进程停在上报 `time: 158.786` 不动，发 `quit` 也没反应，像是死锁。

**排除过程（三个都不是）**：MCI 本身（单独跑正常）、60Hz 的轮询频率、读 stdin 的那个线程。

**真因**：**stdout 管道背压**。测试端只读了约 6 行，而子进程每秒要写 60 行，OS 的管道缓冲区被填满，`sys.stdout.flush()` 就此阻塞。子进程不是死了，是**被自己写不出去的输出堵住了**。换成一个持续排空的驱动脚本后，进度完全正常。

**结论：产品代码不需要改。** `player.py` 里读音频状态的线程是**独立且持续排空**的，真实路径下不会触发。这是**测试脚本的缺陷，不是产品的缺陷**——把这两件事分清楚，是判断测试结果可信度的前提。

### 4.5 GBK 解码崩溃：被集成测试真实抓出来的

见 3.4 第二处。这个坑不是我静态审查预判的，是集成测试实际跑出来的——如果只做代码审查，中文 Windows 上第一次运行就会崩。

### 4.6 `endlocal` 吞掉了 batch 的退出码

**现象**：`播放MV.bat` 在播放器报错（退出码 2）时，自己却返回 0。

**定位手法**：写 6 个最小变体逐个跑，二分出是哪一句吃掉了退出码：

| 变体 | 退出码 |
| --- | --- |
| `exit /b 7` 直接 | 7 |
| `set "C=7"` → `pause` → `exit /b %C%` | 7 |
| `setlocal` → `set "C=7"` → `pause` → `endlocal` → `exit /b %C%` | **0** |
| 同上前提，但写死 `exit /b 7` | 7 |

真因：**`endlocal` 会把作用域内的变量丢掉**，于是 `exit /b %MV_CODE%` 展开成了 `exit /b`（空参数 → 退出码 0）。

**修法**是标准的单行惯用法，利用 `&` 让整行在 `endlocal` 执行前一次性展开：

```bat
rem One line: `&` expands %MV_CODE% before endlocal drops it, so the code survives.
endlocal & exit /b %MV_CODE%
```

## 五、改了之后是怎么验证的

| 层次 | 验什么 |
| --- | --- |
| 单元 | 结构体布局对照 Win32 ABI、真实的 `ReadConsoleInputW` 往返、按键序列翻译、抬起/鼠标事件过滤、`wait_for_keys` 计速、`enable`/`restore` 模式往返、`usable()` 与 NUL 设备的区分 |
| 组件 | `audio-clock.py` 的 JSON 协议与 seek 语义（含 4.3 的「续播 / 保持暂停」矩阵，19 项断言专验这个） |
| 启动器 | 静态检查（全 ASCII、CRLF、`chcp`、路径与参数透传、`endlocal` 同行）+ 4 个运行场景（参数透传、错误码透出、非终端指引、强制缺 Python） |
| 端到端 | 真播放器 + 真音频 + 真音频文件 + 向真实控制台**注入按键**，分别跑 60fps 和 24fps |

**为什么要单独跑一遍 24fps**：60fps 时每帧延迟为 0，`wait_for_keys()` 几乎立即返回，它的 `sleep` 阻塞路径根本没被走到；而双击启动器不传 `--fps`，跑的是默认 24fps（每帧约 41ms），**阻塞路径才是真实路径**。实测帧数和节奏都正确，按键照收。

测试手法值得一提：集成测试用 `CreateFileW` 直接打开 `CONIN$` / `CONOUT$`，再用 `WriteConsoleInputW` 向真实控制台注入按键。**从这一层往下全部是生产代码**——真的主循环、真的音频子进程、真的音频文件，不是 mock。

另外还修掉了 5 个我自己测试脚本的 bug（帧数换算错误、断言里重复调用读键导致打印的详情与实际断言不符、参数类型传错、`exit /b` 提前终止了测试副本、字节串和 str 比较）。**最终报告的失败项全部是被测代码的真问题，没有一个是脚本自身的问题残留**——这个区分是测试可信度的前提。

## 六、如果只记三句话

1. **原项目 macOS 专属，音频时钟是 Swift 二进制**；我把它换成了调用 Win32 MCI 的 Python 实现，**协议完全一致**，所以播放器主程序不需要知道自己跑在哪个平台。
2. **顺手修掉了上游一个潜在缺陷**：读歌词数据用了 locale 编码，在中文 Windows（GBK）上会直接崩溃，macOS 只是被启动脚本里的 `PYTHONUTF8=1` 掩盖了。
3. **过程中最有意思的两个发现**：一是 MCI 因为内嵌了 141KB 专辑封面而拒绝打开歌曲（一开始怀疑中文路径，被实验排除掉）；二是 MCI 的 seek 会中止播放而 macOS 的 AVAudioPlayer 不会——**修在音频层**才能把「暂停中快进仍保持暂停」的语义一起对齐。
