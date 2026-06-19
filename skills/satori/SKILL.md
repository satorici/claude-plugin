---
name: satori
description: Use Satori CI to test anything — write/validate playbooks and run them. Use whenever the user wants to create a YAML test or playbook; fuzz, load-test, scan, or monitor something; run a playbook locally or in the cloud; or list/inspect Satori reports, monitors, or scans. Covers playbook syntax, assertions, inputs, settings, scaling, and the `satori` CLI.
---

# Satori CI

Satori CI is the platform for automated testing at scale. It runs language-agnostic tests defined
in YAML **playbooks** inside ephemeral containers (AWS Fargate) or locally. Any CLI command in any
Docker image becomes a repeatable, assertable, parallelizable test. Playbooks run on demand, on CI
(GitHub push), or on a schedule (monitors).

Two jobs: (1) **author** a playbook from what the user wants to test, and (2) **run** it with the
`satori` CLI and read the results.

---

## Part 1 — Authoring playbooks

A playbook is YAML. `settings:` is optional metadata/config; every other top-level key is a **test
group** with execution steps and (optionally) assertions.

```yaml
settings:
  name: "Playbook Name"
  description: "What it tests"
  image: debian                  # Docker base image (debian, python, alpine, fedora, custom…)

import:                          # optional — reuse public or local playbooks
  - satori://code/trufflehog.yml

install:                         # a setup group (no assertions needed)
  deps:
    - apt-get update && apt-get install -qy nmap

nmap:                            # a test group
  assertReturnCode: 0            # assertion → makes this a pass/fail test
  setSeverity: 1                 # severity 1 (critical) .. 5 (info)
  run:
    - nmap -p- ${{HOST}}
```

### Variables & parameters
- `${{VAR}}` — provided at run time: `satori run playbook.yml -d VAR=value` (or `-df VAR=file`).
- `${{input}}` — current value when iterating an `input:` list (see Inputs).
- `${{step.stdout}}` — stdout of an earlier step (e.g. `${{nmap.stdout}}`).

### Assertions (full set)
Pair with `setSeverity: 1-5` (1 = critical, 5 = info). Not required for setup/data-collection groups.

- **stdout:** `assertStdout` (bool), `assertStdoutEqual`, `assertStdoutNotEqual`,
  `assertStdoutContains` (string or list), `assertStdoutNotContains`, `assertStdoutSHA256`,
  `assertStdoutRegex`, `assertStdoutNotRegex`.
- **stderr:** `assertStderr`, `assertStderrEqual`, `assertStderrNotEqual`, `assertStderrContains`,
  `assertStderrNotContains`, `assertStderrSHA256`, `assertStderrRegex`, `assertStderrNotRegex`.
- **return code:** `assertReturnCode` (int), `assertReturnCodeNot` (int).
- **behavioral:** `assertDifferent` (bool — behavior changes across inputs?), `assertKilled` (bool —
  hit the timeout?).

### Inputs (parameterization & fuzzing)
```yaml
input:
  - - "value1"
    - "value2"
echo:
  run:
    - echo ${{input}}        # runs once per value
```
- **File dictionaries:** `- - file: dict.txt` with `split: "\n"`.
- **Mutation fuzzing:** `value: "seed"`, `mutate: radamsa` (or `zzuf`), `mutate_qty: N` → N mutated
  variants (great with `assertReturnCodeNot: 0` / `assertKilled` to catch crashes).

### settings: reference
- **Identity/docs:** `name`, `description`, `mitigation`, `author` (list of URLs), `gallery` (image
  URLs), `example`.
- **Container:** `image`, `os` (`linux` | `windows`).
- **Scale/resources:** `cpu` (256–16384 = .25–16 vCPU), `memory` (MB), `storage` (GB), `timeout`
  (s, default 3600), `count` (launch N instances in parallel — load/throughput testing).
- **Scheduling (monitors):** `rate` (`10 minutes`, `1 hour`, `7 days`) or `cron` (6-field AWS cron,
  e.g. `"*/5 * * * ? *"`). Mutually exclusive.
- **Notifications:** `log`, `logOnFail`, `logOnPass` → `slack, email, telegram, discord`.
- **Reports/output:** `report: pdf`, `saveReport: false`, `saveOutput: false`, `files: true`
  (downloadable artifacts), `redacted:` (parameter names to mask).

