---
name: paper-explainer-video
description: "基于论文 PDF 或论文 URL 制作 60-90 秒 YouTube 论文解读视频，支持中文或英文解说，适用于 CV、ML、NLP、多模态、机器人等 AI/CS 论文，包括 CVPR、ICML、ICLR、NeurIPS、ACL 等会议或 arXiv 论文。流程包含论文读取、标题/问题/贡献/方法/实验/局限提取、真实 figure/table 抽取、口播脚本与逐句字幕、edge-tts 旁白、HyperFrames 16:9 制作、mp4 渲染、抽帧质检和运行日志记录。触发词：论文解读视频、paper explainer、论文视频、读论文做视频。"
---

# 论文解读视频 Skill

使用 `PDF 解析 + edge-tts + HyperFrames` 把一篇视觉/多模态/AI 论文做成 60-90 秒 YouTube 风格解读视频，支持中文或英文旁白。

## 适用场景

- 用户给出论文 PDF 路径、论文 URL、arXiv URL 或会议论文页面，要求做论文解读视频。
- 目标平台是 YouTube、B 站、网页嵌入或横屏播放器。
- 需要真实论文 figure/table 做画面证据，而不是纯 PPT 或虚构示意图。

不适合：没有论文正文、只想要论文摘要文本、需要逐页讲解的长课件视频。

## 输入要求

- 必需：论文 PDF 路径或可下载 PDF URL。
- 可选：目标时长、解说语言、字幕语言、风格、重点受众、是否需要 BGM、是否要引用作者/机构信息。
- 语言规则：用户指定中文或英文时严格遵守；未指定时默认中文。英文视频应使用英文旁白、英文字幕和英文画面主文案。
- 默认规格：16:9，`1920x1080`，60-90 秒，YouTube 论文解读风格。

## 依赖技能

本 skill 依赖 HyperFrames 官方 Codex skills 的通用视频制作知识，尤其是：

- `hyperframes`：HTML composition、时间线、字幕和画面编排规则。
- `hyperframes-cli`：`npx hyperframes lint/inspect/render` 等开发与渲染流程。
- `hyperframes-media`：TTS、转写、音频预处理相关规则。
- `hyperframes-registry`：可复用 block/component 安装与接线。

这些 skills 通常通过以下命令安装到项目 `.agents/skills/`，不要复制到本 skill 仓库里长期维护：

```bash
npx --yes skills add heygen-com/hyperframes
```

## 硬性质量标准

- 必须先完整读取论文，提取：标题、问题、核心贡献、方法、实验结果、局限。
- 必须从论文中抽取真实 figure/table/page crop 作为视频素材，并在视频中实际出现。
- 必须按目标语言写 YouTube 风格口播脚本，并生成逐句字幕。
- 旁白默认使用 `edge-tts`。中文优先 `zh-CN-YunxiNeural` 或 `zh-CN-YunyangNeural`；英文优先 `en-US-GuyNeural`、`en-US-JennyNeural` 或 `en-US-AriaNeural`。
- 先用 `ffprobe` 确认旁白真实时长，再写 HyperFrames 时间线。
- 渲染后必须抽帧检查：画面非空、图表可见、字幕不溢出、不遮挡核心素材、音频时长和视频时长一致。
- 必须记录命令、文件结构、遇到的问题和修复，推荐写入 `RUN_LOG.md`。

## 推荐文件结构

```text
paper-explainer-YYYYMMDD/
├── SCRIPT.md
├── RUN_LOG.md
├── index.html
├── package.json
├── hyperframes.json
├── metadata.json
├── paper_text.txt
├── assets/
│   ├── paper.pdf
│   ├── audio/
│   ├── subtitles/
│   ├── pages/
│   └── figures/
├── scripts/
├── snapshots/
└── renders/
```

## 流程

### 1. 建立项目目录

```bash
mkdir -p paper-explainer-YYYYMMDD/{assets/figures,assets/pages,assets/audio,assets/subtitles,renders,scripts,snapshots}
```

如果仓库有 `AGENTS.md` 或本地命令包装要求，先遵守项目指令。

### 2. 读取论文与抽取素材

优先使用本机稳定工具：

1. 有 `pdftotext/pdfimages/pdftoppm` 时可用 poppler。
2. 没有 poppler 时安装或使用 PyMuPDF：

```bash
python3 -m pip install --user pymupdf
```

抽取内容：

- `paper_text.txt`：按页保存正文文本。
- `metadata.json`：页数、图片候选、页面尺寸、源 PDF。
- `assets/pages/page-XX.png`：每页整页截图，方便裁 figure/table。
- `assets/figures/*`：论文内嵌图片和裁剪后的 figure/table。

裁剪素材时优先覆盖这些画面：

- Overview figure：论文方法总览或 teaser。
- Qualitative figure：能直观看出任务/方法差异的图。
- Main result table：核心 benchmark 表。
- Human/baseline gap table：如果论文有，优先使用。
- Ablation/generalization table：用于解释迁移、鲁棒性或局限。

