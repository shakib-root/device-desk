# Public GitHub repo

The DeviceDesk repo is public, even though the app is internal. The plan needs branch protection with required status checks (`ci`, `evidence`, `design-critique`) so that the driver can auto-merge safely. GitHub Free only offers those on public repos, and a public repo also gets unlimited Actions minutes. A private repo would need a paid plan and would cap the driver at 2,000 CI minutes a month.

## Consequences

- Secrets live only in GitHub Actions secrets (`FLY_API_TOKEN`, `SMOKE_TOKEN`, Entra client secret), never in the repo.
- The original room photos are never committed; they stay outside the repo. Only the sketch art made from them is committed, and it shows no people.
- No employee names or emails go into the repo. The admin list is set through deploy-time configuration.
- What's public: the code, the process docs, and a sketch of the office layout.
