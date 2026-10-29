# 出片排障：第一次遇到的问题

记录首次出片（主题 `test`，1 镜头、1.0 秒、无声）暴露的问题，按「只会第一次出现」和「后续也会出现」分两层。

---

## 第一层：初始化问题（只会第一次出现）

这类问题在首次装环境时出现，装好后不再复现。

### 1. hyperframes 包未安装，且沙箱阻断 npm 网络

**现象**

```
npm error code EPERM
npm error FetchError: request to https://registry.npmmirror.com/hyperframes failed,
  reason: connect EPERM 127.0.0.1:7890 - Local (0.0.0.0:0)
```

**原因**

- `hyperframes` 是 npx 按需拉取的 npm 包，首次执行时本机没有。
- 沙箱默认路径（`use_default`）网络受限，npm 走本机代理 `127.0.0.1:7890` 被拦。

**解决**

执行 npm 拉包类命令时用 `sandbox_permissions="require_escalated"`，在沙箱外执行。现已全局安装 `hyperframes@0.8.214`，直接用 `hyperframes` 命令，不再走 npx。

---

### 2. `hyperframes init` 因目录非空失败

**现象**

```
Directory already exists and is not empty: .
```

**原因**

制片流程要求先把 `.claude`（skills）和 `transcription.srt` 复制进镜头目录，目录已非空；而 `npx hyperframes init .` 只接受空目录。

**解决**

跳过 `init`，导演直接手写 `index.html`。`init` 只是生成空白脚手架，不影响后续 `render` 命令。模板结构可参照 `exampleFolder/.claude/skills/hyperframes` 下的示例。

---

### 3. Chrome for Testing 首次下载慢、甚至停滞

**现象**

```
[hyperframes] Still getting Chrome ready; the first use downloads it.
✗  Chrome not found
Failed to install chrome-headless-shell 152.0.7977.30:
  The Chrome download from storage.googleapis.com stalled: no data for 60s.
```

**原因**

hyperframes 渲染依赖 `chrome-headless-shell`（约 97.5MB），首次渲染时自动从 `storage.googleapis.com` 下载。

- 走本机代理：约 53KB/s，太慢。
- 直连（绕过代理）：会停滞（`no data for 60s`），下载失败。

**解决**

两条路，最终走的是「本地安装 chrome-headless-shell」：

1. 临时用系统 Chrome：`export HYPERFRAMES_BROWSER_PATH="/Applications/Google Chrome.app/Contents/MacOS/Google Chrome"`，走截图捕获路径，1 秒镜头约 11 秒。
2. 永久方案：在**自己的终端**（非 Hana 沙箱）跑 `npx hyperframes browser ensure`，下载并装好 chrome-headless-shell（97.5MB，Chrome 152.0.7977.30）。装好后渲染自动走更快的 BeginFrame 路径，不再需要 `HYPERFRAMES_BROWSER_PATH`。

---

## 第二层：流程问题（后续每次都会出现）

这类问题与「是否首次」无关，每次出片都要处理。

### 1. 渲染在沙箱内必报 EPERM，始终要 escalate

**现象**

沙箱内（`use_default`）执行渲染报两种 EPERM：

- npx 阶段：`connect EPERM 127.0.0.1:7890`（npx 检查 registry 走代理被拦）
- file server 阶段：`listen EPERM: operation not permitted 127.0.0.1`（渲染时要监听本地端口给浏览器喂页面）

**原因**

两层：npx 每次向 registry 检查包；hyperframes 渲染时起本地 file server 监听 `127.0.0.1` 端口，沙箱阻止监听。

**解决**

- 已全局安装 `hyperframes@0.8.214`（`npm i -g hyperframes@0.8.214`），命令用 `hyperframes` 而非 `npx hyperframes`，消除 npx 这层 EPERM。
- 但 file server 监听端口这层沙箱拦不掉，**渲染仍要 `require_escalated`**。这是平台沙箱的硬限制，无法在应用层绕过。

---

### 2. ~~每次渲染都要指定系统 Chrome~~（已解决）

chrome-headless-shell 已本地装好（`~/.cache/hyperframes/chrome/.../152.0.7977.30`），渲染自动走优化路径，无需再设 `HYPERFRAMES_BROWSER_PATH`。系统 Chrome 仅作 fallback。

---

### 3. 渲染命令不要设硬超时

**现象**

第一次渲染设了 `timeout: 300`，进程被 300 秒硬杀，Chrome 下载半途中断。

**原因**

`exec_command` 的 `timeout` 是硬杀，不是「转后台」。下载/渲染耗时不确定时会被误杀。

**解决**

渲染类长任务不设 `timeout`，让它超过前台 60 秒后自动转后台，拿到 `process_id` 后用 `wait_for_tasks` 等结果；需要现场看进度时用 `write_stdin` 空写入等。

---

### 4. 导演子代理要带绝对路径，且要自行处理沙箱网络

**原因**

- 子代理继承的是仓库根 cwd，不是镜头目录；不写绝对路径，导演读不到镜头目录里的 `.claude/skills` 和 `transcription.srt`。
- 导演在子代理里同样会撞上沙箱网络阻断（拉包、联网搜名词），它需要自己用 `require_escalated` 处理。

**解决**

派工 task 里写明镜头目录**绝对路径**，并说明「读写文件和读 skills 都在这个目录下进行」。

---

### 5. 中文字体映射 lint 告警（非阻塞）

**现象**

渲染时 lint 报 `1 error(s), 1 warning(s)`：

```
[WARN] No deterministic font mapping for: Microsoft YaHei, PingFang SC, SFMono-Regular
```

**原因**

`index.html` 用了系统中文字体名，hyperframes 的内嵌字体映射表里没有，无法确定性内嵌。

**影响**

非阻塞，渲染继续。中文字体渲染可能回退到默认，但这次 `test` 成片文字正常显示。

**处理（决定：维持现状，接受告警）**

- 这是 warning 非阻塞。本地渲染时 hyperframes auto-capture 本机字体（macOS 有 PingFang SC），中文实际正常显示。
- 根因是内嵌字体表里**没有中文字体**（18 个家族里唯一 CJK 是 `Noto Sans JP` 日文），不是 init 的问题。**换空目录跑 `init` 无助于字体告警**——init 模板不含中文、用内嵌英文字体，所以「没告警」是假象，一写中文告警照旧。
- 彻底消除需 `@font-face` 内嵌中文字体文件（几 MB 起，拖慢编译），或用 Noto Sans JP（字形偏日文）。都不采用。

---

## 每次出片的标准动作清单

1. 拆镜后，镜头目录放好 `.claude/skills` + `transcription.srt`。
2. 派导演：task 带镜头目录绝对路径、镜头编号、小数秒时长、文案、输出文件名。
3. 收导演回的渲染命令，用 `hyperframes`（已全局安装）在 `require_escalated` 下执行，不设硬超时；无需再带 `HYPERFRAMES_BROWSER_PATH`。
4. 核对 mp4 规格（1080×1440、30fps、时长、无音轨）后派剪辑拼接。
5. 配音门控：口令无「配音：需要」就不写 brief、不派配音。
