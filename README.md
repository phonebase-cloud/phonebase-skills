<div align="center">
  <h1>PhoneBase Skills</h1>
  <p>AI agent skills for <a href="https://github.com/phonebase-cloud/phonebase-cli">PhoneBase</a> cloud phone CLI</p>
  <p>
    <a href="https://skills.sh/phonebase-cloud/phonebase-skills"><img src="https://img.shields.io/badge/skills.sh-phonebase-2F81F7.svg" alt="skills.sh" /></a>
    <a href="https://github.com/phonebase-cloud/phonebase-cli"><img src="https://img.shields.io/badge/pb%20CLI-1.0.4+-2F81F7.svg" alt="pb CLI 1.0.4+" /></a>
  </p>
  <p><a href="./README.md">English</a> | <a href="./README.zh-CN.md">简体中文</a></p>
</div>

## Overview

This repo provides agent skills that teach AI coding assistants (Claude Code, Cursor, Copilot, etc.) how to operate Android cloud phones through the [`pb` CLI](https://github.com/phonebase-cloud/phonebase-cli). Once installed, your agent can launch apps, tap buttons, type text, browse the web, install APKs, and more — all on a remote cloud device.

## Install

```bash
# All skills
npx skills add phonebase-cloud/phonebase-skills

# Only cloud phone control
npx skills add phonebase-cloud/phonebase-skills@phonebase

# Only skill authoring guide
npx skills add phonebase-cloud/phonebase-skills@phonebase-skill-creator
```

## Skills

| Skill | Description |
|-------|-------------|
| [`phonebase`](./skills/phonebase/SKILL.md) | Control Android cloud phones — launch apps, tap/swipe/type, browse, install APKs, manage files, and automate workflows |
| [`phonebase-skill-creator`](./skills/phonebase-skill-creator/SKILL.md) | Write app skills for pb CLI — directory layout, SDK API, scripting patterns, and testing |

## Requirements

- A [PhoneBase](https://phonebase.cloud) account

