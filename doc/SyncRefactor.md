# 同步模型重构设计：从「单文件快照 + 本地缓存推断」到「带状态的条件写」

状态：草案 v2（经 subagent 代码级评审后修订）
分支：`refactor/sync-atomicity`
范围：`src/lib/Account.ts`、`src/lib/adapters/Git.ts`、`src/lib/strategies/*`、`src/lib/serializers/*`、缓存/映射层
目标后端：以 Git/WebDAV/S3 这类「文件快照」后端为主，同时不破坏 Nextcloud Bookmarks 等 atomic 后端

## 0. v2 修订摘要（评审结论）

v1 的方向成立，但有一处根因判断错误，且 PR 顺序把最严重的风险留在最后。v2 的修正：

1. **第一根因不是「push 失败后 cache 已推进」，而是「同步失败会把 cache/mappings 整体清空」**。
   `Account.ts` 的 `keepsContinuation` 白名单只含 InterruptedSyncError / NetworkError / LocalFolderNotFoundError / 四个 failsafe；
   E026（取消）、E037（锁/push 被拒）、E044（GitPushError）、E060（S3 条件写冲突）、解析失败等都会走到
   `clearContinuation() + init()`，而 `BrowserAccount.init()` 无条件 `initMappings() + initCache()`，把
   `bookmarks[<id>].mappings` 和 `bookmarks[<id>].cache` 写成空。
   下一轮同步 cache 为空 → 两侧都把对方条目当 `CREATE` → **删除的复活、已有的重复**。
   这比「基准错误」更能解释「整份文件回到过去某个状态」。
2. **锁只应作为 best-effort 租约**，正确性交给 `writeSnapshot(expectedRevision)` 的 CAS；不要让锁成为新的失败源。
3. **墓碑要排在事务化（CAS + 提交顺序）之后**，且优先用「git 历史推导」或「独立墓碑文件」，不要内嵌进整份 XBEL（旧客户端/其它程序重写会把它抹掉）。
4. **PR 顺序**：P0 错误分类 → P1 CAS → P2 提交顺序 → P3 墓碑 → P4 reorder → P5 可选。
5. `0/642=0` 与 `No changes...` 是「已收敛到（被复活的）状态」的结果，不能单独当作「客户端空转」的证据。

## 1. 问题陈述

多端（本机 3 个浏览器 + 另一台电脑）同时对同一个 Gitee 仓库读写时，反复出现：

- 删除的书签过一段时间「复活」；
- 批量条目被删掉后又补回（回退）；
- 长时间没有任何新提交，但本地看起来在同步。

历史证据（`gitee.com/xiangagou/bookmark`）：

- 555 次提交中，38 次单次净减少 ≥3 条，其中 27 次之后总量又回到原水平；
- 7 份完全相同的文件内容在不同提交里重复出现（整份文件回到过去的状态）；
- 《Quick》在 11-17、11-18、11-19 被删三次、复活三次；《Atlassian》11-19 09:44 删除、19:41 复活。

## 2. 根因（代码级，已核对）

### 2.1 删除不可见：文件里没有墓碑

`src/lib/serializers/Xbel.ts` / `Html.ts` 只序列化「当前存在的树」（只输出 `bookmark` / `folder`），
没有删除标记、版本号或时间戳。删除一条书签 = 它从文件里消失；服务器无法区分「被删除」和「从未存在」。
解析器会忽略未知元素（`Xbel.ts:78-80`），但**序列化时会丢弃**，所以任何旧客户端或手工重写都会抹掉墓碑。

### 2.2 合并基准在各端本地，cache 是唯一的共同祖先

`Default.getDiffs()`（`Default.ts:1200`）以本地 `cacheTree` 为 base：

- `localScanner`：cache ↔ 本地
- `serverScanner`：cache ↔ 服务器

