# video-render-pdf skills

这个仓库托管多个 Codex skill，用于将视频讲座转换为结构化的中文 LaTeX 讲义和最终 PDF，也支持本地视频理解与处理。

| Skill | 平台 | 说明 |
|-------|------|------|
| `youtube-render-pdf` | YouTube | 原始版本，利用 YouTube CC 字幕和章节结构 |
| `bilibili-render-pdf` | Bilibili (B站) | 适配 B 站的字幕缺失、登录高清、分P视频等特点 |
| `local-video-render-pdf` | 本地视频文件 | 处理 `.mp4/.mov/.mkv/.webm` 等本地视频，结合 sidecar 字幕/转写/抽帧/视觉理解生成讲义 PDF |

这些 skill 共享相同的写作规则、配图策略和 LaTeX 模板，但在素材获取阶段有平台或本地文件特定的差异。

### Bilibili 版的核心差异

- **字幕三级回退**：CC 字幕 → Whisper 语音转写 → 纯视觉模式（B 站大量视频无 CC 字幕）
- **登录获取高清**：1080P+ 需要 cookies（`yt-dlp --cookies-from-browser chrome`）
- **分P视频处理**：自动检测多 P，询问用户处理范围
- **平台话术过滤**：额外排除"一键三连"、"关注投币"等非教学内容
- **额外依赖**：`whisper`（openai-whisper）用于语音转写

### 本地视频版的核心差异

- **本地文件入口**：直接处理本地 `.mp4/.mov/.mkv/.webm/.m4v/.avi` 等文件或包含视频与 sidecar 文件的目录
- **素材自动归集**：优先使用同名 `.srt/.vtt` 字幕、`.txt/.md` 转写稿、`.pdf/.pptx` 讲义、`cover.*`/`thumbnail.*` 图片
- **字幕多级回退**：sidecar 字幕 → 内嵌字幕 → sidecar 转写稿 → Whisper 语音转写 → 纯视觉模式
- **本地抽帧理解**：用 `ffprobe/ffmpeg` 检查媒体信息、抽取候选帧和代表帧，再通过视觉检查筛选关键图
- **真实标注封面**：本地视频若无官方封面，使用视觉确认的代表帧，并在模板中标注为“视频代表帧”

### 共同特点

- 以视频真实教学内容为主，而不是只依赖字幕转写
- 在线视频优先使用原始封面作为首页封面图；本地视频优先使用用户提供封面，否则使用视觉确认的代表帧
- 按教学价值提取关键画面、图表、公式和代码片段
- 生成带 `\section{}` / `\subsection{}` 结构的完整 `.tex`
- 最终必须落到可交付的 PDF

## 仓库结构

```text
.
├── LICENSE
├── README.md
└── skills/
    ├── youtube-render-pdf/
    │   ├── SKILL.md
    │   ├── agents/
    │   │   └── openai.yaml
    │   └── assets/
    │       └── notes-template.tex
    ├── local-video-render-pdf/
    │   ├── SKILL.md
    │   ├── agents/
    │   │   └── openai.yaml
    │   └── assets/
    │       └── notes-template.tex
    └── bilibili-render-pdf/
        ├── SKILL.md
        ├── agents/
        │   └── openai.yaml
        └── assets/
            └── notes-template.tex
```

## 使用方式

如果你想在本地 Codex 环境中使用这些 skill，可以把对应目录放到你的技能目录中：

```bash
mkdir -p ~/.codex/skills

# YouTube 版
cp -R skills/youtube-render-pdf ~/.codex/skills/

# Bilibili 版
cp -R skills/bilibili-render-pdf ~/.codex/skills/

# 本地视频版
cp -R skills/local-video-render-pdf ~/.codex/skills/
```

然后在 Codex 中使用对应 skill 处理视频链接或本地视频文件，请求生成讲义 `.tex` 和最终 PDF。

## 外部依赖

| 工具 | YouTube/Bilibili | 本地视频 | 说明 |
|------|:-:|:-:|------|
| `yt-dlp` | ✓ | | 在线平台下载与字幕抓取 |
| `ffprobe` | | ✓ | 本地视频媒体信息检查，通常随 `ffmpeg` 安装 |
| `ffmpeg` | ✓ | ✓ | 音频提取、字幕提取、抽帧 |
| `xelatex` (TeX Live + CTeX) | ✓ | ✓ | 编译最终 PDF |
| `magick` (ImageMagick) | ✓ | ✓ | 候选帧 contact sheet 和图片处理 |
| `whisper` (openai-whisper) | Bilibili 常用 | ✓ | 无字幕时语音转写 |

此外，运行 skill 的 coding agent 必须具备一定的读图能力，否则很难选择关键帧，很难做到图文align（即至少是一个还不错的 vlm model，ps. MiniMax 2.7 只是一个纯文本模型）。


## subagents 的触发

- codex 中对于 `spwan_agent` 的触发，规定的比较死，"Only use spawn_agent if and only if the user explicitly asks for sub-agents, delegation, or parallel agent work."，即需要我们在 query 中显式地要求，才可以触发 subagents

```
$youtube-render-pdf   https://www.youtube.com/watch?v=vXb2QYOUzl4 请 spwan 多 sub agents 执行，隔离上下文，避免 master agent 的“上下文焦虑”， 形成一个完整全面的 pdf：
  - 1 个 outline agent：先定全局目录、术语、符号表、章节边界等
  - 5 个 writer agents：各自直接写成完整章节草稿，落盘成 section_*.tex
  - 1 个 figure agent：单独负责抽帧、筛图、crop、脚本生成新的示意图、图注和时间脚注等；
  - 1 个 consistency agent：检查重复定义、前后术语不一致、章节衔接断裂
```

## tips

- 强烈建议在 codex 基于这个 skill 给出第一版结果之后，增加一个follow up question：`spwan 一个独立的reviewer agent，基于原始字幕文件，check 是否有重要、细节及有趣等一切有意的信息的漏召回，仅反馈，不修改，不断交互，直至reviewer agent 觉得 tex 信息已完备。`
  - 以缓解 ai extraction/summary 的共性问题，就是召回不足；

## License

仓库保留了根目录下原有的 `LICENSE` 文件。使用、分发或二次修改时，请以该许可证为准。
# video-pdf-skills-main
