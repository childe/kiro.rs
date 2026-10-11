# CacheMeter 容量可配置、水位淘汰与可观测性 Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** 让 `CacheMeter` 的条目上限可配置并显著提高默认值，把容量淘汰改成 `O(n)` 批量水位淘汰，补齐 miss 归因与周期性指标，并把磁盘快照改为可关闭、可降频。

**Architecture:** 四块互相独立但同属一个结构体：(1) `Config` 新增四个 `Option` 字段 + 对应环境变量，解析/钳制在 `cache_metering.rs` 里做成**纯函数**（不读全局状态，便于测试），`main.rs` 启动时解析一次并通过 builder 注入；(2) `enforce_capacity` 的「每次插入全表排序」换成 `shrink_to_watermark`（`select_nth_unstable_by_key`，`O(n)`），主回收交给 60 秒一轮的后台 `sweep()`，`record_entry` 只保留 `1.2C` 的兜底；(3) `Inner` 里加 `Stats`（本周期增量 + 进程累计）与有硬上限的 `tombstones` 墓碑表，`lookup_and_renew` 内部改为返回 `LookupOutcome`，外层保留原签名；(4) 落盘路径靠 `persist_path: Option<PathBuf>` 置 `None` 关闭，落盘间隔与 60 秒巡检周期解耦。

**Tech Stack:** Rust 2024，`parking_lot::Mutex`，`tokio`，`serde` / `serde_json`，`tracing`，`axum`（Admin API）。测试全部放在 `src/anthropic/cache_metering.rs` 的 `mod tests` 与 `src/admin/service.rs`。

**Spec:** `docs/superpowers/specs/2026-10-12-cache-meter-capacity-design.md`

## Global Constraints

- 新配置字段一律 `Option<T>` + `#[serde(default, skip_serializing_if = "Option::is_none")]`，`Config` 用 `serde(rename_all = "camelCase")`，不写就不出现在 `config.json` 里。
- 生效优先级固定为：`config.json 显式值 > 环境变量 > 内置默认`。
- 环境变量名：`KIRO_RS_CACHE_CAPACITY`、`KIRO_RS_CACHE_PERSIST`、`KIRO_RS_CACHE_FLUSH_INTERVAL_SECS`、`KIRO_RS_CACHE_MISS_DIAGNOSTICS`。
- 布尔环境变量口径：trim + 小写后 `0 / off / false / no / disabled` 为假，其余非空为真，空串视为未表态（`None`）。
- 数值环境变量解析失败、为 `0`、为空串一律视为未表态并 `warn!`，**绝不 panic、绝不中止启动**。
- 钳制区间：`capacity ∈ [1_024, 8_388_608]`，`flush_interval_secs ∈ [10, 3_600]`；越界钳制并 `warn!` 原值与生效值。
- 默认值：`capacity = 262_144`，`persist_enabled = false`，`flush_interval_secs = 300`，`miss_diagnostics = true`。
- `TOMBSTONE_CAPACITY = 65_536`，**不随 capacity 变化**；超限整表 `clear()` 并把 `tombstone_clears` +1。
- 比例常量：强制淘汰触发线 `capacity * 6 / 5`，淘汰目标水位 `capacity * 9 / 10`。
- 容量、落盘开关、落盘间隔、miss 诊断开关**启动时固定，运行时不可改**；`PUT /api/admin/cache-metering` 仍只接受 `enabled`。
- 不改哈希/断点/分摊算法（`extract_blocks`、`resolve_breakpoints`、`split_against_total`、`isolation_seed` 原样保留）。
- 不改 `RemoteCacheStore` trait 语义，不引入新的 LRU crate，不做 Prometheus exporter。
- 磁盘快照格式**仍为 JSON，仍只序列化 `entries`**（`HashMap<u64, CacheEntry>`），墓碑与计数器不落盘。
- `src/anthropic/cache_metering.rs` 现有测试**不得改断言**，必须保持通过。
- 每个任务结束前：`cargo fmt`、`cargo clippy -- -D warnings` 无新增告警、`cargo test` 全绿。
- 本仓库**没有 library target**（`Cargo.toml` 无 `[lib]`、无 `src/lib.rs`），只有 bin `kiro-rs`。所有测试命令一律写成 `cargo test --bin kiro-rs <过滤条件>`；`cargo test --lib` 会直接报 `no library targets found` 而不是跑测试。
- 测试/构建命令**不得接 `| tail`**：管道会把 Cargo 的非零退出码吞成 `tail` 的 0，失败被当成通过。需要截断输出时用 `set -o pipefail`，或先看退出码再截断。
- miss 计数口径（Task 3 定规则、Task 5 接线，两者是同一个验收单元）：**一个断点无论 lookback 探测多少个 hash，只记一次账** —— 命中记 1 次 `hits`，全部落空记 1 次 `misses` 并按最深那个 hash 的 `deepest_reason` 分类。底层探测用不计数的 `probe_outcome`，业务计数走 `note_lookup`。

## Review Focus

1. **旧快照文件兼容**：`persistEnabled: true` 的升级用户，磁盘上是旧格式 `{"<hash>": {...CacheEntry}}`。新增的 `tombstones` / `stats` 一旦混进序列化结构，旧文件会整体加载失败并静默丢掉全部缓存。→ 测试落在 Task 6。
2. **`capacity` 取下限 `1_024` 时的整数边界**：`len == target + 1` 时 `drop_n - 1 == 0`，`select_nth_unstable_by_key(0, ..)` 必须合法；`capacity.saturating_mul(6)` 不得溢出。→ 测试落在 Task 2。
3. **远端 Redis 命中不得被记成 miss**：命中走的是 `remote.lookup_and_renew` + `record_entry` 回填，完全绕开本地 `lookup_outcome`，`hits` 不加、`misses` 不加的话整张报表失真。→ 测试落在 Task 5。
4. **Admin 句柄缺失**：`AdminService` 没注入 `cache_meter` 时（单元测试与部分启动路径就是这样），`stats` 必须是 `null` 且配置项回落到 config/env/默认，不得 panic 或 unwrap。→ 测试落在 Task 7。
5. **墓碑必须被重新写入清除**：条目过期 → 写 `Expired` 墓碑 → 同 hash 被重新 `record_entry`，此后 lookup 必须是 `Hit`，墓碑表里不得残留该 hash（否则这条再被淘汰时归因会串）。→ 测试落在 Task 4。

---

### Task 1: 配置项与解析/钳制纯函数

**Files:**
- Modify: `src/model/config.rs:292`（`cache_metering_enabled` 之后插入四个新字段）、`src/model/config.rs:497`（`Default` 实现里补四项）
- Modify: `src/anthropic/cache_metering.rs:31-33`（常量区）、`src/anthropic/cache_metering.rs:484-501`（`metering_enabled_from_env`）
- Test: `src/anthropic/cache_metering.rs` 的 `mod tests`

**Interfaces:**
- Consumes: 无（第一个任务）。
- Produces:
  - 常量 `DEFAULT_CAPACITY: usize = 262_144`、`MIN_CAPACITY`、`MAX_CAPACITY`、`DEFAULT_FLUSH_INTERVAL_SECS: u64`、`MIN_FLUSH_INTERVAL_SECS`、`MAX_FLUSH_INTERVAL_SECS`
  - `CacheMeter::resolve_capacity(from_config: Option<usize>, from_env: Option<usize>) -> usize`
  - `CacheMeter::resolve_flush_interval_secs(from_config: Option<u64>, from_env: Option<u64>) -> u64`
  - `CacheMeter::capacity_from_env() -> Option<usize>`
  - `CacheMeter::flush_interval_from_env() -> Option<u64>`
  - `CacheMeter::persist_enabled_from_env() -> Option<bool>`
  - `CacheMeter::miss_diagnostics_from_env() -> Option<bool>`
  - `Config` 字段 `cache_metering_capacity: Option<usize>`、`cache_metering_persist_enabled: Option<bool>`、`cache_metering_flush_interval_secs: Option<u64>`、`cache_metering_miss_diagnostics: Option<bool>`

- [ ] **Step 1: 写失败测试（解析与钳制）**

加到 `src/anthropic/cache_metering.rs` 的 `mod tests` 末尾：

```rust
    /// 测试内改环境变量会互相干扰，所有碰 env 的测试共用这把锁。
    static ENV_GUARD: std::sync::Mutex<()> = std::sync::Mutex::new(());

    #[test]
    fn capacity_from_config_overrides_env() {
        // config 显式值 > 环境变量 > 内置默认
        assert_eq!(CacheMeter::resolve_capacity(Some(8_192), Some(4_096)), 8_192);
        assert_eq!(CacheMeter::resolve_capacity(None, Some(4_096)), 4_096);
        assert_eq!(CacheMeter::resolve_capacity(None, None), DEFAULT_CAPACITY);
        assert_eq!(DEFAULT_CAPACITY, 262_144, "默认容量提高到 262144");
    }

    #[test]
    fn capacity_invalid_env_falls_back_with_warning() {
        let _guard = ENV_GUARD.lock().unwrap_or_else(|e| e.into_inner());
        for bad in ["abc", "0", "   ", "-1", "1.5"] {
            unsafe {
                std::env::set_var("KIRO_RS_CACHE_CAPACITY", bad);
            }
            assert_eq!(
                CacheMeter::capacity_from_env(),
                None,
                "{bad:?} 应被视为未表态而不是 panic"
            );
        }
        unsafe {
            std::env::set_var("KIRO_RS_CACHE_CAPACITY", "20000");
        }
        assert_eq!(CacheMeter::capacity_from_env(), Some(20_000));
        unsafe {
            std::env::remove_var("KIRO_RS_CACHE_CAPACITY");
        }
        assert_eq!(CacheMeter::capacity_from_env(), None, "未设置时不表态");
        // 未表态时回落默认
        assert_eq!(
            CacheMeter::resolve_capacity(None, CacheMeter::capacity_from_env()),
            DEFAULT_CAPACITY
        );
    }

    #[test]
    fn capacity_out_of_range_is_clamped() {
        assert_eq!(CacheMeter::resolve_capacity(Some(100), None), 1_024);
        assert_eq!(CacheMeter::resolve_capacity(Some(99_999_999), None), 8_388_608);
        assert_eq!(CacheMeter::resolve_capacity(None, Some(100)), 1_024);
        assert_eq!(CacheMeter::resolve_capacity(Some(65_536), None), 65_536);
    }

    #[test]
    fn flush_interval_clamped() {
        assert_eq!(CacheMeter::resolve_flush_interval_secs(Some(1), None), 10);
        assert_eq!(CacheMeter::resolve_flush_interval_secs(Some(99_999), None), 3_600);
        assert_eq!(CacheMeter::resolve_flush_interval_secs(None, Some(600)), 600);
        assert_eq!(
            CacheMeter::resolve_flush_interval_secs(None, None),
            DEFAULT_FLUSH_INTERVAL_SECS
        );
        assert_eq!(DEFAULT_FLUSH_INTERVAL_SECS, 300);
    }

    #[test]
    fn persist_and_diagnostics_env_parsing() {
        let _guard = ENV_GUARD.lock().unwrap_or_else(|e| e.into_inner());
        unsafe {
            std::env::set_var("KIRO_RS_CACHE_PERSIST", "TRUE");
            std::env::set_var("KIRO_RS_CACHE_MISS_DIAGNOSTICS", " off ");
        }
        assert_eq!(CacheMeter::persist_enabled_from_env(), Some(true));
        assert_eq!(CacheMeter::miss_diagnostics_from_env(), Some(false));
        unsafe {
            std::env::remove_var("KIRO_RS_CACHE_PERSIST");
            std::env::remove_var("KIRO_RS_CACHE_MISS_DIAGNOSTICS");
        }
        assert_eq!(CacheMeter::persist_enabled_from_env(), None);
        assert_eq!(CacheMeter::miss_diagnostics_from_env(), None);
    }
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cargo test --bin kiro-rs cache_metering::tests::capacity_`
Expected: 编译失败，`no function or associated item named 'resolve_capacity' found`。

- [ ] **Step 3: 实现常量与解析函数**

把 `src/anthropic/cache_metering.rs` 第 31-33 行的常量块：

```rust
/// 默认条目上限（防止内存无限增长）
const DEFAULT_CAPACITY: usize = 4096;
```

替换为：

