# Nx Affected — Smart Deploys for Nx Monorepos

[![Test Action](https://github.com/mayurrawte/github-nx-affected/actions/workflows/test.yml/badge.svg)](https://github.com/mayurrawte/github-nx-affected/actions/workflows/test.yml)
[![Marketplace](https://img.shields.io/badge/Marketplace-Nx%20Affected-blue?logo=github)](https://github.com/marketplace/actions/nx-affected)

**Stop rebuilding every app on every push.** This action runs `nx affected` against a smart base SHA, then hands you a matrix-ready JSON array of changed apps — drop it straight into `strategy.matrix` and your CI only builds/tests/deploys what actually changed.

## Why this exists

Default Nx CI templates rebuild your entire monorepo on every commit. A 12-app workspace becomes a 45-minute build for a one-line lib change. Existing actions stop at "setup the SHAs for you" — none output a matrix you can fan out on, and none handle the "nothing affected, skip everything" case cleanly.

## Quick start

```yaml
jobs:
  affected:
    runs-on: ubuntu-latest
    outputs:
      apps: ${{ steps.nx.outputs.affected-apps }}
      has-affected: ${{ steps.nx.outputs.has-affected }}
    steps:
      - uses: actions/checkout@v4
        with: { fetch-depth: 0 }  # required: nx affected needs git history
      - id: nx
        uses: mayurrawte/github-nx-affected@v1
        with:
          target: deploy

  deploy:
    needs: affected
    if: needs.affected.outputs.has-affected == 'true'
    runs-on: ubuntu-latest
    strategy:
      matrix:
        app: ${{ fromJSON(needs.affected.outputs.apps) }}
    steps:
      - uses: actions/checkout@v4
      - run: npx nx deploy ${{ matrix.app }} --configuration=production
```

That's it. One affected app → one deploy job. Zero affected → entire `deploy` job is skipped.

## Inputs

| Input | Default | Notes |
|---|---|---|
| `target` | `''` | Filter affected projects to those with this target defined (`build`, `test`, `deploy`, `lint`, …). Empty = all affected. |
| `base` | auto | Comparison base SHA. Auto-detected: PRs → `pull_request.base.sha`, pushes → `event.before`, fallback `HEAD~1`. |
| `head` | `HEAD` | Comparison head SHA. |
| `node-version` | `22` | Node.js version. |
| `package-manager` | auto | `npm` / `yarn` / `pnpm`. Auto-detected from lockfile. |
| `working-directory` | `.` | Where `nx.json` lives (monorepo at repo root by default). |
| `install` | `true` | Run `npm ci` / `yarn install` / `pnpm install` before computing affected. |
| `auto-deepen` | `true` | If the base SHA isn't in the local clone, auto-deepen git history. |

## Outputs

| Output | Type | Notes |
|---|---|---|
| `affected-apps` | JSON array | Matrix-ready: `["app1","app2"]` |
| `affected-libs` | JSON array | Same shape, library projects. |
| `affected` | JSON array | All affected (apps + libs). |
| `has-affected` | `'true'` / `'false'` | Use in `if:` conditions. |
| `count` | number | Total affected count. |
| `base-sha` | string | The resolved base SHA used. |

## Recipes

### Deploy affected apps to Vercel

```yaml
- id: nx
  uses: mayurrawte/github-nx-affected@v1
  with: { target: deploy }

- name: Deploy affected
  if: steps.nx.outputs.has-affected == 'true'
  strategy:
    matrix:
      app: ${{ fromJSON(steps.nx.outputs.affected-apps) }}
  run: vercel deploy apps/${{ matrix.app }} --prod
```

### Test only affected libs on PR

```yaml
- uses: actions/checkout@v4
  with: { fetch-depth: 0 }

- id: nx
  uses: mayurrawte/github-nx-affected@v1
  with: { target: test }

- if: steps.nx.outputs.has-affected == 'true'
  run: npx nx run-many --target=test --projects=${{ steps.nx.outputs.affected }}
```

### Conditional Docker builds

```yaml
- id: nx
  uses: mayurrawte/github-nx-affected@v1
  with: { target: docker-build }

- if: steps.nx.outputs.has-affected == 'true'
  strategy:
    matrix:
      app: ${{ fromJSON(steps.nx.outputs.affected-apps) }}
  run: docker buildx build apps/${{ matrix.app }} --push
```

## Requirements

- **Nx 17+** for `--with-target` filtering. Older Nx works without the `target:` input.
- **`fetch-depth: 0` on actions/checkout** — `nx affected` needs git history to diff. The action auto-deepens if it can, but `fetch-depth: 0` is cleaner.
- An Nx workspace with `nx.json` at the working directory.

## How base SHA is resolved

| Event | Base used |
|---|---|
| `pull_request` | `github.event.pull_request.base.sha` |
| `push` (non-initial) | `github.event.before` |
| `push` (initial commit) / `workflow_dispatch` / others | `HEAD~1` |
| User-provided `base:` input | Always wins |

## License

MIT — see [LICENSE](LICENSE).
