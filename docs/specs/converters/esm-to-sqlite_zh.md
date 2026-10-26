# ESM 到 SQLite + rkyv 转换规范

本文档介绍了《Skyrim》主版 (`.esm`) 和插件 (`.esp` / `.esl`) 记录如何投影到索引 SQLite 数据库 (`skyrim_world.db`) 和零拷贝 `rkyv` 地形缓存中。

默认的内部前端使用 MudCrab 的创作记录模式和通用
Rust 解码器。 `--record-reader legacy` 选择手写阅读器，并且
`--record-reader inhouse` 明确选择默认值。内部解码器
通过现有的方法拥有类型字段验证和 FormID 解析
加载顺序映射。它的规范解码字段驱动现有的 SQLite 和
rkyv 投影，保留现有的 rkyv `records.data` 编码。原来的
解压后的有效负载单独保存在辅助
`inhouse_source_records`表；其嵌入的 FormID 仍然与文件相关。未知类型有
不透明的源记录，不是类型化模式覆盖的声明。参见
[管道规范](pipeline.md) 用于生产者身份、诊断和
所选 `--language` 中的本地化名称解析。

内部输出还保留本地化的事件 ID 和查找状态
`inhouse_localized_fields`，中标LAND实物订单
`inhouse_terrain_source_order`，以及标准化源纹理分配
`inhouse_terrain_layers`。 `cell_cache_preserved.rkyv` 保持原始纹理
身份和所有活动标准化层，没有运行时容量
限制。运行时缓存保留其现有的一个基础加上每个基础的五个覆盖层
象限，删除最低总不透明度叠加层（领带上的最高槽位）
带有有限的每单元格警告。其保留层使用显式占位符
对于不可用的目标，而不降低其混合权重。这些辅助
合约保留超出可空运行时投影的源信息。

---

## 1. 概述和目标

- **输入：** Skyrim 主记录（`Skyrim.esm`、`Update.esm`、`Dawnguard.esm`、`.esp` mod 文件）。
- **输出：**
  1. `skyrim_world.db`（具有 R-Tree 空间索引的 SQLite 数据库）。
  2. `cell_cache.rkyv`（零复制二进制内存映射地形和单元数据）。
- **目标：**
  1. 使用空间和单元索引按世界空间和单元坐标选择放置参考行。
  2、通过32位`FormID`索引记录进行查找。
  3. 支持优先级加权插件加载顺序（`plugins.txt` 覆盖）。

---

## 2. ESM 记录结构到数据库模式映射

Skyrim `.esm` 文件由 24 字节记录头组成（`TES4`、`CELL`、`LAND`、`REFR`、`NPC_`、`WEAP`、 `ARMOR`）。

```
┌────────────────────────────────────────────────────────────────────────────┐
│ 天际.esm 记录 │
│ ┌────────────┐ ┌────────────┐ ┌──────────────┐ ┌──────────┐ │
│ │ CELL 记录 │ ──► │ REFR 记录 │ ──► │ LAND 记录 │ ──► │ NPC_ Rec │ │
│ │ (网格 X, Y) │ │ (3D 位置/旋转) │ │ (高度图) │ │ (统计) │ │
│ └──────────────┘ └──────────────┘ └──────────────┘ └──────────┘ │
└──────────────────────────────────────┬──────────────────────────────────────┘
                                       │
                                       ▼（Rust 名词解析器和转换器）
┌────────────────────────────────────────────────────────────────────────────┐
│ SQLite 3 数据库架构 │
│ │
│ ┌────────────────────────┐ ┐──────────────────────────────────┐ │
│ │ 表：记录 │ │ 虚拟表：external_spatial │ │
│ │ - form_id (主键)│ │ 使用 rtree(id, minX, maxX, │ │
│ │ - 记录类型 (4CHAR) │ │ minY, maxY, │ │
│ │ - load_order (INTEGER) │ │ minZ, maxZ) │ │
│ │ - 数据（BLOB） │ └────────────────────────────────┘ │
│ └──────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 3. Database Schema DDL (`skyrim_world.db`)

The authoritative, complete DDL is in [`db-schema.md`](db-schema.md); the two core tables are
repeated here as implemented in `crates/converter/src/esm/exporter.rs`.

### A. Primary Record Table

```sql
如果不存在记录则创建表（
    form_id INTEGER PRIMARY KEY, -- 中奖记录，加载顺序编号
    record_type TEXT NOT NULL, -- 'CELL', 'REFR', 'NPC_', 'WEAP', 'ARMO' 等
    cell_id INTEGER, -- 已放置参考或 LAND 的父 CELL
    worldspace_id INTEGER, -- 外部 CELL 或引用的父 WRLD
    load_order INTEGER NOT NULL, -- 获胜插件的plugins.id (0 = Skyrim.esm, ...)
    data BLOB NOT NULL -- rkyv ArchivedRecordData（子记录有效负载）