```rust
/// 未配置时的默认条目上限。
///
/// 选型公式：`C >= peak_req_per_sec * entries_per_req * TTL_secs`。
/// 上游 `entries_per_req <= MAX_BREAKPOINTS = 4`、默认 `TTL = 300s`，
/// 262144 支撑约 `262144 / (4 * 300) ≈ 218 req/s`，内存约 16–24MB。
const DEFAULT_CAPACITY: usize = 262_144;
/// 容量下限（再小就不如直接关掉计量）
const MIN_CAPACITY: usize = 1_024;
/// 容量上限：按 64 字节/条估算约 512MB，是「再大就该上 Redis」的界线，不是推荐值
const MAX_CAPACITY: usize = 8_388_608;
/// 默认落盘间隔（秒）
const DEFAULT_FLUSH_INTERVAL_SECS: u64 = 300;
/// 落盘间隔下限（秒）
const MIN_FLUSH_INTERVAL_SECS: u64 = 10;
/// 落盘间隔上限（秒）
const MAX_FLUSH_INTERVAL_SECS: u64 = 3_600;
```

在 `fn now_secs()` 之前（文件内 `impl CacheMeter` 块之外）加三个自由函数：

```rust
/// 解析布尔型环境变量：`0/off/false/no/disabled` 为假，其余非空为真，空串/未设置返回 None。
fn bool_from_env(name: &str) -> Option<bool> {
    let raw = std::env::var(name).ok()?;
    let raw = raw.trim().to_ascii_lowercase();
    if raw.is_empty() {
        return None;
    }
    Some(!matches!(
        raw.as_str(),
        "0" | "off" | "false" | "no" | "disabled"
    ))
}

/// 解析 usize 型环境变量。解析失败、为 0、为空串一律视为「未表态」并 warn，不 panic。
fn usize_from_env(name: &str) -> Option<usize> {
    let raw = std::env::var(name).ok()?;
    let raw = raw.trim().to_string();
    if raw.is_empty() {
        return None;
    }
    match raw.parse::<usize>() {
        Ok(0) => {
            tracing::warn!("环境变量 {}=0 无效，已忽略并回落到默认值", name);
            None
        }
        Ok(v) => Some(v),
        Err(_) => {
            tracing::warn!("环境变量 {}={:?} 不是合法正整数，已忽略", name, raw);
            None
        }
    }
}

/// 解析 u64 型环境变量，口径与 `usize_from_env` 一致。
fn u64_from_env(name: &str) -> Option<u64> {
    usize_from_env(name).map(|v| v as u64)
}
```

把 `metering_enabled_from_env` 的函数体（保留原有文档注释）改为复用 `bool_from_env`：

```rust
    pub fn metering_enabled_from_env() -> Option<bool> {
        bool_from_env("KIRO_RS_CACHE_METERING")
    }
```

紧随其后，在同一个 `impl CacheMeter` 块里加：

```rust
    /// 读取 `KIRO_RS_CACHE_CAPACITY` 表达的容量意图。
    pub fn capacity_from_env() -> Option<usize> {
        usize_from_env("KIRO_RS_CACHE_CAPACITY")
    }

    /// 读取 `KIRO_RS_CACHE_FLUSH_INTERVAL_SECS` 表达的落盘间隔意图。
    pub fn flush_interval_from_env() -> Option<u64> {
        u64_from_env("KIRO_RS_CACHE_FLUSH_INTERVAL_SECS")
    }

    /// 读取 `KIRO_RS_CACHE_PERSIST` 表达的落盘开关意图。
    pub fn persist_enabled_from_env() -> Option<bool> {
        bool_from_env("KIRO_RS_CACHE_PERSIST")
    }

    /// 读取 `KIRO_RS_CACHE_MISS_DIAGNOSTICS` 表达的 miss 归因开关意图。
    pub fn miss_diagnostics_from_env() -> Option<bool> {
        bool_from_env("KIRO_RS_CACHE_MISS_DIAGNOSTICS")
    }

    /// 按 `config.json > 环境变量 > 内置默认` 定出生效容量，并钳到
    /// `[MIN_CAPACITY, MAX_CAPACITY]`。越界时 warn 出原值与生效值。
    pub fn resolve_capacity(from_config: Option<usize>, from_env: Option<usize>) -> usize {
        let raw = from_config
            .filter(|v| *v > 0)
            .or_else(|| from_env.filter(|v| *v > 0))
            .unwrap_or(DEFAULT_CAPACITY);
        let clamped = raw.clamp(MIN_CAPACITY, MAX_CAPACITY);
        if clamped != raw {
            tracing::warn!(
                "cacheMeteringCapacity={} 越界 [{}, {}]，已钳制为 {}",
                raw,
                MIN_CAPACITY,
                MAX_CAPACITY,
                clamped
            );
        }
        clamped
    }

    /// 按 `config.json > 环境变量 > 内置默认` 定出生效落盘间隔，并钳到
    /// `[MIN_FLUSH_INTERVAL_SECS, MAX_FLUSH_INTERVAL_SECS]`。
    pub fn resolve_flush_interval_secs(from_config: Option<u64>, from_env: Option<u64>) -> u64 {
        let raw = from_config
            .filter(|v| *v > 0)
            .or_else(|| from_env.filter(|v| *v > 0))
            .unwrap_or(DEFAULT_FLUSH_INTERVAL_SECS);
        let clamped = raw.clamp(MIN_FLUSH_INTERVAL_SECS, MAX_FLUSH_INTERVAL_SECS);
        if clamped != raw {
            tracing::warn!(
                "cacheMeteringFlushIntervalSecs={} 越界 [{}, {}]，已钳制为 {}",
                raw,
                MIN_FLUSH_INTERVAL_SECS,
                MAX_FLUSH_INTERVAL_SECS,
                clamped
            );
        }
        clamped
    }
```

- [ ] **Step 4: 加 Config 字段**

`src/model/config.rs`，在 `pub cache_metering_enabled: Option<bool>,`（第 292 行）之后插入：

```rust
    /// prompt cache 计量模拟的条目上限。未设置时回落到 `KIRO_RS_CACHE_CAPACITY`
    /// 环境变量，再回落到内置默认 262144。
    ///
    /// 选型公式：`C >= peak_req_per_sec * entries_per_req * TTL_secs`。
    /// 生效值钳到 `[1024, 8388608]`。启动时固定，运行时不可改。
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub cache_metering_capacity: Option<usize>,

    /// 是否把 prompt cache 计量快照落到 `cache_metering.json`。未设置时回落到
    /// `KIRO_RS_CACHE_PERSIST` 环境变量，再回落到默认**关闭**。
    ///
    /// 关闭后启动不读该文件、运行与退出都不写该文件，也不会删除已有文件。
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub cache_metering_persist_enabled: Option<bool>,

    /// prompt cache 计量快照的落盘间隔（秒）。未设置时回落到
    /// `KIRO_RS_CACHE_FLUSH_INTERVAL_SECS` 环境变量，再回落到默认 300。
    /// 生效值钳到 `[10, 3600]`。仅在落盘开启时有意义。
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub cache_metering_flush_interval_secs: Option<u64>,

    /// 是否开启 miss 归因（墓碑表，用于区分 `miss_evicted` / `miss_expired`）。
    /// 未设置时回落到 `KIRO_RS_CACHE_MISS_DIAGNOSTICS` 环境变量，再回落到默认开启。
    ///
    /// 关闭后不建墓碑表（内存归零），`miss_evicted` 退化为 `miss_no_entry`，
    /// 其余计数器与 `entries_len` 仍然输出。
    #[serde(default, skip_serializing_if = "Option::is_none")]
    pub cache_metering_miss_diagnostics: Option<bool>,
```

在 `Default` 实现里（第 497 行 `cache_metering_enabled: None,` 之后）插入：

```rust
            cache_metering_capacity: None,
            cache_metering_persist_enabled: None,
            cache_metering_flush_interval_secs: None,
            cache_metering_miss_diagnostics: None,
```

- [ ] **Step 5: 跑测试确认通过**

Run: `cargo test --bin kiro-rs cache_metering`
Expected: 全部 PASS（含现有测试；`DEFAULT_CAPACITY` 改大不影响任何现有断言，因为现有测试都不到 4096 条）。

- [ ] **Step 6: 格式化与静态检查**

Run: `cargo fmt && cargo clippy --all-targets -- -D warnings`
Expected: 无告警。

- [ ] **Step 7: Commit**

```bash
git add src/model/config.rs src/anthropic/cache_metering.rs
git commit -m "feat(cache): add configurable CacheMeter capacity/persist/flush/diagnostics options"
```

---

### Task 2: 容量字段与水位批量淘汰

**Files:**
- Modify: `src/anthropic/cache_metering.rs:411-426`（`CacheMeter` / `Inner` 结构）、`:436-464`（`with_remote`）、`:558-588`（`record_entry` / `enforce_capacity`）、`:645-648`（`len`）
- Test: `src/anthropic/cache_metering.rs` 的 `mod tests`

**Interfaces:**
- Consumes: Task 1 的 `DEFAULT_CAPACITY`、`MIN_CAPACITY`、`MAX_CAPACITY`。
- Produces:
  - 字段 `CacheMeter::capacity: usize`
  - `CacheMeter::with_capacity(self, capacity: usize) -> Self`（builder，钳制在 `resolve_capacity` 已做，这里直接存）
  - `CacheMeter::capacity(&self) -> usize`
  - `fn hard_limit(capacity: usize) -> usize`、`fn watermark(capacity: usize) -> usize`（自由函数）
  - `CacheMeter::shrink_to_watermark(&self, inner: &mut Inner)`（私有）
  - `CacheMeter::sweep(&self)`（`evict_expired` + 超过 `capacity` 时水位淘汰，供后台与测试调用）
  - `CacheMeter::len(&self)` 由 `#[cfg(test)]` 改为 `pub`（Admin 与日志都要读）

- [ ] **Step 1: 写失败测试（水位淘汰）**

加到 `mod tests`：

```rust
    /// 构造一个容量确定、last_hit_at 可控的 meter。
    ///
    /// 注意：`with_capacity` 自己会把入参钳到 `[MIN_CAPACITY, MAX_CAPACITY]`，
    /// 所以这里只能传 **≥ 1024** 的合法容量。下面统一用 2000：
    /// `hard_limit = 2400`、`watermark = 1800`。
    fn meter_with_capacity(capacity: usize) -> CacheMeter {
        let cache = CacheMeter::new(None).with_capacity(capacity);
        assert_eq!(cache.capacity(), capacity, "测试容量必须落在合法区间内");
        cache
    }

    #[test]
    fn record_entry_does_not_evict_below_hard_ratio() {
        let cache = meter_with_capacity(2_000);
        for i in 0..2_300u64 {
            cache.record_entry(i, 1, 300);
        }
        assert_eq!(cache.len(), 2_300, "未越过 1.2C(=2400) 硬顶，不应强制淘汰");
    }

    #[test]
    fn record_entry_shrinks_to_watermark_above_hard_ratio() {
        let cache = meter_with_capacity(2_000);
        for i in 0..2_401u64 {
            cache.record_entry(i, 1, 300);
        }
        assert_eq!(cache.len(), 1_800, "越过 1.2C(=2400) 应一次清到 0.9C(=1800)");
    }

    #[test]
    fn shrink_evicts_least_recently_hit_first() {
        let cache = meter_with_capacity(2_000);
        for i in 0..2_400u64 {
            cache.record_entry(i, 1, 300);
        }
        // 下面再插一条就是 2401 条，淘汰量正好 601（2401 - 0.9*2000），
        // 所以「很久以前」那组也要正好 601 条，否则 last_hit_at 相同的条目之间
        // 会出现不确定的 tie-break，断言不稳。
        // 0..1799 标成最近命中（1799 条），1799..2400 标成很久以前（601 条）。
        {
            let mut inner = cache.inner.lock();
            let now = now_secs();
            for (k, v) in inner.entries.iter_mut() {
                v.last_hit_at = if *k < 1_799 { now } else { now - 10_000 };
            }
        }
        // 再插一条（last_hit_at = now，属于最近组）越过 1.2C 硬顶：2401 > 2400
        cache.record_entry(9_999, 1, 300);
        assert_eq!(cache.len(), 1_800);
        let inner = cache.inner.lock();
        for k in 0..1_799u64 {
            assert!(inner.entries.contains_key(&k), "最近命中的 {k} 不该被淘汰");
        }
        assert!(inner.entries.contains_key(&9_999), "刚写入的条目不该被淘汰");
        for k in 1_799..2_400u64 {
            assert!(!inner.entries.contains_key(&k), "最久未命中的 {k} 应被淘汰");
        }
    }

    #[test]
    fn background_sweep_shrinks_above_capacity() {
        let cache = meter_with_capacity(2_000);
        for i in 0..2_100u64 {
            cache.record_entry(i, 1, 300);
        }
        assert_eq!(cache.len(), 2_100, "1.0C~1.2C 之间 record_entry 不动手");
        cache.sweep();
        assert_eq!(cache.len(), 1_800, "后台巡检应把超过 1.0C 的表清到 0.9C");
    }

    /// Review Focus 2：capacity 取下限时 drop_n-1 == 0 与 saturating_mul 的边界。
    #[test]
    fn shrink_handles_minimum_capacity_without_panic() {
        let cache = meter_with_capacity(MIN_CAPACITY);
        // 刚好越过硬顶一条：len = 1024*6/5 + 1 = 1229
        for i in 0..1_229u64 {
            cache.record_entry(i, 1, 300);
        }
        assert_eq!(cache.len(), MIN_CAPACITY / 10 * 9);
        // 上限不溢出
        assert_eq!(hard_limit(MAX_CAPACITY), MAX_CAPACITY * 6 / 5);
        assert_eq!(hard_limit(usize::MAX), usize::MAX / 5);
        // 单条越过水位时 drop_n == 1，select_nth_unstable_by_key(0, ..) 必须合法。
        // 注意不能用 sweep()：len = watermark + 1 = 922 还没超过 capacity(1024)，
        // sweep 会直接早退、根本不触发淘汰，覆盖不到这个边界。必须直接调
        // shrink_to_watermark（同模块子 mod，可访问私有方法）。
        let tiny = meter_with_capacity(MIN_CAPACITY);
        let target = watermark(MIN_CAPACITY);
        for i in 0..(target + 1) as u64 {
            tiny.record_entry(i, 1, 300);
        }
        assert_eq!(tiny.len(), target + 1, "此时还没到 capacity，不该自发淘汰");
        {
            let mut inner = tiny.inner.lock();
            tiny.shrink_to_watermark(&mut inner);
        }
        assert_eq!(tiny.len(), target, "drop_n == 1 的边界必须正好清掉一条");
    }
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cargo test --bin kiro-rs cache_metering::tests::record_entry_`
Expected: 编译失败，`no method named 'with_capacity'`。

