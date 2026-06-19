---
description: Do anything with Satori CI from one natural-language command — write and run playbooks, scan, fuzz, monitor, and read reports.
argument-hint: <what you want to do, in plain English>
---

The user wants to do this with Satori CI: **$ARGUMENTS**

Use the `satori` skill for all syntax, assertions, settings, and CLI usage. Figure out the intent
from `$ARGUMENTS` and do it — don't ask which sub-action, just act:

- **List / inspect** (e.g. "lista los reportes", "show monitors", "ver el reporte X") → run the
  matching `satori` command (`satori reports`, `satori report <id>`, `satori report <id> output`,
  `satori monitors`, `satori scans`, `satori playbooks --public`, …) and summarize the result.
- **Run an existing playbook** (e.g. "run satori://scan/nmap.yml", "corré .satori.yml") → run it
  with `satori run <target> --sync --report --output` (pass through any `-d KEY=VALUE`).
- **Describe a test** (e.g. "corre un playbook que escanee internet", "fuzz my parser") → author a
  ready-to-run `.yml` in the cwd (use `import: satori://...` from the catalog when it fits; set
  `settings` for scale/scheduling/resources when implied), then run it and report results.
- **CLI not installed?** Tell the user to `pip install satori-ci` then `satori install`, and stop.

If `$ARGUMENTS` is empty, ask what they want to test or check.
