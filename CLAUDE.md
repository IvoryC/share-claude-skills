# CLAUDE.md

This file provides guidance to Claude Code (claude.ai/code) when working with code in this repository.

## What this repo is

A collection of Claude Code skills shared via git. Skills live under `skills/` (personal/shareable) and `.claude/skills/` (project-level, apply only when working in this repo). Each skill is a directory containing a `SKILL.md` with YAML frontmatter and markdown instructions for Claude.

## Skill structure

Every skill directory must have a `SKILL.md`. Required frontmatter fields: `name` (must match directory name) and `description` (trigger conditions for model-invoked, or `/help` text for user-invoked). Optional fields: `version`, `allowed-tools`, `model`, `disable-model-invocation`, `user-invocable`, `context`, `agent`.

Large skills split reference material into a `references/` subdirectory. The `SKILL.md` links to those files — see `skills/build-q2-pluggins/` as the canonical example.