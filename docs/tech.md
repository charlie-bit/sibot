# sibot — Technical Design

## What it is

sibot is an AI control center: one place to talk to many models from many providers,
switch between them at will, and keep a single continuous conversation while doing so.

## v1 scope

### Control surface

Web UI only: log in, register tasks, enable or disable them, run them on demand, read their
run history.

### Users

Single user. Authentication is required anyway, because the service is reachable from the
internet.

### What a task is

```
task = trigger × skill × model
```

- **Trigger** — a cron schedule, plus a manual run from the panel. Event triggers (webhook)
  are deferred.
- **Skill** — a Claude Code `SKILL.md` directory, supplied as **external configuration**:
  loaded at runtime from a configured location, never part of sibot's own codebase. Adding
  or changing a skill must not require changing or redeploying sibot.
- **Model** — chosen per task, changeable at any time without editing the task.

### Where it runs

Cloud-hosted. Every run executes in its own container, which is discarded afterwards.

## Not yet decided

Everything else.
