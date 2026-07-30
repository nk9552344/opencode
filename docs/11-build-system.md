# Chapter 11: Build System and Tooling

**Sources:** `script/build.ts`, `script/generate.ts`, `packages/script/`, `turbo.json`, `bunfig.toml`, `package.json`

---

## Overview

The build system has three jobs:

1. **Compile** the CLI into standalone binaries for 12 platform/arch targets
2. **Generate** a models.dev snapshot embedded into the binary at build time
3. **Typecheck** all packages via Turbo

No bundler config files (webpack, rollup, vite). The entire build is expressed in `script/build.ts` using Bun's native compile mode.

---

## `script/build.ts` — Cross-Platform Binary Compilation

Run with: `bun run build`

### Binary Targets (12 total)

| OS | Arch | Libc | Baseline |
|---|---|---|---|
| linux | arm64 | glibc | no |
| linux | x64 | glibc | no |
| linux | x64 | glibc | yes (AVX2-off) |
| linux | arm64 | musl | no |
| linux | x64 | musl | no |
| linux | x64 | musl | yes |
| darwin | arm64 | — | no |
| darwin | x64 | — | no |
| darwin | x64 | — | yes |
| win32 | arm64 | — | no |
| win32 | x64 | — | no |
| win32 | x64 | — | yes |

`--single` flag — restricts build to the current platform only.

`--baseline` flag — controls AVX2 selection; baseline variants run on older CPUs without AVX2 instruction support.

