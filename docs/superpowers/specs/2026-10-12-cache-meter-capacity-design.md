# CacheMeter 容量可配置、水位淘汰与可观测性 — 设计

- Issue: #1（`CacheMeter` 的 `DEFAULT_CAPACITY=4096` 在共享部署下静默压缩 prompt cache 有效 TTL）
- 分支: `feat/cache-meter-capacity-20261011-233108`
- 涉及文件: `src/anthropic/cache_metering.rs`、`src/model/config.rs`、`src/main.rs`、`src/admin/service.rs`、`src/admin/types.rs`、`README.md`、`config.example.json`

## 1. 问题陈述

`CacheMeter` 是一张定容 LRU 表，`DEFAULT_CAPACITY = 4096` 是写死的常量。写入速率为 `W` 条/秒时，一条 entry 的实际存活期是

```
T_eff = min(C / W, TTL)
```

`C < W × TTL` 时 TTL 完全失效，命中率由容量决定。上游每请求最多写 `MAX_BREAKPOINTS = 4` 条，`TTL = 300s`，所以 `R > 4096 / (4 × 300) ≈ 3.4 req/s` 的共享实例 `cache_read` 就开始无声衰减。

三件事互相锁死，必须一起改：

1. 容量不可配置且太小 → 有效 TTL 被压缩。
2. `enforce_capacity()` 每次溢出都做全表 `collect + sort_by_key`（`O(n log n)`），且由 `record_entry()` 在持锁状态下每断点调用一次 → 容量一旦调大就成为吞吐瓶颈，于是不敢调大。
3. miss 完全静默，无 `entries_len` / 淘汰计数 / 过期计数 → 现象无法归因。

附带第 4 件：`flush_to_disk()` 每 60 秒全量 JSON 重写，容量调大后 I/O 线性放大。

## 2. 目标与非目标

**目标**

- G1 容量可由 `config.json` + 环境变量配置，默认值提高到足以覆盖常见共享负载；选型公式写进 README。
- G2 容量淘汰从「每次插入全表排序」改为「越过硬顶才批量清到水位」，单次 `O(n)`，摊还到每次插入为 `O(1)`。
- G3 miss 可归因：五种原因可区分；`entries_len` / 淘汰 / 过期计数周期性输出并在 Admin API 暴露。
- G4 落盘可关闭，并降低频率。

**非目标**

- 不改哈希/断点/分摊算法（`extract_blocks`、`resolve_breakpoints`、`split_against_total` 原样保留）。
- 不改 Redis 远端共享层的语义（`RemoteCacheStore` 接口不动）。
- 不引入新的 LRU 依赖 crate（`HashMap` + 批量水位淘汰已足够，见 §4.3 的取舍）。
- 不做 Prometheus / metrics exporter；本轮只做日志 + 已有 Admin API。

## 3. 配置

### 3.1 新增配置项

`Config`（`src/model/config.rs`，`serde(rename_all = "camelCase")`）新增四项，全部 `Option`，`skip_serializing_if = "Option::is_none"`，不写就不出现在 `config.json` 里：

| 字段（Rust） | config.json | 环境变量 | 类型 | 默认 |
|---|---|---|---|---|
| `cache_metering_capacity` | `cacheMeteringCapacity` | `KIRO_RS_CACHE_CAPACITY` | `usize` | `262144` |
| `cache_metering_persist_enabled` | `cacheMeteringPersistEnabled` | `KIRO_RS_CACHE_PERSIST` | `bool` | `false` |
| `cache_metering_flush_interval_secs` | `cacheMeteringFlushIntervalSecs` | `KIRO_RS_CACHE_FLUSH_INTERVAL_SECS` | `u64` | `300` |
| `cache_metering_miss_diagnostics` | `cacheMeteringMissDiagnostics` | `KIRO_RS_CACHE_MISS_DIAGNOSTICS` | `bool` | `true` |

### 3.2 优先级

