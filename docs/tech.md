# sibot — Technical Design

## What it is

sibot is an AI control center: one place to talk to many models from many providers,
switch between them at will, and keep a single continuous conversation while doing so.

## v1 scope

### Control surface

Web UI only. Register tasks, enable or disable them, run them on demand, read their run
history.

### What a task is

```
task = trigger × what runs × which model
```

- **Trigger** — any of: schedule (cron), manual, inbound event (webhook).
- **What runs** — a *skill* in Claude Code `SKILL.md` format: a directory with frontmatter
  and instructions, invoked with parameters.
- **Model** — chosen per task and changeable at any time without editing the task itself.

### Where it runs

Cloud-hosted. Scheduler and executor run as a service, not on the owner's machine.

## Not yet decided

Everything else.