Output: `./dist/<os>-<arch>[-musl][-baseline]/bin/lildax` plus a `package.json` per target with `os` and `cpu` fields (used by npm's optional dependency selection mechanism for binary distribution).

### Bun Compile Mode

```ts
await Bun.build({
  compile: true,
  target: "bun-linux-arm64",   // or bun-darwin-x64, bun-windows-x64, etc.
  entrypoints: ["./src/index.ts"],
  outfile: "./dist/linux-arm64/bin/lildax",
  plugins: [createSolidTransformPlugin()],   // @opentui/solid JSX transform
  define: {
    OPENCODE_VERSION: JSON.stringify(version),
    OPENCODE_CLI_NAME: JSON.stringify("opencode"),
    OPENCODE_MODELS_DEV: JSON.stringify(modelsData),  // embedded JSON snapshot
    OPENCODE_CHANNEL: JSON.stringify(channel),
    OPENCODE_LIBC: JSON.stringify(libc),
    "FFF_LIBC": JSON.stringify(libc),
    "process.env.OPENTUI_LIBC": JSON.stringify(libc),
  },
  sourcemap: flags.sourcemaps ? "linked" : "none",
})
```

`compile: true` — Bun produces a self-contained executable that bundles the JavaScript, its dependencies, and the Bun runtime into a single binary. No Node.js, no `node_modules` needed on the target machine.

`createSolidTransformPlugin()` — the `@opentui/solid` Bun plugin transforms SolidJS JSX into fine-grained reactive calls at build time (not at runtime).

### Define Constants

Constants injected into the binary at compile time:

| Constant | Source | Purpose |
|---|---|---|
| `OPENCODE_VERSION` | `packages/script/src/index.ts` | Reported in `--version` and health check |
| `OPENCODE_CLI_NAME` | hardcoded `"opencode"` | Binary name for display |
| `OPENCODE_MODELS_DEV` | `script/generate.ts` output | models.dev API JSON snapshot |
| `OPENCODE_CHANNEL` | `packages/script/src/index.ts` | Release channel (`"latest"` or branch name) |
| `OPENCODE_LIBC` | build flag | `"musl"` or `"glibc"` |
| `FFF_LIBC` | same | For `@opentui/core` native addon selection |
| `process.env.OPENTUI_LIBC` | same | For opentui internal detection |

---

## `script/generate.ts` — Models Snapshot

```ts
const modelsUrl = process.env.OPENCODE_MODELS_URL || "https://models.dev"

export const modelsData: string = process.env.MODELS_DEV_API_JSON
  ? await Bun.file(process.env.MODELS_DEV_API_JSON).text()
  : await fetch(`${modelsUrl}/api.json`).then(r => r.text())
```

This file is imported by `build.ts`. When the build runs, it fetches the models.dev API JSON (or reads from a local file via `MODELS_DEV_API_JSON` env var for reproducible builds). The result is embedded as the `OPENCODE_MODELS_DEV` define constant.

**Why embed models at build time?**

- The binary works offline with the embedded model list
- The runtime can also refresh the list from models.dev on demand
- Fixes the binary's "baseline" catalog without requiring network access on first run

---

## `packages/script/src/index.ts` — Version and Channel

```ts
export const Script = {
  get channel()  { return CHANNEL },    // git branch name, or "latest" for releases
  get version()  { return VERSION },    // from OPENCODE_VERSION env, or bumped from npmjs
  get preview()  { return IS_PREVIEW }, // true when channel !== "latest"
  get release(): boolean { return !!env.OPENCODE_RELEASE },
  get team()     { return team },       // TEAM_MEMBERS + known bots
}
```

Environment variables read at build time:
- `OPENCODE_CHANNEL` → release channel
- `OPENCODE_BUMP` → version bump type (`patch`, `minor`, `major`)
- `OPENCODE_VERSION` → explicit version override
- `OPENCODE_RELEASE` → whether this is a production release

Also validates that the running Bun version matches the `packageManager` field in root `package.json` (prevents accidental builds with wrong Bun version).

---

## Turbo — Typecheck Orchestration — `turbo.json`

```json
{
  "$schema": "https://v2-8-13.turborepo.dev/schema.json",
  "globalEnv": ["CI", "OPENCODE_DISABLE_SHARE"],
  "globalPassThroughEnv": ["CI", "OPENCODE_DISABLE_SHARE"],
  "tasks": {
    "typecheck": {},
    "build": {
      "dependsOn": [],
      "outputs": ["dist/**"]
    },
    "@opencode-ai/core#test": {
      "dependsOn": ["^build"],
      "outputs": []
    },
    "@opencode-ai/ui#test": {
      "dependsOn": ["^build"],
      "outputs": []
    }
  }
}
```

Turbo is used only for `typecheck` (runs `tsc --noEmit` in each package in dependency order). The actual build uses `bun run build` directly from the root.

Task definitions:
- **`typecheck`** — no deps, no outputs. Turbo just fans it out to every package that has a `typecheck` script. Uses Turbo's dependency-aware parallelism to check packages in topological order.
- **`build`** — no `dependsOn` (packages don't build-depend on each other via Turbo; Bun handles that at bundle time). Outputs `dist/**` for caching.
- **`@opencode-ai/core#test`** and **`@opencode-ai/ui#test`** — the only two test suites that can run from root. Both depend on `^build` (all upstream packages must build first).

---

## Bun Configuration — `bunfig.toml`

```toml
preload = ["@opentui/solid/preload"]

[install]
exact = true
minimumReleaseAge = 259200   # 3 days in seconds
minimumReleaseAgeExcludes = [
  "@ai-sdk/amazon-bedrock",
  "@ai-sdk/anthropic",
  "@opentui/core",
  # ... other fast-moving packages
]

[test]
root = "./do-not-run-tests-from-root"
```

- **`preload`** — `@opentui/solid/preload` runs before every Bun script, setting up SolidJS runtime hooks. Required because Bun evaluates the preload before transpiling, which registers the JSX factory.
- **`exact = true`** — all installs use exact versions (no `^` or `~` ranges in lockfile). This makes the dependency tree fully reproducible.
- **`minimumReleaseAge`** — new package versions must be at least 3 days old before Bun will install them. Protects against supply-chain attacks on recently published versions.
- **`minimumReleaseAgeExcludes`** — AI SDK and opentui packages release frequently and legitimately need to be installed immediately.
- **`[test] root`** — redirects test discovery to a non-existent directory. Prevents `bun test` run from the root from accidentally discovering and running all test suites simultaneously. Individual packages run their own tests via `cd packages/foo && bun test`.

---

## Workspace Structure — `package.json`

The root `package.json` acts as both the workspace root and the CLI package (after the restructure).

```json
{
  "name": "opencode",
  "workspaces": ["packages/*", "packages/sdk/js"],
  "bin": { "opencode": "./bin/lildax.cjs" },
  "scripts": {
    "build":       "bun run script/build.ts",
    "dev":         "bun run src/index.ts",
    "lint":        "...",
    "typecheck":   "turbo typecheck",
    "postinstall": "bun run packages/tui/src/parsers-config.ts",
    "test":        "bun test"
  }
}
```

- `bin/lildax.cjs` — the CJS shim that launches the compiled binary (or falls back to direct execution during development)
- `postinstall` — runs the parsers config script to download Tree-sitter WASM files for syntax highlighting
- `"workspaces": ["packages/*", "packages/sdk/js"]` — Bun workspace setup; `packages/sdk/js` is listed separately because it is nested deeper than one level

---

## Development Workflow

### Local Development

```bash
# Run the TUI directly (no compilation)
bun run dev

# Run typecheck across all packages
bun run typecheck

# Build binaries for all 12 targets
bun run build

# Build only for current platform
bun run build --single
```

### Adding a New Package

1. Create `packages/my-package/package.json` with `"name": "@opencode-ai/my-package"`
2. Add any inter-package deps using workspace protocol: `"@opencode-ai/schema": "workspace:*"`
3. Bun automatically links workspace packages; no separate install step needed
4. Add a `typecheck` script: `"typecheck": "tsc --noEmit"` to participate in `turbo typecheck`

### Package Catalog

`bun.lock` pins all packages to exact versions. The `packages/script/` package is the canonical source for version numbers used across the monorepo.

---

## Binary Distribution

The 12 compiled binaries are published as separate npm packages (one per platform), with the main `opencode` package depending on the correct one via optional dependencies based on `os` and `cpu` fields in each binary's `package.json`. The `bin/lildax.cjs` shim locates and executes the appropriate binary from `node_modules`.

This is the same distribution pattern used by esbuild, Turbo, and other compiled CLI tools distributed via npm.