与已有 `cache_metering_enabled` 完全一致，不另立规矩：

```
config.json 显式值  >  环境变量  >  内置默认
```

环境变量只作为「配置文件未表态」时的初值（适合全新 docker 部署）。布尔类环境变量复用 `metering_enabled_from_env` 的解析口径（`0/off/false/no/disabled` 为假，其余非空为真，trim + 大小写不敏感，空串视为未表态）。数值类新增 `usize_from_env(name)` / `u64_from_env(name)`：解析失败返回 `None` 并 `warn!` 一次，**不 panic、不中止启动**。

解析与回落在 `main.rs` 启动时完成一次，算出的最终值通过 builder 注入 `CacheMeter`，与现有 `.with_enabled(...)` 并列。

### 3.3 校验与钳制

| 项 | 规则 | 越界处理 |
|---|---|---|
| `capacity` | 钳到 `[1_024, 8_388_608]` | 钳制 + `warn!` 打出原值与生效值 |
| `flush_interval_secs` | 钳到 `[10, 3_600]` | 同上 |
| `capacity` 解析为 0 / 负数 / 非数字 | 视为未表态 | `warn!` 后回落到下一优先级 |

上界 `8_388_608` 的依据：单条 `CacheEntry` 为 `u32 + 3×i64 = 28` 字节，加 `u64` key 与 `HashMap` 槽位开销，按 **64 字节/条** 估算，上界约 512MB —— 这是「再大就该上 Redis 而不是堆内存」的界线，不是推荐值。

默认 `262144` 的依据（同一套公式，README 里要写）：

```
C >= peak_req_per_sec × entries_per_req × TTL_secs
```

上游 `entries_per_req <= MAX_BREAKPOINTS = 4`、`TTL = 300`，则 `262144` 支撑 `262144 / (4 × 300) ≈ 218 req/s`，内存约 **16–24MB**。相比旧默认 `4096`（≈3.4 req/s）提高 64 倍，对单实例共享部署留足余量。改过切段策略、`entries_per_req` 远大于 4 的部署，按公式自行调高。

### 3.4 运行时可变性

容量与落盘开关**在启动时固定，运行时不可改**，与 `enabled` 的 `AtomicBool` 不同。理由：改容量会立即触发一次大规模淘汰，让 Admin API 能触发这件事只会增加故障面，而容量本身不是需要热调的旋钮。Admin API 只**只读**暴露当前生效值（§6）。改容量 = 改 `config.json` + 重启，README 里写明。

## 4. 淘汰

### 4.1 `CacheMeter` 结构变化

```rust
pub struct CacheMeter {
    inner: Mutex<Inner>,
    persist_path: Option<PathBuf>,   // persist 关闭时置 None（见 §5）
    remote: Option<Arc<dyn RemoteCacheStore>>,
    enabled: AtomicBool,
    capacity: usize,                 // 生效容量，构造后不变
    miss_diagnostics: bool,          // 构造后不变
}

struct Inner {
    entries: HashMap<u64, CacheEntry>,
    dirty: bool,
    tombstones: HashMap<u64, TombstoneKind>,  // §5.2
    stats: Stats,                             // §5.3
}
```

`DEFAULT_CAPACITY` 常量保留，仅作为「未配置时的默认值」，不再被 `enforce_capacity` 直接引用。

### 4.2 水位淘汰算法

两个阈值，命名写进常量：

```
HARD_RATIO = 1.2   // record_entry 的强制兜底线：len > capacity * 6 / 5 才动手
LOW_RATIO  = 0.9   // 一次清到的目标水位：capacity * 9 / 10
```

`record_entry()`（持锁、每断点一次）：

```rust
if inner.entries.len() > self.capacity.saturating_mul(6) / 5 {
    self.shrink_to_watermark(&mut inner);
}
```

后台任务（每轮，见 §5.1）在 `evict_expired()` 之后：