- [ ] **Step 3: 实现结构字段与水位淘汰**

`src/anthropic/cache_metering.rs`，`pub struct CacheMeter` 加字段：

```rust
pub struct CacheMeter {
    inner: Mutex<Inner>,
    persist_path: Option<PathBuf>,
    remote: Option<Arc<dyn RemoteCacheStore>>,
    /// 计量模拟总开关（运行时可由 Admin API 切换，无需重启）。
    /// 关闭时 `compute_cache_usage` 直接返回全量 input、零缓存，且不读写任何缓存态。
    enabled: std::sync::atomic::AtomicBool,
    /// 生效条目上限，构造后不变（改容量 = 改 config.json + 重启）
    capacity: usize,
}
```

`with_remote` 的构造尾部改为：

```rust
        Self {
            inner: Mutex::new(inner),
            persist_path,
            remote,
            enabled: std::sync::atomic::AtomicBool::new(true),
            capacity: DEFAULT_CAPACITY,
        }
```

在 `with_enabled` 之后加：

```rust
    /// builder 风格设置生效容量。调用方应先用 `resolve_capacity` 算出钳制后的值。
    pub fn with_capacity(mut self, capacity: usize) -> Self {
        self.capacity = capacity.clamp(MIN_CAPACITY, MAX_CAPACITY);
        self
    }

    /// 当前生效容量。
    pub fn capacity(&self) -> usize {
        self.capacity
    }
```

把 `enforce_capacity` 整个替换为：

```rust
    /// 把表清到 `0.9 * capacity` 水位。单次 `O(n)`：用
    /// `select_nth_unstable_by_key` 把 `last_hit_at` 最小的 drop_n 条划到前半段
    /// （顺序不定、不需要有序），取代原来的 `O(n log n)` 全表排序。
    ///
    /// `last_hit_at` 相等时谁被淘汰不确定 —— LRU 本就是近似，接受。
    fn shrink_to_watermark(&self, inner: &mut Inner) {
        let target = watermark(self.capacity);
        let len = inner.entries.len();
        if len <= target {
            return;
        }
        let drop_n = len - target;
        let mut victims: Vec<(u64, i64)> = inner
            .entries
            .iter()
            .map(|(k, v)| (*k, v.last_hit_at))
            .collect();
        // drop_n >= 1 由上面的早退保证，drop_n - 1 不会下溢
        victims.select_nth_unstable_by_key(drop_n - 1, |x| x.1);
        victims.truncate(drop_n);
        for (k, _) in victims {
            inner.entries.remove(&k);
        }
        inner.dirty = true;
    }

    /// 后台巡检一轮：先清过期，再在超过 `capacity` 时清到水位。
    /// `record_entry` 的 `1.2C` 兜底只在突发写入快过本回收时才会触发。
    pub fn sweep(&self) {
        self.evict_expired();
        let mut inner = self.inner.lock();
        if inner.entries.len() > self.capacity {
            self.shrink_to_watermark(&mut inner);
        }
    }
```

`record_entry` 末尾两行改为：

```rust
        inner.dirty = true;
        if inner.entries.len() > hard_limit(self.capacity) {
            self.shrink_to_watermark(&mut inner);
        }
```

`spawn_background` 里的 `cache.evict_expired();` 改为 `cache.sweep();`。

把 `#[cfg(test)] fn len` 改为公开：

```rust
    /// 当前表内条目数（含已过期但尚未被清理的）。
    pub fn len(&self) -> usize {
        self.inner.lock().entries.len()
    }

    /// 表是否为空（clippy 要求与 `len` 成对）。
    pub fn is_empty(&self) -> bool {
        self.len() == 0
    }
```

在 `fn now_secs()` 之前加两个自由函数：

```rust
/// `record_entry` 的强制淘汰触发线：`capacity * 1.2`。
fn hard_limit(capacity: usize) -> usize {
    capacity.saturating_mul(6) / 5
}

/// 一次淘汰清到的目标水位：`capacity * 0.9`。
fn watermark(capacity: usize) -> usize {
    capacity / 10 * 9
}
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cargo test --bin kiro-rs cache_metering`
Expected: 全部 PASS。

- [ ] **Step 5: 格式化与静态检查**

Run: `cargo fmt && cargo clippy --all-targets -- -D warnings`
Expected: 无告警。

- [ ] **Step 6: Commit**

```bash
git add src/anthropic/cache_metering.rs
git commit -m "perf(cache): replace per-insert full sort with O(n) watermark eviction"
```

---

### Task 3: Stats 计数器与 LookupOutcome

**Files:**
- Modify: `src/anthropic/cache_metering.rs`（`Inner`、`lookup_and_renew`、`evict_expired`、`shrink_to_watermark`）
- Test: `src/anthropic/cache_metering.rs` 的 `mod tests`

**Interfaces:**
- Consumes: Task 2 的 `shrink_to_watermark`、`sweep`。
- Produces:
  - `pub enum LookupOutcome { Hit(u32), Expired, Evicted, Absent }`（`Debug + Clone + Copy + PartialEq + Eq`）
  - `pub struct StatsSnapshot`（`Debug + Clone + Copy + Default + PartialEq + Eq`），字段 `hits, misses, miss_no_entry, miss_expired, miss_evicted, miss_no_breakpoint, miss_no_seed, evicted_total, expired_total, tombstone_clears: u64`
  - `StatsSnapshot::is_all_zero(&self) -> bool`
  - `CacheMeter::probe_outcome(&self, hash: u64) -> LookupOutcome`（私有，**不计数**：只查表 + 滑动续期，供 lookback 多次探测用）
  - `CacheMeter::note_lookup(&self, outcome: LookupOutcome)`（按一次业务查找的最终结果记一次账）
  - `CacheMeter::lookup_outcome(&self, hash: u64) -> LookupOutcome`（= `probe_outcome` + `note_lookup`，单次查找的公开入口）
  - `CacheMeter::lookup_and_renew(&self, hash: u64) -> Option<u32>`（签名不变，变成薄封装）
  - `CacheMeter::stats_total(&self) -> StatsSnapshot`
  - `CacheMeter::take_stats_delta(&self) -> StatsSnapshot`
  - 私有 `struct Stats { total: StatsSnapshot, delta: StatsSnapshot }`，`Stats::add(&mut self, pick: fn(&mut StatsSnapshot) -> &mut u64, n: u64)`

本任务**不建墓碑表**：`Evicted` 分支先恒不产生，Task 4 接上。

- [ ] **Step 1: 写失败测试**

```rust
    #[test]
    fn lookup_outcome_expired_vs_absent() {
        let cache = CacheMeter::new(None);
        assert_eq!(cache.lookup_outcome(1), LookupOutcome::Absent, "从未写过");
        cache.record_entry(1, 7, 300);
        assert_eq!(cache.lookup_outcome(1), LookupOutcome::Hit(7));
        {
            let mut inner = cache.inner.lock();
            inner.entries.get_mut(&1).unwrap().expires_at = now_secs() - 1;
        }
        assert_eq!(
            cache.lookup_outcome(1),
            LookupOutcome::Expired,
            "表里还在但已过期"
        );
    }

    #[test]
    fn stats_counters_accumulate_on_hit_and_miss() {
        let cache = CacheMeter::new(None);
        assert_eq!(cache.lookup_and_renew(1), None);
        cache.record_entry(1, 10, 300);
        assert_eq!(cache.lookup_and_renew(1), Some(10));

        let stats = cache.stats_total();
        assert_eq!(stats.hits, 1);
        assert_eq!(stats.misses, 1);
        assert_eq!(stats.miss_no_entry, 1);
        assert_eq!(stats.miss_expired, 0);
    }

    #[test]
    fn stats_delta_resets_but_total_keeps() {
        let cache = CacheMeter::new(None);
        assert_eq!(cache.lookup_and_renew(1), None);
        let first = cache.take_stats_delta();
        assert_eq!(first.misses, 1);
        let second = cache.take_stats_delta();
        assert!(second.is_all_zero(), "增量取走后应清零");
        assert_eq!(cache.stats_total().misses, 1, "累计量不随增量清零");
    }

    #[test]
    fn evict_and_shrink_bump_totals() {
        let cache = CacheMeter::new(None).with_capacity(MIN_CAPACITY);
        cache.record_entry(1, 5, 60);
        {
            let mut inner = cache.inner.lock();
            inner.entries.get_mut(&1).unwrap().expires_at = now_secs() - 1;
        }
        cache.evict_expired();
        assert_eq!(cache.stats_total().expired_total, 1);

        for i in 0..(hard_limit(MIN_CAPACITY) as u64 + 1) {
            cache.record_entry(100_000 + i, 1, 300);
        }
        assert!(cache.stats_total().evicted_total > 0, "强制淘汰应计数");
    }
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cargo test --bin kiro-rs cache_metering::tests::stats_`
Expected: 编译失败，`cannot find type 'LookupOutcome'`。

- [ ] **Step 3: 实现 Stats 与 LookupOutcome**

在 `struct Inner` 定义之前加：

```rust
/// 单次查找的结果与 miss 归因。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum LookupOutcome {
    /// 命中且已滑动续期
    Hit(u32),
    /// 条目还在表里但已过期，或命中 `Expired` 墓碑
    Expired,
    /// 命中 `Evicted` 墓碑：条目曾存在，被容量淘汰
    Evicted,
    /// 表里没有、墓碑里也没有
    Absent,
}

/// 计量模拟的运行计数器快照。
#[derive(Debug, Clone, Copy, Default, PartialEq, Eq)]
pub struct StatsSnapshot {
    pub hits: u64,
    pub misses: u64,
    pub miss_no_entry: u64,
    pub miss_expired: u64,
    pub miss_evicted: u64,
    pub miss_no_breakpoint: u64,
    pub miss_no_seed: u64,
    pub evicted_total: u64,
    pub expired_total: u64,
    pub tombstone_clears: u64,
}

impl StatsSnapshot {
    /// 本周期是否完全没有事情发生（用于抑制空转日志）。
    pub fn is_all_zero(&self) -> bool {
        *self == Self::default()
    }
}

/// 同时维护「本周期增量」与「进程累计」两份计数。
#[derive(Default)]
struct Stats {
    total: StatsSnapshot,
    delta: StatsSnapshot,
}

impl Stats {
    fn add(&mut self, pick: fn(&mut StatsSnapshot) -> &mut u64, n: u64) {
        *pick(&mut self.total) = pick(&mut self.total).saturating_add(n);
        *pick(&mut self.delta) = pick(&mut self.delta).saturating_add(n);
    }

    fn take_delta(&mut self) -> StatsSnapshot {
        std::mem::take(&mut self.delta)
    }
}
```

`Inner` 加字段：

```rust
#[derive(Default)]
struct Inner {
    entries: HashMap<u64, CacheEntry>,
    /// 自上次落盘后是否有变化
    dirty: bool,
    /// 运行计数器（已在 Mutex 保护下，不额外加锁）
    stats: Stats,
}
```

