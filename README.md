# psfly

Bootstrap and operate Elixir apps on the **Fly.io + PlanetScale Postgres + Tigris** stack —
one kit holding the CI/CD workflows, the shared tooling, and the environment-operations CLI,
so a new repo installs the kit instead of porting a sibling repo's copy.

**Status:** the `repave` verb and the reusable gate/deploy workflows are in use against
real repos. The `init` and `doctor` verbs are planned, not built.

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
is consumed by reference. Pin an **immutable release tag** (`v1.0.0`-style) or a commit
SHA — a moving major tag like `@v1` is convenient inside one fleet but is not a stability
guarantee. The caller owns the triggers, paths filter, concurrency, and the gate→deploy
ordering (`needs:`); the kit owns the runner, toolchain, caching, and the pinning rules.

- **`.github/workflows/gate.yml`** — the quality gate. The gate *surface* is the consuming
  repo's own gate command (default `bin/precommit`), so CI and the local pre-commit hook
  can never drift. Elixir/OTP come from the repo's `mise.toml` via mise — no version
  literal in CI. Inputs: `postgres-db` (required), `postgres-port` (`0` = a free random port, exported as `DB_PORT`/`PGPORT`; have config/test.exs read `DB_PORT` so several runner instances can share one machine), `postgres-image`,
  `gate-command`, `parallel-dialyzer` (a separate dialyzer lane for gate commands that
  skip it — wall-clock becomes max, not sum), `timeout-minutes`, `runs-on` (runner label;
  default `ubuntu-latest`).
- **`.github/workflows/deploy.yml`** — Fly.io deploy. SHA-pinned `setup-flyctl` with the
  token scoped to the deploy step only, and **no `--strategy` flag** — the strategy lives
  in the fly config's `[deploy]` section, because the CLI flag silently overrides the toml
  and that pair drifts. Inputs: `fly-config`, `timeout-minutes`; secret: `FLY_API_TOKEN`.
  Callers must `needs:` a gate job — a red revision never deploys.

### `psfly compile-check` / `psfly warnings`

Token- and noise-frugal compile loop for the nearest enclosing Mix project, from anywhere
inside it. `compile-check` compiles quietly to `tmp/compile.log`, failing only on a hard
error and printing a one-line warning count; `warnings` shows only the warnings not
already accepted in the repo's `priv/known_warnings.txt`.

## Configuration

Each consuming repo commits flat `KEY=VALUE` files under `.psfly/<env>.conf` naming its own
apps, databases, and buckets. The kit reads config; it hardcodes nothing about any fleet.

## Planned

- `psfly init` — wire a new repo: caller workflows, fly toml skeletons, `.psfly/` config.
- `psfly doctor` — read-only environment/wiring readiness check.
- An ephemeral PR preview-environment reusable workflow.
