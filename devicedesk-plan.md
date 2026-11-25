# DeviceDesk plan

Input for `/grill-with-docs` and `/to-spec`. Decisions here are settled; grill only what is open.

## Goal

Prove a coding agent can run a full product cycle (plan → build → verify → release → feedback → retro) with minimal human time, and ship a desk and meeting-room booking tool the team uses daily.

Done by Sunday night:

- v1 is deployed and the release gate is green.
- Every `agent-ready` ticket is closed or labelled `agent-failed`.
- The persona round, the real-feedback round and `/retro` have each run once.

## Product (v1)

- Sign-in: Microsoft Entra ID, single tenant (the company's).
- Test login: dev only, for agents and tests. CI fails if it can be enabled in production.
- Floor plan: interactive 2D SVG of two shared rooms and the meeting room(s), for any date.
  - Rendered on the server.
  - Each desk and room is an element with `data-desk-id` / `data-room-id` and a state: free, mine, taken, blocked.
  - Layout lives in `layout.yaml` (rooms, desks, coordinates), derived from room photos the human supplies. The layout is data; changing it touches no code.
  - Include one hidden smoke-test desk for release checks.
  - Visual bar: polished and clean. Users pick a desk or room directly on the map.
- Desks: booked by full day.
- Rooms: booked in 30-minute slots, 09:00–18:00 Asia/Dhaka.
- Work week: Sunday–Thursday.
- My bookings page, with cancel.
- Roles:
  - Member: books and cancels own bookings.
  - Admin (emails in config): cancels any booking; blocks a desk or room for a date range.

### Invariants

The Hypothesis stateful tests and the fuzz run check these.

- One booking per desk per day.
- Room bookings never overlap.
- One desk per person per day.
- Room booking: at most 2 h, slot-aligned, inside working hours, on a work day.
- Bookings fall between today and 14 days ahead.
- Blocked desks and rooms reject bookings.
- Cancelling frees the slot immediately.
- Only the owner or an admin can cancel.
- Times are stored in UTC and shown in Asia/Dhaka.

### Later and excluded

Stretch, in order:

1. Room check-in with release after a 15-minute no-show.
2. 3D view.
3. Outlook invites.
4. Recurring bookings.
5. In-app layout editor.

Excluded: LLM calls at runtime, canary releases, formal methods.

## Stack

- Python, FastAPI, HTMX, Jinja; SVG rendered on the server.
- SQLite with Alembic.
  - Migration order: ship code that tolerates the new schema, then the schema change, then code that uses it.
- MSAL for Python.
- One deployed environment with persistent storage; choose the host after checking current free tiers.
- `uv` for dependencies.
- GitHub Issues, PRs and Actions.

## Guardrails (hard: CI fails the build)

- Types: pyright strict.
- Lint and format: ruff.
- Custom rules: semgrep. Each message tells the agent what to do instead.
- Feature boundaries: import-linter.
  - One directory per feature.
  - Cross-feature imports go through each feature's public module.
- Code carries no comments and no suppressions (`type: ignore`, `noqa`, `nosemgrep`, `pyright: ignore`).
- Test login is unreachable in production.

Correction rule: the second time the same correction appears, encode it as a lint rule, type, test or API change, and log it in `corrections.md`.

## Verification

Ticket 1 builds the verification lever. Every later ticket depends on it.

### `devicedesk-ctl`

The project CLI. It wraps Playwright for Python, and every session, skill and persona drives the app through it.

Commands:

- Setup:
  - `doctor`
  - `seed --scenario <name>`
  - `reset`
  - `login --as <member|admin|email>`
- Navigate:
  - `goto <floor-plan|my-bookings|admin> [--date]`
- Act:
  - `book desk <id> --date`
  - `book room <id> --date --start --end`
  - `cancel <id>`
  - `block <desk|room> <id> --from --to`
  - `click <selector>`
  - `type <selector> <text>`
- Inspect:
  - `state --date` (JSON of every desk and room)
  - `snapshot` (accessibility tree)
  - `screenshot <path> --viewport desktop|phone`
  - `wait-settle`
- Logs:
  - `console`
  - `network-log`
  - `server-log --tail`
- Release:
  - `release-check --url --baseline`
  - `rollback --to <sha>`, which triggers the `rollback` workflow
- `cleanup`

Contract, covered by CLI tests:

- JSON output by default; `--human` for people.
- `--dry-run` on every command that writes.
- Each error message names the next command to run.
- `--help` on every subcommand.

### Feature map

Generate it with `/create-verification-skill`, under `.claude/skills/verify-devicedesk/`:

- `SKILL.md`: how to verify; points to the CLI and the map.
- `references/features/README.md`: an index linking one file per feature.
- Each feature file: sub-features, user path, CLI commands, selectors, gotchas.

A PR that changes a feature updates its file. `/maintain-verification-skill` runs at the end of each build day.

### Evidence

Every PR attaches evidence from `devicedesk-ctl`:

- screenshots, with desktop and phone views for UI changes;
- a `state` JSON diff;
- a trace, where relevant.

Tests:

- Hypothesis stateful tests cover the invariants.
- One fuzz run drives `devicedesk-ctl` with random booking sequences against the running app and checks `state` against the invariants.

### Release gate

The app exposes:

- `GET /healthz`: liveness.
- `GET /readyz`: DB, migrations and auth config, as per-check JSON.
- `GET /metrics`: token-protected JSON.
  - Requests and 5xx count; error rate.
  - p50 and p95 latency per route group (floor plan, booking, auth).
  - Rejected booking conflicts.
  - Git SHA.
- JSON logs. Each unhandled exception carries route, request id and SHA.

The `deploy` workflow runs `release-check` after every merge to `main`:

1. `/healthz` and `/readyz` turn green within 60 s.
2. Smoke test:
   - test login is off;
   - the sign-in page loads;
   - the floor plan renders;
   - a booking on the smoke-test desk round-trips.
3. Send 2 minutes of smoke traffic.
4. Compare `/metrics` with `release-baselines/<previous-sha>.json`. The gate passes when:
   - error rate is below 1%;
   - no new exception types appear;
   - p95 for each route group is at most 1.5× baseline and at most 800 ms.
5. Result:
   - Pass: write the new baseline.
   - Fail: redeploy the previous SHA, re-run the check, and open a `release-rollback` issue with the metrics and log excerpt.

Rehearse rollback once: ship a build with an injected delay and confirm the gate rolls it back unaided.

## Process

1. `/grill-with-docs`: produces `GLOSSARY.md` and ADRs. Record the model routing below as an ADR.
2. `/to-spec`: the spec becomes a GitHub issue. Human gate.
3. `/to-tickets`: tracer-bullet tickets, each listing its blockers. Human gate.
   - Ticket 1: walking skeleton, test login, `devicedesk-ctl` setup/navigate/inspect commands, feature-map skeleton.
4. Bootstrap session (interactive): builds the orchestration in the next section.
5. The driver runs each ticket through `/implement` (red-green TDD, evidence, feature-map update), then `/code-review`, then design critique. Auto-merge follows.
6. Feedback rounds, then `/retro`.

## Orchestration

### Bootstrap creates

- `scripts/drive.py`: the driver.
- `scripts/model-routing.toml`: the routing table below.
- `.claude/settings.json`: permissions, below.
- Labels:
  - `agent-ready`
  - `needs-human`
  - `agent-failed`
  - `release-rollback`
- Branch protection on `main`, with no human approval required. Required checks:
  - `ci`: ruff, pyright, import-linter, semgrep, tests.
  - `evidence`: fails unless the PR carries `devicedesk-ctl` evidence.
  - `design-critique`: a status posted by the critique session; required on PRs that touch templates or static files.
- Workflows:
  - `ci`
  - `deploy`: runs on merge and includes the release gate.
  - `rollback`: manual trigger, takes a SHA.

### Driver loop

1. Pick the lowest-numbered open `agent-ready` issue whose blockers are all closed. If none remain, go to the phase transitions below.
2. Implement in a fresh `claude -p` session, with the routed model, a `--max-turns` cap and a 45-minute wall-clock limit.
3. Review in a fresh session. Blocking findings start the next attempt.
4. Run the design critique in a fresh session, on UI PRs only.
5. Run `gh pr merge --auto --squash`. GitHub merges once the required checks pass.
6. On a failed check, blocking review or timeout, start a new attempt in a fresh session with the failure log. At most 3 attempts.
   - After attempt 2, Opus triage returns JSON with one decision:
     - `retry`, with guidance;
     - `split`, with sub-tickets, filed as a `needs-human` proposal;
     - `escalate`, with a summary the human can read in two minutes.
   - After attempt 3: label the issue `agent-failed` and `needs-human`, log the case in `corrections.md`, and move on to the next unblocked ticket.
7. At the end of each build day:
   - run `/maintain-verification-skill`;
   - build `gallery/<date>.html` from the day's screenshots;
   - open a `needs-human` issue that links the gallery, and keep working.

Triage and summaries advise the driver; gates, checks and merges stay with the script and GitHub.

Phase transitions. The driver pauses at each until a human acts.

- When v1 is merged and the gate is green:
  1. Run the persona round.
  2. Run the product-owner ranking.
  3. Open a `needs-human` issue. The human labels chosen proposals `agent-ready`.
- When `feedback/real.md` appears:
  1. Run the reproduction agent.
  2. File reproduced bugs as proposals.
  3. The human labels the chosen ones `agent-ready`.
- After the last approved ticket:
  1. Run `/retro`.
  2. Open a `needs-human` issue. Approved changes become `agent-ready` tickets.

Controls:

- A `STOP` file in the repo root halts the driver before its next step.
- `run-status.md`, updated every step: ticket, attempt, result, model, tokens, time, and human minutes at gates.
- Production credentials live only in GitHub Actions; agents trigger workflows.

### Permissions (`.claude/settings.json`)

Mode: `acceptEdits`, plus an allowlist.

Allow:

- `uv` and `uv run`, for pytest, pyright, ruff, semgrep, import-linter and alembic.
- `python scripts/devicedesk-ctl` and `python scripts/drive.py`.
- `git`: status, diff, add, commit, switch, branch, fetch, pull, and push to feature branches.
- `gh`: issue, pr, label, `workflow run`, `run view`.
- `playwright install`.

Deny:

- force-push; pushing to `main`; `git reset --hard` on `main`;
- `gh repo delete` and `gh secret`;
- `rm -rf` outside the repo;
- reading `.env*`.

A command outside the allowlist fails in headless mode. The driver logs it to `corrections.md` for `/retro`.

### Model routing

Keep each role on its model for the whole weekend. Change routes only through `/retro`, between phases.

| Role | Model |
|---|---|
| Driver control flow | script |
| `/grill-with-docs`, `/to-spec`, `/to-tickets`, bootstrap | Opus |
| `/implement`, building `devicedesk-ctl` | Sonnet, high effort |
| `/code-review`, failure triage, `needs-human` summaries, product-owner ranking, `/retro` | Opus |
| Personas, design critique, reproduction agent | Sonnet, high effort |
| Feature-map maintenance, gallery, log and metric summaries | Haiku, high effort |

## Feedback

Stakeholder agents produce proposals only. Code changes reach the codebase through approved tickets.

Personas run through `/persona-walkthrough` against the deployed app, via `devicedesk-ctl`:

- a new trainee, booking on a phone;
- a regular who wants "their" desk;
- a meeting organizer booking for six;
- an office manager on the admin view: blocks, no-shows, space use.

The product owner merges, deduplicates and ranks the personas' findings.

Evidence rule: each item cites its steps and a screenshot. Items without evidence are dropped.

Reproduction agent: for each item in `feedback/real.md`, record one outcome:

- reproduced, with evidence;
- not reproducible, with the steps tried;
- already fixed on `main`.

A reproduced bug becomes a proposal, with its reproduction steps as the failing test.

## Human gates

1. Answer the grilling.
2. Approve the spec.
3. Approve the tickets.
4. Run bootstrap.
5. Review the daily gallery. Dislikes go into `corrections.md`.
6. Choose proposals at each phase transition.
7. Supply `feedback/real.md`.
8. Choose `/retro` changes.
9. Answer `needs-human` issues.

## Human-supplied inputs

- Entra app registration, with redirect URIs for local and deployed.
- Room photos.
- Admin email list.
- GitHub repo.

## Metrics

Recorded in `corrections.md` and `run-status.md`. Each `corrections.md` entry has: date, ticket, correction, layer the fix went into.

1. v1 is deployed and used by the team.
2. Every repeated correction became a check.
3. Interventions per ticket trend down from the first ticket to the last.
4. Persona feedback compared with real feedback: overlap, issues only humans found, and persona noise.
5. Every deploy passed the gate or was rolled back by it unaided, and the rehearsal succeeded.
6. Human minutes at gates, and `needs-human` issues raised outside the gates (trending to zero).
7. Every PR's evidence after ticket 1 came from `devicedesk-ctl`.
8. Share of real-feedback items resolved by the reproduction agent unaided.