把 `lookup_and_renew` 整个替换为：

```rust
    /// 探测单个哈希，返回命中结果与 miss 归因。命中时按 entry 自身 TTL 滑动续期。
    ///
    /// **不碰任何计数器**：lookback 一个断点要探测多达 20 个 hash，
    /// 逐个记账会把「一次成功命中」记成「多条 miss + 一条 hit」。
    /// 业务计数统一由调用方在断点结束后调一次 `note_lookup`。
    fn probe_outcome(&self, hash: u64) -> LookupOutcome {
        let now = now_secs();
        let mut inner = self.inner.lock();

        // 先结束对 entries 的可变借用，再去碰 tombstones / stats
        let probe = inner.entries.get_mut(&hash).map(|entry| {
            if entry.expires_at > now {
                let ttl = entry.ttl_secs.clamp(60, MAX_TTL_SECS);
                entry.ttl_secs = ttl;
                entry.last_hit_at = now;
                entry.expires_at = now + ttl;
                Some(entry.tokens)
            } else {
                None
            }
        });

        let outcome = match probe {
            Some(Some(tokens)) => LookupOutcome::Hit(tokens),
            Some(None) => LookupOutcome::Expired,
            None => LookupOutcome::Absent,
        };

        if matches!(outcome, LookupOutcome::Hit(_)) {
            inner.dirty = true;
        }

        outcome
    }

    /// 给**一次业务查找**记一次账：命中记 1 次 `hits`，未命中按原因记 1 次
    /// `misses` + 对应分项。一个断点无论探测过多少个 hash，只调用一次。
    pub fn note_lookup(&self, outcome: LookupOutcome) {
        let mut inner = self.inner.lock();
        match outcome {
            LookupOutcome::Hit(_) => inner.stats.add(|s| &mut s.hits, 1),
            LookupOutcome::Expired => {
                inner.stats.add(|s| &mut s.misses, 1);
                inner.stats.add(|s| &mut s.miss_expired, 1);
            }
            LookupOutcome::Evicted => {
                inner.stats.add(|s| &mut s.misses, 1);
                inner.stats.add(|s| &mut s.miss_evicted, 1);
            }
            LookupOutcome::Absent => {
                inner.stats.add(|s| &mut s.misses, 1);
                inner.stats.add(|s| &mut s.miss_no_entry, 1);
            }
        }
    }

    /// 查一个哈希并记一次账。单 hash 查找（含 `lookup_and_renew`）走这里；
    /// 多 hash 回溯探测请用 `probe_outcome` + 末尾一次 `note_lookup`。
    pub fn lookup_outcome(&self, hash: u64) -> LookupOutcome {
        let outcome = self.probe_outcome(hash);
        self.note_lookup(outcome);
        outcome
    }

    /// 查询单个哈希是否在缓存中且未过期。命中则滑动续期并返回 tokens。
    /// 归因信息请用 `lookup_outcome`。
    pub fn lookup_and_renew(&self, hash: u64) -> Option<u32> {
        match self.lookup_outcome(hash) {
            LookupOutcome::Hit(tokens) => Some(tokens),
            _ => None,
        }
    }

    /// 进程累计计数器快照。
    pub fn stats_total(&self) -> StatsSnapshot {
        self.inner.lock().stats.total
    }

    /// 取走本周期增量并清零（周期性日志用）。
    pub fn take_stats_delta(&self) -> StatsSnapshot {
        self.inner.lock().stats.take_delta()
    }
```

`shrink_to_watermark` 在 `inner.dirty = true;` 之前加：

```rust
        inner.stats.add(|s| &mut s.evicted_total, drop_n as u64);
```

`evict_expired` 整个替换为：

```rust
    /// 删除已过期条目（lookup 不命中过期时只是返回 miss，不会顺手清理；
    /// 这里在后台周期里清一次，避免内存膨胀）。
    pub fn evict_expired(&self) {
        let now = now_secs();
        let mut inner = self.inner.lock();
        let dead: Vec<u64> = inner
            .entries
            .iter()
            .filter(|(_, v)| v.expires_at <= now)
            .map(|(k, _)| *k)
            .collect();
        if dead.is_empty() {
            return;
        }
        for k in &dead {
            inner.entries.remove(k);
        }
        inner.stats.add(|s| &mut s.expired_total, dead.len() as u64);
        inner.dirty = true;
    }
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cargo test --bin kiro-rs cache_metering`
Expected: 全部 PASS（`lookup_miss_then_record_then_hit`、`ttl_expiry_makes_entry_miss` 等现有测试不受影响）。

- [ ] **Step 5: 格式化与静态检查**

Run: `cargo fmt && cargo clippy --all-targets -- -D warnings`
Expected: 无告警。

- [ ] **Step 6: Commit**

```bash
git add src/anthropic/cache_metering.rs
git commit -m "feat(cache): add hit/miss stats counters and LookupOutcome attribution"
```

---

### Task 4: 墓碑表与 evicted / expired 归因

**Files:**
- Modify: `src/anthropic/cache_metering.rs`（`Inner`、`CacheMeter`、`record_entry`、`shrink_to_watermark`、`evict_expired`、`probe_outcome`）
- Test: `src/anthropic/cache_metering.rs` 的 `mod tests`

**Interfaces:**
- Consumes: Task 3 的 `LookupOutcome`、`Stats`、`lookup_outcome`。
- Produces:
  - `pub enum TombstoneKind { Evicted, Expired }`（`Debug + Clone + Copy + PartialEq + Eq`）
  - 常量 `TOMBSTONE_CAPACITY: usize = 65_536`
  - 字段 `CacheMeter::miss_diagnostics: bool`
  - `CacheMeter::with_miss_diagnostics(self, enabled: bool) -> Self`
  - `CacheMeter::miss_diagnostics(&self) -> bool`
  - 私有自由函数 `fn note_tombstone(inner: &mut Inner, enabled: bool, hash: u64, kind: TombstoneKind)`

- [ ] **Step 1: 写失败测试**

```rust
    #[test]
    fn shrink_records_evicted_tombstone() {
        let cache = CacheMeter::new(None).with_capacity(MIN_CAPACITY);
        for i in 0..(hard_limit(MIN_CAPACITY) as u64 + 1) {
            cache.record_entry(i, 1, 300);
        }
        // 被淘汰的那批里至少有一个 hash 能查到 Evicted
        let evicted_seen = (0..(hard_limit(MIN_CAPACITY) as u64 + 1))
            .any(|i| cache.lookup_outcome(i) == LookupOutcome::Evicted);
        assert!(evicted_seen, "容量淘汰应留下 Evicted 墓碑");
    }

    #[test]
    fn evict_expired_records_expired_tombstone() {
        let cache = CacheMeter::new(None);
        cache.record_entry(5, 1, 60);
        {
            let mut inner = cache.inner.lock();
            inner.entries.get_mut(&5).unwrap().expires_at = now_secs() - 1;
        }
        cache.evict_expired();
        assert_eq!(cache.lookup_outcome(5), LookupOutcome::Expired);
        assert_eq!(cache.stats_total().miss_expired, 1);
    }

    #[test]
    fn tombstones_are_capped_and_cleared() {
        let cache = CacheMeter::new(None).with_capacity(MIN_CAPACITY);
        // 每轮插满越过硬顶，反复触发淘汰，累计淘汰量远超 TOMBSTONE_CAPACITY
        let mut next: u64 = 0;
        for _ in 0..80 {
            for _ in 0..hard_limit(MIN_CAPACITY) {
                cache.record_entry(next, 1, 300);
                next += 1;
            }
        }
        let inner = cache.inner.lock();
        assert!(
            inner.tombstones.len() <= TOMBSTONE_CAPACITY,
            "墓碑表必须有硬上限，实际 {}",
            inner.tombstones.len()
        );
        assert!(
            inner.stats.total.tombstone_clears >= 1,
            "超限应整表清空并计数"
        );
    }

    #[test]
    fn miss_diagnostics_disabled_keeps_tombstones_empty() {
        let cache = CacheMeter::new(None)
            .with_capacity(MIN_CAPACITY)
            .with_miss_diagnostics(false);
        for i in 0..(hard_limit(MIN_CAPACITY) as u64 + 1) {
            cache.record_entry(i, 1, 300);
        }
        assert!(cache.inner.lock().tombstones.is_empty(), "关闭后不建墓碑表");
        // 被淘汰的条目退化为 Absent
        let evicted_seen = (0..(hard_limit(MIN_CAPACITY) as u64 + 1))
            .any(|i| cache.lookup_outcome(i) == LookupOutcome::Evicted);
        assert!(!evicted_seen, "关闭后不应产出 Evicted");
        assert!(cache.stats_total().miss_no_entry > 0, "退化为 miss_no_entry");
        assert_eq!(cache.stats_total().miss_evicted, 0);
    }

    /// Review Focus 5：条目被重新写入后，同 hash 的旧墓碑必须消失。
    #[test]
    fn record_entry_clears_stale_tombstone() {
        let cache = CacheMeter::new(None);
        cache.record_entry(42, 1, 60);
        {
            let mut inner = cache.inner.lock();
            inner.entries.get_mut(&42).unwrap().expires_at = now_secs() - 1;
        }
        cache.evict_expired();
        assert_eq!(cache.lookup_outcome(42), LookupOutcome::Expired);

        cache.record_entry(42, 99, 300);
        assert_eq!(cache.lookup_outcome(42), LookupOutcome::Hit(99));
        assert!(
            !cache.inner.lock().tombstones.contains_key(&42),
            "重新写入后旧墓碑必须被删除，否则下次淘汰归因会串"
        );
    }
```

- [ ] **Step 2: 跑测试确认失败**

Run: `cargo test --bin kiro-rs cache_metering::tests::tombstones_`
Expected: 编译失败，`no field 'tombstones' on type 'Inner'`。

- [ ] **Step 3: 实现墓碑表**

常量区（Task 1 新增的常量之后）加：

```rust
/// 墓碑表硬上限。**不随 capacity 变化**：按 48 字节/条估算约 3MB 封顶。
/// 超限整表清空，归因会在清空后的一小段时间里把本应 `miss_evicted` 的记成
/// `miss_no_entry` —— 这是诊断信息，不影响计量正确性。
const TOMBSTONE_CAPACITY: usize = 65_536;
```

在 `LookupOutcome` 之后加：

```rust
/// 条目消失的原因，用于区分 `miss_evicted` / `miss_expired` / `miss_no_entry`。
#[derive(Debug, Clone, Copy, PartialEq, Eq)]
pub enum TombstoneKind {
    /// 被容量水位淘汰
    Evicted,
    /// 被 TTL 过期清理
    Expired,
}
```

`Inner` 加字段：

```rust
    /// miss 归因墓碑表，上限 `TOMBSTONE_CAPACITY`；`miss_diagnostics` 关闭时恒为空
    tombstones: HashMap<u64, TombstoneKind>,
```

`CacheMeter` 加字段 `miss_diagnostics: bool`，`with_remote` 构造尾部补 `miss_diagnostics: true,`。

在 `with_capacity` 之后加：

```rust
    /// builder 风格设置 miss 归因开关。关闭后完全不建墓碑表，内存归零。
    pub fn with_miss_diagnostics(mut self, enabled: bool) -> Self {
        self.miss_diagnostics = enabled;
        if !enabled {
            self.inner.lock().tombstones.clear();
        }
        self
    }

    /// miss 归因是否开启。
    pub fn miss_diagnostics(&self) -> bool {
        self.miss_diagnostics
    }
```

在 `fn now_secs()` 之前加自由函数（写成自由函数是为了在 `&mut Inner` 已被借出时仍可调用）：

```rust
/// 记一条墓碑。超过硬上限时整表清空并把 `tombstone_clears` +1。
fn note_tombstone(inner: &mut Inner, enabled: bool, hash: u64, kind: TombstoneKind) {
    if !enabled {
        return;
    }
    if inner.tombstones.len() >= TOMBSTONE_CAPACITY {
        inner.tombstones.clear();
        inner.stats.add(|s| &mut s.tombstone_clears, 1);
    }
    inner.tombstones.insert(hash, kind);
}
```

`record_entry` 的 `inner.dirty = true;` 之前插入：

```rust
        inner.tombstones.remove(&hash);
```

`shrink_to_watermark` 的淘汰循环改为：

```rust
        let diagnostics = self.miss_diagnostics;
        for (k, _) in victims {
            inner.entries.remove(&k);
            note_tombstone(inner, diagnostics, k, TombstoneKind::Evicted);
        }
```

`evict_expired` 的删除循环改为：

```rust
        let diagnostics = self.miss_diagnostics;
        for k in &dead {
            inner.entries.remove(k);
            note_tombstone(&mut inner, diagnostics, *k, TombstoneKind::Expired);
        }
```

