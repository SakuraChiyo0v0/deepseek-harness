# Agent Note: Client tsdown preset locates the repository root by marker

Status: implemented

English | [中文](2026-08-20-tsdown-preset-repository-root.zh.md)

## Problem

The shared client tsdown preset (`packages/client/tsdown.client.ts`) derived the repository root as `new URL('../..', import.meta.url)`, correct only while the preset file itself was the module tsdown loaded. With the unrun config-loader fallback, tsdown bundles each package config and rewrites `import.meta.url` inside the preset to the package config's path (one level deeper than the preset). The relative depth then resolved to `packages/` instead of the root, so `workspaceManifest()` globbed `packages/*/*/package.json` from the wrong cwd, matched nothing, and the host build failed with `tsdown: no packages/*/*/package.json declares the name @deepseek-ai/dsh-api-remotes` — blocking the pre-push `typecheck` gate.

## Decision

The preset locates the root by walking up from its own location to the workspace marker (`pnpm-workspace.yaml`) instead of assuming a fixed relative depth. Both loaders satisfy the walk: a direct load keeps `import.meta.url` at the preset file, and the unrun bundle rewrites it to a package config path inside the same repository, so either start reaches the marker. A missing marker throws at load rather than failing later with a misleading manifest error.

## Alternatives considered

**Pin the tsdown config loader to `tsx` in build scripts.** The loader selection is tsdown's own auto-detection; a script flag fights the tool instead of making the preset loader-agnostic, and it does nothing for direct preset use outside the workspace build.

**Compute the root from `process.cwd()`.** tsdown happens to evaluate every workspace package config with the repository root as cwd, but that is an implicit build-runtime contract the preset cannot assert; a standalone package build from its own directory would silently resolve the wrong root.

**Compensate for the rewritten depth.** Hard-coding `'../../..'` only moves the breakage to the direct-load path, where the original two levels were correct.

## Consequences

The preset no longer depends on how tsdown loads config files or where the preset sits relative to the root. The walk is a few `stat` calls per build; the workspace marker is the same marker the repo already uses to define the workspace.

## Testing

`pnpm run typecheck` (the pre-push gate that failed) passes with the unrun loader. Both tsdown faces build (`build:lib:host`, `build:lib:client`), and the focused client-preset specs (`client-bundle-purity`, `client-bundle-css`, `client-build-environment.client`) pass.
