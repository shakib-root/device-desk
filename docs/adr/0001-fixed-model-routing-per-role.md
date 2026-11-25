# Fixed model routing per role

Each agent role runs on one model for the whole build weekend, and routes change only through `/retro`, between phases. Judgement-heavy roles that gate or shape work (grilling, spec, tickets, bootstrap, code review, failure triage, `needs-human` summaries, product-owner ranking, retro) run on Opus; volume work that is checked by those gates (implementation, building `devicedesk-ctl`, personas, design critique, reproduction) runs on Sonnet at high effort; summarising and upkeep (feature-map maintenance, gallery, log and metric summaries) runs on Haiku at high effort; the driver's control flow is a plain script with no model. Holding routes fixed keeps the run's metrics (interventions per ticket, tokens, time) comparable from the first ticket to the last.

| Role | Model |
|---|---|
| Driver control flow | script |
| `/grill-with-docs`, `/to-spec`, `/to-tickets`, bootstrap | Opus |
| `/implement`, building `devicedesk-ctl` | Sonnet, high effort |
| `/code-review`, failure triage, `needs-human` summaries, product-owner ranking, `/retro` | Opus |
| Personas, design critique, reproduction agent | Sonnet, high effort |
| Feature-map maintenance, gallery, log and metric summaries | Haiku, high effort |

## Consequences

- Switching a role's model mid-phase is not a fix for a failing ticket; the driver's retry, split and escalate paths are.
- The routing table lives in `scripts/model-routing.toml`; this ADR records why it is fixed, not its current values.