`probe_outcome` 里 `None => LookupOutcome::Absent,` 这一支改为查墓碑（`lookup_outcome` 是它的薄封装，归因自动跟着变）：

```rust
            None => match inner.tombstones.get(&hash) {
                Some(TombstoneKind::Expired) => LookupOutcome::Expired,
                Some(TombstoneKind::Evicted) => LookupOutcome::Evicted,
                None => LookupOutcome::Absent,
            },
```

- [ ] **Step 4: 跑测试确认通过**

Run: `cargo test --bin kiro-rs cache_metering`
Expected: 全部 PASS。

- [ ] **Step 5: 格式化与静态检查**

Run: `cargo fmt && cargo clippy --all-targets -- -D warnings`
Expected: 无告警。

- [ ] **Step 6: Commit**

```bash
git add src/anthropic/cache_metering.rs
git commit -m "feat(cache): add capped tombstone table to distinguish evicted vs expired misses"
```

---

### Task 5: compute_cache_usage 归因与 miss debug 日志

**Files:**
- Modify: `src/anthropic/cache_metering.rs:880-993`（`compute_cache_usage`）、`:995-1081`（`compute_cache_usage_sync`）
- Test: `src/anthropic/cache_metering.rs` 的 `mod tests`

**Interfaces:**
- Consumes: Task 3/4 的 `probe_outcome`、`note_lookup`、`StatsSnapshot`、`stats_total`。
- Produces:
  - `CacheMeter::note_miss_no_breakpoint(&self)`（计数 + `debug!`）
  - `CacheMeter::note_miss_no_seed(&self)`（计数 + `debug!`）

**计数口径（本任务与 Task 3 是同一个验收单元）**：lookback 用不计数的 `probe_outcome` 探测，
一个断点探完后**只调一次** `note_lookup` —— 命中记 1 次 hit（远端命中同样走这里，不再需要
单独的 `note_remote_hit`），全部落空记 1 次 miss 并用 `deepest_reason` 分类。这样计数器口径与
`if !hit { debug! }` 日志口径完全一致：**每断点一条账、每断点最多一条 miss 日志**。

- [ ] **Step 1: 写失败测试**

```rust
    #[tokio::test]
    async fn compute_cache_usage_counts_miss_no_breakpoint() {
        use super::super::types::Message;
        let cache = CacheMeter::new(None);
        let req = MessagesRequest {
            model: "claude-sonnet-4-5-20250929".to_string(),
            max_tokens: 32,
            messages: vec![Message {
                role: "user".to_string(),
                content: serde_json::Value::String("Hello".to_string()),
            }],
            stream: false,
            ..Default::default()
        };
        let usage = compute_cache_usage(&cache, &req, 1).await;
        assert_eq!(usage.cache_read, 0);
        assert!(usage.prompt_total_est > 0);
        let stats = cache.stats_total();
        assert_eq!(stats.miss_no_breakpoint, 1);
        assert_eq!(stats.miss_no_seed, 0);
        assert_eq!(stats.misses, 1);
    }

    #[tokio::test]
    async fn compute_cache_usage_counts_miss_no_seed() {
        // 有断点但 key_id == 0 且没有 metadata.user_id → isolation_seed 返回 None
        let cache = CacheMeter::new(None);
        let req = build_request_with_system_breakpoint();
        let usage = compute_cache_usage(&cache, &req, 0).await;
        assert_eq!(usage.cache_read, 0);
        let stats = cache.stats_total();
        assert_eq!(stats.miss_no_seed, 1);
        assert_eq!(stats.miss_no_breakpoint, 0);
        assert_eq!(stats.misses, 1);
    }

    /// Review Focus 3：远端命中走 remote 路径，必须记成 hit，不能记成 miss。
    #[tokio::test]
    async fn remote_hit_counts_as_hit_not_miss() {
        let remote = Arc::new(FakeRemoteStore::default());
        let cache = CacheMeter::with_remote(None, Some(remote.clone()));
        let req = build_request_with_system_breakpoint();

        // 第一次：本地远端都 miss，写入两边
        let _ = compute_cache_usage(&cache, &req, 1).await;
        let after_cold = cache.stats_total();
        assert!(after_cold.misses > 0);

        // 清掉本地表（墓碑一并清），只留远端 → 下一次必须走远端命中
        {
            let mut inner = cache.inner.lock();
            inner.entries.clear();
            inner.tombstones.clear();
        }
        let before = cache.stats_total();
        let warm = compute_cache_usage(&cache, &req, 1).await;
        let after = cache.stats_total();

        assert!(warm.cache_read > 0, "应从远端命中");
        assert_eq!(after.hits, before.hits + 1, "远端命中必须计 1 次 hit");
        assert_eq!(after.misses, before.misses, "远端命中不得计入 misses");
    }

    /// 构造 user 消息为 text block 数组的请求。
    /// `bp_on_first = true`：只有第 0 块、断点在第 0 块（用来预热浅层前缀）。
    /// `bp_on_first = false`：第 0、1 两块、断点在第 1 块 —— 它的 lookback 能回溯到
    /// 第 0 块，正是「深层不存在、浅层命中」要用的形状。两者第 0 块内容完全一致，
    /// 所以 cum_hash[0] 相同。
    fn build_request_with_two_blocks(bp_on_first: bool) -> MessagesRequest {
        use super::super::types::Message;
        let cc = serde_json::json!({"type": "ephemeral"});
        let mut first = serde_json::json!({"type": "text", "text": "A".repeat(400)});
        let mut second = serde_json::json!({"type": "text", "text": "B".repeat(400)});
        if bp_on_first {
            first["cache_control"] = cc;
        } else {
            second["cache_control"] = cc;
        }
        let content = if bp_on_first {
            serde_json::Value::Array(vec![first])
        } else {
            serde_json::Value::Array(vec![first, second])
        };
        MessagesRequest {
            model: "claude-sonnet-4-5-20250929".to_string(),
            max_tokens: 32,
            messages: vec![Message {
                role: "user".to_string(),
                content,
            }],
            stream: false,
            system: None,
            tools: None,
            tool_choice: None,
            thinking: None,
            output_config: None,
            metadata: None,
            cache_control: None,
        }
    }

    /// 全部落空：一个断点探测多个 hash 也只记 **1 次** miss。
    #[test]
    fn lookback_all_miss_counts_one_miss_per_breakpoint() {
        let cache = CacheMeter::new(None);
        let req = build_request_with_two_blocks(false);
        let usage = compute_cache_usage_sync(&cache, &req, 1);
        assert_eq!(usage.cache_read, 0, "冷启动不该有 read");

        let stats = cache.stats_total();
        assert_eq!(stats.misses, 1, "一个断点只记一次 miss，不是每次探测一条");
        assert_eq!(stats.miss_no_entry, 1);
        assert_eq!(stats.hits, 0);
    }

    /// 深层不存在、浅层命中：记 1 次 hit、0 次 miss。
    #[test]
    fn lookback_deep_miss_shallow_hit_counts_one_hit_only() {
        let cache = CacheMeter::new(None);
        // 先用「断点在第 0 块」的请求写入浅层前缀
        let seed_req = build_request_with_two_blocks(true);
        let _ = compute_cache_usage_sync(&cache, &seed_req, 1);
        let before = cache.stats_total();

        // 再用「断点在第 1 块」的请求：最深 hash 没写过，回溯到第 0 块命中
        let req = build_request_with_two_blocks(false);
        let usage = compute_cache_usage_sync(&cache, &req, 1);
        assert!(usage.cache_read > 0, "应回溯命中浅层前缀");

        let after = cache.stats_total();
        assert_eq!(after.hits, before.hits + 1, "一次成功命中只记 1 次 hit");
        assert_eq!(
            after.misses, before.misses,
            "命中的断点不得同时记 miss —— 深层探测落空不是业务 miss"
        );
    }

    /// 远端路径下同样成立：深层落空、浅层命中只记 1 次 hit。
    #[tokio::test]
    async fn lookback_deep_miss_shallow_hit_counts_one_hit_remote() {
        let remote = Arc::new(FakeRemoteStore::default());
        let cache = CacheMeter::with_remote(None, Some(remote.clone()));
        let seed_req = build_request_with_two_blocks(true);
        let _ = compute_cache_usage(&cache, &seed_req, 1).await;

        // 清掉本地表，强制走远端
        {
            let mut inner = cache.inner.lock();
            inner.entries.clear();
            inner.tombstones.clear();
        }
        let before = cache.stats_total();
        let req = build_request_with_two_blocks(false);
        let usage = compute_cache_usage(&cache, &req, 1).await;

        assert!(usage.cache_read > 0, "应从远端回溯命中");
        let after = cache.stats_total();
        assert_eq!(after.hits, before.hits + 1);
        assert_eq!(after.misses, before.misses);
    }
```

> `build_request_with_two_blocks` 落地时先 `cargo test --bin kiro-rs lookback_` 跑一遍：
> 若 `estimate_tokens` 把 400 字符估成 0 token 会让 block 不可缓存，按需要加长文本。
> 两个请求的第 0 块内容必须逐字节相同（`without_cache_control` 会剥掉 `cache_control`
> 再算 signature），否则 cum_hash[0] 对不上，测试会退化成「全部落空」。

> 注：若 `MessagesRequest` 没有 `Default` 实现，请改为照搬 `build_request_with_system_breakpoint` 的字段逐项构造，并把 `system` 字段置为 `None`、所有 `cache_control` 置为 `None`。先 `cargo check` 确认哪种可用。

- [ ] **Step 2: 跑测试确认失败**

Run: `cargo test --bin kiro-rs cache_metering::tests::compute_cache_usage_counts`
Expected: 编译失败或断言失败（`miss_no_breakpoint` 为 0）。

- [ ] **Step 3: 实现归因入口**

在 `impl CacheMeter` 的 `take_stats_delta` 之后加：

```rust
    /// 请求没有任何可用断点（无 cache_control / 超 MAX_BREAKPOINTS / 混合 TTL 顺序非法）。
    pub fn note_miss_no_breakpoint(&self) {
        {
            let mut inner = self.inner.lock();
            inner.stats.add(|s| &mut s.misses, 1);
            inner.stats.add(|s| &mut s.miss_no_breakpoint, 1);
        }
        tracing::debug!("cache_meter miss reason=no_breakpoint");
    }

    /// 请求拿不到会话隔离种子（无 metadata.user_id 且 key_id == 0）。
    pub fn note_miss_no_seed(&self) {
        {
            let mut inner = self.inner.lock();
            inner.stats.add(|s| &mut s.misses, 1);
            inner.stats.add(|s| &mut s.miss_no_seed, 1);
        }
        tracing::debug!("cache_meter miss reason=no_seed");
    }
```

> 两处 `debug!` 写在锁作用域之外：`tracing` 的 subscriber 是用户代码，不在持锁期间调用。
> 加上 lookback 的 `reason=Absent/Expired/Evicted` 三类，`RUST_LOG=debug` 下五种 miss 原因都有日志。

`compute_cache_usage`（async）里两处早退加计数：

```rust
    let breakpoints = resolve_breakpoints(&blocks, req.cache_control.as_ref());
    if breakpoints.is_empty() {
        // 无断点：官方不会缓存，全部计入 input
        cache.note_miss_no_breakpoint();
        return CacheUsage {
            prompt_total_est: prompt_total_est as i32,
            ..Default::default()
        };
    }

    // 会话隔离种子
    let Some(seed) = isolation_seed(req, key_id) else {
        cache.note_miss_no_seed();
        return CacheUsage {
            prompt_total_est: prompt_total_est as i32,
            ..Default::default()
        };
    };
```

lookback 循环整体替换为：