```rust
if inner.entries.len() > self.capacity {
    self.shrink_to_watermark(&mut inner);
}
```

`shrink_to_watermark()`：

```rust
let target = self.capacity * 9 / 10;
let len = inner.entries.len();
if len <= target { return; }
let drop_n = len - target;
let mut victims: Vec<(u64, i64)> = inner.entries.iter().map(|(k, v)| (*k, v.last_hit_at)).collect();
victims.select_nth_unstable_by_key(drop_n - 1, |x| x.1);   // O(n)，不是 O(n log n)
for (k, _) in victims.into_iter().take(drop_n) {
    inner.entries.remove(&k);
    note_tombstone(&mut inner, k, TombstoneKind::Evicted);
}
inner.stats.evicted_total += drop_n as u64;
inner.dirty = true;
```

要点：

- `select_nth_unstable_by_key(drop_n - 1, ..)` 把 `last_hit_at` 最小的 `drop_n` 条划到前半段（顺序不定、不需要有序），`O(n)` 期望时间取代 `O(n log n)` 排序。`drop_n >= 1` 由上面的 `len > target` 早退保证。
- `last_hit_at` 相等时谁被淘汰不确定 —— LRU 本就是近似，接受。
- **主回收交给后台 + 兜底在 `record_entry`**（issue 建议 2 的两个选项都要）：稳态下表长在 `0.9C ~ 1.0C` 之间由每 60 秒一轮的后台任务维持，`record_entry` 的 `1.2C` 兜底只在突发写入速率超过后台回收速度时触发。
- 摊还复杂度：两次强制淘汰之间至少插入 `0.3 × capacity` 条，单次代价 `O(n)`，故每次插入摊还 `O(1)`，且常数远小于原来「满表后每次插入都排序」。
- `record_entry` 的热路径只多出一次 `len()` 比较（`HashMap::len` 是 `O(1)`），锁持有时间不变。

### 4.3 为什么不换 LRU crate

`lru::LruCache` 能给出 `O(1)` 精确淘汰，但：entry 还要按 `expires_at` 做 TTL 清理（LRU 链表帮不上），落盘要整表序列化，远端回填路径要 `get_mut` 续期 —— 换完仍然要维护一张并行结构，净收益为负。保留 `HashMap` + 批量水位淘汰。

## 5. 可观测性与落盘

### 5.1 后台任务

`spawn_background()` 的周期从写死 60 秒改为 `max(60, flush_interval_secs)` 的 gcd 式单周期简化为：**固定 60 秒一轮做 `evict_expired` + 水位检查 + 指标输出；落盘按 `flush_interval_secs` 独立计时**（记录上次落盘时间，到点才 `flush_to_disk()`）。这样过期清理与指标输出的及时性不被落盘降频拖累。

每轮输出（仅当本分钟有非零增量时，`info!`，避免空转刷屏）：

```
cache_meter stats entries_len=... capacity=... hits=.. misses=.. evicted=.. expired=..
  miss_no_entry=.. miss_expired=.. miss_evicted=.. miss_no_breakpoint=.. miss_no_seed=..
```

数值为**本周期增量**（`entries_len` / `capacity` 为瞬时值），输出后把增量计数器清零，累计量另存供 Admin API 读取。

### 5.2 miss 归因与其内存上限

五种原因：

| 原因 | 判定点 | 判定依据 |
|---|---|---|
| `miss_no_breakpoint` | `compute_cache_usage` | `resolve_breakpoints` 返回空（无断点 / 超 `MAX_BREAKPOINTS` / 混合 TTL 顺序非法） |
| `miss_no_seed` | `compute_cache_usage` | `isolation_seed` 返回 `None` |
| `miss_expired` | `lookup` | entry 还在表里但 `expires_at <= now`；或命中 `Expired` 墓碑 |
| `miss_evicted` | `lookup` | 命中 `Evicted` 墓碑 |
| `miss_no_entry` | `lookup` | 表里没有、墓碑里也没有 |