### 3. 论文内容提炼

在 `SCRIPT.md` 先写信息提取，再写口播。最少包含：

```markdown
- 标题：
- 问题：
- 核心贡献：
- 方法：
- 实验结果：
- 局限：
```

口播建议 10-12 句；中文约 450-650 字，英文约 130-180 words。结构：

1. 开场问题：这篇论文到底解决什么。
2. 为什么难：任务或痛点。
3. Benchmark/data：论文怎么把问题定义清楚。
4. Method：核心方法用一句话讲透。
5. Results：给关键数字。
6. Limitation/takeaway：承认边界，说明意义。

### 4. 生成旁白和逐句字幕

先落盘口播源稿：

```bash
python3 - <<'PY'
from pathlib import Path
# 从 SCRIPT.md 提取或手动写入每句旁白
Path("assets/audio/narration.txt").write_text("第一句\n第二句\n", encoding="utf-8")
PY
```

根据目标语言选择 voice。中文示例：

```bash
edge-tts --voice zh-CN-YunxiNeural --rate +20% \
  --text "$(tr '\n' ' ' < assets/audio/narration.txt)" \
  --write-media assets/audio/narration.mp3 \
  --write-subtitles assets/subtitles/edge-tts.vtt
```

英文示例：

```bash
edge-tts --voice en-US-GuyNeural --rate +8% \
  --text "$(tr '\n' ' ' < assets/audio/narration.txt)" \
  --write-media assets/audio/narration.mp3 \
  --write-subtitles assets/subtitles/edge-tts.vtt
```

检查时长：

```bash
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 assets/audio/narration.mp3
```

如果超过 90 秒，优先压缩脚本；只差一点时再提高语速。`edge-tts` 可能在过高语速报 `NoAudioReceived`，此时删除坏音频并降低 rate 重试。

逐句字幕可按句长比例生成初版 `sentence-subtitles.json/srt`，再按抽帧效果微调。字幕必须跟随旁白出现，不能只在画面上放标题摘要。

### 5. 制作 HyperFrames 视频

创建 `package.json`：

```json
{
  "private": true,
  "type": "module",
  "scripts": {
    "check": "npx --yes hyperframes@0.6.25 lint && npx --yes hyperframes@0.6.25 inspect --samples 18",
    "render": "npx --yes hyperframes@0.6.25 render --quality standard --output renders/paper-explainer.mp4"
  }
}
```

`index.html` 要点：

- root composition 使用 `data-width="1920"`、`data-height="1080"`。
- `data-duration` 跟随旁白真实时长，通常加 0-0.5 秒余量。
- 每个 scene 和 caption 都加稳定 `id`。
- 论文素材使用 `<img src="assets/figures/...">`，确保真实图表在画面中可识别。
- 字幕放底部安全区，使用半透明深色底或描边，避免遮挡表格关键行。
- 可用 GSAP 做轻量进出场，但不要让动画干扰论文数字和图表可读性。

### 6. 检查、渲染、抽帧

```bash
npm run check
npm run render
ffprobe -v error -show_entries format=duration -of default=nw=1:nk=1 renders/paper-explainer.mp4
ffprobe -v error -select_streams a:0 -show_entries stream=duration -of default=nw=1:nk=1 renders/paper-explainer.mp4
```

抽帧：

```bash
ffmpeg -y -i renders/paper-explainer.mp4 \
  -vf "select='eq(n,60)+eq(n,300)+eq(n,660)+eq(n,1140)+eq(n,1680)+eq(n,2160)+eq(n,2520)',scale=960:-1,tile=1x7" \
  -frames:v 1 snapshots/render-contact-sheet.png
```

检查清单：

- 标题页能在 2 秒内说明论文主题。
- 每个核心段落都有真实 figure/table 或论文页面截图。
- 字幕无溢出、无错位、无遮挡关键数据。
- 视频和音频时长基本一致，最后一句没有被截断。
- HyperFrames `lint/inspect` 没有 error 或 layout issue；维护性 warning 可记录。

## 失败处理

- PDF 无法下载：尝试论文页面、arXiv abs/pdf、OpenReview 附件；仍失败则让用户提供本地 PDF。
- PDF 解析工具缺失：优先 PyMuPDF；若安装失败，再用浏览器截图或系统预览导出页面图片。
- figure/table 裁剪不准：先保存整页截图，再按页面比例裁剪；必要时在视频中使用整页局部放大。
- 口播超时：压缩脚本，减少背景介绍和重复数字；不要把 TTS 加速到听感失真。
- 字幕过长：拆成两句或缩短字幕文案，但保留关键信息。
- 渲染 404：检查 `src` 路径是否相对项目根目录，确认素材文件存在。
- 抽帧发现遮挡表格：调整字幕底部安全区、改素材缩放或将关键表格行放大。

## 交付格式

最终回复应包含：

- 成片 mp4 路径。
- 项目目录路径。
- 旁白/字幕/素材/抽帧检查结果。
- 仍存在的 warning 或风险。
- 如用户要求，附上关键论文结论摘要。
