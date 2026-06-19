# Satori CI — Claude Code plugin

Author, validate, and run [Satori CI](https://satori.ci) playbooks straight from Claude Code.

[Satori CI](https://docs.satori.ci) is a platform for automated testing at scale: language-agnostic
tests defined in YAML **playbooks** that run inside ephemeral containers — locally, on every GitHub
push, or on a schedule — and assert on output, exit codes, behavior, and more. This plugin teaches
Claude the playbook DSL and the `satori` CLI so you can describe what you want to test in plain
English and get a ready-to-run playbook plus the command to execute it.

## What's included

**Just talk to it.** The plugin ships one `satori` skill that Claude loads automatically the moment
your request involves Satori — no slash, no syntax to remember:

- "satori, listá los reportes"
- "corré un playbook que escanee internet con satori"
- "write a Satori playbook that fuzzes my parser and flag any crash"
- "run satori://code/semgrep.yml and show me the report"

The skill covers the whole platform: playbook syntax, the full assertion set, inputs/fuzzing,
`settings` (scale, scheduling, notifications), imports and the public catalog, and the entire
`satori` CLI (run locally or in the cloud, reports, monitors, scans, teams, shell).

**Slash command** (optional, explicit entrypoint):
- `/satori:do <what you want to do>` — one command, plain English, does the rest
  (e.g. `/satori:do run a playbook that scans example.com`).

## Install

```
/plugin marketplace add satorici/claude-plugin
/plugin install satori@satori
```

(Once listed in the community directory you can also install it from there.)

## Prerequisite

Running playbooks (and the slash command) uses the Satori CLI:

```bash
pip install satori-ci
satori install          # log in / configure your token
```

Authoring playbooks works without it; running them needs it (or use `--local`).

## Examples

- "Write a Satori playbook that runs nmap against example.com and fails if telnet is open."
- "Create a monitor that curls my site every 5 minutes and alerts Slack on failure."
- "Fuzz my parser binary with radamsa and flag any crash."
- `/satori:do run satori://code/semgrep.yml and summarize the report`

## Links

- Docs: https://docs.satori.ci
- Public playbook catalog (270+): https://github.com/satorici/playbooks
- CLI: https://github.com/satorici/satori-cli

## License

MIT. See [LICENSE](./LICENSE).
