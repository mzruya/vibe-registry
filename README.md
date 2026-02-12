# vibe-registry

Formula registry for the [Vibe](https://github.com/mzruya/vibe) AI-powered package manager.

## What is this?

This repository contains formulas - versioned prompts that tell AI coding agents how to generate and build software. Each formula consists of:

- `formula.toml` - Package metadata (name, version, description, build config)
- `prompt.md` - The prompt sent to the AI agent to generate the code

## Usage

Install packages using the [Vibe CLI](https://github.com/mzruya/vibe):

```sh
vibe install <package-name>
```

## Structure

```
formulas/
  <package-name>/
    formula.toml
    prompt.md
```