`lookup_and_renew` 内部改为返回 `LookupOutcome { Hit(u32), Expired, Evicted, Absent }`；现有 `pub fn lookup_and_renew(&self, hash) -> Option<u32>` 保留为薄封装（测试与远端回填路径不受影响）。

**墓碑（tombstone）是区分 `evicted` / `expired` / `no_entry` 的唯一机制，必须有硬上限**：

- `tombstones: HashMap<u64, TombstoneKind>`，上限 `TOMBSTONE_CAPACITY = 65_536` 条，**不随 `capacity` 变化**。
- 写入时机：`shrink_to_watermark` 删条目写 `Evicted`；`evict_expired` 删条目写 `Expired`；条目被重新 `record_entry` 写入时删除同 key 的墓碑。
- 超限处理：`len() > TOMBSTONE_CAPACITY` 时**整表 `clear()`**，并把 `tombstone_clears` 计数 +1 一并输出。归因在清空后的一小段时间里会把本应 `miss_evicted` 的记成 `miss_no_entry` —— 这是诊断信息，不影响计量正确性，用「整表清空 + 明示清空次数」换「绝不无界增长」。
- 内存上限：`65_536 × (8 + 1)` 字节加 `HashMap` 开销，按 48 字节/条估算 **≈ 3MB 封顶**。
- `cache_metering_miss_diagnostics = false` 时完全不建墓碑表（`miss_evicted` 退化为 `miss_no_entry`），内存归零；计数器与 `entries_len` 仍然输出。

miss 日志：每个断点在 lookback 探测全部落空后打一条 `debug!`，原因取**最深那个 hash**（`check_idx == bp.block_idx`，即第一次探测）的判定结果 —— 它才是本可命中的那一条。单请求最多 4 条 debug 日志。

### 5.3 计数器

`Stats` 放在 `Inner` 里（已在 `Mutex` 保护下，不额外加锁）：`hits`、`misses`、`miss_no_entry`、`miss_expired`、`miss_evicted`、`miss_no_breakpoint`、`miss_no_seed`、`evicted_total`、`expired_total`、`tombstone_clears`，各带一份「本周期增量」与一份「进程累计」。

### 5.4 落盘

- `cache_metering_persist_enabled = false`（新默认）时：`main.rs` 传 `persist_path = None`。于是 **启动不读文件、运行不写文件、退出不写文件，也不删除已有文件**（留给回滚）。启动时若该路径存在，`info!` 一行「持久化已关闭，忽略 `<path>`（<size>）」，避免用户以为文件在生效。
- 为真时：行为与今天一致（启动加载未过期条目、`dirty` 时落盘），但间隔默认 `300s` 而非 `60s`。
- **格式仍为 JSON，不换二进制。** 理由：默认关闭后这条路径不再是热点；换格式要带迁移与兼容分支，收益不抵复杂度。issue 建议 4 的「或降频」分支即此选择，README 里写明。
- 默认值从「总是落盘」翻成「默认不落盘」是行为变更，依据是 issue 的论证：TTL 只有 300 秒，快照只在重启后 5 分钟内有价值，而容量调大后单次全量 JSON 重写代价线性放大。需要原行为的用户显式置 `cacheMeteringPersistEnabled: true` 即可恢复，README 的升级说明里单列一条。

## 6. Admin API

复用已有的 `GET /api/admin/cache-metering`，`CacheMeteringConfigResponse` 从 `{ enabled }` 扩成：

```json
{
  "enabled": true,
  "capacity": 262144,
  "persistEnabled": false,
  "flushIntervalSecs": 300,
  "missDiagnostics": true,
  "stats": {
    "entriesLen": 13421,
    "hits": 90120, "misses": 10233,
    "missNoEntry": 8120, "missExpired": 1900, "missEvicted": 13,
    "missNoBreakpoint": 180, "missNoSeed": 20,
    "evictedTotal": 0, "expiredTotal": 240311, "tombstoneClears": 0
  }
}
```

