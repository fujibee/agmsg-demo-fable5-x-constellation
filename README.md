# AI Constellation — what X is talking about right now

An interactive 3D constellation map of AI-model chatter on X. Each model is a constellation, each topic is a star (size = mention volume, pulse = heat). Drag to rotate, scroll to zoom, click a star to read the actual posts behind it.

**Live demo: https://fujibee.github.io/agmsg-demo-fable5-x-constellation/**

Built end-to-end by a team of AI agents from **a single human instruction**, inside the [agmsg](https://github.com/fujibee/agmsg) desktop app.

## How it was made

One instruction to Fable 5. No human intervention after that.

1. **Fable 5** (Claude Code) wrote a 21,283-character implementation-ready design doc → [`DESIGN.md`](DESIGN.md)
2. **Grok** collected live topic data from X → [`data.json`](data.json) — Fable inspected it and sent it back once for more posts per topic
3. **Opus 4.8** (Claude Code) implemented the whole app as a single HTML file → [`index.html`](index.html)
4. **Codex** polished it (bloom, line quality, starfield, entrance choreography, mobile)
5. **Fable 5** did the final visual review — measured actual screen pixels, caught a shader color-space double-conversion bug (background 10x brighter than spec), prescribed the exact fix, re-measured, approved

The full, unedited team-room conversation: [`TRANSCRIPT.md`](TRANSCRIPT.md)
Review screenshots (including the before/after of Fable's color-space fix): [`review/`](review/)

## Why

Fable 5 is brilliant and expensive. So it only did the two things that need its visual reasoning: the design and the final review. Everything hands-on went to cheaper teammates (Opus / Codex / Grok), coordinated over [agmsg](https://github.com/fujibee/agmsg) — all visible in one window of the agmsg desktop app.

## Run locally

It's one self-contained HTML file (three.js via CDN):

```bash
python3 -m http.server 8931
# open http://localhost:8931/index.html
```

---

Built with [agmsg](https://github.com/fujibee/agmsg) 🪐
