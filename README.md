# srht-builds

A skill for AI coding agents that adds expert knowledge of [builds.sr.ht](https://builds.sr.ht/) — SourceHut's CI service.

## What it does

- **Write manifests** — generate `.build.yml` files from a project description
- **Debug builds** — diagnose failing manifests and fix common pitfalls (secrets leaking, `cd` scope, wrong package managers)
- **Migrate CI** — convert GitHub Actions, GitLab CI, or CircleCI configs to sr.ht manifests
- **Submit jobs** — submit builds via the REST API or `hut` CLI, poll for status, download artifacts

## Install

### Claude Code

```bash
claude skill add https://github.com/M1n-74316D65/srht-builds
```

Or manually:

```bash
mkdir -p ~/.claude/skills
git clone https://github.com/M1n-74316D65/srht-builds.git ~/.claude/skills/srht-builds
```

### OpenCode

```bash
mkdir -p ~/.config/opencode/skills
git clone https://github.com/M1n-74316D65/srht-builds.git ~/.config/opencode/skills/srht-builds
```

Or project-local:

```bash
mkdir -p .opencode/skills
git clone https://github.com/M1n-74316D65/srht-builds.git .opencode/skills/srht-builds
```

### Mistral Vibe

```bash
mkdir -p ~/.vibe/skills
git clone https://github.com/M1n-74316D65/srht-builds.git ~/.vibe/skills/srht-builds
```

Or project-local (trusted folders only):

```bash
mkdir -p .vibe/skills
git clone https://github.com/M1n-74316D65/srht-builds.git .vibe/skills/srht-builds
```

### Pi Agent

```bash
mkdir -p ~/.pi/agent/skills
git clone https://github.com/M1n-74316D65/srht-builds.git ~/.pi/agent/skills/srht-builds
```

Or project-local:

```bash
mkdir -p .pi/skills
git clone https://github.com/M1n-74316D65/srht-builds.git .pi/skills/srht-builds
```

### Any Agent Skills-compatible agent

The `.agents/skills/` directory in this repo follows the [Agent Skills Specification](https://agentskills.io). Clone the repo and point your agent at it:

```bash
mkdir -p ~/.agents/skills
git clone https://github.com/M1n-74316D65/srht-builds.git ~/.agents/skills/srht-builds
```

Or for a project, add it as a submodule:

```bash
git submodule add https://github.com/M1n-74316D65/srht-builds.git .agents/skills/srht-builds
```

## Usage

Once installed, the skill activates automatically when you mention builds.sr.ht, SourceHut CI, `.build.yml`, or related topics. Example prompts:

- "Write me a .build.yml for my Go project on Alpine"
- "My build manifest keeps printing my API key in the logs — how do I fix it?"
- "Migrate this GitHub Actions workflow to sr.ht"
- "How do I submit a build from a bash script using the API?"

## Repo mirrors

- **GitHub:** https://github.com/M1n-74316D65/srht-builds
- **SourceHut:** https://git.sr.ht/~m1n/srht-builds
