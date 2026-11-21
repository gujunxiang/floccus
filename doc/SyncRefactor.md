# 同步模型重构设计：从「单文件快照 + 本地缓存推断」到「带状态的条件写」

状态：草案（refactor/sync-atomicity）
范围：`src/lib/adapters/Git.ts`、`src/lib/serializers/*`、`src/lib/strategies/*`、`src/lib/Account.ts`、缓存/映射层
目标后端：以 Git/WebDAV/S3 这类「文件快照」后端为主，同时不破坏 Nextcloud Bookmarks 等 atomic 后端

## 1. 问题陈述

多端（本机 3 个浏览器 + 另一台电脑）同时对同一个 Gitee 仓库读写时，反复出现：

- 删除的书签过一段时间「复活」；
- 批量条目被删掉后又补回（回退）；
- 长时间没有任何新提交，但本地看起来在同步。

历史证据（`gitee.com/xiangagou/bookmark`）：

- 555 次提交中，38 次单次净减少 ≥3 条，其中 27 次之后总量又回到原水平；
- 7 份完全相同的文件内容在不同提交里重复出现（整份文件回到过去的状态）；
- 《Quick》在 11-17、11-18、11-19 被删三次、复活三次；《Atlassian》11-19 09:44 删除、19:41 复活；
- 2026-11-20 11:02 之后 9.5 小时零提交，而客户端每 15 分钟同步一次且报告 `0/642=0`、`No changes to the server version necessary`。

## 2. 根因（代码级）

### 2.1 删除不可见：文件里没有墓碑

`src/lib/serializers/Xbel.ts` / `Html.ts` 只序列化「当前存在的树」，没有删除标记、版本号或时间戳。
删除一条书签 = 它从文件里消失；服务器无法区分「被删除」和「从未存在」。

### 2.2 合并基准在各端本地

`Default.getDiffs()`（`src/lib/strategies/Default.ts:1200`）以本地 `cacheTree` 为 base：

- `localScanner`：cache ↔ 本地
- `serverScanner`：cache ↔ 服务器

`Scanner.diffFolder()` 把「cache 里有、新树里没有」的子项记为 `REMOVE`。
因此删除的传播完全依赖每个客户端的 `bookmarks[<accountId>].cache`（`src/lib/browser/BrowserAccountStorage.js`）与 mappings。
任一端的 cache 丢失/错位（重装、升级、清存储、同步中断、新设备），删除就会被误读成「新增」而复活。

### 2.3 Git 锁不是互斥

`src/lib/adapters/Git.ts`：

- `obtainLock()` 只是查询远端是否存在「新鲜的」`floccus-lock-*` tag；
- `setLock()` 各自 push 一个 `floccus-lock-<Date.now()>` 的**不同** tag。

多个客户端在彼此看不到锁的情况下会各自创建自己的 tag，**都能认为自己拿到了锁**——没有在同一个 ref 上做 compare-and-set。

### 2.4 读-改-写-推不是原子的

`onSyncStart()` fetch → 本地执行计划 → `onSyncComplete()` commit + push。
push 失败抛 `GitPushError`，此时本地已改、远端未改、cache/mappings 已推进，下一轮可能基于错误基准合并。

### 2.5 设计取向

`CONSIDERATIONS.md` 明确：目标优先级是「最终一致 > 保留意图」，且 WebExtensions 书签 API 没有事务/CAS 原语。
这是设计选择，不是单纯 bug；但在多写端场景下会放大成用户可见的回退。

## 3. 目标与非目标

目标：

- 多端并发写：不丢更新、不复活已删除条目；
- 同步中断可自愈，不产生「删了又回来」；
- 文件后端（Git/WebDAV/S3）在单写与多写下都可用；
- 保持现有合并引擎（`Scanner`/`Diff`/`strategies`）与后端兼容。

非目标：

- 重写整个 merge 引擎；
- 引入强依赖的中央服务；
- 一次做完 op-log/CRDT（作为后续可选阶段）。

## 4. 目标架构

### 4.1 存储契约：revision + 条件写（CAS）

新增/扩展适配器能力（参考现有 atomic adapter 区分）：

```ts
interface IVersionedStorage {
  // 读取整份快照及其版本标识
  readSnapshot(): Promise<{ tree: Folder<Server>; revision: string }>
  // 仅当服务器仍处于 expectedRevision 时写入，否则抛 ConflictError
  writeSnapshot(tree: Folder<Server>, expectedRevision: string): Promise<{ revision: string }>
}
```

- Git：`revision` = commit sha；写入 = commit(parent = expectedRevision) + 非 force push；服务端 ref 移动则 push 被拒 → `ConflictError`。
- WebDAV：`revision` = ETag，写入用 `If-Match`。
- S3：`revision` = VersionId / ETag，写入用条件头。
- Nextcloud Bookmarks：已有逐条语义，走现有 atomic 路径。

### 4.2 Git 锁：单一 ref + CAS

