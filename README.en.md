[🇨🇳 中文](README.md) · [🇺🇸 English](README.en.md)

# openclaw-plugin-dev

A guide skill for the OpenClaw plugin (extension) **development pipeline and its core gotchas**: from defining purpose, configuration, and build, through local verification, publishing to GitHub, then to ClawHub, and post-publish verification — all following the official docs.

## What problem it solves

Official OpenClaw plugin docs are scattered (17+ pages), and it's easy to hit the host's deliberate security boundaries (sandbox / permissions / iframe). This skill distills a real-plugin pipeline that has been verified end-to-end, and organizes the four plugin forms, full manifest fields, hooks & permissions, install & dependencies, architecture & loading, and build & publish into `references/` — **every section cites its official source**, so the agent can follow along and avoid detours.

## Preview

![Skill directory structure](docs/screenshots/structure.svg)

![Model routing & lookup](docs/screenshots/routing.svg)

## Install

**Option 1: ClawHub (after publish)**

Once published to ClawHub, install with one command:

```bash
clawhub install <your-owner>/openclaw-plugin-dev
```

> Note: the slug `openclaw-plugin-dev` is already taken on ClawHub by a community skill with the same name. It will be renamed or scoped under an owner before publishing; use the actual published name in the command.

**Option 2: Manual placement**

Copy the `openclaw-plugin-dev/` folder from this repo into your OpenClaw skills directory. No extra enable step is needed — the skill is available immediately.

## Usage

1. In a conversation, ask the agent to "develop / debug / publish an OpenClaw plugin" and this skill activates automatically.
2. `SKILL.md` is always loaded, providing the verified pipeline overview and a reference index; the 6 files under `references/` are read on demand — only the one matching the topic is loaded.
3. Jump straight to the right file by topic:

| What you want to do | Where to look |
|---|---|
| Build a tool plugin / what's next | `SKILL.md` pipeline → if needed `references/plugin-forms-and-manifest.md` |
| Fill the manifest / tell the four forms apart | `references/plugin-forms-and-manifest.md` |
| Add a hook / intercept with `before_tool_call` | `references/hooks-and-permissions.md` |
| Install a plugin / missing dependency | `references/install-and-dependencies.md` |
| Debug load failures / understand architecture | `references/architecture-and-loading.md` |
| Publish to ClawHub / build errors | `references/build-and-publish.md` + `SKILL.md` pipeline (second half) |
| Scripts won't run in the sandbox | `references/sandbox-noscript-guide.md` |

4. Every reference section links its official source (`docs.openclaw.ai/zh-CN/plugins/...`) — click through to verify authoritative details.

## License

MIT