### Imports & the public catalog
```yaml
import:
  - satori://code/semgrep.yml      # public playbook from the Satori catalog
  - file://local/playbook.yml      # a local playbook
```
The catalog has 270+ playbooks (https://github.com/satorici/playbooks): `code`, `web`, `scan`,
`dns`, `secrets`, `container`, `cve`, `compliance`, `llm`, `dos`, `osint`, `monitor`, … Prefer
importing over reinventing. List with `satori playbooks --public`. New playbooks welcome via PR.

### Scale patterns
- **Parallel instances:** `settings: { count: 50 }` → 50 copies at once (each its own report).
- **Sharding huge datasets:** `satori shards --shard X/Y --input ips.txt` splits IPs/CIDRs/domains
  across Y workers — e.g. scan all of IPv4 in 500 shards.
- **Per-run override:** `satori run … --cpu 4096 --memory 8192 --storage 30 --timeout 600`.

### Validation
Playbooks validate against `satorici/playbook-validator` schemas (`settings.json`, `command.json`,
`import.json`, `input.json`, `test.json`). Validate via `pip install satori-playbook-validator`, or
just run with `--local` (validates before executing). Always hand the user the exact `satori run`
command.

---

## Part 2 — Running playbooks & reading results (the `satori` CLI)

If `satori` isn't installed: `pip install satori-ci` then `satori install` (log in / set token).
`satori update` upgrades it; `satori whoami` shows identity.

### Run
`satori run [PATH] [flags]` — `PATH` is a `.yml`, a dir with `.satori.yml`, or a `satori://` public
playbook. Cloud by default.
- `-d KEY=VALUE` (repeatable; `-df KEY=FILE`), `-s/--sync` (block + show result), `-r/--report`,
  `-o/--output`, `-f/--files`, `--visibility …`, `--redacted PARAM`.
- Scale: `--cpu --memory --storage --timeout --os --image`. Monitor: `--rate "5 minutes"` /
  `--cron "…"` with `--count N`.
- `satori run … --local` (or `satori local …`) runs on this machine with live output.

```bash
satori run nmap.yml -d HOST=example.com --report --output
satori run satori://code/semgrep.yml --sync
satori run . --cpu 4096 --memory 8192 --count 20
satori run ./ --rate "every 1 hour"
```

### Read results
- `satori reports` — list (`-p` page, `-l` limit, `--public`). `satori reports search` filters by
  `--result {pass,fail,unknown}`, `--status`, `--execution {local,run,ci,scan,monitor}`,
  `--from/--to`, `--severity`, `--query`, `--repo`, `--monitor`.
- `satori report <id>` — show one. Actions: `output`, `files`, `stop`, `status`, `delete`,
  `visibility <v>`, `issue <template>` (open a GitHub issue).
- `--json` or global `--export {html,svg,text}` on most commands.

### Catalog, monitors, scans, repos, teams
- `satori playbooks --public` (browse 270+); `satori playbook <id|satori://…>` (`--original` raw YAML).
- `satori monitors`; `satori monitor <id> {start|stop|clean|delete}`.
- `satori scan <owner/repo>` (`-c` coverage, `-b` branch, `--playbook`, `-s -o -r`); `satori scans`.
- `satori repo <owner/repo> run`; GitHub App auto-runs `.satori.yml` on every push.
- `satori team` / `satori teams` / `satori settings` (notification channels).
- `satori shell --image debian [--cpu N --memory N]` — interactive container to prototype steps.
- `satori feedback "<message>"` — send feedback to the Satori team.

---

## References
- Cheatsheet (commands + playbook syntax at a glance): https://docs.satori.ci/cheatsheet
- Full docs: https://docs.satori.ci (asserts: /playbooks/asserts, settings: /playbooks/settings,
  inputs: /playbooks/inputs, modes: run, monitor, scan, shards, CI/GitHub)
- Public playbook catalog: https://github.com/satorici/playbooks
- CLI source: https://github.com/satorici/satori-cli

When unsure about an exact flag, assertion, or settings key, consult the cheatsheet/docs above
rather than guessing.
