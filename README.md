# daana-modeler

A Claude Code skill that interviews you to build DMDL model files.

[![License: MIT](https://img.shields.io/badge/License-MIT-yellow.svg)](LICENSE)
[![Built for Claude Code](https://img.shields.io/badge/Built_for-Claude_Code-6f42c1.svg)](https://docs.anthropic.com/en/docs/claude-code)

## What It Does

`/daana` is a slash command for [Claude Code](https://docs.anthropic.com/en/docs/claude-code) that walks you through an interactive interview to define business entities, attributes, and relationships — then generates a valid DMDL `model.yaml` file. Learn more about Daana and DMDL at [docs.daana.dev](https://docs.daana.dev).

## Installation

**As a Claude Code plugin** (recommended):

```bash
claude plugin add https://github.com/mattiasthalen/daana-modeler
```

**As a local skill:**

Copy the `skills/daana/` directory into your project's `.claude/skills/` directory.

## Usage

Run the `/daana` slash command in Claude Code. The skill will interview you to define your data model step by step, then write a `model.yaml` file to your project.

## Documentation

- [Daana CLI](https://docs.daana.dev) — Daana CLI documentation
- [DMDL Specification](https://docs.daana.dev/dmdl) — DMDL language reference

## License

This project is licensed under the MIT License — see the [LICENSE](LICENSE) file for details.
