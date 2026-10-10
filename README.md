# Codessin capture action

Freeze your rendered UI in CI and upload it to Codessin, so design drift shows up
as a diff instead of a surprise.

> **Status: pre-release.** The `@codessin/capture` runner this action invokes is not
> yet published to npm. Until it is, the capture step will fail to resolve the
> package. Watch this repository for the `v1` tag.

## Quick start

Two things to get right:

- **`CODESSIN_CAPTURE: "1"` goes on the build step**, not the capture step. That is
  what turns on the anchors plugin, and it is why your production builds never
  carry source paths.
- **Pin the action to a commit SHA.** A tag can be moved to point at different
  code; a SHA cannot. Pin it, and let Dependabot raise the bumps.

```yaml
# .github/workflows/codessin-capture.yml
name: Codessin capture

on:
  push:
    branches: [main]
  pull_request:

jobs:
  capture:
    runs-on: ubuntu-latest
    continue-on-error: true
    steps:
      - uses: actions/checkout@v5
      - uses: actions/setup-node@v5
        with:
          node-version: 22
          cache: npm
      - run: npm ci
      - run: npm run build
        env:
          CODESSIN_CAPTURE: "1"
      - uses: codessin-dev/capture-action@<commit-sha>
        with:
          api-key: ${{ secrets.CODESSIN_API_KEY }}
```

Capture never fails your build — the runner exits `0` for any capture outcome.
`continue-on-error: true` is belt and braces for the surrounding steps.

## Inputs

| Input | Default | Description |
|---|---|---|
| `api-key` | — | Project API key with the `capture:write` scope. Absent on fork PRs, where the run is skipped rather than failed. |
| `config` | `codessin.config.json` | Path to the config file, relative to `working-directory`. |
| `working-directory` | `.` | Directory to run in. Set this for a package inside a monorepo. |
| `api-url` | `https://api.codessin.dev` | Codessin API base URL. Override only for self-hosted instances. |
| `version` | `0.1.0` | Exact `@codessin/capture` version to run. Pinned deliberately: a capture runner that changes under you changes what your captures look like. |

## Outputs

| Output | Description |
|---|---|
| `run-url` | Link to this run in Codessin, when one was uploaded. |

## Configuration

`codessin.config.json` tells the runner how to start your app and what to capture:

```jsonc
{
  "version": 1,
  "projectId": "your-project-id",
  "app": {
    "start": "npm run preview -- --port 4173",
    "url": "http://127.0.0.1:4173"
  },
  "viewports": {
    "mobile": { "width": 375, "height": 812 },
    "desktop": { "width": 1440, "height": 900 }
  },
  "routes": [{ "path": "/", "figma": { "desktop": "12:345" } }]
}
```

## Framework setup

### Next.js (webpack or Turbopack)

```ts
// next.config.ts
import { codessinAnchors } from "@codessin/anchors";

export default codessinAnchors.next({
  // your existing settings
});
```

`codessinAnchors.next()` detects Turbopack and configures it accordingly, so the
workflow and config are unchanged either way.

```jsonc
{ "app": { "start": "npm run start -- --port 3000", "url": "http://127.0.0.1:3000" } }
```

### Nuxt

```ts
// nuxt.config.ts
import { codessinAnchors } from "@codessin/anchors";

export default defineNuxtConfig({
  vite: { plugins: [codessinAnchors.nuxt()] },
});
```

```jsonc
{ "app": { "start": "node .output/server/index.mjs", "url": "http://127.0.0.1:3000" } }
```

### Storybook only

No app server: point `storybook.staticDir` at the build and omit `routes`. The
runner serves that directory itself, on loopback.

```yaml
      - run: npm run build-storybook
        env:
          CODESSIN_CAPTURE: "1"
      - uses: codessin-dev/capture-action@<commit-sha>
        with:
          api-key: ${{ secrets.CODESSIN_API_KEY }}
```

```jsonc
{
  "version": 1,
  "projectId": "your-project-id",
  "app": { "url": "http://127.0.0.1:6006" },
  "viewports": { "desktop": { "width": 1440, "height": 900 } },
  "routes": [],
  "storybook": {
    "staticDir": "storybook-static",
    "include": ["Components/**"],
    "exclude": ["**/Deprecated/**"]
  }
}
```

### A Turborepo monorepo

Build the one package you are capturing, and run the action inside it.

```yaml
      - run: npx turbo run build --filter=@acme/web
        env:
          CODESSIN_CAPTURE: "1"
      - uses: codessin-dev/capture-action@<commit-sha>
        with:
          api-key: ${{ secrets.CODESSIN_API_KEY }}
          working-directory: apps/web
```

`codessin.config.json` lives in `apps/web/`, and its paths are relative to it.

## Pull requests from forks

Leave the `pull_request` trigger in place. GitHub withholds secrets from fork
runs, so the capture step prints

```
Skipped: no CODESSIN_API_KEY available to this run.
```

and exits `0`. Your build passes; there is simply no capture for that PR.

**Do not use `pull_request_target` to work around this.** It runs the fork's code
with your repository's secrets — it would hand any contributor who opens a PR
your Codessin key, and everything else in that environment with it.

## License

MIT