`stats` 为进程累计值。`PUT` 仍只接受 `enabled`（§3.4）；收到其它字段时忽略并在响应里原样回当前生效值，不报错。`cache_meter` 句柄缺失时 `stats` 为 `null`，配置项回落到 config/env/默认 —— 与现有 `get_cache_metering_config` 的回落口径一致。

## 7. 文档

README 新增「Prompt cache 计量容量选型」小节：公式 `C >= peak_req_per_sec × entries_per_req × TTL_secs`、`T_eff = min(C/W, TTL)` 的解释、默认值对应的 req/s 与内存、四个新配置项与环境变量、`persistEnabled` 默认翻转的升级提示、以及「怎么用日志里的 `miss_evicted` / `entries_len` 判断该调容量」。`config.example.json` 补上四项。

## 8. 验证标准

实现按 TDD：每条先写失败测试再改实现。

**单元测试（`src/anthropic/cache_metering.rs` 的 `mod tests`）**

1. `capacity_from_config_overrides_env`：config 显式值 > 环境变量 > 默认，三层各一例。
2. `capacity_invalid_env_falls_back_with_warning`：`abc` / `0` / 空串 → 回落默认，不 panic。
3. `capacity_out_of_range_is_clamped`：`100` → `1_024`；`99_999_999` → `8_388_608`。
4. `flush_interval_clamped`：`1` → `10`；`99_999` → `3_600`。
5. `record_entry_does_not_evict_below_hard_ratio`：`capacity = 1000`，插 1150 条，`len() == 1150`（未触发强制淘汰）。
6. `record_entry_shrinks_to_watermark_above_hard_ratio`：同上插到 1201 条 → `len() == 900`。
7. `shrink_evicts_least_recently_hit_first`：构造 `last_hit_at` 可区分的条目，断言被留下的是最近命中的那批。
8. `background_sweep_shrinks_above_capacity`：`len` 在 `1.0C~1.2C` 之间时手动调一次后台回收路径 → 降到 `0.9C`。
9. `evict_expired_records_expired_tombstone` / `shrink_records_evicted_tombstone`：随后 lookup 该 hash 分别得到 `Expired` / `Evicted`。
10. `tombstones_are_capped_and_cleared`：制造 > 65_536 次淘汰 → 墓碑表长 `<= 65_536`，`tombstone_clears >= 1`。
11. `miss_diagnostics_disabled_keeps_tombstones_empty`：关闭开关后墓碑恒为空，且 lookup 返回 `Absent`。
12. `lookup_outcome_expired_vs_absent`：表内过期条目 → `Expired`；从未写过 → `Absent`。
13. `compute_cache_usage_counts_miss_no_breakpoint` / `..._no_seed`：构造无断点、无 `metadata.user_id` 的请求，断言对应计数器 +1。
14. `stats_counters_accumulate_on_hit_and_miss`：一次 miss + 一次 hit 后 `hits == 1 && misses == 1`。
15. `persist_disabled_does_not_read_or_write_file`：预置一份快照文件 → 启动后 `len() == 0`，跑完一轮落盘后文件内容字节不变。
16. `persist_enabled_round_trips`：现有落盘/加载测试保持通过（回归）。

**回归**：`src/anthropic/cache_metering.rs` 现有全部测试不得改断言、必须保持通过（`compute_cache_usage_first_miss_then_hit`、`evict_expired_removes_dead_entries` 等）。

**整体**：`cargo test` 全量通过；`cargo clippy -- -D warnings` 无新增告警；`cargo fmt --check` 通过。

**行为验收**（人工，MR 描述里给出）：以 `capacity = 262144` 启动，日志出现 `cache_meter stats` 行且 `entries_len` 随负载上升、`evicted=0`；把 `capacity` 调到 `2048` 复跑同一负载，`miss_evicted` 明显非零 —— 证明归因机制确实指向容量。
