# UPSTREAM

- 上游仓库：https://github.com/Dely0/dsh-personal-workbench
- 固定版本（HEAD SHA）：`f1be68586c3278f0105514a4d5c5da9d634a27ff`
- 上游版本：1.17.1（npm 包 `@dely0/dsh-personal-workbench`）
- 许可证：MIT（见仓库根 `LICENSE`，与上游逐字一致；另有第三方许可文件 `LICENSE-novotnyllc-dotnet-artisan`、`LICENSE-K-Dense-scientific-agents`、`THIRD_PARTY_NOTICES.md`，均原样保留）
- 拉取日期：2026-10-23
- 原作者：Dely0（Du Penglai）

## 仓库身份核对（交付前披露）

`gh api repos/dely0/dsh-personal-workbench` 返回的 `full_name` 为 `Dely0/dsh-personal-workbench`，
与任务给定的 `dely0/dsh-personal-workbench` 仅大小写之差，为同一仓库，无改名/重定向。

## 本 fork 的改动范围

仅 `src/db/repo/plans.ts` 一处逻辑（见 BUGFIX.md）、`package.json` 的 `version`
（1.17.1 → 1.17.2）与 `repository.url`（改指本 fork 落点仓）、README 顶部 fork 说明头；
新增本文件与 BUGFIX.md。上游其余文件一字未动，`LICENSE` 原样保留。
pnpm-lock.yaml 无根包版本字段，上游亦无独立版本戳常量（buildId 于构建时生成），无其他待同步项。


---

## 修复版版权

本修复版的修改部分 © 2026 lhg-dsh-plugins（刘洪光），修改内容见 [BUGFIX.md](./BUGFIX.md)；原始代码版权归原作者所有，见仓库内 LICENSE 与上方出处。