```rust
    // Lookup: 从每个断点向后检查最多 20 个 block，寻找此前真正写入过的最长前缀
    let mut max_read_tokens: u32 = 0;

    for bp in &breakpoints {
        let start_idx = lookback_start_index(&blocks, bp.block_idx);
        // 归因取最深那个 hash（第一次探测）的判定结果 —— 它才是本可命中的那一条
        let mut deepest_reason = LookupOutcome::Absent;
        // 一个断点只记一次账：探测期间一律用不计数的 probe_outcome，
        // 循环结束后按最终结果调一次 note_lookup。
        let mut hit_tokens_final: Option<u32> = None;

        for check_idx in (start_idx..=bp.block_idx).rev() {
            let hash = cum_hashes[check_idx];
            let mut hit_tokens = None;

            // 优先查远端 Redis 共享后端
            if let Some(remote) = &cache.remote {
                if let Some(tokens) = remote.lookup_and_renew(hash).await {
                    // 回填本地保持同步
                    cache.record_entry(hash, tokens, bp.ttl_secs);
                    hit_tokens = Some(tokens);
                }
            }

            // 若远端未配置或 miss / 出错降级，查本地
            if hit_tokens.is_none() {
                let outcome = cache.probe_outcome(hash);
                if check_idx == bp.block_idx {
                    deepest_reason = outcome;
                }
                if let LookupOutcome::Hit(tokens) = outcome {
                    hit_tokens = Some(tokens);
                }
            }

            if let Some(tokens) = hit_tokens {
                if tokens > max_read_tokens {
                    max_read_tokens = tokens;
                }
                // 当前断点已找到最深命中，不必继续向更浅的 block 回溯
                hit_tokens_final = Some(tokens);
                break;
            }
        }

        match hit_tokens_final {
            // 本地或远端命中都走这里：一个断点记 1 次 hit
            Some(tokens) => cache.note_lookup(LookupOutcome::Hit(tokens)),
            None => {
                // 全部落空：记 1 次 miss，原因取最深那次探测的判定
                cache.note_lookup(deepest_reason);
                tracing::debug!(
                    "cache_meter miss block_idx={} reason={:?} ttl={}",
                    bp.block_idx,
                    deepest_reason,
                    bp.ttl_secs
                );
            }
        }
    }
```

> `deepest_reason` 的初值是 `Absent`，而全部落空时最深那次探测一定执行过
> （`check_idx` 从 `bp.block_idx` 开始），所以 `note_lookup(deepest_reason)` 不会是
> `Hit` 分支 —— 远端命中会在上面 `break` 掉。

`compute_cache_usage_sync` 做同样的三处改动（两个早退加 `note_miss_no_breakpoint()` / `note_miss_no_seed()`，lookback 循环用 `probe_outcome` 探测、循环后一次 `note_lookup` + `debug!`；该函数没有远端路径，去掉 `remote` 那段）。两个函数的计数口径必须完全一致，否则同步/异步两条路径的报表对不上。

- [ ] **Step 4: 跑测试确认通过**

Run: `cargo test --bin kiro-rs cache_metering`
Expected: 全部 PASS（`compute_cache_usage_first_miss_then_hit`、`no_cache_control_does_not_cache_at_all`、`max_breakpoints_limit_enforced` 等现有测试断言不变）。

- [ ] **Step 5: 格式化与静态检查**

Run: `cargo fmt && cargo clippy --all-targets -- -D warnings`
Expected: 无告警。

- [ ] **Step 6: Commit**

```bash
git add src/anthropic/cache_metering.rs
git commit -m "feat(cache): attribute misses per breakpoint with debug logging"
```

---

### Task 6: 落盘开关、间隔与后台任务重构

**Files:**
- Modify: `src/anthropic/cache_metering.rs:591-631`（`flush_to_disk` / `spawn_background`）、`CacheMeter` 结构
- Modify: `Cargo.toml`（新增 `[dev-dependencies]`，给测试打开 tokio 的 `test-util`，用于时间调度测试）
- Test: `src/anthropic/cache_metering.rs` 的 `mod tests`

**Interfaces:**
- Consumes: Task 1 的 `DEFAULT_FLUSH_INTERVAL_SECS` / 钳制常量，Task 2 的 `sweep`，Task 3 的 `take_stats_delta`。
- Produces:
  - 常量 `SWEEP_INTERVAL_SECS: u64 = 60`
  - 字段 `CacheMeter::flush_interval_secs: u64`
  - `CacheMeter::with_flush_interval_secs(self, secs: u64) -> Self`
  - `CacheMeter::flush_interval_secs(&self) -> u64`
  - `CacheMeter::persist_enabled(&self) -> bool`（即 `persist_path.is_some()`）
  - `CacheMeter::log_stats_delta(&self)`

**落盘必须用独立定时器**：巡检固定 60 秒一轮，落盘按 `flush_interval_secs` 自己计时，
两者在 `tokio::select!` 里并行。若把落盘挂在 60 秒巡检里判断「距上次落盘是否够久」，
配 10 秒实际最快 60 秒才落一次、配 90 秒实际 120 秒，和对外声明的 `[10, 3600]` 可配置区间不符。

落盘关闭 = `main.rs` 传 `persist_path = None`（Task 8 接线）。本任务只保证 `None` 时既不读也不写。

- [ ] **Step 1: 写失败测试**

```rust
    #[test]
    fn persist_disabled_does_not_read_or_write_file() {
        let tmp = std::env::temp_dir().join(format!("kiro-pc-off-{}.json", now_secs()));
        // 预置一份有效快照
        let seeded = CacheMeter::new(Some(tmp.clone()));
        seeded.record_entry(7, 42, 600);
        seeded.flush_to_disk();
        let before = std::fs::read(&tmp).expect("预置快照应存在");
        assert!(!before.is_empty());

        // persist 关闭（persist_path = None）：不读
        let off = CacheMeter::new(None);
        assert_eq!(off.len(), 0, "关闭后启动不应加载任何条目");
        assert!(!off.persist_enabled());

        // 不写：跑完一轮落盘后文件字节不变
        off.record_entry(1, 1, 300);
        off.flush_to_disk();
        let after = std::fs::read(&tmp).expect("关闭后不得删除已有文件");
        assert_eq!(before, after, "关闭后不得改写已有快照文件");

        let _ = std::fs::remove_file(&tmp);
    }

    /// Review Focus 1：快照格式不变，旧版本写的文件必须仍能加载。
    #[test]
    fn legacy_snapshot_format_still_loads() {
        let tmp = std::env::temp_dir().join(format!("kiro-pc-legacy-{}.json", now_secs()));
        let future = now_secs() + 600;
        let legacy = format!(
            r#"{{"7":{{"tokens":42,"ttl_secs":600,"expires_at":{future},"last_hit_at":{future}}}}}"#
        );
        std::fs::write(&tmp, legacy).unwrap();

        let cache = CacheMeter::new(Some(tmp.clone()));
        assert_eq!(
            cache.lookup_and_renew(7),
            Some(42),
            "旧格式快照必须仍能加载：序列化结构只含 entries"
        );

        // 回写后仍是同一结构（顶层对象的值里不得出现 tombstones / stats）
        cache.record_entry(8, 1, 300);
        cache.flush_to_disk();
        let raw = std::fs::read_to_string(&tmp).unwrap();
        assert!(!raw.contains("tombstones"), "墓碑不得落盘");
        assert!(!raw.contains("miss_no_entry"), "计数器不得落盘");
        let reread = CacheMeter::new(Some(tmp.clone()));
        assert_eq!(reread.lookup_and_renew(8), Some(1));

        let _ = std::fs::remove_file(&tmp);
    }

    #[test]
    fn flush_interval_builder_round_trips() {
        let cache = CacheMeter::new(None).with_flush_interval_secs(600);
        assert_eq!(cache.flush_interval_secs(), 600);
        // 钳制：builder 自身也兜一次底
        let low = CacheMeter::new(None).with_flush_interval_secs(1);
        assert_eq!(low.flush_interval_secs(), MIN_FLUSH_INTERVAL_SECS);
    }

    #[test]
    fn log_stats_delta_consumes_the_delta() {
        let cache = CacheMeter::new(None);
        assert_eq!(cache.lookup_and_renew(1), None);
        cache.log_stats_delta();
        assert!(
            cache.take_stats_delta().is_all_zero(),
            "输出后增量应被清零"
        );
        assert_eq!(cache.stats_total().misses, 1, "累计量保留");
    }

    /// 落盘定时器必须独立于 60 秒巡检：配 10 秒就要在 10 秒量级落盘，
    /// 配 90 秒就要在 90 秒落盘（而不是被巡检对齐到 60/120 秒）。
    /// 用 tokio 的虚拟时钟，不真的等。
    async fn flush_count_within(interval_secs: u64, advance_secs: u64, persist: bool) -> usize {
        let tmp = std::env::temp_dir().join(format!(
            "kiro-pc-sched-{}-{}-{}.json",
            interval_secs,
            advance_secs,
            fastrand::u64(..)
        ));
        let path = if persist { Some(tmp.clone()) } else { None };
        let cache = Arc::new(
            CacheMeter::new(path)
                .with_flush_interval_secs(interval_secs),
        );
        cache.record_entry(1, 10, 3_600);
        cache.clone().spawn_background();

        let mut writes = 0usize;
        let mut last: Option<std::time::SystemTime> = None;
        for _ in 0..advance_secs {
            tokio::time::advance(std::time::Duration::from_secs(1)).await;
            // 让后台任务有机会被调度
            tokio::task::yield_now().await;
            if let Ok(meta) = std::fs::metadata(&tmp) {
                let mtime = meta.modified().ok();
                if mtime != last {
                    writes += 1;
                    last = mtime;
                }
            }
        }
        let _ = std::fs::remove_file(&tmp);
        writes
    }

    #[tokio::test(start_paused = true)]
    async fn flush_timer_is_independent_of_sweep_period() {
        // 10 秒间隔：在 50 秒内（第一次巡检都还没到）必须已经落过盘
        assert!(
            flush_count_within(10, 50, true).await >= 1,
            "flush_interval=10 不应被 60 秒巡检拖到 60 秒后才落盘"
        );
        // 90 秒间隔：100 秒内应恰好落过一次（而不是被对齐到 120 秒）
        assert!(
            flush_count_within(90, 100, true).await >= 1,
            "flush_interval=90 应在 90 秒落盘，而不是 120 秒"
        );
        // 300 秒（默认）：200 秒内不该落盘
        assert_eq!(
            flush_count_within(300, 200, true).await,
            0,
            "flush_interval=300 在 200 秒内不该落盘"
        );
        // 关闭持久化：无论多久都不落盘
        assert_eq!(
            flush_count_within(10, 200, false).await,
            0,
            "persist 关闭时落盘分支必须完全不执行"
        );
    }
```

> 这组测试用的是 tokio 的虚拟时钟（`start_paused`），需要 `test-util` feature；
> 若 `tokio::time::advance` 下 `flush_to_disk` 的文件 mtime 分辨率不足以区分两次写入，
> 改为统计落盘次数的测试钩子（`AtomicUsize` 计数器）而不是比对 mtime —— 口径不变：
> **10 秒的配置不得被 60 秒巡检对齐**。

- [ ] **Step 2: 跑测试确认失败**

Run: `cargo test --bin kiro-rs cache_metering::tests::persist_disabled`
Expected: 编译失败，`no method named 'persist_enabled'`。

- [ ] **Step 3: 实现**

`Cargo.toml` 末尾加（只影响 `cargo test` 的构建，不进发布二进制）：

```toml
[dev-dependencies]
# 时间调度测试用虚拟时钟（#[tokio::test(start_paused = true)]）；
# test-util 不在 tokio 的 "full" 里，必须显式打开。
tokio = { version = "1.0", features = ["full", "test-util"] }
```

`CacheMeter` 加字段 `flush_interval_secs: u64`，`with_remote` 构造尾部补 `flush_interval_secs: DEFAULT_FLUSH_INTERVAL_SECS,`。

在 `with_miss_diagnostics` 之后加：

```rust
    /// builder 风格设置落盘间隔（秒），自动钳到 `[10, 3600]`。
    pub fn with_flush_interval_secs(mut self, secs: u64) -> Self {
        self.flush_interval_secs = secs.clamp(MIN_FLUSH_INTERVAL_SECS, MAX_FLUSH_INTERVAL_SECS);
        self
    }

    /// 当前生效的落盘间隔（秒）。
    pub fn flush_interval_secs(&self) -> u64 {
        self.flush_interval_secs
    }

    /// 是否开启磁盘快照。关闭即 `persist_path == None`：
    /// 启动不读文件、运行不写文件、退出不写文件，也不删除已有文件。
    pub fn persist_enabled(&self) -> bool {
        self.persist_path.is_some()
    }

    /// 输出本周期增量指标。本周期无任何增量时不输出，避免空转刷屏。
    pub fn log_stats_delta(&self) {
        let (entries_len, delta) = {
            let mut inner = self.inner.lock();
            (inner.entries.len(), inner.stats.take_delta())
        };
        if delta.is_all_zero() {
            return;
        }
        tracing::info!(
            "cache_meter stats entries_len={} capacity={} hits={} misses={} evicted={} expired={} \
             miss_no_entry={} miss_expired={} miss_evicted={} miss_no_breakpoint={} \
             miss_no_seed={} tombstone_clears={}",
            entries_len,
            self.capacity,
            delta.hits,
            delta.misses,
            delta.evicted_total,
            delta.expired_total,
            delta.miss_no_entry,
            delta.miss_expired,
            delta.miss_evicted,
            delta.miss_no_breakpoint,
            delta.miss_no_seed,
            delta.tombstone_clears,
        );
    }
```

