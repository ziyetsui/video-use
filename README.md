<p align="center">
  <img src="static/video-use-banner.png" alt="video-use" width="100%">
</p>

# video-use

Introducing **video-use** — edit videos with Claude Code. 100% open source.

Drop raw footage in a folder, chat with Claude Code, get `final.mp4` back. Works for any content — talking heads, montages, tutorials, travel, interviews — without presets or menus.

Cloud-hosted video-use is coming. [Join the waitlist at video-use.com](https://video-use.com?utm_source=github).

## 使用前必须配置 API（Required API setup）

**本项目不附带 API 密钥。首次转写前，请自行配置 ElevenLabs API；仅安装依赖不能完成语音转写。**

1. 在 [ElevenLabs API Keys](https://elevenlabs.io/app/settings/api-keys) 创建自己的密钥。
2. 在本仓库根目录执行 `cp .env.example .env`，用本地编辑器填写：

   ```dotenv
   ELEVENLABS_API_KEY=your_own_elevenlabs_api_key
   ```

   也可以设置同名环境变量。若 `.env` 中已有此变量，请先填写或删除空值，因为当前转写脚本优先读取 `.env`。
3. 密钥仅保存在本地；不要发到聊天、Issue 或提交到 GitHub。`.env` 已被 Git 忽略，`.env.example` 仅提供空模板。

ElevenLabs Scribe 用于语音转写，需有可用额度，调用费用由你自己的账户承担。未配置时先完成配置，再运行转写；FFmpeg 本地剪辑与已有转写缓存的复用不新增转写调用。运行 Claude Code、Codex 等助手还需准备对应平台的账号/额度。此仓库的基础剪辑流程没有内置数字人服务，另加数字人或云端生成服务时需单独配置对应 API。

## What it does

- **Cuts out filler words** (`umm`, `uh`, false starts) and dead space between takes
- **Auto color grades** every segment (warm cinematic, neutral punch, or any custom ffmpeg chain)
- **30ms audio fades** at every cut so you never hear a pop
- **Burns subtitles** in your style — 2-word UPPERCASE chunks by default, fully customizable
- **Generates animation overlays** via [HyperFrames](https://github.com/heygen-com/hyperframes), [Remotion](https://www.remotion.dev/), [Manim](https://www.manim.community/), or PIL — spawned in parallel sub-agents, one per animation
- **Self-evaluates the rendered output** at every cut boundary before showing you anything
- **Persists session memory** in `project.md` so next week's session picks up where you left off

## Setup prompt

Paste into Claude Code, Codex, Hermes, Openclaw, or any agent with shell access:

```text
Set up https://github.com/ziyetsui/video-use for me.

Read install.md first to install this repo, wire up ffmpeg, register the skill with whichever agent you're running under, and set up the ElevenLabs API key — tell me how to enter it locally in .env; never ask me to paste secrets into chat. Then read SKILL.md for daily usage, and always read helpers/ because that's where the editing scripts live. After install, don't transcribe anything on your own — just tell me it's ready and wait for me to drop footage into a folder.
```

The agent handles the clone, dependencies, skill registration, and guides you to configure your own ElevenLabs API key locally (grab one at [elevenlabs.io/app/settings/api-keys](https://elevenlabs.io/app/settings/api-keys)).

Then point your agent at a folder of raw takes:

```bash
cd /path/to/your/videos
claude    # or codex, hermes, etc.
```

For always-on editing from your own VPS or Telegram, run the agent through [Browser Use Box](https://browser-use.com/bux). [Watch the 15-second demo](https://www.tiktok.com/@browser_use/video/7639824093721758989).

And in the session:

> edit these into a launch video

It inventories the sources, proposes a strategy, waits for your OK, then produces `edit/final.mp4` next to your sources. All outputs live in `<videos_dir>/edit/` — the skill directory stays clean.

## Manual install

If you'd rather do it by hand:

```bash
# 1. Clone and symlink into your agent's skills directory
git clone https://github.com/ziyetsui/video-use ~/Developer/video-use
ln -sfn ~/Developer/video-use ~/.claude/skills/video-use        # Claude Code
# ln -sfn ~/Developer/video-use ~/.codex/skills/video-use       # Codex

# 2. Install deps
cd ~/Developer/video-use
uv sync                         # or: pip install -e .
brew install ffmpeg             # required
brew install yt-dlp             # optional, for downloading online sources

# 3. Add your ElevenLabs API key
cp .env.example .env
$EDITOR .env                    # ELEVENLABS_API_KEY=...
```

## How it works

The LLM never watches the video. It **reads** it — through two layers that together give it everything it needs to cut with word-boundary precision.

<p align="center">
  <img src="static/timeline-view.svg" alt="timeline_view composite — filmstrip + speaker track + waveform + word labels + silence-gap cut candidates" width="100%">
</p>

**Layer 1 — Audio transcript (always loaded).** One ElevenLabs Scribe call per source gives word-level timestamps, speaker diarization, and audio events (`(laughter)`, `(applause)`, `(sigh)`). All takes pack into a single ~12KB `takes_packed.md` — the LLM's primary reading view.

```
## C0103  (duration: 43.0s, 8 phrases)
  [002.52-005.36] S0 Ninety percent of what a web agent does is completely wasted.
  [006.08-006.74] S0 We fixed this.
```

**Layer 2 — Visual composite (on demand).** `timeline_view` produces a filmstrip + waveform + word labels PNG for any time range. Called only at decision points — ambiguous pauses, retake comparisons, cut-point sanity checks.

> Naive approach: 30,000 frames × 1,500 tokens = **45M tokens of noise**.
> Video Use: **12KB text + a handful of PNGs**.

Same idea as browser-use giving an LLM a structured DOM instead of a screenshot — but for video.

## Pipeline

```
Transcribe ──> Pack ──> LLM Reasons ──> EDL ──> Render ──> Self-Eval
                                                              │
                                                              └─ issue? fix + re-render (max 3)
```

The self-eval loop runs `timeline_view` on the _rendered output_ at every cut boundary — catches visual jumps, audio pops, hidden subtitles. You see the preview only after it passes.

## Design principles

1. **Text + on-demand visuals.** No frame-dumping. The transcript is the surface.
2. **Audio is primary, visuals follow.** Cuts come from speech boundaries and silence gaps.
3. **Ask → confirm → execute → self-eval → persist.** Never touch the cut without strategy approval.
4. **Zero assumptions about content type.** Look, ask, then edit.
5. **12 hard rules, artistic freedom elsewhere.** Production-correctness is non-negotiable. Taste isn't.

See [`SKILL.md`](./SKILL.md) for the full production rules and editing craft.
