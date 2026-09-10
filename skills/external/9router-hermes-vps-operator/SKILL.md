---
name: 9router-hermes-vps-operator
description: Operate an existing Linux VPS running 9Router and Hermes through SSH. Use for auditing services, diagnosing rate limits and provider failures, validating model catalogs, building safe fallback combos, installing a Hermes SOUL.md personality, restarting gateways, testing Telegram delivery, and rolling back configuration changes.
---

# 9Router + Hermes VPS Operator

Use this skill when the user asks to inspect, repair, configure, expand, or stabilize a VPS running Hermes Agent behind a local 9Router OpenAI-compatible gateway.

## Operating contract

Keep the architecture intact:

```text
Telegram or another Hermes channel → Hermes → custom provider → 9Router localhost API → combo → upstream provider/model
```

Do not replace 9Router with a direct model unless explicitly requested. Never hard-code passwords, API keys, OAuth tokens, cookies, private keys, or Telegram tokens in this skill, shell history, logs, screenshots, commits, or chat. Use interactive login or a current-session environment variable. Never print secret-bearing configuration values.

Treat every catalog entry as unverified until it returns a successful request through local 9Router. A listed model is not necessarily connected, supported, free, or within quota.

## Required inputs

Confirm the SSH target, port, user, 9Router base URL, Hermes provider name, combo name, and cost policy without asking the user to paste secrets into chat. Common values are `http://127.0.0.1:20128/v1`, provider `custom`, and combo `hermes`. If a password appeared in chat or an image, recommend rotation but do not repeat it.

## Read-only audit

Connect with strict host-key handling and a short timeout. Inspect service state, process ownership, listening ports, recent service logs, Hermes configuration, the active `SOUL.md`, 9Router database, model catalog, and gateway logs. Discover paths instead of assuming them. Classify `429` as quota/rate limit, `401` as authentication, `400` as request incompatibility, `404` as unavailable model/endpoint, `402` as credit required, `5xx` as upstream/gateway failure, and timeout as latency or a stuck provider. Save a redacted audit note.

Useful checks are:

```bash
systemctl --user is-active hermes-gateway.service
ss -ltnp | grep -E '20128|hermes' || true
journalctl --user -u hermes-gateway.service -n 200 --no-pager
```

## Backup before edits

Before changing a database, config, or SOUL file, create timestamped copies and verify them:

```bash
cp -a /root/.hermes/SOUL.md /root/.hermes/SOUL.md.backup-YYYYMMDD-HHMMSS
sqlite3 /root/.9router/db/data.sqlite ".backup /root/.9router/db/data-pre-change-YYYYMMDD-HHMMSS.sqlite"
```

Never use SQLite `readfile()` to write a JSON text field: it creates a BLOB and can cause `a.includes is not a function`. Write JSON as TEXT and verify:

```sql
select name, typeof(models), models from combos where name='hermes';
```

The expected type is `text`.

## Enumerate and validate models

Read active provider connections and the local model catalog without printing credentials. Separate native namespaces from aliases. For short native lists, test every candidate; for huge catalogs, test a representative shortlist and state the scope. Avoid probing thousands of models because this consumes quota and can overload the gateway.

Probe through the local endpoint with a bounded timeout and a tiny request:

```bash
curl -sS --max-time 18 http://127.0.0.1:20128/v1/chat/completions \
  -H "Authorization: Bearer $NINE_ROUTER_KEY" \
  -H 'Content-Type: application/json' \
  -d '{"model":"PROVIDER/MODEL","messages":[{"role":"user","content":"Reply OK only"}],"max_tokens":8}'
```

Classify each result as `200 validated`, `429 quota`, `402 paid/credit`, `401 credential`, `400 incompatible`, `404 unavailable`, `5xx upstream`, or `timeout`. Prefer independent upstream pools. Multiple aliases in one free pool do not create extra quota.

## Build the combo

Use the 9Router dashboard/API when available. Direct SQLite editing is a last resort: back up first, write a JSON array as TEXT, and verify it. A free-only combo must not contain `openrouter/auto`, because it can select a paid route. Add it only after explicit approval of possible charges and a spending limit.

Example:

```sql
update combos
set models = cast('["gemini/gemini-3.5-flash-lite","glm/glm-4.7-flash","oc/big-pickle"]' as text),
    updatedAt = datetime('now')
where name = 'hermes';
```

Put the most reliable model first, then independent fallbacks, and free-pool models last when they are known to rate-limit. Keep the list short enough to avoid long retry chains.

## Configure Hermes and SOUL.md

Hermes must call the combo through its configured provider. In this installation the working reference was:

```text
provider = custom
model = hermes
endpoint = http://127.0.0.1:20128/v1
```

Do not change it to `9router/hermes` unless the actual provider registry contains that name. A Telegram session may persist a `/model` override; check the effective model in logs and select the combo, not an exhausted model directly.

When requested, back up `/root/.hermes/SOUL.md` and use `templates/SOUL.md`. The personality should be proactive in Linux/VPS, SSH, Docker, systemd, APIs, databases, coding, debugging, deployment, UI/UX, branding, prompts, photo editing, and design critique. It must distinguish facts from assumptions, make reversible backups, and never expose credentials, bypass security, perform destructive actions without confirmation, or claim untested success.

Restart only after validated changes:

```bash
systemctl --user restart hermes-gateway.service
systemctl --user is-active hermes-gateway.service
```

## Verify and recover

Run a direct Hermes smoke test, then ask the user to send a fresh Telegram message. Confirm service `active`, combo JSON type `text`, a short valid response, and no new 400/401/402/404/429/5xx on the tested route. A local smoke test does not prove that a Telegram session uses the same model because persistent overrides and long context differ.

If a change causes silence, 500 errors, invalid requests, or long retries, restore the last known-good database backup, restart Hermes, and test. Never delete provider credentials while debugging.

## Final report

Report the architecture, changed files, backup paths, validated models, rejected models with error classes, service status, smoke-test result, Telegram status, and quota/cost limitations. Never report raw tokens or secret-bearing configuration.