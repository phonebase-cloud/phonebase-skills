<div align="center">
  <h1>PhoneBase Skills</h1>
  <p><a href="https://github.com/phonebase-cloud/phonebase-cli">PhoneBase</a> 云手机 CLI 的 AI agent 技能</p>
  <p>
    <a href="https://skills.sh/phonebase-cloud/phonebase-skills"><img src="https://img.shields.io/badge/skills.sh-phonebase-2F81F7.svg" alt="skills.sh" /></a>
    <a href="https://github.com/phonebase-cloud/phonebase-cli"><img src="https://img.shields.io/badge/pb%20CLI-1.0.4+-2F81F7.svg" alt="pb CLI 1.0.4+" /></a>
  </p>
  <p><a href="./README.md">English</a> | <a href="./README.zh-CN.md">简体中文</a></p>
</div>

## 概述

这个仓库提供 AI agent 技能，让 AI 编程助手（Claude Code、Cursor、Copilot 等）学会通过 [`pb` CLI](https://github.com/phonebase-cloud/phonebase-cli) 操控 Android 云手机。安装后，你的 AI 助手可以启动应用、点击按钮、输入文字、浏览网页、安装 APK 等 — 所有操作都在远程云设备上执行。

## 安装

```bash
# 安装全部技能
npx skills add phonebase-cloud/phonebase-skills

# 只安装云手机操控
npx skills add phonebase-cloud/phonebase-skills/phonebase

# 只安装 skill 编写指南
npx skills add phonebase-cloud/phonebase-skills/phonebase-skill-creator
```

## 技能列表

| 技能 | 说明 |
|------|------|
| [`phonebase`](./skills/phonebase/SKILL.md) | 操控 Android 云手机 — 启动应用、点击/滑动/输入、浏览网页、安装 APK、管理文件、自动化工作流 |
| [`phonebase-skill-creator`](./skills/phonebase-skill-creator/SKILL.md) | 编写 pb CLI 的应用技能 — 目录结构、SDK API、脚本模式、测试流程 |

## 前置要求

- 拥有 [PhoneBase](https://phonebase.cloud) 账号