`spawn_background` 整个替换为：

```rust
    /// 启动后台周期任务。
    ///
    /// **两个互相独立的定时器**：
    /// - 巡检 `SWEEP_INTERVAL_SECS`（60 秒）一轮，做「清过期 + 水位检查 + 指标输出」；
    /// - 落盘按 `flush_interval_secs`（10–3600 秒）自己计时。
    ///
    /// 不把落盘挂在巡检轮里判断「距上次够久没有」：那样 10 秒的配置实际最快 60 秒
    /// 才落一次、90 秒的配置实际 120 秒，与对外声明的可配置区间不符。
    /// 落盘关闭时该分支用 `select!` 的条件直接禁用，定时器不参与竞争。
    pub fn spawn_background(self: Arc<Self>) {
        let sweep_period = std::time::Duration::from_secs(SWEEP_INTERVAL_SECS);
        let flush_period = std::time::Duration::from_secs(self.flush_interval_secs);
        let persist = self.persist_enabled();
        let weak = Arc::downgrade(&self);
        drop(self);
        tokio::spawn(async move {
            let mut sweep_tick = tokio::time::interval(sweep_period);
            let mut flush_tick = tokio::time::interval(flush_period);
            // interval 的第一次 tick 立刻返回，先吃掉，避免启动瞬间就落一次盘
            sweep_tick.tick().await;
            flush_tick.tick().await;
            // 落后时补齐而不是追打一串 tick（进程被挂起/机器休眠后尤其重要）
            sweep_tick.set_missed_tick_behavior(tokio::time::MissedTickBehavior::Delay);
            flush_tick.set_missed_tick_behavior(tokio::time::MissedTickBehavior::Delay);
            loop {
                tokio::select! {
                    _ = sweep_tick.tick() => {
                        let Some(cache) = weak.upgrade() else { return };
                        cache.sweep();
                        cache.log_stats_delta();
                    }
                    _ = flush_tick.tick(), if persist => {
                        let Some(cache) = weak.upgrade() else { return };
                        cache.flush_to_disk();
                    }
                }
            }
        });
    }
```

常量区加：

```rust
/// 后台巡检周期（秒）：清过期 + 水位检查 + 指标输出。与落盘间隔无关。
const SWEEP_INTERVAL_SECS: u64 = 60;
```

`flush_to_disk` 不改（`persist_path == None` 时第一行就 `return`，快照仍只序列化 `inner.entries`）。

- [ ] **Step 4: 跑测试确认通过**

Run: `cargo test --bin kiro-rs cache_metering`
Expected: 全部 PASS（`flush_and_reload_round_trip` 作为回归保持通过）。

- [ ] **Step 5: 格式化与静态检查**

Run: `cargo fmt && cargo clippy --all-targets -- -D warnings`
Expected: 无告警。

- [ ] **Step 6: Commit**

```bash
git add Cargo.toml src/anthropic/cache_metering.rs
git commit -m "feat(cache): make snapshot persistence optional and decouple flush from sweep"
```

---

### Task 7: Admin API 暴露容量与指标

**Files:**
- Modify: `src/admin/types.rs:636-650`（`CacheMeteringConfigResponse`）
- Modify: `src/admin/service.rs:730-769`（`get_cache_metering_config`）
- Test: `src/admin/service.rs` 的 `#[cfg(test)] mod tests`（若该文件尚无测试模块，在文件末尾新建）

**Interfaces:**
- Consumes: Task 1 的 `resolve_capacity` / `resolve_flush_interval_secs` / `*_from_env`，Task 2 的 `capacity()` / `len()`，Task 3 的 `stats_total()` / `StatsSnapshot`，Task 4 的 `miss_diagnostics()`，Task 6 的 `persist_enabled()` / `flush_interval_secs()`。
- Produces:
  - `CacheMeteringStatsResponse`（camelCase 序列化）
  - `CacheMeteringConfigResponse` 新增 `capacity: usize`、`persist_enabled: bool`、`flush_interval_secs: u64`、`miss_diagnostics: bool`、`stats: Option<CacheMeteringStatsResponse>`

`PUT` 行为不变：`SetCacheMeteringConfigRequest` 仍只有 `enabled`，收到其它字段因 serde 默认忽略未知键而被丢弃，响应里原样回当前生效值。

- [ ] **Step 1: 写失败测试**

在 `src/admin/service.rs` 末尾（若已有 `mod tests` 就加进去）：

测试必须**真的调用 `AdminService::get_cache_metering_config`**，不能手工 new 一个响应结构体
再断言 serde —— 那样句柄优先、配置回落、`stats=null` 这三条本任务的核心行为一条都没被验证。
同文件已有的 `#[tokio::test]` 已经演示了怎么构造真 service（`MultiTokenManager::new(Config::default(), ...)`
+ `AdminService::new(manager, Vec::new())`），照抄即可。

```rust
#[cfg(test)]
mod cache_metering_config_tests {
    use super::*;
    use crate::anthropic::cache_metering::CacheMeter;

    fn service_with(config: Config) -> AdminService {
        let manager = Arc::new(MultiTokenManager::new(config, Vec::new(), None, None, false).unwrap());
        AdminService::new(manager, Vec::new())
    }

    /// Review Focus 4：句柄缺失时 stats 为 null、配置回落到 config/env/默认，不得 panic。
    #[tokio::test]
    async fn config_falls_back_without_cache_meter_handle() {
        let service = service_with(Config::default());
        let resp = service.get_cache_metering_config();
        let json = serde_json::to_value(&resp).unwrap();

        assert_eq!(json["capacity"], 262_144, "回落到内置默认容量");
        assert_eq!(json["persistEnabled"], false);
        assert_eq!(json["flushIntervalSecs"], 300);
        assert_eq!(json["missDiagnostics"], true);
        assert!(json["stats"].is_null(), "句柄缺失时 stats 必须是 null");
    }

    /// config.json 显式值在句柄缺失时也要生效（而不是永远吐默认值）。
    #[tokio::test]
    async fn config_explicit_values_win_without_handle() {
        let mut config = Config::default();
        config.cache_metering_capacity = Some(4_096);
        config.cache_metering_persist_enabled = Some(true);
        config.cache_metering_flush_interval_secs = Some(600);
        config.cache_metering_miss_diagnostics = Some(false);

        let json = serde_json::to_value(service_with(config).get_cache_metering_config()).unwrap();
        assert_eq!(json["capacity"], 4_096);
        assert_eq!(json["persistEnabled"], true);
        assert_eq!(json["flushIntervalSecs"], 600);
        assert_eq!(json["missDiagnostics"], false);
        assert!(json["stats"].is_null());
    }

    /// 有句柄时一律以运行时真实值为准，config.json 里写什么都不算数。
    #[tokio::test]
    async fn runtime_handle_overrides_config_values() {
        let mut config = Config::default();
        // 故意和句柄对着写，验证优先级
        config.cache_metering_capacity = Some(262_144);
        config.cache_metering_flush_interval_secs = Some(300);
        config.cache_metering_miss_diagnostics = Some(true);

        let meter = Arc::new(
            CacheMeter::new(None)
                .with_capacity(4_096)
                .with_miss_diagnostics(false)
                .with_flush_interval_secs(600),
        );
        meter.record_entry(1, 10, 300);
        assert_eq!(meter.lookup_and_renew(1), Some(10));
        assert_eq!(meter.lookup_and_renew(2), None);

        let service = service_with(config).with_cache_meter(meter);
        let json = serde_json::to_value(service.get_cache_metering_config()).unwrap();

        assert_eq!(json["capacity"], 4_096, "以句柄为准");
        assert_eq!(json["flushIntervalSecs"], 600);
        assert_eq!(json["missDiagnostics"], false);
        assert_eq!(json["persistEnabled"], false);
        assert_eq!(json["stats"]["entriesLen"], 1);
        assert_eq!(json["stats"]["hits"], 1);
        assert_eq!(json["stats"]["missNoEntry"], 1);
    }

    /// PUT 只改 `enabled`，其余只读字段必须原样保留（回归：别被顺手覆写成默认值）。
    #[tokio::test]
    async fn put_only_toggles_enabled() {
        let meter = Arc::new(
            CacheMeter::new(None)
                .with_capacity(4_096)
                .with_flush_interval_secs(600)
                .with_miss_diagnostics(false),
        );
        let service = service_with(Config::default()).with_cache_meter(meter);

        let before = serde_json::to_value(service.get_cache_metering_config()).unwrap();
        let resp = service
            .set_cache_metering_config(SetCacheMeteringConfigRequest {
                enabled: Some(false),
            })
            .expect("PUT 应成功");
        let after = serde_json::to_value(&resp).unwrap();

        assert_eq!(after["enabled"], false, "enabled 必须被改掉");
        for key in ["capacity", "flushIntervalSecs", "missDiagnostics", "persistEnabled"] {
            assert_eq!(after[key], before[key], "{key} 是只读字段，PUT 不得改动");
        }
    }
}
```

> 落地时先 `cargo check --tests` 对一下三处签名：`MultiTokenManager::new` 的参数、
> `with_cache_meter` 接受的是 `SharedCacheMeter`（`Arc<CacheMeter>` 的别名）、
> `set_cache_metering_config` 是同步方法且参数是 `Option<bool>`，以及它调
> `update_config_file` 在没有配置文件时的返回值。若它在无配置文件时返回 Err，就把
> `put_only_toggles_enabled` 改成直接断言该 Err 不影响只读字段，或给 service 一个临时
> 配置文件路径 —— **不要**把这条测试退化成手工构造响应结构体。

`src/admin/service.rs` 顶部的 `use crate::admin::types::{...}` 需加 `CacheMeteringStatsResponse`。

- [ ] **Step 2: 跑测试确认失败**

Run: `cargo test --bin kiro-rs admin::service::cache_metering_config_tests`
Expected: 编译失败，`struct CacheMeteringConfigResponse has no field named 'capacity'`。

- [ ] **Step 3: 扩展响应类型**

`src/admin/types.rs`，把 `CacheMeteringConfigResponse` 替换为：

```rust
/// prompt cache 本地计量模拟的运行计数器（进程累计值）。
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct CacheMeteringStatsResponse {
    /// 当前表内条目数（瞬时值）
    pub entries_len: usize,
    pub hits: u64,
    pub misses: u64,
    pub miss_no_entry: u64,
    pub miss_expired: u64,
    pub miss_evicted: u64,
    pub miss_no_breakpoint: u64,
    pub miss_no_seed: u64,
    pub evicted_total: u64,
    pub expired_total: u64,
    pub tombstone_clears: u64,
}

/// prompt cache 本地计量模拟配置响应。
///
/// 除 `enabled` 外均为**只读**：容量、落盘开关、落盘间隔、miss 诊断开关
/// 启动时固定，改这些等于改 `config.json` + 重启。
#[derive(Debug, Clone, Serialize, Deserialize)]
#[serde(rename_all = "camelCase")]
pub struct CacheMeteringConfigResponse {
    /// 计量模拟是否启用
    pub enabled: bool,
    /// 生效条目上限
    pub capacity: usize,
    /// 是否落盘快照
    pub persist_enabled: bool,
    /// 落盘间隔（秒）
    pub flush_interval_secs: u64,
    /// miss 归因是否开启
    pub miss_diagnostics: bool,
    /// 运行计数器；运行时句柄缺失时为 `null`
    pub stats: Option<CacheMeteringStatsResponse>,
}
```

- [ ] **Step 4: 改 service 的查询实现**

`src/admin/service.rs`，`get_cache_metering_config` 整个替换为：

