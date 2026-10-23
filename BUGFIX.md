# BUGFIX — 1.17.2

## Bug 标题

既有每日计划的 `items_json` 不可解析时，全量编辑（PUT）与草稿确认会把它当成「空计划」静默覆盖销毁，与另两条写路径「拒绝且原数据未改动」的口径不一致。

## 证据链

模块自身口径（修复前）：

- `parseDailyPlan` 对坏数据的口径是「坏数据**原样保留**（一个字节都不动）」，读成 `readable=false`，让容量显示「不可计算」而不是假装没有计划；
- `addDailyPlanItem`（原子追加）对不可解析计划明确拒绝：`计划 … 的数据无法解析，不能追加（原数据未改动，请先备份后修复或清空）`；
- `updateDailyPlanItem`（项级 PATCH）同样拒绝：`… 不能修改（原数据未改动）`；
- PUT 路由对「清空计划」另有显式入口（items 为空时 400 并提示「清空计划请用 DELETE」）。

但 `updateDailyPlan`（全量编辑）与 `confirmDailyPlanDraft`（草稿确认）共用的 `mergeAndCheck()`
把不可解析的既有计划按 `previousItems = []` 处理，随后走 upsert 分支把提交内容写回同一行：
坏数据被一字节不剩地覆盖，返回的计划 `readable=true`、`diagnostics=[]`，调用方无从知晓数据曾损坏、更无从备份修复。

最小复现（内存 SQLite，走真实仓储层；脚本在交付工作区 `repro.mjs`）：先直插一条
`items_json = '{"items": [broken'` 的计划行（模拟手工改库/写坏的历史数据），然后：

```
seeded readable = false | raw intact = true
addDailyPlanItem -> {"ok":false,"error":"计划 2026-10-24 的数据无法解析，不能追加（原数据未改动，请先备份后修复或清空）"}
updateDailyPlanItem -> {"ok":false,"error":"计划 2026-10-24 的数据无法解析，不能修改（原数据未改动）"}
updateDailyPlan -> {"threw":false,"items":1,"readable":true,"diagnostics":[]}
raw after PUT intact = false
confirmDailyPlanDraft -> {"threw":false} | raw intact = false | draft still pending = false
```

脚本 exit 1。`git stash` 只回退本版修复、用同一脚本复跑，输出逐字一致（exit 1），证明缺陷为上游既有、非本次引入。

上游 open issue 仅 2 条：#13（浮层被宿主 overlay 层盖住）经核对在上游 HEAD 已由
`src/client/overlayMount.ts` 修复落地，不可用；#11 无正文。故走静态定位 + 最小复现路线。

## 根因

`mergeAndCheck()` 只用 `previous?.readable === true ? previous.items : []` 决定合并基线，
不可解析时「基线为空」与「没有既有计划」被混为一谈；下游 `updateDailyPlan` /
`saveDailyPlan` 的 upsert 又不设 readable 守卫，于是「读不出来」被静默转成「覆盖重写」。

## 改动文件

- `src/db/repo/plans.ts`（唯一逻辑改动，+11 行）：`mergeAndCheck()` 开头新增守卫——
  `previous !== undefined && !previous.readable` 时抛错
  `计划 <date> 的数据无法解析，不能保存（原数据未改动，请先备份后修复或清空）`。
  该函数是全量编辑与草稿确认的共同收口，一处修复同时护住两条路径；
  草稿确认在 `withDraftConfirm` 事务内执行，抛错即回滚且草稿保留 pending（已复验）。
  底层原语 `saveDailyPlan` 按其自述「不做任何校验」的契约未动。
- `package.json`：version 1.17.1 → 1.17.2；`repository.url` 改指 fork 落点仓（diff 恰两行）。
- `README.md`：顶部追加中文 fork 头，上游原文一字未删。
- 新增 `UPSTREAM.md` / 本文件。`LICENSE` 与上游逐字一致，未动。

## 复验结果

- 复现脚本（同上）：修复后 exit 0 —— `updateDailyPlan` 与 `confirmDailyPlanDraft` 均抛错拒绝，
  `items_json` 原串一字节未动，草稿仍为 pending。
- 全量测试 `pnpm test`（build + `node --test test/*.test.mjs`）：1310 条，1308 过 / 0 失败 / 2 跳过，
  与修复前基线逐项一致，零回归（2 跳过为 Windows 平台专属用例）。
- `pnpm typecheck`（`tsc --noEmit`）退出 0。

## 验证边界

- 未做真实 DSH 宿主端到端验证；复现走仓储层真实代码 + 内存 SQLite，与上游自有测试
  （`test/dailyEffort.test.mjs` 等）的仓储层证据口径一致。
- 本版只拒绝「经 `mergeAndCheck` 的两条受校验写路径」对不可解析计划的保存；
  显式清空仍走 DELETE（`deleteDailyPlan`），与上游既有口径一致。
- 沙箱基线说明：修复前基线为 typecheck 0、`pnpm test` 1310 条全过（1308 过 / 2 跳过），
  本缺陷不在既有测试覆盖内（坏 JSON 的既有测试只钉读侧行为）。
