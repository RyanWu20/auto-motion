---
name: auto-motion-director
description: 为当前镜头目录编写 1080x1440 的 HyperFrames HTML，并给出渲染命令。只在任务给出了镜头编号、时长和文案时使用。
---

# 导演

先读当前目录的 transcription.srt，判断这一镜在全片中的位置。文案不必逐字上屏。不认识的名词先用浏览器搜索。有明确 logo 时，把 SVG 或图片下载进当前目录，再写代码。

还没有 index.html 时执行：

npx hyperframes init . --non-interactive --example blank

不要加 --resolution portrait。

根节点写成 data-width="1080"、data-height="1440"，data-duration 等于任务给出的小数秒。每段画面写 data-start、data-duration、data-track-index。动效使用暂停的 GSAP 时间轴，并挂到 window.__timelines。同时阅读已加载的 hyperframes、hyperframes-motion、hyperframes-design，以及 motion-graphics 或 general-video。

任务已经给齐编号、时长和文案时，直接写文件。HyperFrames 技能里「渲染前停在 preview」这一条不执行，也不要自己把渲染等到结束。不要使用 Date.now()、运行时随机数，也不要在渲染时访问网络。

回复只留两样：index.html 已写好，以及一条在当前目录执行的命令：

npx hyperframes render --fps 30 --quality high --output <镜头文件名>.mp4
