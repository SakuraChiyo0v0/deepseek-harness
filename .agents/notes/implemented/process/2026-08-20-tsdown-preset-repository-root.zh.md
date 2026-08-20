# Agent Note: Client tsdown 预设按标记定位仓库根目录

Status: implemented

[English](2026-08-20-tsdown-preset-repository-root.md) | 中文

## Problem

共享的 client tsdown 预设（`packages/client/tsdown.client.ts`）用 `new URL('../..', import.meta.url)` 推导仓库根目录，这仅在预设文件本身是 tsdown 加载的模块时成立。引入 unrun 配置加载器回退后，tsdown 会把每个包配置打包，并把预设内部的 `import.meta.url` 改写为包配置的路径（比预设深一层）。相对深度于是解析到 `packages/` 而不是仓库根目录，`workspaceManifest()` 从错误的 cwd 扫描 `packages/*/*/package.json`，一无所获，host 构建报 `tsdown: no packages/*/*/package.json declares the name @deepseek-ai/dsh-api-remotes`，阻塞了推送前的 `typecheck` 门禁。

## Decision

预设不再假定固定相对深度，而是从自身位置向上走到工作区标记（`pnpm-workspace.yaml`）来定位仓库根目录。两种加载器都满足该向上查找：直接加载时 `import.meta.url` 停留在预设文件，unrun 打包时改写为同一仓库内的包配置路径，两种起点都能到达标记。找不到标记时在加载阶段报错，而不是稍后以误导性的 manifest 错误失败。

## Alternatives considered

**在构建脚本中把 tsdown 配置加载器固定为 `tsx`。** 加载器选择由 tsdown 自身自动检测；用脚本参数与工具对抗，而不是让预设与加载器无关，并且对工作区构建之外直接使用预设的情形毫无帮助。

**用 `process.cwd()` 计算根目录。** tsdown 恰好以仓库根目录作为每个工作区包配置的 cwd，但这是预设无法断言的隐式构建运行时约定；从包目录单独构建时会在静默中解析到错误的根目录。

**补偿被改写的深度。** 硬编码 `'../../..'` 只是把问题移到直接加载路径，那里原来的两层才是正确的。

## Consequences

预设不再依赖 tsdown 加载配置文件的方式，也不再依赖预设相对于仓库根目录的位置。每次构建只多几次 `stat` 调用；工作区标记与仓库定义工作区时使用的标记相同。

## Testing

`pnpm run typecheck`（此前失败的推送前门禁）在 unrun 加载器下通过。两个 tsdown face 均可构建（`build:lib:host`、`build:lib:client`），聚焦的 client 预设测试（`client-bundle-purity`、`client-bundle-css`、`client-build-environment.client`）全部通过。