`Scanner.diffFolder()` 把「cache 里有、新树里没有」的子项记为 `REMOVE`（`Scanner.ts:136`），
「新树里有、cache 里没有」记为 `CREATE`（`Scanner.ts:144-147`）。
因此删除的传播完全依赖每个客户端的 `bookmarks[<accountId>].cache` 与 mappings。cache 一丢，删除意图就没了。

### 2.3 Git 锁不是互斥

`src/lib/adapters/Git.ts`：

- `obtainLock()` 只是查询远端是否存在「新鲜的」`floccus-lock-*` tag；
- `setLock()` 各自 push 一个 `floccus-lock-<Date.now()>` 的**不同** tag。

多个客户端在彼此看不到锁时会各自创建自己的 tag，都能认为自己拿到了锁——没有在同一个 ref 上做 CAS。
另外 `forceLock`（`Account.ts:286-291`，超过 2h 或用户强制同步）会调用 `clearAllLocks()` 删除**所有** floccus 锁，
这本身也是一个并发写来源。

### 2.4 【第一根因】同步失败会清空 cache/mappings

`Account.ts:511-521` 的 `keepsContinuation` 白名单只包含：

- `InterruptedSyncError`
- `NetworkError`
- `LocalFolderNotFoundError`
- 四个 failsafe 错误

E026（取消）、E037（ResourceLockedError / push 被拒）、E044（GitPushError）、E060（S3 条件写冲突）、
文件不可读/解析失败等都不在白名单。结果：

```
keepsContinuation = false
  → clearContinuation()
  → init()
  → storage.initMappings() + storage.initCache()   // BrowserAccount.ts:57-58
  → cache/mappings 变为 {}
```

下一轮同步 cache 为空 → `Scanner` 对每个本地条目和每个服务器条目都产出 `CREATE`；
`reconcileDiffs` 的并发创建配对只按 `canMergeWith`（书签比 URL、文件夹比标题）重新配对，
净效果是**只在一侧存在的条目被重新创建**——别处删掉的复活，本地删掉的回到本地。

日常可触发的例子：

- 用户点「停止同步」或 background 页重载 → `CancelledSyncError`(E026) → 清空；
- `Git.onSyncComplete` push 被拒 → `ResourceLockedError`(E037) → 清空；
- `git.push` 其它失败 → `GitPushError`(E044) → 清空；
- `git.fetch` 网络异常（非 FloccusError）→ 清空；
- 远端文件被改名/删除 → `pullFromServer` 返回 false → `init()` → 清空。

补充：Git 的 `isAtomic()` 继承 `CachingAdapter` 的 `true`（`Caching.ts:381`，Git 未覆写），
所以 `persistProgress` 在 `Account.ts:779-788` 直接 return——同步中途不写 cache。
失败后的真实后果是「cache 被销毁」，不是「cache 已推进」。

### 2.5 读-改-写-推不是原子的

`onSyncStart()` fetch → 本地执行计划 → `onSyncComplete()` commit + push。
push 失败抛 `GitPushError`，而按 2.4 会清空 cache。且 `Mappings` 在服务端计划执行时就被改写（`Default.ts:1763-1767`），
没有回滚机制；`highestId`、`alwaysUpload`、`initialTreeHash` 也都随执行推进。

### 2.6 设计取向

`CONSIDERATIONS.md` 明确：目标优先级是「最终一致 > 保留意图」，且 WebExtensions 书签 API 没有事务/CAS 原语。
这是设计选择，不是单纯 bug；但在多写端场景下会放大成用户可见的回退。

## 3. 目标与非目标

目标：

- 多端并发写：不丢更新、不复活已删除条目；
- 同步失败/中断不清空 cache/mappings，不产生「删了又回来」；
- 文件后端（Git/WebDAV/S3）在单写与多写下都可用；
- 保持现有合并引擎（`Scanner`/`Diff`/`strategies`）与后端兼容。

非目标：

- 重写整个 merge 引擎；
- 引入强依赖的中央服务；
- 一次做完 op-log/CRDT（作为后续可选阶段）。

