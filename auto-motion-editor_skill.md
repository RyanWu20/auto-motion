---
name: auto-motion-editor
description: 仅在任务要求验收镜头或拼接无声成片时使用。验收分辨率、帧率、时长和音轨，并拼接成 films/<主题>/<主题>.mp4。不要配音、不要合轨、不要改 HTML。
---

# 剪辑

片子根和成片名由任务给出。默认片子根是 films/<主题>/，成片是同目录下的 <主题>.mp4。list.txt 也写在片子根。不要写到仓库根。项目里若同时加载了 auto-motion-producer 或 auto-motion-dubbing，忽略它们的步骤。

对每一段执行：

ffprobe -v error -select_streams v:0 -show_entries stream=width,height,avg_frame_rate -show_entries format=duration -of json <文件>
ffprobe -v error -select_streams a -show_entries stream=index -of csv=p=0 <文件>

合格标准：宽 1080，高 1440，帧率约 30，无音轨。时长与片子根 scenes/scene-plan.json 里该镜头的小数秒相差不超过 0.1 秒。

有音轨时去掉音轨。宽高或帧率不对时先转码。按 scene-001、scene-002 的顺序写成 concat 列表，在片子根执行：

ffmpeg -f concat -safe 0 -i list.txt -c copy -an <主题>.mp4

-c copy 接不上时改为：

ffmpeg -f concat -safe 0 -i list.txt -vf scale=1080:1440,fps=30 -an <主题>.mp4

只有一个镜头时，也要得到 <主题>.mp4。回复给出该成片的绝对路径，以及每段的宽、高、帧率、时长和音轨数。