- 固定锁 ref，例如 `refs/tags/floccus-lock`（或专用 branch）；
- 获取 = 以「不存在」为前提创建（非 force push；已存在则失败 = 被占用）；
- 锁内容：owner id、时间戳、持有者 commit；
- 过期清理也走 CAS（先读到旧 oid，再以 old-oid 为条件删除），避免误删新锁；
- 获取失败 → 有界退避重试；**不再使用按时间戳命名的多 tag**。

### 4.3 墓碑（tombstones）

在同步文件中显式记录删除，消除「靠 cache 推断」：

```xml
<floccus:tombstones>
  <floccus:tombstone id="..." deleted-at="..." hash="..."/>
</floccus:tombstones>
```

规则：

- 合并时墓碑优先：客户端看到墓碑 → 本地删除该条，而不是把它当新增推回；
- 墓碑随文件提交，任何客户端（包括 cache 全新的）都能正确执行删除；
- GC：默认保留 30 天，或按「所有已知客户端 revision 都已越过该墓碑」回收（先实现时间策略）；
- 兼容：读取旧文件（无墓碑）时回退到现有 cache 推断；写入新文件时带墓碑。

### 4.4 事务化同步流程

```
acquire lock (CAS)
  → fetch latest snapshot + revision
  → 以 latest 为 base 重算三方合并（cache 仅作辅助）
  → 生成新树 + 新墓碑
  → writeSnapshot(tree, expectedRevision)   // CAS
      ├─ 成功 → 应用本地计划 → release lock
      └─ ConflictError → 释放/重取锁 → 重新 fetch/merge（有界重试）
```

要点：

- **服务端提交成功后再执行本地计划**（或本地计划可回滚），避免「本地已改、远端没改」；
- 合并 base 使用刚 fetch 到的 revision，而非过期的本地 cache；
- 重试用指数退避 + 上限，失败则本次同步失败并保留现场（不写入错误状态）。

### 4.5 顺序（reorder）降噪

- 将 REORDER 视为低优先级元数据；
- 引入稳定排序键（例如按 id/创建时间），避免同一棵树在不同客户端产生不同顺序 → 假 diff；
- 或在比较时忽略纯顺序差异，仅在用户显式排序时提交 REORDER。

## 5. 分阶段实施（PR 拆分）

| PR | 内容 | 影响面 | 可独立验证 |
|---|---|---|---|
| PR1 | Git 锁改单 ref + CAS + 冲突重试 | `adapters/Git.ts` | 双客户端同时同步不丢更新 |
| PR2 | 墓碑格式 + 序列化/反序列化 + 合并应用 + 迁移 | `serializers/*`, `strategies/*`, `Tree.ts` | 删除在 cache 全新客户端上也生效、不复活 |
| PR3 | 事务化 `writeSnapshot(expectedRevision)` + 本地执行顺序调整 | `adapters/*`, `Account.ts`, `strategies/*` | push 冲突后重试收敛，无半同步状态 |
| PR4 | reorder 降噪 / 稳定排序键 | `Diff.ts`, `strategies/*` | 纯重排不再产生提交 |
| PR5（可选） | op-log / CRDT | 新增 | 长期演进 |

PR1 与 PR2 互不依赖，可并行；PR3 依赖 4.1 的契约。

## 6. 测试策略

- 复用 `test/` 与 `adapters/Fake.ts` / `FakeNcBookmarks.ts`；
- 新增「双客户端并发」模拟：两个独立 cache/adapter 实例，随机 add/delete/move，交错同步；
  - 断言：最终一致、删除不复活、无重复；
- Git adapter 集成测试：本地 bare repo + isomorphic-git，模拟 push 冲突/中断；
- 中断测试：在 commit 后、push 前注入失败，断言重试后状态正确；
- 属性测试：随机操作序列 + 随机同步顺序 → 不变量（删除墓碑存在则不复活）。

## 7. 风险与兼容

- 文件格式变更：旧客户端读到带墓碑的文件需能忽略未知字段（现有解析器需验证）；
- 墓碑 GC 误删：保守保留期 + 可配置；
- 锁饥饿/重试放大：有界退避，必要时提示用户；
- 与上游 divergence：fork 维护成本，尽量把改动做成可上游的独立 PR；
- 性能：整文件读写仍是 O(n)，墓碑会增大文件；后续可考虑分片。

## 8. 立即可做的第一步（PR1）

把 `Git.ts` 的锁替换为单 ref CAS：

- `obtainLock()`：尝试以非 force 方式创建固定锁 ref；失败则读锁内容判断是否过期；
- `setLock()`：只创建一次固定 ref，携带 owner/timestamp/oid；
- `freeLock()`：以 old-oid 为条件删除（CAS），避免删掉别人的新锁；
- `onSyncStart/onSyncComplete` 的 push 冲突：捕获后释放锁、重新 fetch、重算、有限次重试；
- 增加单元测试：两个客户端同时 obtainLock，只有一个成功。

