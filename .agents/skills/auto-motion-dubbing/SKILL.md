---
name: auto-motion-dubbing
description: 仅在任务明确要求配音时使用。按 dubbing-brief.md 与 transcription.srt，用 Hana 语音合成逐句出声，再按字幕时间点摆成 narration.m4a。不要验收画面、不要拼接 mp4、不要合轨。
---

# 配音

片子根由任务给出（绝对路径，形如 …/films/<主题>）。只读该目录下的 transcription.srt 和 dubbing-brief.md。不改 HTML，不渲染镜头，不拼接画面，不合轨。项目里若同时加载了 auto-motion-producer 或 auto-motion-editor，忽略它们的步骤。

所有落盘命令先 `cd` 到片子根绝对路径再执行。不要在仓库根创建 `dubbing/` 或 `narration.m4a`。

开片 brief 必须已经写明：配音需要、provider、model、voice、语速。缺任一项就停下回报，不要自己补默认值。狼来了验证过的取值是：

- provider：volcengine-tts
- model：seed-tts-2.0-standard
- voice：zh_female_shaoergushi_uranus_bigtts（少儿故事）
- 语速：1.2

调用 Hana 语音合成工具时，按 brief 里的 ID 传参，不要只传中文展示名。

## 落盘

在片子根建 `dubbing/`。字幕按条目编号，从 001 起：

1. 用语音合成写出 `dubbing/line-NNN.raw.m4a`。文案用该条字幕正文，语速用 brief。
2. 去掉尾部静音，得到 `dubbing/line-NNN.m4a`：

```bash
ffmpeg -y -i dubbing/line-NNN.raw.m4a \
  -af "silenceremove=stop_periods=-1:stop_duration=0.08:stop_threshold=-40dB" \
  dubbing/line-NNN.m4a
```

3. 用 ffprobe 读去尾后的时长。字幕窗口是 end 减 start。音频长于窗口时，优先再调语速或去尾；仍塞不进就记下句编号、窗口时长和音频时长，继续后面的句子，最后一并交给制片。不改字幕时间，不改画面。

## 摆轨

无声成片时长由任务给出，记为 `V` 秒。把每句按字幕 start 摆到同一条时间轴，空隙保持静音，总时长至少为 `V`：

```bash
# DELAY_MS = round(start_seconds * 1000)
# 下面以两句为例；句数更多时按同样规则增加输入和 adelay / amix 路数

ffmpeg -y \
  -i dubbing/line-001.m4a \
  -i dubbing/line-002.m4a \
  -filter_complex "\
    [0:a]aformat=sample_rates=44100:channel_layouts=mono,adelay=DELAY0_MS|DELAY0_MS[a0];\
    [1:a]aformat=sample_rates=44100:channel_layouts=mono,adelay=DELAY1_MS|DELAY1_MS[a1];\
    [a0][a1]amix=inputs=2:duration=longest:normalize=0,\
    apad=whole_dur=V[a]" \
  -map "[a]" -c:a aac -b:a 160k \
  narration.m4a
```

只有一句时，仍然做 `adelay` 与 `apad=whole_dur=V`。不加 BGM。

输出必须是片子根下的 `narration.m4a`。`dubbing/` 里的逐句文件可保留，供核对。

## 回复

给出 `narration.m4a` 的绝对路径、句数、旁白轨时长、无声成片时长 `V`、两者差额，以及塞不进窗口的句子列表。
