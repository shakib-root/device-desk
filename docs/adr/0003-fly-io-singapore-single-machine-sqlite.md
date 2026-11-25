# Host on one auto-suspending Fly.io machine in Singapore, with SQLite on a volume

DeviceDesk runs as the Fly app `nc-devicedesk` (`https://nc-devicedesk.fly.dev`) on a single Fly.io Machine (shared-cpu-1x, 256 MB) in Singapore (`sin`). The Machine auto-suspends after a few minutes with no traffic and wakes on the next request, so it runs only for the hours people use it. The SQLite database lives on a 1 GB Fly volume. The goal is to stay well inside the Free plan's $5 of usage a month. Development happens locally in WSL (Ubuntu 26.04), and the local copy is never production. GitHub Actions deploys with `flyctl`, using a `FLY_API_TOKEN` secret. Each merge's image is tagged with its git SHA in the Fly registry, so a rollback is a redeploy of an earlier SHA's image.

## Considered options

- **Always on, with auto-stop off.** Rejected: the human only needs the server up for a few hours at a time, and paying for idle hours buys nothing.
- **Manual start and stop.** Rejected: someone would have to remember to start it, and teammates couldn't open it whenever they wanted, which the real-feedback round depends on.
- **Hosting locally in WSL behind a tunnel (Tailscale Funnel).** Rejected: the machine would have to stay on during office hours, deploys would need a self-hosted runner, and that runner would need a private repo.
- **Oracle Always Free VM.** Rejected: we'd run TLS, the proxy and backups ourselves, and Oracle reclaims VMs it counts as idle.
- **Render, Railway, Koyeb, Azure F1, Cloud Run.** Rejected: on their free tiers there's no persistent disk for SQLite, or waking up takes tens of seconds instead of under a second.

## Consequences

- **Cost:** the Machine is billed only while it's awake. The always-on ceiling is about $2.80 a month (Singapore runs about 27% above the published US price of $2.19). Add $0.15 for the volume, which is billed even while the Machine is suspended, and a few cents of outbound traffic at $0.04/GB. Snapshots (first 10 GB free), builders, the registry and the shared IPv4 cost nothing today.
- **The first request after a quiet spell waits for the Machine to wake,** usually under a second. Deploys wake it, and the Release Gate's 2 minutes of smoke traffic keep it awake, so the latency comparison against the Baseline is measured on a warm Machine.
- **Bootstrap must confirm that suspend works with a volume at this Machine size.** If it doesn't, fall back to auto-stop, which wakes in a few seconds, and record that in this ADR.
- **Fly has no spending cap and no billing alert.** If usage goes over $5, the *whole* month is billed at standard rates, which at this size is still only a few dollars. Fly emails when $1 or less remains. A CI check fails if `fly.toml` allows more than one Machine, a size other than 256 MB, a region other than `sin`, or any auto-stop setting except `suspend`, so no agent can scale it up by accident. Only GitHub Actions holds the Fly token.
- **If 256 MB proves too small, add swap first.** 512 MB is affordable while the Machine is suspended most of the time, but needs a new decision.
- **There's only one Machine with a volume,** so every deploy causes a short outage and nothing fails over. Daily volume snapshots, kept for 5 days, are the backup.
- **The release check runs from GitHub's US runners,** so p95 latency includes the round trip from the US to Singapore. The Baselines are measured the same way, so comparisons still hold.
- **Fly changed its free allowance in 2024 and says it may start charging for builders and registry storage.** Keeping the app in one container, with settings from environment variables and the SQLite path from config, keeps a move to another host cheap.
- **Fly requires a card on file,** which the human adds.
