# psfly

Bootstrap and operate Elixir apps on the **Fly.io + PlanetScale Postgres + Tigris** stack —
one kit holding the CI/CD workflows, the shared tooling, and the environment-operations CLI,
so a new repo installs the kit instead of porting a sibling repo's copy.

**Status: pre-release.** The `repave` verb works and the reusable gate/deploy workflows
exist; composite actions and the `init`/`doctor` verbs are being assembled.

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

## The CI kit — reusable workflows

Consuming repos carry only thin caller shims (see `templates/`); the logic lives here and
is consumed by reference, pinned to a tag. The caller owns the triggers, paths filter, and
concurrency; the kit owns the runner, toolchain, caching, and the safety rules.

- **`.github/workflows/gate.yml`** — the quality gate. The gate *surface* is the consuming
  repo's own gate command (default `bin/precommit`), so CI and the local pre-commit hook
  can never drift. Elixir/OTP come from the repo's `mise.toml` via mise — no version
  literal in CI. Inputs: `postgres-db` (required), `postgres-port`, `postgres-image`,
  `gate-command`, `parallel-dialyzer` (a separate dialyzer lane for gate commands that
  skip it — wall-clock becomes max, not sum), `timeout-minutes`.
- **`.github/workflows/deploy.yml`** — Fly.io deploy. SHA-pinned `setup-flyctl` with the
  token scoped to the deploy step only, and **no `--strategy` flag** — the strategy lives
  in the fly config's `[deploy]` section, because the CLI flag silently overrides the toml
  and that pair drifts. Inputs: `fly-config`, `timeout-minutes`; secret: `FLY_API_TOKEN`.
  Callers must `needs:` a gate job — a red revision never deploys.

## Configuration

Each consuming repo commits flat `KEY=VALUE` files under `.psfly/<env>.conf` naming its own
apps, databases, and buckets. The kit reads config; it hardcodes nothing about any fleet.

## Planned

- `psfly init` — wire a new repo: caller workflows, fly toml skeletons, `.psfly/` config.
- `psfly doctor` — read-only environment/wiring readiness check.
- Reusable GitHub Actions workflows and composite actions: gate (compile, format, credo,
  tests), deploy from a repo's fly config, ephemeral PR preview environments.
