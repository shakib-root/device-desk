# Smoke Principal for release checks in production

The Release Gate must book the Smoke-test Desk in production, where test login is off. Entra sign-in needs a real person, often with MFA. So the release check signs in as a Smoke Principal instead: any request carrying a `SMOKE_TOKEN`, a secret stored only in GitHub Actions, acts as a fixed non-human identity. That identity can book and cancel the Smoke-test Desk and nothing else. A semgrep rule and a test enforce that scope, and `/metrics` counts every use.

## Considered options

- **A real Entra test account with stored credentials.** Rejected: password sign-in breaks under MFA and Conditional Access, and it would put a person's credentials in CI.
- **Skip the booking round-trip in production.** Rejected: that round-trip is the only check that exercises the write path against the real database after a deploy.

## Consequences

- This is a deliberate, narrow exception to "no auth bypass in production". Widening it (to more Spaces, or to reading other people's Bookings) needs a new ADR.
- Rotating `SMOKE_TOKEN` is a GitHub Actions secret change plus a restart; the app reads it from the environment.
