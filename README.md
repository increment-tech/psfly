# psfly

Bootstrap and operate Elixir apps on the **Fly.io + PlanetScale Postgres + Tigris** stack —
one kit holding the CI/CD workflows, the shared tooling, and the environment-operations CLI,
so a new repo installs the kit instead of porting a sibling repo's copy.

**Status: pre-release.** The `repave` verb works; the CI kit (reusable workflows and
composite actions) is being assembled.

## The CLI

`bin/psfly` is the entry point — clone this repo once and symlink `bin/psfly` onto your PATH.

### `psfly repave <env>`

Hard-reset a **disposable** environment (staging, preview) of a repo that follows the shared
topology: Fly app + PlanetScale database (`DATABASE_URL` secret) + Tigris bucket (`AWS_*` /
`BUCKET_NAME` secrets) + CD on push to main.

- A read-only **preflight** validates machine, identity, repo wiring, and environment state
  before anything destructive is offered.
- The destructive path — PlanetScale database delete + recreate with a fresh role and staged
  Fly secret; bucket wipe with verified-empty accounting — sits behind **type-to-confirm**
  (you retype the resource names).
- **Re-run convergent:** progress is journaled, so re-running after any partial failure
  resumes — completed steps are skipped, credentials are re-minted fresh.

## Configuration

Each consuming repo commits flat `KEY=VALUE` files under `.psfly/<env>.conf` naming its own
apps, databases, and buckets. The kit reads config; it hardcodes nothing about any fleet.

## Planned

- `psfly init` — wire a new repo: caller workflows, fly toml skeletons, `.psfly/` config.
- `psfly doctor` — read-only environment/wiring readiness check.
- Reusable GitHub Actions workflows and composite actions: gate (compile, format, credo,
  tests), deploy from a repo's fly config, ephemeral PR preview environments.
