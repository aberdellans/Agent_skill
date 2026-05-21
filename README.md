# Agent Skills for the Inveniam.io Platform

Agent Skills that teach an AI agent how to work with the **Inveniam.io platform** — its deals, data rooms, and workflows. Install a skill into your AI agent, and it gains that know-how automatically, without you re-explaining the platform each time.

Skills follow the open **Agent Skills** standard and work across multiple hosts — Claude Code, OpenAI Codex, and ChatGPT among them. These skills are intended for people who work with the Inveniam.io platform.

## What's in here

| Skill | What it covers |
|---|---|
| [`inveniam/`](inveniam/) | Operating the Inveniam platform — deals and portfolios, data-room folders and documents, extracted data artifacts, file-veracity checks, and per-deal workflows (tasks, checklists, RACI, comments). Includes the resource model, task-status transition rules, data-room write limits, known idiosyncrasies, and common multi-step recipes. It is host-neutral and covers reaching the platform via an MCP server or the V2 REST API directly. |

## What is an Agent Skill?

A skill is a folder containing a `SKILL.md` file — frontmatter (`name`, `description`) plus Markdown instructions. A supporting agent host discovers the skill and automatically pulls in its guidance whenever a task is relevant, so the agent already knows the platform's structure, quirks, and workflows instead of rediscovering them by trial and error. You don't invoke a skill by hand; it triggers itself from its description.

## Installing a skill

First get the files:

```bash
git clone https://github.com/aberdellans/Agent_skill.git
```

(No git? Use GitHub's **Code → Download ZIP** button instead.)

Then place the `inveniam` folder where your host looks for skills:

**Claude Code**
```bash
mkdir -p ~/.claude/skills
cp -r Agent_skill/inveniam ~/.claude/skills/
```
Start a new session — skills are loaded when a session begins.

**OpenAI Codex**
Codex scans `.agents/skills/` directories. Copy the folder into the user-level one (available everywhere), or a repo-level `.agents/skills/` for a single project:
```bash
mkdir -p ~/.agents/skills
cp -r Agent_skill/inveniam ~/.agents/skills/
```
Codex detects it automatically; restart Codex if it doesn't appear.

**ChatGPT**
ChatGPT supports Skills under the same open standard. Upload the `inveniam` skill — zipping the folder if an archive is required — following OpenAI's current [ChatGPT Skills instructions](https://help.openai.com/en/articles/20001066).

## Using it

Once installed, just describe your task in plain language — *"list my Inveniam deals,"* *"summarize the open workflow tasks on the deal"* — and the agent loads the skill on its own. There is no command to run.

## A skill is know-how, not access

Installing a skill teaches the agent *how* the platform works; it does not by itself connect the agent to your Inveniam data. You still need a way for the agent to reach the platform — the `inveniam` skill's own "Reaching the platform" section explains the options (an Inveniam MCP server, or the Inveniam V2 REST API directly).

## Updates

This repository is updated as the platform evolves. Pull the latest and re-copy the skill folder to stay current.
