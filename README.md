# product-film

A Claude skill for making product launch and promo videos. The film is one HTML file: a 1920×1080 stage, a scene timeline, a pure `render(t)` function, and a soundtrack synthesized from the same timeline. You preview it in the browser, and a small script exports a frame-exact 1080p60 MP4 with audio through headless Chrome and ffmpeg.

Once installed, ask Claude for a launch video, promo, teaser or animated product demo and it follows the skill: gather inspiration, storyboard, build, score, check frames, export.

Bring your own reference videos (links, files or screenshots) and Claude studies them and shows you what it plans to borrow before building. With no references, it offers the two house references the skill ships with.

## Install

**Claude Code (recommended)**

```
/plugin marketplace add Kinfe123/product-film-skill
/plugin install product-film@product-film-skill
```

**Manual**

```bash
git clone https://github.com/Kinfe123/product-film-skill.git
mkdir -p ~/.claude/skills
cp -r product-film-skill/plugins/product-film/skills/product-film ~/.claude/skills/
```

**Claude desktop app or claude.ai:** download `SKILL.md` from `plugins/product-film/skills/product-film/` and upload it under Settings → Capabilities → Skills.

## Requirements

- Chrome or Chromium
- `ffmpeg` with libx264
- Node 22+

## What's inside

| Path | What it is |
|---|---|
| `plugins/product-film/skills/product-film/SKILL.md` | The skill: workflow, rules that keep the export exact, pacing, sound, the export script |
| `examples/film-template.html` | A working 16s starter film (title, app demo with a cursor click, end card). Open it in a browser and press space |

## Try the example

```bash
open examples/film-template.html        # space = play, ←/→ = frame step, ?t=8 jumps to 8s
```

To export it, save the capture script from `SKILL.md` as `film-capture.mjs`, then:

```bash
node film-capture.mjs examples/film-template.html film.mp4
```

## License

[MIT](LICENSE)
