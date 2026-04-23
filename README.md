# claude-design-to-video

> **Claude Design HTML export → MP4 video** — deterministic frame-by-frame rendering via timecut.

Claude Design (Anthropic Labs) exports animated prototypes as **standalone HTML** (React + Babel + JSX scenes). This skill converts that export to an MP4 video without frame drops, glitches, or quality loss — ready for YouTube, Instagram, LinkedIn, or anywhere else.

---

## Why this skill exists

Recording a Claude Design animation with Playwright/Puppeteer/OBS **doesn't work**. Browsers render frames when they can, skip frames under load, and tie animations to wall-clock time. If a screenshot takes 200ms but your animation expects 16ms frames, you get a stuttery mess.

**timecut** solves this by **virtualizing time** — it overrides `requestAnimationFrame`, `setTimeout`, `Date.now`, and `performance.now` inside the browser, then renders each frame at its exact virtual timestamp. Screenshots can take as long as they need; the resulting video is perfectly smooth.

The skill also **auto-patches known CSS bugs** in Claude Design exports (scrubber, blink keyframes, fade transitions, webcam idle motion) before rendering — so you don't need to hand-edit the export.

---

## What you get

- **MP4 output**, 1920×1080 @ 30fps by default (or 4K, or custom resolution/fps)
- **YouTube-grade quality** — H.264, yuv420p, CRF 18, preset slow
- **Zero drop frames** — every frame rendered deterministically
- **Auto-patched CSS bugs** — see [Auto-patches](#auto-patches) below
- **Cleanup out of the box** — temp files deleted via `trap`, only the final MP4 stays
- **Persistent cache** — timecut installed once, reused forever

---

## Requirements

| Tool | Minimum | Check |
|---|---|---|
| **Claude Code** | latest | skill loads via `.claude/skills/` |
| **ffmpeg** | ≥ 4 | `ffmpeg --version` |
| **Node.js** | ≥ 18 | `node --version` |
| **Python 3** | ≥ 3.8 | `python3 --version` (for HTTP server) |
| **Chrome / Chromium** | any recent | auto-detected on macOS / Linux / Windows |

First run installs `timecut` globally-free (cached in `~/.cache/claude-design-to-video/timecut/`, ~150 MB).

**OS support:** macOS ✅ (fully tested) · Linux ✅ (Chrome path auto-detected) · Windows ⚠️ (Git Bash / WSL recommended, untested).

---

## Installation

### Option A — manual copy (easiest)

```bash
git clone https://github.com/YOUR_USER/claude-design-to-video.git
cp -R claude-design-to-video/skill ~/.claude/skills/claude-design-to-video
```

Restart Claude Code. The skill appears as `/claude-design-to-video`.

### Option B — `.skill` package

Download `claude-design-to-video.skill` from the [releases page](https://github.com/YOUR_USER/claude-design-to-video/releases), then install via Claude Code's skill installer.

---

## Quick start

### 1. Export your animation from Claude Design
Share → **Download standalone HTML**. You'll get a folder like:

```
My Animation/
├── index.html          # React + Babel entry
├── animations.jsx      # <Stage>, useTime, useTimeline engine
├── scenes/             # individual scene components
└── assets/             # images, fonts
```

### 2. Call the skill

```
You: wyrenderuj mi animację z Downloads/My Animation
```

or explicitly:

```
You: /claude-design-to-video "~/Downloads/My Animation"
```

### 3. Wait
- **Preview** (15s): ~1 min
- **Full 74s @ 1080p30 fast:** ~10 min
- **Full 74s @ 1080p30 HQ:** ~15 min
- **Full 74s @ 4K30 HQ:** ~40–60 min

Output lands in `~/Downloads/<slug>-<WxH>-<fps>-<mode>.mp4`, auto-opens in the default player.

---

## Flags

| flag | effect | default |
|---|---|---|
| `--hq` | CRF 18, preset slow (YouTube source grade) | ✓ on |
| `--fast` | CRF 20, preset medium (iteration, -30% time) | off |
| `--preview N` | render only first N seconds (quick check) | off |
| `--4k` | 3840×2160 (default 1920×1080) | off |
| `--fps N` | frame rate | 30 |
| `--no-patch` | skip auto-patch CSS (if bundle is already clean) | off |
| `--in-place` | patch the original folder instead of a temp copy | off (safer) |
| `--output PATH` | override output location | `~/Downloads/...` |

---

## Auto-patches

Claude Design exports contain CSS timing that breaks frame-by-frame capture (timecut controls JS time only, not CSS `@keyframes` or `transition:`). The skill auto-detects and rewrites these patterns before rendering:

| Pattern | Fix |
|---|---|
| `<PlaybackBar>` scrubber in `<Stage>` | Hidden via `?render=1` URL flag |
| `@keyframes blink` + `animation: 'blink Ns infinite'` | JS `opacity` from `useTime()` with linear fade |
| `transition: opacity Nms cubic-bezier` on webcam | JS ease-out cubic interpolation |
| Webcam idle motion (`sway`, `breathe` via `Math.sin`) | Disabled (subpixel jitter looks bad in pre-rendered video) |

Unknown patterns → warning + skip (render proceeds, user can hand-fix if needed). New patterns can be added to the skill's patch list over time.

---

## Known limitations

- **CSS transitions with dynamic layout** (e.g. `transition: flex 600ms cubic-bezier`) aren't auto-patched — flex interpolation requires knowing final dimensions, too risky. If you hit one, either port it to JS `interpolate()` manually, or accept the minor glitch.
- **Windows** isn't fully tested. Git Bash / WSL should work, PowerShell/CMD may need tweaks.
- **Intel Macs** render ~1.5–2× slower than Apple Silicon.

---

## Troubleshooting

See the **Failure modes** section in `skill/SKILL.md` — it covers port conflicts, Chrome timeouts, mid-render crashes, offline npm install, and frame count mismatches.

---

## How it works under the hood

```
1. Preflight       → check ffmpeg, Chrome, Node, Python3
2. Locate bundle   → arg or autodetect in ~/Downloads/
3. Parse metadata  → DURATION, viewport from HTML
4. Copy to temp    → mktemp -d, preserves original
5. Auto-patch CSS  → regex-based, 4 known bugs
6. Install timecut → one-time, cached
7. Serve bundle    → python3 http.server on random port
8. Render          → timecut → PNG frames → ffmpeg → MP4
9. Cleanup (trap)  → kill HTTP server, rm -rf temp
10. Report + open  → ffprobe stats, open MP4
```

Full flow lives in [`skill/SKILL.md`](skill/SKILL.md).

---

## Credits

- **[timecut](https://github.com/tungs/timecut)** by Steve Tung — the virtualized-time browser rendering engine that makes this possible.
- **ffmpeg** — for the final encode.
- **Anthropic** — for Claude Design.

---

## License

MIT — see [LICENSE](LICENSE).

---

## Contributing

Issues and PRs welcome. Especially interested in:
- New CSS pattern auto-patches (as Claude Design export evolves)
- Windows testing + path fixes
- Linux distro compatibility reports
- Performance improvements (parallel render? canvas capture mode?)