## 4. 目标架构

### 4.1 存储契约：revision + 条件写（CAS）

参考 S3 adapter 已有的完整先例（`S3.ts:39-43, 301-330, 337-375, 546-580`：`If-Match` / `If-None-Match`、
`conditionalWritesUnsupported` 降级），抽象为：

```ts
interface IVersionedStorage {
  readSnapshot(): Promise<{ tree: Folder<Server>; revision: string }>
  writeSnapshot(tree: Folder<Server>, expectedRevision: string): Promise<{ revision: string }>
}
```

- Git：`revision` = fetch 后 `origin/<branch>` 的 oid；写入 = commit(parent = expectedRevision) + 非 force push；
  push 前/后各校验一次远端 head；冲突抛 `TransientError`（不要复用 E037）。
- WebDAV：ETag/`If-Match`；很多服务器是弱 ETag 或不支持，需要能力探测 + 降级。
- S3：复用现有实现。
- Nextcloud Bookmarks：已有逐条语义，走现有 atomic 路径。

### 4.2 Git 锁：best-effort 租约（不承担正确性）

- 固定 ref（建议 `refs/heads/floccus-lock`，而不是 tag：tag 的非 fast-forward 更新在部分服务器上会被允许）；
- 获取 = 以「不存在」为前提创建；租约带 owner/timestamp/oid；
- 过期接管与清理都要能证明没有误删别人的新锁；`forceLock` 不再 `clearAllLocks()`；
- 过渡期同时识别旧的 `floccus-lock-*` tag，避免混合版本完全失去互斥；
- 获取失败 → 有界退避；**正确性由 4.1 的 CAS 保证，锁只减少无用功**。

### 4.3 墓碑（tombstones）——放在 CAS/提交顺序之后

优先方案（Git 特化，不改文件格式）：

- 客户端记住 `lastRevision`；
- adapter 提供 `getTombstonesSince(revision)`，通过 `git log` / 对比 `revision..HEAD` 的两棵树得到「被删除的 server id 集合」；
- 优点：旧客户端抹不掉、GC 由保留深度控制、无需改格式；
- 代价：浅 fetch 深度要覆盖最老的 `lastRevision`（或按 revision 定点 fetch），且只适用于 Git。

备选方案（文件内墓碑）：

- 放**独立文件** `bookmarks.tombstones`（旧客户端不会重写它），而不是内嵌 XBEL；
- 两次 CAS 写：先写树、再写墓碑（顺序反了会在失败时误删）；
- 匹配键：server id 优先，`url` + 父路径兜底；新 id 不被墓碑压制（重新添加语义）；
- 只对「真正执行过的 server REMOVE」立墓碑，排除 `filterOutInvalidBookmarks` / `filterUnacceptedBookmarks` 与 Unidirectional overwrite；
- 文件夹墓碑要覆盖子树；
- GC 默认不启用（642 条规模可接受），或保留期 > 最老设备离线时间；
- 把「墓碑是否变化」计入 `Git.ts:214` 的写入门槛。

实现落点：做成 adapter 能力 + 扫描结果后置过滤，不改 `Tree.ts`/`Scanner.ts` 核心逻辑，便于回退与上游。

### 4.4 提交顺序：服务端先落地

需要四处改动（v1 低估了）：

1. `IAdapter` 拆出 `onSyncCommit()`（写文件+push）与 `onSyncComplete()`（清理/解锁），
   在 server stage3 之后由策略调用（`Default.ts:824-830`）；
2. `Mappings` 需要快照/回滚（`Mappings.ts:105-110` 目前直接改），`highestId`/`alwaysUpload`/`initialTreeHash` 同理；
3. continuation 记录 `committedRevision`，恢复时跳过已提交的 server 计划，避免重复创建；
4. 本地计划失败时 cache 要写「服务端树」而非「本地树」——需要给 `ICacheStore`/`CachingTreeWrapper`
   一个新的「只合并服务端侧变更」入口。

