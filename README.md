# Agent Skills for the Inveniam.io Platform

Claude Agent Skills that teach an AI agent how to work with the **Inveniam.io platform** — its deals, data rooms, and workflows. Install a skill into Claude Code, and every session gains that know-how automatically, without you re-explaining the platform each time.

These skills are intended for people who work with the Inveniam.io platform.

## What's in here

| Skill | What it covers |
|---|---|
| [`inveniam/`](inveniam/) | Operating the Inveniam platform — deals and portfolios, data-room folders and documents, extracted data artifacts, file-veracity checks, and per-deal workflows (tasks, checklists, RACI, comments). Includes the resource model, task-status transition rules, data-room write limits, known idiosyncrasies, and common multi-step recipes. |

## What is an Agent Skill?

A skill is a folder containing a `SKILL.md` file. Claude Code discovers it at startup and automatically pulls in its guidance whenever a task is relevant — so the agent already knows the platform's structure, quirks, and workflows instead of rediscovering them by trial and error. You don't invoke a skill by hand; it triggers itself from its description.

## Installing a skill

Copy the skill folder into your personal Claude Code skills directory:

```bash
git clone https://github.com/aberdellans/Agent_skill.git
mkdir -p ~/.claude/skills
cp -r Agent_skill/inveniam ~/.claude/skills/
```

(No git? Use GitHub's **Code → Download ZIP** button, then copy the `inveniam` folder out of the ZIP into `~/.claude/skills/`.)

Then **start a new Claude Code session** — skills are loaded when a session begins, so an already-open session won't pick up a newly added one.

## Using it

Once installed, just describe your task in plain language — *"list my Inveniam deals,"* *"summarize the open workflow tasks on deal X"* — and Claude Code loads the skill on its own. There is no command to run.

## A skill is know-how, not access

Installing a skill teaches Claude *how* the platform works; it does not by itself connect Claude to your Inveniam data. You still need a way for the agent to reach the platform — the `inveniam` skill's own "Reaching the platform" section explains the options (the official Inveniam MCP server, the Inveniam V2 REST API directly, or a local MCP server).

## Updates

This repository is updated as the platform evolves. Pull the latest and re-copy the skill folder to stay current.
