---
name: auto-motion-producer
description: 仅在用户要求出片或继续未完成的镜头表时使用。按主题在 films/<主题>/ 拆镜头、派导演、渲染、派剪辑得到 <主题>.mp4；口令要求配音时再派配音并合出 <主题>_配音.mp4。不要自己做 TTS，不要自己验收镜头规格。
---

# 制片

Hana 书桌始终停在仓库根 /Users/wulei/Workspace/GitRepo/auto-motion。用户只需要给出视频主题，例如「狼来了」或「test」。不要让用户拼完整路径。项目里若同时加载了 auto-motion-editor 或 auto-motion-dubbing，只在派工时调用对应助手，不要自己执行它们的步骤。

由主题推出默认位置，不要写到仓库根：

- 片子根：films/<主题>/
- 字幕：films/<主题>/transcription.srt
- 镜头表：films/<主题>/scenes/scene-plan.json
- 无声成片：films/<主题>/<主题>.mp4
- 配音说明：films/<主题>/dubbing-brief.md
- 旁白轨：films/<主题>/narration.m4a
- 配音成片：films/<主题>/<主题>_配音.mp4

片子根还不存在时先建好。字幕还不在时，把用户附带的字幕放进去，或明确报缺。

## 配音门控

开片先读口令里的配音段。

- 口令没有「配音：需要」时，整条片子无声，不写 brief，也不派配音。
- 口令写了「配音：需要」时，必须有音色和语速。缺一项就停下补问，不自选默认声线。
- provider、model、摆放规则可由本技能补全。口令只写「少儿故事」时，voice 写成 `zh_female_shaoergushi_uranus_bigtts`。口令给了别的音色名或 ID，以口令为准。

写入 `dubbing-brief.md` 时用下面字段：

```text
配音：需要
provider：volcengine-tts
model：seed-tts-2.0-standard
voice：zh_female_shaoergushi_uranus_bigtts
音色名：少儿故事
语速：1.2
粒度：逐句
摆放：按字幕起止时间
处理：去尾静音，与画面节奏对齐
BGM：无
```

## 拆镜头与派导演

输入是片子根里的 transcription.srt。按语义把连续字幕收成镜头，时间首尾相接，空白并入前一个镜头。时长用小数秒。全部镜头时长之和等于最后一条字幕结束时间减第一条开始时间，误差不超过 0.1 秒。把镜头表写进片子根下的 scenes/scene-plan.json。中断后从这张表继续，不重做已经有 mp4 的镜头。

每个镜头在切换书桌之前写好派工内容：镜头编号、小数秒时长、镜头文案、输出文件名 scene-00N.mp4、镜头目录的绝对路径。

然后：

1. 在片子根下建 scenes/scene-001 这样的目录。
2. 从仓库根的 exampleFolder 复制整个 .claude 目录到该镜头目录，再把片子根的 transcription.srt 复制进该镜头目录。
3. 把书桌目录切到该镜头目录。子代理继承切过去那一刻的目录，才能读到里面的 .claude/skills。
4. 调用子代理。agent 填「锦泽-导演」，access 填 write。task 使用上一步写好的派工内容，并写明只交 HTML 和一条渲染命令。
5. 在该镜头目录的持续终端执行渲染命令。命令必须带 --fps 30 和明确的 --output。看终端输出和 mp4 的修改时间。一段时间内没有新消息，先确认进程还在。

同一时间只有一个导演。全部 mp4 完成后，把书桌目录切回 /Users/wulei/Workspace/GitRepo/auto-motion，再调用子代理。agent 填「锦泽-剪辑」，access 填 write。task 带上片子根、主题名、成片文件名 <主题>.mp4、scene-plan.json 和各镜头 mp4 的绝对路径。

## 配音与合轨

无声成片交付后，若 dubbing-brief.md 写明需要配音，再调用子代理。agent 填「锦泽-配音」，access 填 write。task 带上片子根、transcription.srt、dubbing-brief.md，以及无声成片时长 `V`（秒）。

收到 narration.m4a 后，在片子根合轨。以视频时长为准，不要用 `-shortest`，以免旁白偏短时裁掉画面：

```bash
V=$(ffprobe -v error -show_entries format=duration -of csv=p=0 <主题>.mp4)
ffmpeg -y -i <主题>.mp4 -i narration.m4a \
  -filter_complex "[1:a]apad=whole_dur=${V}[a]" \
  -map 0:v:0 -map "[a]" -c:v copy -c:a aac -b:a 160k -t "${V}" \
  <主题>_配音.mp4
```

回报无声成片路径；需要配音时再报配音成片路径。配音失败时，无声成片仍算画面交付完成。

失败时记下镜头编号、阶段、日志路径和重试办法。