同时要决定 Git 是否从 atomic 改为 non-atomic（影响 continuation 持久化与 #2362 风险）。

### 4.5 顺序（reorder）降噪

顺序是用户语义（`preserveOrder: true`），不能简单忽略。可行做法：

- 纯 REORDER 单独成低优先级提交，减少与内容变更混在一起；
- 双方都改顺序时取一方（已有 `reconcileConcurrentReorderings`，`Default.ts:907-916`）；
- 引入稳定排序键作为兜底。

## 5. 分阶段实施（修订后的 PR 顺序）

| 序 | 内容 | 影响面 | 直接解决 |
|---|---|---|---|
| **P0** | 错误分类：E026/E037/E044/E060/网络/解析失败一律 transient 且 `keepsContinuation`；`init()` 只在显式重置时调用 | `Account.ts:511-521` | **cache 被清空 → 复活/重复（第一根因）** |
| **P1** | `writeSnapshot(expectedRevision)` CAS + 冲突重试 | `adapters/*`, `Account.ts` | 多端并发写丢更新 |
| **P2** | 提交顺序拆分（`onSyncCommit`）+ continuation `committedRevision` + mappings 快照/回滚 | `Account.ts`, `strategies/*`, `interfaces/*` | 中断后的半同步状态 |
| **P3** | 墓碑（git 历史优先 / 独立文件备选） | adapter 能力 + 扫描后置过滤 | 删了又复活 |
| **P4** | reorder 降噪 | `Diff.ts`, `strategies/*` | 纯重排产生提交 |
| P5（可选） | op-log / CRDT | 新增 | 长期演进 |

P0 必须最先做：一行策略级改动，直接止住最大的复活来源，且可独立验证。

## 6. 测试策略

现有测试已覆盖大量同步场景：`atomic_interrupts.test.js`、`continuation_resume.test.js`、`sync_basic*.test.js`、
`sync_advanced.test.js`、`serializers.test.js`、`s3.test.js` 等。补充：

- **P0 回归**：push 被拒/取消/网络失败后，cache/mappings 未被清空，下一轮不产生 CREATE；
- **双客户端并发**：需要一个新的「整文件 + 非 atomic」fake（现有 `Fake` 是 atomic），交错 add/delete/move，
  断言最终一致、删除不复活、无重复；
- **Git 集成**：isomorphic-git 不能对 `file://` push，需要 smart HTTP（`git-http-backend`）或本地 HTTP git 服务；
  扩展现有 Apache 配置（`test/apache-vhost.conf`）；
- **中断测试**：在 commit 后、push 前注入失败，断言重试后状态正确；
- **属性测试**：随机操作序列 + 随机同步顺序 → 不变量（有墓碑则不复活）。

## 7. 风险与兼容

- 格式变更：旧客户端会抹掉内嵌墓碑（解析忽略、序列化丢弃）；因此优先 git 历史方案；
- 墓碑误删：只对真正执行的 server REMOVE 立墓碑；默认不 GC；
- 锁：过期锁会导致长时间不能同步，故降级为 best-effort；
- WebDAV 条件写不可用：探测 + 降级，行为退化为当前水平；
- 上游 divergence：P0/P1 尽量做成可上游的独立 PR；P3 是硬分叉；
- 浏览器自带同步（Firefox Sync / Chrome Sync）可能把删除的条目加回来，需要与 floccus 自身问题区分。

## 8. 立即可做的第一步（P0）

改 `Account.ts` 的错误分类：

- 把 E026/E037/E044/E060/网络/解析失败加入 `keepsContinuation`；
- `init()`（清空 cache/mappings）只在显式「重置」或 `LocalFolderNotFoundError` 时调用；
- 增加回归测试：push 被拒后 cache/mappings 仍在，下一轮不产生 CREATE。

验证方式：在现有 `atomic_interrupts.test.js` / `sync_basic.test.js` 基础上加一个「push 被拒」用例。