）；

如果不存在则创建索引 idx_records_type ON 记录(record_type);
```

There is no `editor_id` column; a record's Editor ID is its `EDID` subrecord in `data`.

### B. World Cell & Cell References Table

```sql
如果单元格不存在则创建表（
    id INTEGER PRIMARY KEY, -- CELL FormID
    worldspace_id INTEGER, -- 父世界空间表单ID；室内装饰为 NULL
    grid_x INTEGER, -- 外部单元网格 X（如果内部为 NULL）
    grid_y INTEGER, -- 外部单元网格 Y（如果内部为 NULL）
    Interior_name TEXT, -- 单元的 EDID，而不是其显示名称
    flags INTEGER NOT NULL, -- 记录头标志
    数据 BLOB，--rkyv ArchivedRecordData
    full_name TEXT -- 解析的完整显示名称
）；
```

Placed references have their own `references` table (positions, rotations, base object); see
`db-schema.md` §5.

### C. Hybrid Spatial Indexing Architecture (Exterior vs. Interior)

Exterior references use a spatial index with a worldspace filter; interior references use direct cell lookup:

1. **Exterior Worldspace R-Tree (`exterior_spatial`):** The exporter inserts each reference's global X/Y/Z position as a point (equal minimum and maximum bounds). SQLite stores these bounds as single-precision values; the exporter applies no coordinate magnitude limit. `cell_id` and `worldspace_id` are auxiliary fields rather than spatial dimensions. Separate reference columns retain cell-local X/Y offsets.
2. **Interior Cell Index (`idx_references_cell`):** Interior cells do not use the R-Tree. The exporter creates an ordinary SQLite index on `references(cell_id)`, which the engine uses to select reference rows for an interior cell. This is an illustrative query shape; it does not fetch complete objects or establish a lookup-time guarantee.

```sql
-- 用于外部 3D 边界框空间查询的 R-Tree 虚拟表
如果不存在则创建虚拟表 external_spatial 使用 rtree(
    id, -- 匹配 REFR FormID
    minX, maxX, -- 全局参考X位置
    minY, maxY, -- 全局参考Y位置
    minZ, maxZ, -- 世界 Z 高度单位
    +cell_id, -- 外部 CELL FormID
    +worldspace_id -- 父世界空间 FormID（例如 Tamriel 的 0x0000003C）
）；

-- 内部引用查找使用的普通 SQLite 索引
如果不存在则创建索引 idx_references_cell ON "references"(cell_id);
```

#### Exterior reference query shape:

```sql
-- 在一个 4096 单元的外部网格单元中选择参考行。
-- 绑定世界空间 ID 和网格派生的下/上 X 和 Y 边界。
选择 r.id、r.cell_id、r.base_form_id、r.pos_x、r.pos_y、r.pos_z
来自外部空间 x
JOIN“参考文献” r ON r.id = x.id
哪里 x.worldspace_id = ?1
  且 x.minX >= ?2 且 x.minX < ?3
  且 x.minY >= ?4 且 x.minY < ?5;
```

The engine uses half-open bounds `[grid * 4096, (grid + 1) * 4096)` for each axis and filters by
worldspace. The R-Tree stores point bounds (`minX = maxX`, and likewise for Y and Z), so the engine
tests the point's minimum coordinates. Its full projection selects explicit reference columns and
joins the base model plus optional door, emittance and enable-state data; the query above only
illustrates spatial candidate selection. Cell-local X/Y offsets remain in `references.local_x` and
`references.local_y`.

#### Interior reference query shape:

```sql
-- 选择一个内部 CELL FormID 的参考行。
选择 r.id、r.cell_id、r.base_form_id、r.pos_x、r.pos_y、r.pos_z
来自“参考文献”r
其中 r.cell_id = ?1;
```

The engine may join the same base-model and optional reference data described above. This example
selects reference rows; it does not represent full object loading or promise a lookup latency.

---

## 4. `rkyv` Zero-Copy Hot Storage (`cell_cache.rkyv`)

For raw terrain heightmaps (`LAND` records) and ultra-dense exterior grid geometry, parsing SQL rows introduces minor memory copies. We use `rkyv` binary buffers mapped with `mmap`.

### Rust Struct Definition:

The implemented record is `shared::CachedLand` (`crates/shared/src/lib.rs`), inside a versioned
`shared::CellCache` (`CELL_CACHE_VERSION` 3):

```rust
#[导出（调试、克隆、存档、序列化、反序列化、PartialEq）]
#[rkyv(bytecheck())]
pub 结构 CachedLand {
    酒吧 cell_id: u32,
    酒吧宽度：u16，
    酒吧高度：u16，
    /// 行优先，绝对创建引擎高度单位。
    酒吧高度：Vec<f32>，
    /// 打包带符号的 XYZ 法线，每个顶点三个字节。
    酒吧法线：Vec<i8>，
    /// 打包 RGB 颜色，每个顶点三个字节。
    pub vertex_colors: Vec<u8>,
    pub 层：Vec<TerrainLayer>,
    酒吧水高度：选项<f32>，
    pub water_type_form_id：选项<u32>，
}
```

### Reading Zero-Copy at Runtime:

The runtime's `CellCache::open` in `crates/engine/src/world/cache.rs` maps the file,
validates the archive, checks `CELL_CACHE_VERSION`, and builds its cell index, returning errors
on failure. This helper illustrates the archive-validation step and returns its result;
version checking remains a separate step before using the cache.

```rust
使用 rkyv::access；
使用共享::ArchivedCellCache；

pub fn load_cell_cache<'a>(
    mmap: &'a memmap2::Mmap,
) -> 结果<&'a ArchivedCellCache, rkyv::rancor::Error> {
    // 验证访问：在就地读取存档之前对其进行检查。
    访问::<ArchivedCellCache, rkyv::rancor::Error>(&mmap[..])
}
```

---

## 5. Implemented Export Path

The record-reader entry points in `crates/converter/src/esm/mod.rs` and `esm/inhouse.rs` prepare
the database schema and pass winning records to the exporter. The transaction in
`crates/converter/src/esm/exporter.rs` writes the canonical `records` archive and its runtime
projections together.

For placed references, `insert_reference_with_canonical_data` writes a `references` row with
the owning cell, worldspace, base object, position, rotation, scale, flags and canonical data.
Exterior rows also retain cell-local X/Y offsets. It then removes any previous spatial entry
for that reference, including when a subset update moves it into an interior. For an exterior
reference, it inserts the global position as equal minimum and maximum bounds, with cell and
worldspace auxiliary values:

```sql
从 external_spatial 删除，其中 id = ?1;

-- 仅为外部参考而执行。
-- 绑定：参考 ID、全局 X、全局 Y、全局 Z、单元 ID、世界空间 ID。
插入或替换到 external_spatial
    （id、minX、maxX、minY、maxY、minZ、maxZ、cell_id、worldspace_id）
值（?1、?2、?2、?3、?3、?4、?4、?5、?6）；
```

Interior references remain in `references` and use its cell index. The exporter writes the
canonical record and runtime reference rows in addition to these spatial statements; the
R-Tree entries supply candidates for the queries in section 3.