```rust
    /// 查询 prompt cache 计量模拟配置与运行指标。
    ///
    /// 运行时句柄优先（反映当前真实生效状态）；句柄缺失时回落到 config.json
    /// 显式值、再回落到环境变量与默认值，与启动时的优先级一致，`stats` 为 `null`。
    pub fn get_cache_metering_config(&self) -> CacheMeteringConfigResponse {
        use crate::anthropic::cache_metering::CacheMeter;

        if let Some(meter) = self.cache_meter.as_ref() {
            let stats = meter.stats_total();
            return CacheMeteringConfigResponse {
                enabled: meter.is_enabled(),
                capacity: meter.capacity(),
                persist_enabled: meter.persist_enabled(),
                flush_interval_secs: meter.flush_interval_secs(),
                miss_diagnostics: meter.miss_diagnostics(),
                stats: Some(CacheMeteringStatsResponse {
                    entries_len: meter.len(),
                    hits: stats.hits,
                    misses: stats.misses,
                    miss_no_entry: stats.miss_no_entry,
                    miss_expired: stats.miss_expired,
                    miss_evicted: stats.miss_evicted,
                    miss_no_breakpoint: stats.miss_no_breakpoint,
                    miss_no_seed: stats.miss_no_seed,
                    evicted_total: stats.evicted_total,
                    expired_total: stats.expired_total,
                    tombstone_clears: stats.tombstone_clears,
                }),
            };
        }

        let config = self.token_manager.config();
        CacheMeteringConfigResponse {
            enabled: config
                .cache_metering_enabled
                .or_else(CacheMeter::metering_enabled_from_env)
                .unwrap_or(true),
            capacity: CacheMeter::resolve_capacity(
                config.cache_metering_capacity,
                CacheMeter::capacity_from_env(),
            ),
            persist_enabled: config
                .cache_metering_persist_enabled
                .or_else(CacheMeter::persist_enabled_from_env)
                .unwrap_or(false),
            flush_interval_secs: CacheMeter::resolve_flush_interval_secs(
                config.cache_metering_flush_interval_secs,
                CacheMeter::flush_interval_from_env(),
            ),
            miss_diagnostics: config
                .cache_metering_miss_diagnostics
                .or_else(CacheMeter::miss_diagnostics_from_env)
                .unwrap_or(true),
            stats: None,
        }
    }
```

`set_cache_metering_config` 不改（仍只写 `cache_metering_enabled`，末尾 `Ok(self.get_cache_metering_config())` 自动带上新字段）。

- [ ] **Step 5: 跑测试确认通过**

Run: `cargo test --bin kiro-rs admin`
Expected: 全部 PASS。

- [ ] **Step 6: 格式化与静态检查**

Run: `cargo fmt && cargo clippy --all-targets -- -D warnings`
Expected: 无告警。

- [ ] **Step 7: Commit**

```bash
git add src/admin/types.rs src/admin/service.rs
git commit -m "feat(admin): expose cache metering capacity and runtime stats"
```

---

### Task 8: main.rs 启动接线

**Files:**
- Modify: `src/main.rs:263-281`
- Test: 无新单测（接线靠 `cargo build` + 人工启动验收）；本任务靠全量 `cargo test` 回归把关。

**Interfaces:**
- Consumes: Task 1 的 `resolve_capacity` / `resolve_flush_interval_secs` / `*_from_env`，Task 2 的 `with_capacity`，Task 4 的 `with_miss_diagnostics`，Task 6 的 `with_flush_interval_secs`。
- Produces: 运行时生效的 `CacheMeter`。

- [ ] **Step 1: 替换启动段**

把 `src/main.rs` 第 263–281 行整段替换为：

```rust
    // CacheMeter：本地计量模拟，支持可选 Redis 共享元数据层；不承载真实 KV Cache。
    // 容量 / 落盘 / 落盘间隔 / miss 诊断四项均为「config.json > 环境变量 > 内置默认」，
    // 在这里解析一次后通过 builder 注入，运行时不可改（改 = 改配置 + 重启）。
    use anthropic::cache_metering::CacheMeter;
    let cache_metering_enabled = config
        .cache_metering_enabled
        .or_else(CacheMeter::metering_enabled_from_env)
        .unwrap_or(true);
    let cache_metering_capacity =
        CacheMeter::resolve_capacity(config.cache_metering_capacity, CacheMeter::capacity_from_env());
    let cache_metering_persist = config
        .cache_metering_persist_enabled
        .or_else(CacheMeter::persist_enabled_from_env)
        .unwrap_or(false);
    let cache_metering_flush_interval = CacheMeter::resolve_flush_interval_secs(
        config.cache_metering_flush_interval_secs,
        CacheMeter::flush_interval_from_env(),
    );
    let cache_metering_miss_diagnostics = config
        .cache_metering_miss_diagnostics
        .or_else(CacheMeter::miss_diagnostics_from_env)
        .unwrap_or(true);

    let cache_metering_snapshot = cache_dir.join("cache_metering.json");
    let cache_metering_persist_path = if cache_metering_persist {
        Some(cache_metering_snapshot.clone())
    } else {
        // 关闭时不读、不写，也不删除已有文件（留给回滚）
        if let Ok(meta) = std::fs::metadata(&cache_metering_snapshot) {
            tracing::info!(
                "prompt cache 快照持久化已关闭，忽略 {}（{} 字节）",
                cache_metering_snapshot.display(),
                meta.len()
            );
        }
        None
    };

    let cache_meter = std::sync::Arc::new(
        CacheMeter::from_env(cache_metering_persist_path)
            .await
            .with_enabled(cache_metering_enabled)
            .with_capacity(cache_metering_capacity)
            .with_flush_interval_secs(cache_metering_flush_interval)
            .with_miss_diagnostics(cache_metering_miss_diagnostics),
    );
    cache_meter.clone().spawn_background();
    if !cache_metering_enabled {
        tracing::info!("本地 prompt cache 计量模拟已关闭，usage 将全量计入 input_tokens");
    } else {
        tracing::info!(
            "本地 prompt cache 计量模拟已开启：capacity={}（约 {} req/s @ 4 断点 / 300s TTL），\
             persist={}，flush_interval={}s，miss_diagnostics={}",
            cache_metering_capacity,
            cache_metering_capacity / (4 * 300),
            cache_metering_persist,
            cache_metering_flush_interval,
            cache_metering_miss_diagnostics
        );
    }
```

- [ ] **Step 2: 编译确认**

Run: `cargo build`
Expected: 编译通过，无 warning。

- [ ] **Step 3: 全量回归**

Run: `cargo test`
Expected: 全部 PASS。

- [ ] **Step 4: 格式化与静态检查**

Run: `cargo fmt && cargo clippy --all-targets -- -D warnings`
Expected: 无告警。

- [ ] **Step 5: Commit**

```bash
git add src/main.rs
git commit -m "feat(cache): wire cache metering capacity/persist options at startup"
```

---

### Task 9: README 与 config.example.json

**Files:**
- Modify: `README.md:353-387`（常用字段表）、`README.md:680-713`（用量、缓存与日志一节）
- Modify: `config.example.json`

**Interfaces:**
- Consumes: Task 1–8 的全部配置项与默认值。
- Produces: 文档（无代码接口）。

- [ ] **Step 1: 补常用字段表**

`README.md` 的「常用字段」表，在 `| traceEnabled | ... |` 这一行之前插入四行：

```markdown
| `cacheMeteringCapacity` | `262144` | prompt cache 计量条目上限，钳到 `[1024, 8388608]`；见下方「计量容量选型」 |
| `cacheMeteringPersistEnabled` | `false` | 是否把计量快照落到 `cache_metering.json`（**0.9.x 起默认关闭**） |
| `cacheMeteringFlushIntervalSecs` | `300` | 快照落盘间隔（秒），钳到 `[10, 3600]`，仅在落盘开启时有意义 |
| `cacheMeteringMissDiagnostics` | `true` | 是否开启 miss 归因墓碑表（约 3MB 封顶），关闭后 `missEvicted` 退化为 `missNoEntry` |
```

- [ ] **Step 2: 新增「Prompt cache 计量容量选型」小节**

在「用量、缓存与日志」一节的末尾（当前第 712 行 `- output_tokens` 之后、`<a id="admin-ui"></a>` 之前）插入：

````markdown
### Prompt cache 计量容量选型

`CacheMeter` 是一张定容表。写入速率为 `W` 条/秒时，一条 entry 的实际存活期是

```text
T_eff = min(C / W, TTL)
```

`C < W × TTL` 时 TTL 完全失效，命中率由容量而不是 TTL 决定，表现为 `cache_read`
随负载**无声衰减**。上游每请求最多写 4 条断点、默认 TTL 300 秒，所以容量应满足：

```text
C >= peak_req_per_sec × entries_per_req × TTL_secs
```

默认 `262144` 支撑约 `262144 / (4 × 300) ≈ 218 req/s`，内存约 16–24MB
（单条按 64 字节估算）。上界 `8388608` 约 512MB —— 那是「再大就该上 Redis
共享层而不是堆内存」的界线，不是推荐值。改过切段策略、`entries_per_req`
远大于 4 的部署，按公式自行调高。

容量、落盘开关、落盘间隔、miss 诊断开关**启动时固定，运行时不可改**：
改容量会立即触发一次大规模淘汰，不适合做热调旋钮。改这些 = 改 `config.json`
（或环境变量）+ 重启。Admin API `GET /api/admin/cache-metering` 只读暴露当前生效值。

环境变量（优先级：`config.json` 显式值 > 环境变量 > 内置默认）：

| 环境变量 | 对应字段 | 默认 |
|---|---|---:|
| `KIRO_RS_CACHE_CAPACITY` | `cacheMeteringCapacity` | `262144` |
| `KIRO_RS_CACHE_PERSIST` | `cacheMeteringPersistEnabled` | `false` |
| `KIRO_RS_CACHE_FLUSH_INTERVAL_SECS` | `cacheMeteringFlushIntervalSecs` | `300` |
| `KIRO_RS_CACHE_MISS_DIAGNOSTICS` | `cacheMeteringMissDiagnostics` | `true` |

数值解析失败、为 `0` 或为空串都视为「未表态」并打一条 warn，回落到下一优先级，不会中止启动。

#### 怎么判断该不该调容量

后台任务每 60 秒输出一行（本周期无任何增量时不输出）：

```text
cache_meter stats entries_len=13421 capacity=262144 hits=90120 misses=10233 evicted=0 expired=240311
  miss_no_entry=8120 miss_expired=1900 miss_evicted=13 miss_no_breakpoint=180 miss_no_seed=20 tombstone_clears=0
```

- `miss_evicted` 持续非零 + `entries_len` 贴着 `capacity` → **容量不够**，按上面的公式调高。
- `miss_expired` 占多数、`evicted=0` → 是真实 TTL 过期，调容量没用。
- `miss_no_breakpoint` / `miss_no_seed` 占多数 → 客户端没声明 `cache_control`，
  或请求缺少 `metadata.user_id` 且用的是 `id=0` 系统 Key，与容量无关。

单条 miss 的归因还会在 `RUST_LOG=debug` 下输出：有断点的请求按断点逐条打
`cache_meter miss block_idx=.. reason=Absent|Expired|Evicted`（一个断点最多一条，
单请求最多 4 条）；整条请求级的两种原因打 `cache_meter miss reason=no_breakpoint`
与 `cache_meter miss reason=no_seed`。五种原因都可在 debug 日志里看到。

> 计数口径：**一个断点只记一次账**。lookback 会向后探最多 20 个 hash，
> 「深层没有、浅层命中」是正常的部分前缀命中，只记 1 次 `hits`，不会同时产生
> 一堆 `misses` —— 否则命中率报表会被探测次数稀释。

#### 升级提示：快照默认不再落盘

`cacheMeteringPersistEnabled` 默认为 `false`，即**默认不再读写 `cache_metering.json`**。
依据：计量 TTL 只有 300 秒，快照只在重启后 5 分钟内有价值，而容量调大后单次全量
JSON 重写的代价线性放大。关闭时启动不读该文件、运行与退出都不写该文件，
也**不会删除**已有文件（留给回滚）。需要原行为显式写：

```json
{ "cacheMeteringPersistEnabled": true }
```

快照格式仍是 JSON（未换二进制）：默认关闭后这条路径不再是热点，换格式要带
迁移与兼容分支，收益不抵复杂度。旧版本写的快照文件可直接被新版本加载。
````

- [ ] **Step 3: 补 config.example.json**

把 `config.example.json` 的 `"traceEnabled": true,` 之前插入四行：

```json
  "cacheMeteringCapacity": 262144,
  "cacheMeteringPersistEnabled": false,
  "cacheMeteringFlushIntervalSecs": 300,
  "cacheMeteringMissDiagnostics": true,
```

- [ ] **Step 4: 校验示例配置能被解析**

Run: `python3 -m json.tool config.example.json > /dev/null && echo OK`
Expected: 打印 `OK`。

- [ ] **Step 5: 全量回归**

Run: `cargo fmt --check && cargo clippy --all-targets -- -D warnings && cargo test`
Expected: 三项全通过。

- [ ] **Step 6: Commit**

```bash
git add README.md config.example.json
git commit -m "docs: document cache metering capacity sizing formula and new options"
```

---

## 完成后的人工行为验收（写进 MR 描述）

1. 以默认配置启动，跑一段真实负载：日志出现 `cache_meter stats` 行，`entries_len` 随负载上升，`evicted=0`。
2. 把 `cacheMeteringCapacity` 调到 `2048` 复跑同一负载：`miss_evicted` 明显非零、`entries_len` 贴着容量 —— 证明归因机制确实指向容量。
3. `curl -H "x-api-key: <adminApiKey>" http://127.0.0.1:8990/api/admin/cache-metering` 返回新字段与 `stats`。
