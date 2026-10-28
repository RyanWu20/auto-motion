# 用 Hana 跑 auto-motion：从拉仓库到出片

这份手册按 [vibe-motion/auto-motion](https://github.com/vibe-motion/auto-motion) 的真实流程来写。成片不是视频生成模型画出来的。模型只写一页能跳到任意时间的 HTML。HyperFrames 在无头 Chrome 里把时间拨到每一帧，ffmpeg 把这些帧编成 MP4，再把镜头接起来。

原理演示页：`docs/auto-motion-render-principle.html`。

仓库上游的入口是 `codex exec < PROMPT.md`，再由 `claude -p` 写镜头。本手册不用这两条命令。调度、写镜头、验收、配音分别交给 Hana 里的四个助手。

## 1. 剧组设四个岗位

| 显示名 | 电影岗位 | 在这个仓库里做什么 | 对话模型 | 元 |
|---|---|---|---|---|
| 锦泽-制片 | 执行制片 | 读字幕、拆镜头、建目录、一次派一个导演、在自己的终端里渲染、派剪辑；口令要求配音时再派配音并合轨 | `deepseek` / `deepseek-v4-pro` | kong |
| 锦泽-导演 | 导演，兼任美术和动画 | 只做当前镜头的 HTML。不在子代理里等渲染结束 | `moonshot` / `kimi-k2.6` | ming |
| 锦泽-剪辑 | 剪辑兼场记 | 用 ffprobe 验收，用 ffmpeg 接成无声 `<主题>.mp4`。不改画面 | `deepseek` / `deepseek-v4-flash` | kong |
| 锦泽-配音 | 配音 | 按 brief 和字幕逐句合成，按时间点摆轨，交出 `narration.m4a` | `deepseek` / `deepseek-v4-flash` | kong |

无声成片 `<主题>.mp4` 始终没有音轨。只有开片口令写明「配音：需要」，并写齐音色和语速时，才派锦泽-配音，再合出 `<主题>_配音.mp4`。口令没写配音，就整条片子无声，也不派配音。

不要把可灵、即梦、Seedance 配成任何一个助手的对话模型。仓库里的 `image-gen` 技能只出静图，调用 MiniMax 的 `image-01`，而且要单独的 `MINIMAX_API_KEY`。主流程不需要它。品牌标志优先在写代码之前下载 SVG。

Hana 的子代理限时 30 分钟，写在 `lib/tools/subagent-tool.ts` 的 `SUBAGENT_TIMEOUT_MS`。所以导演只交 HTML。`npx hyperframes render` 由制片在自己的持续终端里跑。这条终端不受那 30 分钟限制。

## 2. 机器上要先有的东西

个人 Mac 已登录，Hana 已安装。模型密钥只写在 Hana 的供应商设置里，不写进仓库，不贴进微信或 QQ。

终端里确认：

```bash
node --version    # 必须是 v22 或更高
ffmpeg -version
ffprobe -version
jq --version
```

缺 Node 22 时先装好再继续。ffmpeg 和 ffprobe 一般在同一个安装包里。`jq` 用来看 ffprobe 的 JSON。

第一次渲染时，`npx hyperframes` 会下载 CLI，并拉起 headless Chrome。需要能访问外网。

## 3. 拉仓库

书桌目录固定为 `/Users/wulei/Workspace/GitRepo/auto-motion`。目录已经存在就不要再克隆，直接进去。

```bash
mkdir -p /Users/wulei/Workspace/GitRepo
cd /Users/wulei/Workspace/GitRepo
git clone https://github.com/vibe-motion/auto-motion.git
cd auto-motion
```

### 3.1 仓库目录怎么分

书桌始终是仓库根。字幕、镜头、成片都进 `films/<主题>/`，不要写回根目录。根上也不再放闲置的 `assets/`。

```text
/Users/wulei/Workspace/GitRepo/auto-motion/
  .agents/skills/                 书桌在仓库根时加载
    auto-motion-producer/
    auto-motion-editor/
    auto-motion-dubbing/
  exampleFolder/                  上游带来的名字，当作导演技能母版，不要改名
    run-claude-ai.sh              上游 Claude 入口，本手册不执行
    .claude/skills/
      auto-motion-director/       导演岗位技能
      hyperframes*/               HyperFrames 全家桶，随镜头目录复制
      ...
  films/                          每一部片子一个子目录
    <主题>/
      transcription.srt
      dubbing-brief.md            仅配音=需要；由制片按口令写出
      scenes/
        scene-plan.json
        scene-00N/                从 exampleFolder 复制来的 .claude + 本片字幕
      list.txt
      <主题>.mp4                  无声成片
      dubbing/                    逐句 TTS 中间文件
      narration.m4a
      <主题>_配音.mp4
  auto-test/transcription.srt     约 1 秒试片字幕
  docs/                           出片手册副本
  PROMPT.md / README* / AGENTS.md 上游说明，本手册不执行 PROMPT.md
  auto-hyper- / readme-assets/    上游自带，出片主流程不用
```

岗位技能为什么分两处：制片、剪辑、配音在仓库根干活，读 `.agents/skills`。导演被派到镜头目录时，只能看见该目录下的技能，所以母版放在 `exampleFolder/.claude/skills`，建镜头时整份 `.claude` 复制进去。导演技能保持这个位置，不挪进 `.agents`。

用户只给视频主题。制片按主题推出路径：片子根 `films/<主题>/`，字幕 `transcription.srt`，无声成片 `<主题>.mp4`，配音成片 `<主题>_配音.mp4`。口令里不必拼完整路径。

磁盘上已有的 `films/test`、`films/wolf`、`films/demo` 是早期英文目录名。`films/wolf` 里还可能留着旧名 `final.mp4`、`final_narrated.mp4`；新约定以 `<主题>.mp4` / `<主题>_配音.mp4` 为准。新片优先用主题原文建目录，例如 `films/狼来了/`。

`.gitignore` 已忽略 `films/**/*.mp4`、`films/**/*.m4a`，以及镜头里复制出来的 `films/**/scenes/**/.claude/`。

### 3.2 第一次用试片

```bash
mkdir -p films/test
cp auto-test/transcription.srt films/test/transcription.srt
```

正式片先把字幕放进 `films/<主题>/transcription.srt`。

## 4. Hana 的供应商和全局模型

打开 Hana，进入 **设置 → 供应商**。

1. 添加供应商 `deepseek`，填 API key。
2. 添加供应商 `moonshot`，填 Moonshot 的 API key。
3. 确认模型列表里能看到 `deepseek-v4-pro`、`deepseek-v4-flash` 和 `kimi-k2.6`。导演用 `moonshot` 下内置的 `kimi-k2.6`。不要手动加 `kimi-k3.1-pro`，也不要选 `kimi-coding` 里的 `k3`。

当前 Hana 调不通 `kimi-k3.1-pro`。官方要求 K3 系列始终思考，只用顶层 `reasoning_effort`（`low`、`high`、`max`），不要 K2 的 `thinking` 参数，`max_tokens` 也已弃用。Hana 的 Kimi 适配器仍会在推理打开时写入 `thinking`，并且只对模型 ID `k3`、`k3-256k` 去掉 `temperature`。这两个 ID 属于 Kimi Coding Plan，地址是 `https://api.kimi.com/coding/v1`，不是开放平台的 K3.1 Pro。开放平台 Key 配上去会 401，模型也不是同一个。
4. 在同一页下半的 **其他模型** 里：
   - 小工具模型：`deepseek` / `deepseek-v4-flash`，点测试。
   - 大工具模型：`deepseek` / `deepseek-v4-flash`，点测试。
   - 视觉辅助：关闭。`kimi-k2.6` 本身支持看图。

小工具模型和大工具模型是全局的，不跟着某个助手走。

配音还要配全局语音合成。进入 **设置 → 媒体 → 语音合成**：

1. 供应商选 **豆包语音合成**，provider 是 `volcengine-tts`。
2. 合成模型选 **豆包语音合成 2.0（标准版）**，model 是 `seed-tts-2.0-standard`。下拉完整写法是 `volcengine-tts / 豆包语音合成 2.0（标准版）`。
3. 把它设成 **默认语音合成模型**。
4. 音色不在这一页写死。开片时写进片子目录的 `dubbing-brief.md`。狼来了验证过的音色 ID 是 `zh_female_shaoergushi_uranus_bigtts`（少儿故事），语速是 `1.2`。

「Ta 的模型」是对话调度；「默认语音合成模型」才是出声通道。锦泽-配音不要把豆包 TTS 填进「Ta 的模型」。

改完后，进入 **设置 → 安全**。这一页没有叫「沙盒网络」的开关，对应的是沙盒区块里的第二行。

- **沙盒**保持打开。说明是「限制 AI 的文件系统写入范围；普通外部文件可只读访问，工作目录内可读写」。
- **允许沙盒联网**保持打开。它在「沙盒」正下方。说明是「开启后，沙盒内的命令可以访问公网；文件写入限制仍然生效」。`npx hyperframes` 走的是这条。默认就是开。沙盒关掉时这一行会变灰，命令改按普通进程联网。
- **出站代理**的代理模式用「跟随系统 / 环境」或「直连」。说明写着它只管应用内 API、OAuth、Bridge 和网页请求，模型调用走这里。

## 5. 创建四个助手

在 **设置 → Agent** 用卡片上的添加按钮，按下面的显示名各建一个：`锦泽-制片`、`锦泽-导演`、`锦泽-剪辑`、`锦泽-配音`。这四个名字在团队里唯一。锦泽-制片派工时，子代理的 `agent` 参数就填这些显示名。

Hana 把一个助手的提示词分成三层，不要混在同一段里：

| 层 | 写在哪里 | 装什么 |
|---|---|---|
| 身份简介 | 设置里的身份简介，对应 `identity.md` | 标题加一两句。用 `{{agentName}}`、`{{userName}}`，运行时会替换。花名册摘要由这段和下面的人格生成，其他助手靠摘要认识他 |
| 元 | 新建时选的性格 | 只决定要不要先写内心区块。kong 不写，ming 先写沉思 |
| AGENTS.md | 设置里的 AGENTS.md | 只写性格、语气和边界，标题用「人格定义」。这是助手自己的人格，不是仓库根目录那份项目 AGENTS.md。两份都会注入，不要互相覆盖 |

出片步骤写在技能里。Hana 只扫描当前书桌目录下的 `.agents/skills` 或 `.claude/skills`。书桌一切到镜头子目录，仓库根的技能就离开视野。所以制片、剪辑、配音放在仓库根 `.agents/skills`；导演放在会被复制进镜头的 `exampleFolder/.claude/skills`，并保持这一分法，不挪进 `.agents`。书桌始终停在仓库根；只有派导演时才临时切到镜头目录。

仓库里这四份技能已经就位。Hana 打开「加载项目技能」后会自动读到，不必再粘贴进助手设置。改步骤只改磁盘上的文件：

| 技能 | 路径 | 只做什么 |
|---|---|---|
| auto-motion-producer | `.agents/skills/auto-motion-producer/SKILL.md` | 拆镜头、派导演、渲染、派剪辑；需要时再派配音并合轨 |
| auto-motion-director | `exampleFolder/.claude/skills/auto-motion-director/SKILL.md` | 写单镜头 HTML，交渲染命令 |
| auto-motion-editor | `.agents/skills/auto-motion-editor/SKILL.md` | 验收镜头，拼无声 `<主题>.mp4` |
| auto-motion-dubbing | `.agents/skills/auto-motion-dubbing/SKILL.md` | TTS 逐句合成，按字幕时间摆成 `narration.m4a` |

配音摆轨与合轨的硬规则写在技能里：逐句落在 `dubbing/line-NNN.m4a`，用 `adelay` 对齐字幕 start，用 `apad` 把旁白垫到无声片时长；合轨以视频时长为准，不用 `-shortest`。

新建时先选元，再填显示名。四个显示名分别是 `锦泽-制片`、`锦泽-导演`、`锦泽-剪辑`、`锦泽-配音`。

| 助手 | 元 | 原因 |
|---|---|---|
| 锦泽-制片 | kong | 界面写着「无思考区块，直接回答」。kong 没有人格模板，回落到空。步骤在技能里，人格不写灵感 |
| 锦泽-导演 | ming | 界面写着「更理性冷静」。先核对时长和尺寸，再写 HTML。hanako、butter 在不确定时会追问 |
| 锦泽-剪辑 | kong | 只报 ffprobe 读到的数字 |
| 锦泽-配音 | kong | 只执行合成和落盘。不需要沉思，也不需要联想 |

每个助手在设置里粘贴身份简介和 AGENTS.md。工具开关在保存后，要新开一轮对话才生效。四个助手都打开「加载项目技能」。只有锦泽-导演打开「加载其他工具的项目技能」。配完再出片。

### 5.1 锦泽-制片

| 项 | 值 |
|---|---|
| 显示名 | 锦泽-制片 |
| 元 | kong |
| Ta 的模型 | `deepseek` / `deepseek-v4-pro` |
| 技能页 | 不启用 hyperframes、hyperframes-motion、hyperframes-design、general-video、motion-graphics |
| 工作区 | 书桌目录设为 `/Users/wulei/Workspace/GitRepo/auto-motion` |
| 加载项目技能 | 开 |
| 加载其他工具的项目技能 | 关 |
| 工具 | 浏览器、安装技能、Workflow、修改设置都关 |

身份简介：

```markdown
# {{agentName}}

{{userName}}的执行制片。擅长把字幕拆成首尾相接的镜头，并按顺序交付成片。不设计画面。
```

AGENTS.md：

```markdown
# 人格定义

- 你说话克制、短、可核对。先给结论和路径，再给数字
- 出片步骤只认 auto-motion-producer。不要自己执行剪辑或配音技能里的步骤
- 镜头的图形、配色和动效交给导演。你不改画面
- 时长用小数秒。对不上就停下并报差额
- 同一时间只推进一个镜头
- 少用破折号
- 不用「总的来说」「希望对你有帮助」收尾
- 如非必要，别用「不是...是...」及其相似的句式
```

### 5.2 锦泽-导演

| 项 | 值 |
|---|---|
| 显示名 | 锦泽-导演 |
| 元 | ming |
| Ta 的模型 | `moonshot` / `kimi-k2.6` |
| 技能页 | 不在技能页另开一份 HyperFrames。技能跟着镜头目录走 |
| 工作区 | 书桌目录先设为 `/Users/wulei/Workspace/GitRepo/auto-motion`。真正干活时由锦泽-制片切到镜头目录 |
| 加载项目技能 | 开 |
| 加载其他工具的项目技能 | 开。技能在 `.claude/skills`，这项不开就读不到 |
| 工具 | 只开浏览器。安装技能、Workflow、修改设置关 |

身份简介：

```markdown
# {{agentName}}

{{userName}}的单镜头导演，兼任美术和动画。擅长把一镜文案写成可逐帧跳转的 HTML。不拼接成片。
```

AGENTS.md：

```markdown
# 人格定义

- 你先核对时长、尺寸和这一镜在全片中的位置，再决定视觉
- 构图步骤以 auto-motion-director 技能为准，并遵守已加载的 HyperFrames 技能
- 任务已经给齐编号、时长和文案时，直接写文件，不追问
- 你不把渲染等到结束，不拼接成片，不改其他镜头
- 少用破折号
- 不用「总的来说」「希望对你有帮助」收尾
- 如非必要，别用「不是...是...」及其相似的句式
```

### 5.3 锦泽-剪辑

| 项 | 值 |
|---|---|
| 显示名 | 锦泽-剪辑 |
| 元 | kong |
| Ta 的模型 | `deepseek` / `deepseek-v4-flash` |
| 技能页 | 不启用 HyperFrames |
| 工作区 | 书桌目录设为 `/Users/wulei/Workspace/GitRepo/auto-motion` |
| 加载项目技能 | 开。剪辑技能在仓库根的 `.agents/skills` |
| 加载其他工具的项目技能 | 关 |
| 工具 | 浏览器、安装技能、Workflow、修改设置都关 |

身份简介：

```markdown
# {{agentName}}

{{userName}}的剪辑，兼任场记。擅长核对分辨率、帧率、时长和音轨，并把镜头接成成片。不改画面。
```

AGENTS.md：

```markdown
# 人格定义

- 你只报测得到的数字：宽、高、帧率、时长、音轨数、文件路径
- 验收和拼接只认 auto-motion-editor。不要配音，不要合轨
- 规格不合格就先转码再接。你不改 HTML，不重做画面
- 少用破折号
- 不用「总的来说」「希望对你有帮助」收尾
- 如非必要，别用「不是...是...」及其相似的句式
```

### 5.4 锦泽-配音

| 项 | 值 |
|---|---|
| 显示名 | 锦泽-配音 |
| 元 | kong |
| Ta 的模型 | `deepseek` / `deepseek-v4-flash`。只做读 brief、逐句调度、摆轨和回报 |
| 语音合成 | 全局默认 provider `volcengine-tts`，model `seed-tts-2.0-standard`（界面显示为豆包语音合成 2.0 标准版）。在设置 → 媒体 → 语音合成里配置，不要填进「Ta 的模型」 |
| 音色与语速 | 开片写进 `dubbing-brief.md`。狼来了验证过：voice `zh_female_shaoergushi_uranus_bigtts`（少儿故事），语速 `1.2` |
| 技能页 | 不启用 HyperFrames |
| 工作区 | 书桌目录设为 `/Users/wulei/Workspace/GitRepo/auto-motion`。真正干活时由锦泽-制片派工 |
| 加载项目技能 | 开。配音技能在仓库根的 `.agents/skills/auto-motion-dubbing` |
| 加载其他工具的项目技能 | 关 |
| 工具 | 浏览器、安装技能、Workflow、修改设置都关 |

身份简介：

```markdown
# {{agentName}}

{{userName}}的配音。擅长按字幕时间轴逐句合成旁白轨，并与画面节奏对齐。不改画面。
```

AGENTS.md：

```markdown
# 人格定义

- 你只报测得到的数字：句数、每句起止、音频时长、与成片时长的差额、文件路径
- 合成与摆轨只认 auto-motion-dubbing。不要读制片或剪辑技能里的步骤
- 音色、语速、摆放规则只认 dubbing-brief.md
- 开片 brief 缺「配音：需要」、音色或语速时，停下回报，不自选默认声线
- 你不改 HTML，不渲染镜头，不拼接画面，不合轨，不加 BGM
- 少用破折号
- 不用「总的来说」「希望对你有帮助」收尾
- 如非必要，别用「不是...是...」及其相似的句式
```

无声拼接命令不用在 Hana 里单独配置，写在 `auto-motion-editor` 里。合轨命令写在 `auto-motion-producer` 里，以视频时长为准，不用 `-shortest`。

## 6. 开始出片

1. 在 Hana 里选中 **锦泽-制片**，新开一轮对话。
2. 确认这一轮的书桌目录是 `/Users/wulei/Workspace/GitRepo/auto-motion`。不要把书桌改成片子目录，否则读不到仓库根的项目技能。
3. 用 Owner 单聊，或在本机这一轮里，对整个仓库目录做一次写入和跑命令的授权。后面一个镜头会写很多文件，不要逐个文件确认。
4. 把下面这段发给锦泽-制片。口令只写主题；需要配音时再写音色和语速。路径、provider、model、摆放规则由技能补全。

试片：先确认 `films/test/transcription.srt` 已就位，再发：

```text
主题：test。
配音：不需要。
按 auto-motion-producer 出片。完成后把成片路径和 ffprobe 结果发回来。
```

正式片无声：先把字幕放进 `films/<主题>/transcription.srt`，再发：

```text
主题：狼来了。
配音：不需要。
按 auto-motion-producer 出片。
```

正式片需要配音时，开片写主题、音色和语速即可：

```text
主题：狼来了。
配音：需要。
音色：少儿故事。
语速：1.2。
按 auto-motion-producer 出片。
```

## 7. 跑起来之后目录里会有什么

一部配音片跑完后，片子目录大致是这样（主题示例用「狼来了」；若目录仍叫 `films/wolf`，把路径里的主题名换成 `wolf`）：

```text
films/狼来了/
  transcription.srt
  dubbing-brief.md                 制片按口令写出；派配音前必须存在
  scenes/
    scene-plan.json
    scene-001/
      .claude/skills/              从 exampleFolder 复制：导演技能 + HyperFrames
      transcription.srt
      index.html
      gsap.min.js                  若导演改用本地 GSAP，会落在镜头目录
      scene-001.mp4
    scene-002/
      ...
  list.txt
  狼来了.mp4                       无声成片
  dubbing/
    line-001.raw.m4a
    line-001.m4a
    ...
  narration.m4a
  狼来了_配音.mp4
```

仓库根在出片过程中不应再出现 `transcription.srt`、`list.txt`、`final.mp4`、`narration.m4a` 或闲置的 `assets/`。技能母版仍在根上的 `.agents/skills` 与 `exampleFolder/.claude/skills`。

`scene-plan.json` 至少要能看出每个镜头的编号、起止时间、小数秒时长和文案。制片中断后靠这张表接着做，不要把已经渲染好的镜头再派一次。

试片预期只有 `scene-001`，成片大约 1 秒。上游的 `auto-test/validate.sh` 也是按这条约 1 秒的试片写的。正式字幕不要用那个脚本的「大约 1 秒」作为合格线，以剪辑的 ffprobe 结果和该片子目录里的 `scene-plan.json` 为准。

## 8. 你要自己看的三件事

渲染进行中，制片的终端里应能看到 `npx hyperframes render`。mp4 的修改时间会变。子代理那边安静，不代表渲染停了。

一个镜头失败时，先看这个镜头目录里的 `index.html` 是否存在，以及终端里的渲染退出码。不要先改别的镜头。

成片到手后抽查，路径换成实际片子目录：

```bash
ffprobe -v error \
  -select_streams v:0 \
  -show_entries stream=width,height,avg_frame_rate \
  -show_entries format=duration \
  -of json films/狼来了/狼来了.mp4

ffprobe -v error -select_streams a -show_entries stream=index -of csv=p=0 films/狼来了/狼来了.mp4
```

宽应为 1080，高应为 1440，帧率约为 30，第二条命令没有输出才是没有音轨。时长应接近该片子 `scene-plan.json` 里各镜头时长之和。

若做了配音，再抽查：

```bash
ffprobe -v error -show_entries stream=codec_type,codec_name -show_entries format=duration -of json films/狼来了/狼来了_配音.mp4
ffprobe -v error -show_entries format=duration -of json films/狼来了/narration.m4a
```

配音成片应有视频轨和 AAC 音轨，时长应与无声成片一致，不能比无声片更短。旁白轨用静音垫到无声片时长。差额过大或有句子塞不进字幕窗口时，先看 `dubbing-brief.md` 和配音回报，不要先改画面。

## 9. 和上游技能冲突的地方，以本手册为准

HyperFrames 技能默认 16:9，点名竖屏才用 9:16，并要求检查通过后停在 preview 等用户确认。本片是 1080×1440，也就是 3:4。尺寸和「只交渲染命令、不等待渲染结束」写在 auto-motion-director 技能里。

`--fps` 的可选取值是 24、30、60，默认 30。交付命令写 `--fps 30`。不要为了「更高质量」改成 60。`--quality high` 只提高编码质量。

`--resolution portrait` 不是 1080×1440。尺寸写在 HTML 的 `data-width` 和 `data-height` 上。
