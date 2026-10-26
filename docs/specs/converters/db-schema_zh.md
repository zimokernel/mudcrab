# Mudcrab SQLite 3 数据库架构 (`skyrim_world.db`)

本规范详细介绍了 `skyrim_world.db` 的规范 DDL 架构、表、索引和列约束，如 [`crates/converter/src/esm/exporter.rs`](../../../crates/converter/src/esm/exporter.rs) 中实现。

默认使用内部阅读器； `--record-reader legacy` 选择
手写前端。两个记录前端使用相同的运行时列合约和 TXST 槽
解释：漫反射TX00、正常TX01、遮罩TX02、发光
TX03、高度 TX04、环境 TX05、细节 TX06 和镜面反射 TX07。现有的
`detail_path`列承载TX06多层数据； TX03 是源的
发光/细节贴图插槽和 TX07 的背光遮罩/镜面反射插槽。这些角色名称
保留运行时近似值，并不意味着完整的着色器支持。
ARMO 模型投影使用 MOD2 和 MOD4 作为女性世界模型后备
对于两个前端。 MOD3 是 ARMA 模型插槽，而不是 ARMO 世界模型路径。

---

## 1. 架构概述

`skyrim_world.db`是由`crates/converter`通过解析主文件（`Skyrim.esm`）和插件文件（`.esp`/`.esl`）构建的。使用 `PipelineConfig.plugins_file` (CLI `--plugins-file`)，转换器读取 Windows-1252 选择，保留最后一个重复条目的位置和复选框，并不区分大小写地匹配已安装的文件名。安装的《Skyrim》、《Update》、《Dawnguard》、《HearthFires》和《Dragonborn》大师始终按固定顺序激活，然后是可读的 `Skyrim.ccc` 的已安装条目。 `skyrim_ccc_file` (`--skyrim-ccc-file`) 覆盖数据旁边的默认文件。仅加载已检查的玩家条目，包括模组大师；官方和创作条目保留其隐含位置。显式的 `skyrim_ini_file` (`--skyrim-ini-file`) 提供常规 `sTestFile1` 到 `sTestFile10`：即使未选中，列出的插件也会保留其列出的位置，而未列出的条目则按升序文件修改时间附加（文件名打破联系）。不会读取任何环境用户配置文件。缺少条目和可选的 CCC/INI 读取错误发出警告并继续。缺少、禁用或不可用的主插件及其传递依赖项的插件将被忽略，并带有有限的通知；空的可用选择会给出明显的错误。

MO2 配置文件在获胜覆盖路径上重用共享选择规则和转换器的 header/missing-master 闭包恢复。安装的官方大师和游戏文件夹 CCC 条目仍然隐式处于活动状态；其他选定的插件遵循 `loadorder.txt`。缺失的有序条目按最后出现的 `plugins.txt` 顺序附加，并带有警告。两个配置文件列表均使用 Windows-1252，并仅保留最后一个副本。丢失的文件和不可用的依赖闭包会跳过配置文件上下文通知，从而保留可用的邻居。经典天际仍然激活裸个人资料条目。插件名称密钥在玩家列表、MO2 列表、合并数据查找、主匹配和身份映射中使用共享 Unicode `normalize_plugin_name` 函数 (`to_lowercase`)。 Windows-1252 `CAFÉ.esp` 和 `café.esp` 因此匹配；未实现 Unicode 规范化和完整的 Unicode 大小写折叠。记录签名、EditorID 和其他资产密钥保留其现有规则。 MO2 保留其现有的最终顺序验证，而不是应用玩家列表优先级标准化。

现有的显式顺序规范化为带有 ESM 标记的插件和 `.esm`/`.esl` 文件提供优先级，保留每个类别中列出的顺序，但常规依赖项移到需要它们的主文件之前。在规范化之前，循环插件及其所有依赖项都会被省略。既定的最终订单最多允许 254 个完整插件和 4,096 个轻型插件；溢出及其传递依赖项被省略，而不回填后面的条目或更改幸存者顺序。一个有界警告总结了每个恢复原因；转换报告的 `skipped_plugins` 保留通过选择恢复删除的每个原始文件名，每个文件名都按恢复前的选择顺序，包括尝试的不明确文件名、不可用的标头和丢失的主关闭。未检查的插件不会被跳过。 `load_order` 按照与发布的 `plugins` 表相同的顺序记录幸存的原始文件名。当旧报告中不存在时，这两个向量默认为空。规范化顺序仍然放置在其主控之一之前的插件与其依赖项一起被省略而不是移动。幸存路径提供槽位分配、存档优先级、数据库合并和地形缓存。带有 ESL 标记的 `.esp` 占据了一个较轻的位置，但仍属于常规插件。

当 `Skyrim.ccc` 丢失或不可读时，播放器和 MO2 选择使用共享后备：`ccBGSSSE001-Fish.esm`、`ccQDRSSE001-SurvivalMode.esl`、`ccBGSSSE037-Curios.esl`、`ccBGSSSE025-AdvDSGS.esm` 和 `_ResourcePack.esl` 的已安装子集，按顺序位于五个官方大师之后。可读的 CCC，即使是空的，也具有权威性。 `selection_warnings` 记录fallback的CCC路径以及读取失败与转换失败分开； CLI/progress/launcher 将其显示为警告，而不会使可用输出不完整。对于较旧的报告，可选向量默认为空。

存档所有权和加载顺序优先级共享相同的精确或空格/破折号/下划线词干匹配器。安装时间最长的所有者必须在恢复后幸存下来，然后同伴才能提供输入，包括修复和元数据重建。活跃的`_`和`-`同伴在MO2中被接纳；无主档案仍然被排除在外，并在 MO2 之外被接纳。相同的活动茎保留其第一个槽。获胜层中的直接 MO2 插件名称歧义是可以恢复的；更高层的独特替换保留正常的覆盖行为。

如果没有 `plugins_file` 或 MO2 配置文件，则仅选择直接在数据中的插件，并且依赖项优先于依赖项。在现成的插件中，带有 ESM 标记的插件和 `.esm`/`.esl` 文件优先，其次是五个官方文件的常规顺序和 Unicode 规范化文件名顺序。不可用的标头和丢失的主闭包被省略，并带有有限的通知，如在玩家选择中，并且该路径共享周期和时隙溢出恢复。非空插件选择减少到没有可用插件显然会失败，而仅资产输入保留其现有行为。自动发现无法推断不相关模组之间的覆盖优先级；嵌套备份/可选插件将被忽略，而嵌套资产仍然可发现。

数据库将其版本标记为 `schema_info`；当前版本是 **9**
（`shared::WORLD_DATABASE_SCHEMA_VERSION`）。 Schema 4 添加了灯光和
`references.radius_override`。 Schema 5 在#152 中添加了草地数据；
联合生产商输出草和 LOD 表。 Schema 6 添加了 LOD 起源、块元数据、
它的空间索引和构建标识。 Schema 9 结合了修正后的纹理集槽
投影：`mask_path`来自TX02，`glow_path`来自TX03，`height_path`
来自 TX04 和来自 TX05 的 `environment_path`。 TX00 和 TX01 提供漫射和
正常路径。两个前端将 TX06 投影到 `detail_path` 中，将 TX07 投影到
`specular_path`，如上所述。模式 9 还
仅将 XESP 字节 4 投影到 `enable_parent_flags` 中。单独的候选修复
将生产者 25/世界 8 用于不同的合约；首先发货的组合输出
为 26/9，生产者 27 保持世界模式 9。
引擎、`world-inspect` 和启动器接受世界模式 **3 到 9**，使用
`shared::supports_runtime_world_database_schema`。完整的转换器包
支持模式 **15 到 28**。传统世界无需渲染完整细节
详细程度；宣传的 LOD 包需要当前的数据库合同。

转换器生产者 26 还纠正从这些纹理编码中选择的纹理编码。
列。 Producer 27 添加球体、多球体和圆柱体碰撞形状
GLB 碰撞数据的更多物理层；数据库停留在
模式 9. 制作者 28 更改 KTX2 mip 布局和对齐方式；数据库
仍然使用模式 9。Producer-27 KTX2 文件需要重新生成，而
Producer-27 GLB 仍然可重用，因为它们的字节未更改。在制作人处
28、早于27的GLB生产者不兼容。正常转换重建
数据库、单元缓存和派生的 LOD 输出，并重新生成其资产
生产者合同或源/配置/输出证明不兼容。
未更改的脚本和存档摄取在其自己的源时仍然可重用，
依赖性、配置和输出字节证明匹配。
仅元数据重建保留原始保留的生产者/配置
以及网格和纹理字节。运行正常转换以修复保留的内容
编码和碰撞数据。

旧版 TX06/TX07 对齐更改了 SQL 角色分配，同时保留
当前的纹理字节编码：多层/细节和镜面/背光是
两种颜色均用于 `TextureEncoding::from_semantics`。交换这些角色
保持所选编码不变，包括正常/数据优先级
具有多种用途的纹理。 ARMO 模型路径对齐也改变了 SQL
投影；网格转换独立于该列发现 NIF 输入。
这些投影更改需要重建元数据。任何后续更改
转换后的网格或纹理字节必须使受影响的资源缓存键无效；
元数据保留保留了保留字节的生产者身份。

```
┌────────────────────────────────────────────────────────────────────────────┐
│ `skyrim_world.db` 实施方案 │
│ ┌────────────────────────┬────────────────────┬────────────────────┐ │
│ │ `plugins` │ `records` │ `worldspaces` │ │
│ │ (活动插件订单) │ (原始 FormID 数据) │ (世界空间 EDID) │ │
│ ├────────────────────────┼────────────────────┼────────────────────┤ │
│ │ `cells` │ `references` │ `exterior_spatial` │ │
│ │ （单元格网格和名称） │ （3D 世界布局）│ （3D 空间 R 树） │ │
│ ├────────────────────────┼────────────────────┼────────────────────┤ │
│ │ `land` │ `lod` │ `scripts` │ │
│ │ (地形高度图) │ (未使用的 LOD 表) │ (Papyrus 字节码) │ │
│ ├────────────────────────┴────────────────────┴────────────────────┤ │
│ │ `formid_map` & `conversion_cache` │ │
│ │ （32 位到 64 位 ID 桥接器和缓存哈希） │ │
│ └──────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Table Definitions

### 1. Active Plugin Registry (`plugins`)

Stores loaded `.esm`/`.esp`/`.esl` plugin file metadata, load order priority, and checksums.

```sql
如果不存在则创建表插件（
    id INTEGER PRIMARY KEY, -- 加载顺序索引； Records.load_order 指的是它
    name TEXT NOT NULL UNIQUE, -- 安装的插件文件名
    优先级 INTEGER NOT NULL, -- 与 id 相同
    校验和 BLOB NOT NULL -- 插件文件的 SHA-256
）；
```

---

### 2. Primary Record Database (`records`)

Stores subrecord byte payloads indexed by 32-bit Skyrim `FormID` and 4-character
record type codes, encoded as rkyv `ArchivedRecordData`. Both frontends resolve
known embedded links to load-order numbering. The in-house frontend projects
validated decoded fields into this existing encoding. Its auxiliary experimental
`inhouse_source_records(form_id, source_form_id, load_order, record_type, flags,
payload)` table holds original
decompressed source bytes, including file-relative IDs and unused framing. Those
source bytes are provenance, not decoded runtime fields. `load_order` names the
winning source plugin through `plugins.id`. Required movement and terrain
semantics are validated before winner selection; rejected candidates leave an
earlier usable winner intact and are counted in per-plugin diagnostics. The pack's
`record-reader.json` identifies the frontend and schema.

```sql
如果不存在记录则创建表（
    form_id INTEGER PRIMARY KEY, -- 中奖记录，加载顺序编号
    record_type TEXT NOT NULL, -- 'CELL'、'REFR'、'NPC_'、'WEAP'、'ARMO'、'SPEL' 等。
    cell_id INTEGER, -- 已放置参考或 LAND 的父 CELL；否则为 NULL
    worldspace_id INTEGER, -- 外部 CELL 或引用的父 WRLD；否则为 NULL
    load_order INTEGER NOT NULL, -- 提供获胜记录的插件的plugins.id
    data BLOB NOT NULL -- rkyv ArchivedRecordData：子记录有效负载
）；

如果不存在则创建索引 idx_records_type ON 记录(record_type);
如果不存在，则创建索引 idx_records_cell_id ON 记录（cell_id），其中 cell_id 不为空；
```

---

### Draft: optional in-house override history (`record_overrides`)

The prototype in-house producer adds a tooling table containing each physically
framed source record, including singletons, deleted versions and rejected
candidates. Runtime readers continue to query the existing winning-record
tables. No losing payload or decoded field set is retained for history during
conversion. Singletons are explicit: this makes complete physical-header audits
and lazy version lookup uniform without a second implicit representation.

`record_overrides` has a physical primary key `(plugin_id, source_record_offset)`
and a `(form_id, chain_position)` lookup index. Columns are canonical `form_id`,
zero-based `chain_position`, `plugin_id` naming `plugins.id`, native
`source_form_id`, `deleted`, `record_type`, `flags`, `source_record_offset`,
`source_record_size`, `status`, `reason`, native enclosing cell/world/topic IDs
and topic-group offset. The preservation extension also stores proven
`original_owner`, unmasked `full_local`, `source_identity_state`, exclusive
`source_group_end`, `frame_complete`, and exact four-byte `record_signature`
BLOB. A source lookup index covers `(original_owner,full_local,plugin_id,
source_record_offset)`. The record signature does not create a separate source
identity namespace; changing kinds at one owner/local remains a type conflict.
`source_record_offset` points at the 24-byte native
header; size is the declared on-disk payload length, including the compressed
size word when present and excluding the header. Unmappable identities are NULL
and remain identifiable by plugin, offset and native FormID. Positions follow
load order, then physical source offset; duplicate physical IDs stay distinct.

`accepted` versions participate in the existing winner/deletion policy;
`skipped` versions retain a reason and do not displace usable predecessors.
The last accepted version is the source winner unless it deletes the record.
The existing torn-override rule skips an incomplete payload that would replace
an earlier complete version; its physical history row retains that reason.
GMST history uses the reader's canonical EditorID aliases, including an already
decoded EditorID when a later mandatory field rejects that version. Broken
EditorID framing retains the raw numeric alias; history does not guess an ID.
Each TES4 row uses a NULL canonical identity
and status `header`, and stores source path and the authored schema hash.
`inhouse_plugin_masters(plugin_id,master_index,master_name)` stores original MAST
order as scalar rows with contiguous zero-based indices; a header with no child
rows has no masters. New snapshots have no `plugin_masters` JSON column.
The history loader still accepts that column in retained packs when the child
table is absent. These small metadata rows allow replay to reconstruct
original full/light mapping without reading another missing or changed plugin.
They are excluded from ordinary record-chain counts. Their deterministic positions
in the NULL-identity group are header ordering, not a semantic record chain.

`inhouse-records history <pack> <hex FormID> [--data DIR] [--json]` opens the
database read-only and verifies each source against `plugins.checksum` before
decoding the requested offset with the producing schema. Changed, missing,
oversized or undecodable versions remain local diagnostics; other versions
continue. The text view aligns authored field names and occurrences and marks
differences. This is decoded source provenance, not a merge or patch format;
partial decoding and unsupported semantic roles retain their existing limits.
Retained 18-column history layouts remain queryable. If any of the three
physical replay safety columns (group end, frame completeness or exact record
signature) is absent or NULL, that version is reported as undecodable with its
stored provenance; replay never invents missing bounds or reads unchecked bytes.
Older ASCII-folded owner spellings are accepted only through an alias derived
from the authenticated stored plugin basename and its original master list.

These optional tooling tables do not change the runtime database
contract, accepted version range or asset encodings. World and converter asset
schema constants therefore remain unchanged. The existing committed history
contribution uses in-house adapter 5; subsequent adapter identities cover the
preservation extension and required-header master-index compatibility. The
independent `diagnostics_version` in the reader identity covers complete scalar
diagnostics and the normalization of stored tooling lists. Older
in-house provenance output is invalidated through the existing reader/configuration
identity. Runtime schema constants and reusable asset encodings remain unchanged,
and older texture or mesh bytes are not relabeled. The broad configuration hash
includes the reader identity for metadata provenance. Normal conversion can
reuse compatible assets through separately recorded per-kind settings and source
proofs. Adapter-11 metadata recognizes the exact adapter-10 combined settings
proof for independently eligible asset producers, without changing retained
bytes or their recorded producer. The metadata-only path accepts the current
reader identity and that explicit adapter-10 retention contract with matching
schema/add-on bytes and complete reader fields. Older in-house packs without
`diagnostics_version` remain refused rather than relabeled with the new tooling
contract; unknown adapter versions cannot authorize retained settings.
Header rows bind the exact authored
schema; a different schema is reported as undecodable rather than silently
interpreting an old pack with new definitions. Deterministic sorted insertion
and an explicit physical primary key make ordered history contents reproducible.

`inhouse-records history-source <pack> <owner basename> <hex full local>
[--data DIR]` also reaches nullable runtime identities. Proven source ownership
comes only from the original supplier's MAST list: lower high-byte indices name
masters, the equal index names self, and a generic greater index remains
`master_index_out_of_range` with no invented self owner. The existing GMST
greater-index convention is separately labelled `gmst_editor_id_convention`;
its physical rows and decoded EDID/runtime aliases do not establish native
owner proof. Source words of zero remain a separate null state, while nonzero
self/master words with a local component of zero retain that exact source key.
Missing-master load-order validation remains a producer limitation: this
extension does not admit or scan a wholly rejected plugin sequence.

`history-physical <pack> <decimal plugin id> <decimal source offset> [--data DIR]`
reaches every stored header, including unproven owner words. Source/physical
access requires the caller to authenticate the complete database against a
trusted completed-producer receipt/pin and preserve that integrity through
the call, retaining before/after bundle guards. Read-only SQL is not integrity
validation. Original-file checksums prove read bytes, not edited stored MAST,
slot, group-bound or candidate/winner context. These APIs do not independently
verify arbitrary edited databases; their context authority is the authenticated
producing bundle. Callers must consume/drop streamed versions to keep one-body
memory, and must reject partial delivery without a successful complete summary.
Both new commands
emit one version JSONL row at a time followed by a summary. The Rust streaming
callback exposes exact original 24-byte header and on-disk stored payload
(including compression prefix) with full original-file checksum guards before
and after reading through the same handle. It does not inflate, normalize,
repair flags, mask locals, or manufacture typed/runtime decoding. Complete
stored frames have a 64 MiB body access ceiling. A frame crossing its original
containing-group end yields header-only material even when later outer siblings
would satisfy its nominal size. Changed or unavailable originals remain local
access failures. Source queries have an explicit 4096-version capacity; callback
errors and exceeded query capacity return partial/error rather than claiming
complete access. The caller must retain the emitted partial report. Files are
not embedded or copied during conversion, so immutable originals are required
for future body access, and each queried version entails whole-original
checksum passes. The conversion retains header structs plus a sorted-reference
vector; this is not a disk-bounded all-candidate conversion design.

The report distinguishes the latest physical source candidate, latest
reader-admitted candidate, current accepted-source winner, runtime presence,
and surviving earlier runtime fallback. Deletions remain admitted history but
do not become a live source/runtime winner. Wide light locals remain proven
owner/full-local source keys with nullable runtime IDs and exact raw access;
they never spill or mask into an adjacent light slot. Generic required header
indices beyond the original MAST list use the existing supplier fallback for
runtime mapping and increment a separate bounded compatibility diagnostic.
Original ownership stays unresolved, and the full 24-bit local component is
retained in source history. Optional links keep their existing clear-or-reject
policy. Width and null-runtime checks still reject unusable candidates locally,
with their header/reason retained and valid neighbors continuing.
Full conversion atomically recreates only the owned
tooling table within the provenance transaction, so its older optional schema
does not silently survive with missing columns. There are no new sidecar files,
asset conversions or runtime schema changes. Normal asset-cache identity
invalidation remains an existing global adapter coupling for full conversion,
despite unchanged reusable asset encodings.

### 3. Worldspace Registry (`worldspaces`)

Stores worldspace hierarchy and parent world relations (e.g. Tamriel `0x0000003C`, Solstheim).

```sql
如果世界空间不存在则创建表（
    id INTEGER PRIMARY KEY, -- WorldSpace FormID
    editor_id TEXT NOT NULL, -- EDID 字符串（例如“Tamriel”）
    Parent_world INTEGER, -- 父世界空间 FormID（如果是子世界空间）
    标志 INTEGER NOT NULL，
    lod_origin_x INTEGER, -- 已验证的 sidecar/显式来源；如果未解决则为 NULL
    lod_origin_y 整数，
    full_name TEXT -- 显示名称（完整，例如“Skyrim”）；请参阅“显示名称”
）；
```

---

### 4. Cell Registry (`cells`)

Stores exterior cell grid coordinates and each cell's Editor ID.

```sql
如果单元格不存在则创建表（
    id INTEGER PRIMARY KEY, -- CELL FormID
    worldspace_id INTEGER, -- 父世界空间表单ID；室内装饰为 NULL
    grid_x INTEGER, -- 外部单元网格 X（如果内部为 NULL）
    grid_y INTEGER, -- 外部单元网格 Y（如果内部为 NULL）
    Interior_name TEXT, -- 单元格的 EDID（编辑器 ID），当它有一个、外部或
                                        ——室内；不是显示名称
    标志 INTEGER NOT NULL，
    data BLOB, -- 可选单元二进制有效负载
    full_name TEXT -- 显示名称（完整）；请参阅下面的“显示名称”
）；
```

#### Display names (`full_name`)

`worldspaces.full_name`, `cells.full_name` and `statics.full_name` hold the record's `FULL`, the
name the game shows (door prompts, the crosshair label, container menus, the map). `npcs.full_name`
follows the same rules. `cells.interior_name` is still the cell's Editor ID (`EDID`), not its
name; it keeps its column name for compatibility.

`FULL` is a localized string. In a plugin with the TES4 `Localized` flag (0x80) it is a four-byte
ID into `Strings/<plugin>_<language>.STRINGS`, looked up in the configured language's table for the
plugin that supplied the winning record: a loose `Data/Strings` file first, then the archives.
English text uses Windows-1252; other languages use UTF-8 with Windows-1252 fallback. Plain plugin
text uses Windows-1252. Both record readers publish the same UTF-8 text:

- The in-house reader resolves every field its schema types as `lstring`.
- The legacy reader (`--record-reader legacy`) resolves the subrecords the in-house schema types
  as `lstring` in every occurrence for that record type, using the same table lookup
  (`crates/converter/src/esm/legacy_text.rs`). That covers `FULL` on every record type, and
  `DESC`, `RNAM` and others where the subrecord is unambiguous. This changed the legacy
  projection, so its reader identity is `projection_version` 4 and includes language/encoding.
  Version-3 retained proofs are accepted only for their historical English setting; all older
  metadata rebuilds use the configured language. Retained assets keep their independent proof.

A name whose ID is 0 is the existing null sentinel and is NULL without a loss warning. A nonzero ID
missing from the table, or a table that cannot be read, leaves NULL and adds one bounded warning
per plugin with a count and first example. The canonical `records.data` payloads hold the same
resolved text for both readers.

The columns are additive and optional. They did not bump `shared::WORLD_DATABASE_SCHEMA_VERSION`:
`create_tables` adds them to an older reused table, and a reader must probe for them.

`cells.flags` and `worldspaces.flags` are the record-header flags. The `DATA` flags (interior,
has water, show sky, ...) are in `cell_environment.data_flags` and
`worldspace_environment.data_flags` below.

#### Cell and Worldspace Environment (`cell_environment`, `cell_regions`, `cell_lighting`, `worldspace_environment`)

Authored lighting, image space, climate, music, ownership and flags (#269 gap 1). Before these
tables existed the data was only in `cells.data` / `records.data`, and the engine hard-coded
`SkyrimClear` lighting for every space (#264). Layouts follow xEdit 4.1.5f
(`Core/wbDefinitionsTES5.pas`, `CELL` and `WRLD`).

```sql
CREATE TABLE IF NOT EXISTS cell_environment ( -- 每个单元格一行
    cell_id 整数主键，
    data_flags INTEGER, -- DATA（旧插件中的u8，SSE中的u16）：0x1内部，0x2有水，
                                 -- 0x4可以从这里出发，0x8没有LOD水，0x20公共区域，
                                 -- 0x40 手动更改，0x80 显示天空，0x100 使用天空照明
    image_space_id INTEGER, -- XCIM -> IMGS
    Acoustic_space_id INTEGER, -- XCAS -> ASPC
    music_id INTEGER, -- XCMO -> MUSC
    sky_region_id INTEGER, -- XCCM -> REGN 单元格获取其天空和天气
    location_id INTEGER, -- XLCN -> LCTN
    遭遇_区域_id INTEGER, -- XEZN -> ECZN
    owner_id INTEGER, -- XOWN -> FACT, NPC_ 或 ACHR
    owner_rank INTEGER, -- XRNK，拥有它的派系等级
    water_id INTEGER, -- XCWT -> WATR
    water_height REAL -- XCLW；当不存在或非有限时为 NULL
）；
CREATE TABLE IF NOT EXISTS cell_regions ( -- XCLR，每个区域一行
    cell_id 整数非空，
    Region_id 整数非空，
    主键（小区 ID、区域 ID）
) 没有 ROWID；
CREATE TABLE IF NOT EXISTS cell_lighting ( -- 每个 CELL 一行，带有 XCLL 或 LTMP
    cell_id 整数主键，
    lighting_template_id INTEGER, -- LTMP -> LGTM
    xcll_size INTEGER，-- 92，或缩写形式为 72；没有 XCLL 的 NULL
    ambient_r INTEGER、ambient_g INTEGER、ambient_b INTEGER、-- XCLL[0..3]
    方向_r 整数、方向_g 整数、方向_b 整数、 -- XCLL[4..7]
    og_near_r 整数，fog_near_g 整数，fog_near_b 整数，--XCLL[8..11]
    雾_近真实，雾_远真实，-- XCLL [12]，[16]
    direction_rotation_xy INTEGER, orientation_rotation_z INTEGER, -- XCLL[20], [24], 度
    direction_fade REAL、fog_clip_distance REAL、fog_power REAL、-- XCLL[28]、[32]、[36]
    dalc_x_pos_r INTEGER、dalc_x_pos_g INTEGER、dalc_x_pos_b INTEGER、 -- XCLL[40..72]：方向
    dalc_x_neg_r INTEGER、dalc_x_neg_g INTEGER、dalc_x_neg_b INTEGER、 -- 环境 X+、X-、Y+、Y-、
    dalc_y_pos_r INTEGER, dalc_y_pos_g INTEGER, dalc_y_pos_b INTEGER, -- Z+、Z-、镜面反射、比例
    dalc_y_neg_r 整数、dalc_y_neg_g 整数、dalc_y_neg_b 整数、
    dalc_z_pos_r 整数、dalc_z_pos_g 整数、dalc_z_pos_b 整数、
dalc_z_neg_r 整数、dalc_z_neg_g 整数、dalc_z_neg_b 整数、
    dalc_specular_r 整数、dalc_specular_g 整数、dalc_specular_b 整数、
    dalc_scale REAL，
    og_far_r 整数，fog_far_g 整数，fog_far_b 整数，--XCLL[72..75]
    folg_max REAL、light_fade_begin REAL、light_fade_end REAL、-- XCLL[76]、[80]、[84]
    继承标志INTEGER——XCLL[88]
）；
如果不存在则创建表 worldspace_environment ( -- 每个 WRLD 一行
    worldspace_id 整数主键，
    data_flags INTEGER, -- DATA: 0x1 小世界，0x2 不能快速旅行，0x8 没有 LOD 水，
                                 -- 0x10 无风景，0x20 无天空，0x40 固定尺寸，0x80 无草
    parent_flags INTEGER, -- PNAM 字节 0: 0x1 使用土地, 0x2 LOD, 0x4 地图, 0x8 水,
                                 -- 0x10 气候，0x40 来自父级的天空单元数据 (WNAM)
    Climate_id INTEGER, -- CNAM -> CLMT
    water_id INTEGER, -- NAM2 -> WATR
    lod_water_id INTEGER, -- NAM3 -> WATR
    lod_water_height REAL, -- NAM4
    default_land_height REAL, -- DNAM[0]
    default_water_height REAL, -- DNAM[4]
    lighting_template_id INTEGER, -- LTMP -> LGTM（室内照明）
    music_id INTEGER, -- ZNAM -> MUSC
    location_id INTEGER, -- XLCN -> LCTN
    遭遇_区域_id INTEGER, -- XEZN -> ECZN
    bounds_min_x REAL、bounds_min_y REAL、bounds_max_x REAL、bounds_max_y REAL、 -- NAM0、NAM9
    lod_multiplier REAL, -- NAMA
    center_cell_x INTEGER, center_cell_y INTEGER -- WCTR，固定尺寸中心单元
）；
```

Rules:

- Colours are 8-bit sRGB; each one's fourth byte is unused and not stored. The `dalc_*` columns
  are the cell's directional ambient lighting colours (what `LGTM` and `WTHR` keep in a `DALC`).
- `XCLL` is 92 bytes in Skyrim SE. xEdit accepts it from 72 bytes, ending after the directional
  ambient block; the fields past the end are then NULL. An `XCLL` under 72 bytes is ignored.
  Floats that are not finite are NULL. `inherit_flags` says which fields the lighting template
  supplies instead: 0x1 ambient, 0x2 directional, 0x4 fog colour, 0x8 fog near, 0x10 fog far,
  0x20 directional rotation, 0x40 directional fade, 0x80 clip distance, 0x100 fog power,
  0x200 fog max, 0x400 light fade distances. The template is in `lighting_templates` below.
- Every link column holds a load-order FormID, and only when the winning record it names exists
  and has the type listed; otherwise it is NULL (a `cell_regions` row is skipped). The in-house
  reader already clears links that do not resolve and the legacy reader keeps them, so this check
  is what makes both readers publish the same rows.
- A typed field whose length the in-house schema rejects (for example a `DATA` that is neither 1
  nor 2 bytes, or an `XCLW` that is not 4) is NULL for both readers: the in-house reader leaves it
  out of its runtime projection, so the projection applies the same sizes to legacy records.
- The tables are derived data: a full load-order export recreates them from the winning records.
  The movement-annotation subset export leaves them alone, and diagnostic archives leave them
  empty (they are deferred runtime views, like the room tables).

#### Weather, Climate, Image Space and Lighting Templates (`weathers`, `weather_colors`, `weather_directional_ambient`, `climates`, `climate_weathers`, `region_weathers`, `image_spaces`, `lighting_templates`)

The records Skyrim's sky and interior lighting come from (#269 gap 2). Together with the tables
above, this is the data the engine needs to replace its hard-coded `SkyrimClear` day (#264).

```sql
CREATE TABLE IF NOT EXISTS 天气 ( -- 每个 WTHR 一行
    id 整数主键，editor_id 文本，
    fog_day_near REAL、fog_day_far REAL、fog_night_near REAL、fog_night_far REAL、--FNAM（32 字节）
    fog_day_power REAL、fog_night_power REAL、fog_day_max REAL、fog_night_max REAL、
    Wind_speed INTEGER、transition_delta INTEGER、sun_glare INTEGER、sun_damage INTEGER、-- DATA（19 字节）、
    降水开始 INTEGER, 降水结束 INTEGER, -- 原始 u8 值
    闪电开始整数、闪电结束整数、闪电频率整数、
    classification_flags INTEGER, -- DATA[11]: 0x1 宜人, 0x2 阴天, 0x4 下雨, 0x8 下雪, ...
    闪电_r 整数、闪电_g 整数、闪电_b 整数、
    视觉效果开始整数，视觉效果结束整数，
    风向整数，风向范围整数，
    disabled_cloud_layers INTEGER, -- NAM1 位
    降水量_id INTEGER, -- MNAM -> SPGD
    Visual_effect_id INTEGER, -- NNAM -> RFCT
    sky_static_id INTEGER, -- TNAM -> STAT
    sun_glare_lens_id INTEGER, -- GNAM -> 镜头
    image_space_sunrise_id INTEGER、image_space_day_id INTEGER、-- IMSP -> IMGS
    image_space_sunset_id INTEGER、image_space_night_id INTEGER、
    volumetric_lighting_sunrise_id INTEGER、volumetric_lighting_day_id INTEGER、-- HNAM -> VOLI
    volumetric_lighting_sunset_id INTEGER、volumetric_lighting_night_id INTEGER
）；
如果不存在则创建表weather_colors（--NAM0：每种颜色类型和一天中的时间一行
    Weather_id 整数非空，
    color_type TEXT NOT NULL，-- sky_upper，fog_near，未知，环境，阳光，太阳，
                                               -- 星星、sky_lower、地平线、effect_lighting、
                                               -- cloud_lod_diffuse、cloud_lod_ambient、fog_far、
                                               -- 天空静态、水倍增、太阳眩光、月亮眩光
    time_of_day TEXT NOT NULL, -- 日出、白天、日落、夜晚
    r 整数非空，g 整数非空，b 整数非空，
    主键（天气 ID、颜色类型、当天时间）
) 没有 ROWID；
CREATE TABLE IF NOT EXISTSweather_direction_ambient ( -- 四个 DALC 子记录，按顺序排列
    Weather_id INTEGER NOT NULL，time_of_day TEXT NOT NULL，
x_pos_r 整数, x_pos_g 整数, x_pos_b 整数, ..., -- x_pos, x_neg, y_pos, y_neg, z_pos,
    specular_r INTEGER、specular_g INTEGER、specular_b INTEGER、-- z_neg，然后是镜面反射和缩放
    真实规模，
    主键（weather_id、time_of_day）
）；
CREATE TABLE IF NOT EXISTS 气候（--每个 CLMT 一行
    id 整数主键、editor_id 文本、sun_texture 文本、sun_glare_texture 文本、-- FNAM、GNAM
    sunrise_begin INTEGER, sunrise_end INTEGER, -- TNAM（6 字节），以 10 分钟为单位
    日落开始整数，日落结束整数，
    波动性 INTEGER、moon_phase INTEGER -- TNAM[4]、TNAM[5]（月亮和相位长度位）
）；
如果不存在则创建表climate_weathers（--CLMT WLST，按文件顺序
    Climate_id INTEGER NOT NULL，位置 INTEGER NOT NULL，
    Weather_id INTEGER, -- -> WTHR
    机会整数不为空，
    opportunity_global_id INTEGER, -- -> GLOB
    主键（climate_id、位置）
）；
CREATE TABLE IF NOT EXISTS Region_weathers ( -- REGN RDWT，按记录中的文件顺序
    Region_id INTEGER NOT NULL，位置 INTEGER NOT NULL，
    Weather_id INTEGER、机会 INTEGER NOT NULL、chance_global_id INTEGER、
    主键（region_id，位置）
）；
如果不存在则创建表 image_spaces ( -- 每个 IMGS 一行
    id 整数主键，editor_id 文本，
    eye_adaptation_speed REAL、bloom_blur_radius REAL、bloom_threshold REAL、bloom_scale REAL、-- HNAM
    receive_bloom_threshold REAL、white REAL、sunlight_scale REAL、sky_scale REAL、--（36 字节）
    eye_adaptation_strength REAL，
    饱和度 REAL、亮度 REAL、对比度 REAL、-- CNAM
    Tint_amount REAL、tint_r REAL、tint_g REAL、tint_b REAL、-- TNAM、颜色为 0..1 浮点数
    dof_strength REAL、dof_distance REAL、dof_range REAL、--DNAM（12 或 16 字节）
    dof_sky_blur_radius INTEGER -- DNAM[14]，原始 u16 代码
）；
如果不存在则创建表 lighting_templates ( -- 每个 LGTM 一行
    id 整数主键，editor_id 文本，
    data_size INTEGER, -- 64、68、72、76、80、88 或 92；否则为 NULL
    ..., -- DATA: 与 cell_lighting 相同的列，来自
                                                --ambient_r 到 light_fade_end (DATA[88..92] 未知)
    subrecord_dalc_x_pos_r INTEGER, ..., -- 单独的 DALC 子记录，布局如下
subrecord_dalc_scale REAL --weather_direction_ambient
）；
```

Rules beyond the ones above:

- `weather_colors` has a row for each colour type `NAM0` holds: 13 to 17 types of 16 bytes (sunrise,
  day, sunset, night; each RGB and an unused byte). A `NAM0` of another length gives no rows.
- A `DALC` is 24, 28 or 32 bytes (the specular colour and the scale are optional); missing fields
  are NULL.
- Links follow the same existence and type check as above. `CLMT` `WLST` and `WTHR` `IMSP`, `HNAM`,
  `MNAM`, `NNAM`, `TNAM`, `GNAM` and `SNAM` are now remapped by the legacy reader too, as the
  in-house schema already did (see the remapped-field table).

**These tables are optional and did not bump `shared::WORLD_DATABASE_SCHEMA_VERSION`.**
`create_tables` adds them to an older reused database. The engine reads them into
`world::environment::EnvironmentCatalog` (cell lighting, worldspace environment, lighting
templates, weathers with their colours and directional ambient, image spaces, climate weather
lists). A table that is missing or lacks a column the reader needs loads as empty, with one log
line. Nothing renders from the catalog yet: the hard-coded `SkyrimClear` lighting in `sky.rs` is
unchanged.

| Case | Converter writes | Engine |
|---|---|---|
| Old database, tables absent | n/a | empty catalog, one log line, built-in lighting |
| Table of another shape | n/a | empty catalog for that table, one log line |
| Cell without XCLL or LTMP | `cell_environment` row only | no authored lighting for it |
| Link to a missing or wrong-type record | NULL | n/a |

---

### 5. Placed World References (`references`)

Stores 3D positions, rotations, scales, and cell parentage for all placed world objects (`REFR`, `ACHR`, `ACRE`, `PGRE`, `PMIS`).

```sql
如果不存在“引用”则创建表（
    id 整数主键，
    cell_id INTEGER NOT NULL, -- 父单元格表单ID
    worldspace_id INTEGER, -- 父世界空间表单ID；内部为 NULL
    base_form_id INTEGER NOT NULL, -- 基础对象 FormID (NAME)
    is_exterior INTEGER NOT NULL, -- 当父单元格具有世界空间或网格时为 1
    pos_x REAL NOT NULL, -- 3D X 坐标
    pos_y REAL NOT NULL, -- 3D Y 坐标
    pos_z REAL NOT NULL, -- 3D Z 坐标
    local_x REAL, -- 仅外部： pos_x 在其 4096 单元网格单元内
    local_y REAL, --（来自位置，而不是所属的 CELL）；里面为空
    rot_x REAL NOT NULL, -- 旋转 X（弧度）
    rot_y REAL NOT NULL, -- 旋转 Y（弧度）
    rot_z REAL NOT NULL, -- 旋转 Z（弧度）
    scale REAL NOT NULL DEFAULT 1.0, -- 比例乘数
    radius_override REAL, -- 以创建单位为单位的 XRDS 半径（当 REFR 没有时为 NULL）
    header_flags INTEGER NOT NULL DEFAULT 0, -- 获胜记录标题标志，原始存储
    enable_parent_id INTEGER, -- 重新映射的 XESP 父 FormID；不存在时为 NULL
    enable_parent_flags INTEGER, -- XESP 字节 4（0 到 255）；不存在时为 NULL
    emittance_id INTEGER, -- 重新映射的 XEMI 光或区域 FormID；不存在时为 NULL
    数据 BLOB——子记录负载
）；

-- 用于内部单元格引用查找的普通 SQLite 索引
如果不存在则创建索引 idx_references_cell ON "references"(cell_id);
```

The reference flag and enable-parent columns preserve winning raw-record
metadata for initial spawn filtering (#165) and later object LOD (#106).
The engine resolves parent chains before spawning. With no parent, header bit
`0x800` marks an initially disabled reference; the documented rule and
runtime limits are in [reference enable-state evidence](../../research/reference-enable-state-20261005.md).
Current terrain LOD does not consume them. `header_flags` comes from the record header. `enable_parent_id` is
XESP's little-endian parent FormID (bytes 0-3), resolved through plugin load order;
`enable_parent_flags` is byte 4 alone in both readers. Bit 0 inverts the parent's
state and bit 1 enables pop-in. The three unused bytes (5-7) remain unchanged
in the canonical subrecords serialized in `data`. Older legacy projections could
include padding in the normalized flags value; reconversion corrects that
projection. An invalid parent link is published as
`enable_parent_id = 0` (no parent) with its flags kept; a malformed `XESP` is
dropped, so both columns are NULL. See the remapped-field table below.

`emittance_id` is the reference's `XEMI`: the light or region whose colour the
`External_Emittance` surfaces of its model take (§16). A cleared link is stored as NULL.

#### Base object models (`statics`)

`references.base_form_id` joins `statics.id`. The runtime spawns a reference only when its base
has a `statics` row whose `model_path` it can resolve, so a base type missing from this table
never renders.

```sql
如果不存在则创建表静态数据（
    id INTEGER PRIMARY KEY, -- 基本记录 FormID，加载顺序编号
    editor_id TEXT, -- EDID
    model_path TEXT, -- MODL（ARMO：MOD2，然后是 MOD4）；不存在时为 NULL
    flags INTEGER NOT NULL, -- 记录头标志
    full_name TEXT, -- 完整的、解析的本地化文本（如果可用）
    bounds_min_x REAL NOT NULL DEFAULT -64，bounds_min_y REAL NOT NULL DEFAULT -64，
    bounds_min_z REAL NOT NULL DEFAULT -64，bounds_max_x REAL NOT NULL DEFAULT 64，
    bounds_max_y REAL NOT NULL DEFAULT 64，bounds_max_z REAL NOT NULL DEFAULT 64，
    bounds_valid INTEGER NOT NULL DEFAULT 0 -- 1 一旦集成测量了转换后的 GLB
）；
```

Rows come from every placeable base type with a world model: `STAT`, `MSTT`, `FURN`, `TREE`,
`FLOR`, `CONT`, `DOOR`, `ACTI`, `TACT`, `WEAP`, `MISC`, `BOOK`, `AMMO`, `ALCH`, `INGR`, `SLGM`,
`KEYM`, `SCRL`, `ARMO`, `APPA`, `ADDN`, `ARTO`, and `LIGH` when it has a model. `TACT`
(talking activators), `APPA` (apparatus), `ADDN` (addon nodes) and `ARTO` (art objects) were
added for #269; before that their references had no row and did not render. The engine still
skips models under `effects/`, `sky/` or `markers/` and files named `*marker*`
(`converted_model_path` in `crates/engine/src/streaming.rs`), which covers many `ADDN` and `ARTO`
models; that is render policy, not data the table lacks. They are additive
rows, so no `shared::WORLD_DATABASE_SCHEMA_VERSION` bump: an older database keeps loading and
simply lacks them until reconverted. Mesh conversion discovers NIF inputs independently of this
table, so their models were already converted. Both record readers produce the same rows.

#### Reference Links and Details (`reference_linked_refs`, `reference_activate_parents`, `reference_details`)

What a placed reference is linked to, what activates it, who owns it, how it is locked and
whether it is a map marker (#269 gap 3). Patrols, idle markers, traps, levers, pressure plates,
stealing and locked doors need these; before they were only in `references.data`. Written for
every exported placed reference (`REFR`, `ACHR`, `ACRE`, `PGRE`, `PMIS`). Layouts follow xEdit
4.1.5f (`Core/wbDefinitionsTES5.pas`, `REFR`).

```sql
CREATE TABLE IF NOT EXISTS reference_linked_refs ( -- XLKR，按文件顺序每个子记录一行
    Reference_id INTEGER NOT NULL, --references.id
    位置 INTEGER NOT NULL,
    key_id INTEGER, -- -> KYWD; 4 字节形式为 NULL
    linked_ref_id INTEGER, -- -> 放置的引用
    主键（参考 ID、位置）
）；
如果不存在则创建表reference_activate_parents（--XAPR，每个子记录一行
    Reference_id 整数非空，
    位置 INTEGER NOT NULL,
    Parent_ref_id INTEGER, -- -> 放置的引用
    延迟 REAL，-- 秒；如果不是有限则为 NULL
    主键（参考 ID、位置）
）；
如果不存在则创建表reference_details（--每个引用一行，其中任何一个
    Reference_id 整数主键，
    owner_id INTEGER, -- XOWN -> FACT, NPC_ 或 ACHR
    owner_rank INTEGER, -- XRNK，拥有它的派系等级
    activate_parent_flags INTEGER, -- XAPD: 0x1 仅父激活
    lock_level INTEGER, -- XLOC[0]: 1 新手, 25 学徒, 50 熟练者, 75 专家,
                                         -- 100 个主控，255 个需要密钥
    lock_key_id INTEGER, -- XLOC[4..8] -> KEYM，在此处重新映射（见下文）
    lock_flags INTEGER, -- XLOC[8]: 0x4 级别锁
    open_by_default INTEGER NOT NULL DEFAULT 0, -- ONAM 存在（门）
    map_marker INTEGER NOT NULL DEFAULT 0, -- XMRK 存在
    map_marker_flags INTEGER, -- FNAM: 0x1 可见，0x2 可以前往，0x4“显示全部”隐藏
    map_marker_type INTEGER, -- TNAM[0]，标记图标枚举
    location_id INTEGER, -- XLCN -> LCTN（持久位置）
    location_ref_type_id INTEGER, -- XLRL -> LCRT 或 LCTN
    遭遇_区域_id INTEGER, -- XEZN -> ECZN
    Attached_ref_id INTEGER -- XATR -> 放置的引用
）；
```

Rules:

- These detail rows are projected from winning source references of types `REFR`, `ACHR`, `ACRE`,
  `PGRE` and `PMIS`. Target types are field-specific and are not the same set as source types:
  `XLKR.keyword_id` accepts `KYWD`, `PLYR`, `ACHR`, `REFR`, `PGRE`, `PHZD`, `PMIS`, `PARW`,
  `PBAR`, `PBEA`, `PCON` and `PFLA`; `XLKR.linked_ref_id` and `XAPR.parent_ref_id` accept
  `PLYR`, `ACHR`, `REFR`, `PGRE`, `PHZD`, `PMIS`, `PARW`, `PBAR`, `PBEA`, `PCON` and `PFLA`.
  `XATR.attached_ref_id` accepts `REFR`, `PGRE`, `PHZD`, `PMIS`, `PARW`, `PBAR`, `PBEA`, `PCON`
  and `PFLA` (not `ACHR` or `ACRE`). `owner_id` accepts `FACT`, `ACHR` or `NPC_`; `lock_key_id`
  accepts `KEYM`; `XLCN` accepts `LCTN`; `XLRL` accepts `LCRT` or `LCTN`; and `XEZN` accepts
  `ECZN`. A zero ID resolves to NULL. A nonzero hardcoded ID below `0x800` is retained if no
  plugin record exists; a present winner must be non-deleted and have an allowed type. Other
  targets resolve to NULL. Valid-length `XLKR` and `XAPR` rows are kept in source order with
  contiguous positions among accepted-length entries, even when a target resolves to NULL. This
  makes legacy projection match the in-house reader's invalid-link clearing behavior.
- A field whose length the in-house schema rejects is skipped for both readers: `XLKR` must be 4
  or 8 bytes, `XAPR` 8, `XAPD` 1, `XOWN`/`XRNK`/`XLCN`/`XLRL`/`XEZN`/`XATR` 4, `FNAM` 1 and `TNAM`
  2.
- `XLOC` is untyped bytes in the in-house schema, so neither reader remaps its key in
  `references.data`. The projection remaps the key through the supplying plugin's master list
  (`records::source_identity::header_runtime_id`) and applies the normal `KEYM` target check,
  including retention of a nonzero absent hardcoded ID below `0x800`. A subset export without a
  load order leaves `lock_key_id` NULL.
- The map marker's name (`FULL`, a localized string) is not projected into these tables yet.
  Both readers retain its resolved text in the canonical record payload after #271.
- Derived data like the other runtime views: a full load-order export recreates the tables, and
  diagnostic archives leave them empty.

**These tables are optional and did not bump `shared::WORLD_DATABASE_SCHEMA_VERSION`.**
`create_tables` adds them to an older reused database. No engine code reads them yet; a reader
must probe for them first.


---

### 6. Hybrid Spatial Indexing (`exterior_spatial` & Interior `cell_id` Index)

Exterior references use a spatial index with a worldspace filter; interior references use direct cell lookup:

1. **Exterior Worldspace R-Tree (`exterior_spatial`):** Bounds are the reference's global X/Y/Z point position, with equal minimum and maximum values. The exporter does not constrain coordinate magnitudes. SQLite stores single-precision bounds; `cell_id` and `worldspace_id` are auxiliary columns, not spatial dimensions. Cell-local X/Y offsets remain separate columns in `references`.
2. **Interior Cell Index (`idx_references_cell`):** Interior cells do not use the R-Tree. An ordinary SQLite index on `references(cell_id)` supports the engine's interior reference selection; this description makes no latency or complexity claim.

```sql
-- 用于外部 3D 边界框空间查询的 R-Tree 虚拟表
如果不存在则创建虚拟表 external_spatial 使用 rtree(
    id, -- 匹配内部参考 ID
    minX, maxX, -- 全局参考X位置
    minY, maxY, -- 全局参考Y位置
    minZ, maxZ, -- 世界 Z 高度单位
    +cell_id, -- 外部 CELL FormID
    +worldspace_id -- 父世界空间 FormID（例如 Tamriel 的 0x0000003C）
）；
```

---

### 7. Terrain Heightmaps (`land`)

Stores 33x33 terrain heightmap data, vertex textures (`vtex`), vertex colors (`vclr`) and
vertex normals extracted from the winning `LAND` record of each cell.

```sql
如果不存在则创建表 (
    cell_id INTEGER PRIMARY KEY, -- 父单元格表单ID
    高度图 BLOB NOT NULL, -- VHGT: 33x33 高度增量
    vtex BLOB, -- VTEX：土地纹理 FormID
    vclr BLOB, -- VCLR: 顶点颜色
    法线 BLOB -- VNML：顶点法线
）；
```

---

### 8. Level of Detail (`lod_chunks`, `lod_chunks_spatial`, `lod_build`)

Schema 5 stores spatial metadata and file-backed GLB payloads, not geometry
blobs. Source-cell names are serialized text used for full-detail handoff.
Bounds are world-space Creation units; `payload_path` is relative to the asset
root and `content_hash` is the payload's SHA-256. The single build row matches
`lod-manifest.json`.

```sql
如果不存在则创建表 lod_chunks (
    worldspace_id INTEGER NOT NULL，层 INTEGER NOT NULL，
    anchor_x 整数非空，anchor_y 整数非空，
    负载路径文本不为空，内容哈希文本不为空，
    bounds_min_x REAL NOT NULL、bounds_min_y REAL NOT NULL、bounds_min_z REAL NOT NULL、
    bounds_max_x REAL NOT NULL、bounds_max_y REAL NOT NULL、bounds_max_z REAL NOT NULL、
    source_cells TEXT NOT NULL DEFAULT '',
    主键（worldspace_id、层、anchor_x、anchor_y）
）；
如果不存在则创建虚拟表 lod_chunks_spatial 使用 rtree(
    id、minX、maxX、minY、maxY、+worldspace_id、+tier、+anchor_x、+anchor_y
）；
如果不存在则创建表 lod_build (
    id 整数主键检查（id = 1），build_identity 文本不为空
）；
```

The unused pre-schema-6 `lod` blob placeholder is retained. Terrain LOD
uses `lod_chunks` for indexing and external GLB files for payloads. See
[ADR-0010](../../adr/0010-lod-chunk-payload-format.md).

---

### 9. Script attachments (`scripts`)

In-house output stores attached Papyrus script names and canonical VMAD bytes.
These are property bindings, not compiled PEX bytecode. Script status and
authored order remain in VMAD. The last exact-name attachment for a record keeps
the existing SQL overwrite behavior.

```sql
创建表脚本（
    form_id 整数非空，
    script_name 文本不为空，
    vmad BLOB 不为空，
    主键（表单 ID、脚本名称）
）；
```

`script_properties` has `(form_id,script_name,property_index)` as its primary key,
with scalar property name, native `property_type`, status and `value_count`.
`script_property_values` adds a zero-based `value_index` to that key and stores
nullable `object_form_id`, `alias_id`, `string_value`, `integer_value`,
`float_bits` and `boolean_value` columns. Scalar properties have one value row;
arrays keep their authored element order. Native type zero and empty arrays
have no value rows and remain distinguishable by their property headers. Float
bits preserve all f32 representations without JSON or SQLite numeric coercion.

These tables replace `scripts.properties_json` for in-house exports inside the
existing record transaction. Complete exports refresh their owned script
snapshot; record archives leave all three runtime script tables empty. The
legacy reader's existing JSON projection remains separate. Runtime engine
consumers continue using the canonical record/VMAD data. Public partial exports
into a normalized database use scalar script rows without refreshing unrelated
attachments; fresh legacy databases retain the legacy JSON projection.

---

### 10. FormID Translation Map (`formid_map`)

Maps each resolved 32-bit FormID to its stable owning plugin and plugin-local
ID. `plugin_name` is the lowercase owning filename, such as `skyrim.esm`;
`internal_id` is the low 24 bits for full plugins or the low 12 bits for light
plugins. Full and light slots are assigned independently. The pair
`(plugin_name, internal_id)` survives changes to the load-order slots.
`plugin_name` uses the reader's shared Unicode lowercase filename key; join to
`plugins` by applying that same normalization to `plugins.name`. SQLite's
built-in `lower` alone does not normalize accented Windows-1252 case pairs.

Ownership differs from override provenance: `records.load_order` identifies
the winning plugin's priority. Game settings override by case-insensitive
EditorID and retain the first definition's identity. A deletion with no EDID
resolves through any previously encountered non-null FormID alias; an unknown
header-only deletion is skipped with a warning. Later restorations keep the
original identity. Ambiguous aliases and live settings without an EDID are
errors, rather than silently replacing or dropping another record.

Known FormID fields inside the published subrecord blobs (`records.data`,
`cells.data`, `references.data`) are rewritten into the same load-order numbering
as the record keys, so a consumer can look them up directly. That covers the
single-FormID fields in the converter's `is_form_id_subrecord`, the
LAND/GRAS/LTEX payloads, the primary VMAD script-property references, and these
reference and cell fields, validated against their expected sizes:

| Field | Records | FormIDs |
| --- | --- | --- |
| `XTEL` | placed references | door destination reference (bytes 0-3 of 32) |
| `XESP` | placed references | enable parent (bytes 0-3 of 8; flags and unused bytes preserved) |
| `XLKR` | placed references | keyword and linked reference (8 bytes), or the linked reference alone (legacy 4 bytes) |
| `XNDP` | placed references | navmesh (bytes 0-3 of 8) |
| `XEMI` | placed references | emitted light or region |
| `XATR` | placed references | attached reference (the in-house reader already remapped it; the legacy reader does since #269) |
| `SNAM`, `LNAM` | LIGH | sound and lens flare (the in-house reader already remapped them; the legacy reader does since #269) |
| `XAPR` | placed references | activate-parent reference (one 8-byte subrecord per parent) |
| `XLRT` | placed references | location reference types (array) |
| `XHOR` | ACHR | horse reference |
| `LTMP` | CELL, WRLD | lighting template |
| `XCIM`, `XCMO`, `XCAS` | CELL | image space, music type, acoustic space |
| `XCCM` | CELL | region the cell takes its sky and weather from |
| `XCLR` | CELL | regions (array) |
| `ZNAM` | WRLD | music (the in-house reader already remapped it; the legacy reader does since #269) |
| `RDWT` | REGN | weathers: (weather, chance, global) entries of 12 bytes; weather and global are FormIDs |
| `WLST` | CLMT | weathers, laid out like `RDWT` |
| `MNAM`, `NNAM`, `TNAM`, `GNAM` | WTHR | precipitation, visual effect, sky static, sun glare lens |
| `IMSP`, `HNAM` | WTHR | image spaces and volumetric lighting at sunrise, day, sunset, night (16 bytes) |
| `SNAM` | WTHR | sound descriptor (bytes 0-3 of 8; one subrecord per sound) |
| `XPOD` | REFR | the rooms a portal marker joins: an array of (origin, destination) pairs; every u32 a FormID (xEdit 4.1.5f defs) |
| `XLRM` | REFR | linked room of a room marker |
| `LNAM`, `INAM` | REFR | room marker lighting template, image space (other record types keep their own handling) |

For blob remapping, placed references are REFR, ACHR, ACRE, PGRE, PMIS, PHZD,
PARW, PBAR, PBEA, PCON and PFLA. This does not expand the record types exported
to the `references` table; the six newly covered types retain their blobs in
`records` without adding `references` rows.
Fields not covered may still hold plugin-local FormIDs. In particular, do not
consume `WRLD.RNAM` large-reference lists, `XLOC` lock keys (use `reference_details.lock_key_id`), `XPWR` water
reflections as resolved IDs. Other
unconverted fields include `XLIB`, `XMBR`, `XTNM`, `PDTO` and
`CELL.XILL`. Check the converter's field-specific handling before using
these payloads; this list is not exhaustive.

The fields in this table validate master indices and light-plugin local IDs.
An out-of-range index or light-plugin local ID wider than 12 bits is an invalid
optional link: the converter sets that FormID to zero and reports a warning,
aggregated per source plugin with the count and first record/field diagnostic.
Other records and valid links continue to convert. Malformed field lengths cause
the entire subrecord to be dropped and counted in the same per-plugin warning,
since its FormID offsets cannot be decoded safely. An array with an incomplete
entry is dropped in full; valid repeated subrecords are retained. Existing
validation of record headers and required fields is unchanged. Zero FormIDs
remain zero. Other bytes in retained subrecords (such as teleport coordinates,
enable flags, navmesh triangles and activation delays) are preserved. Unlisted fields remain opaque,
not an assurance that all FormIDs in arbitrary Skyrim or mod subrecords have
been resolved.

Existing packs must be converted again to obtain these corrected links. This
does not change the schema-4 table or blob layout, so the database and asset
cache versions are unchanged. Every pipeline run, including resume, rebuilds
the database from source plugins while reusing unaffected assets. Old packs
are still accepted by the runtime: schema 4 alone does not certify these links
were remapped. A future consumer of these fields must account for old packs.
`export_to_db` without a load order refreshes already-resolved records (for
example movement annotations), preserving established `formid_map` ownership;
it neither resolves plugin-local IDs nor upgrades an old pack.

```sql
如果不存在则创建表 formid_map (
    form_id INTEGER PRIMARY KEY, -- 解析的 32 位 Skyrim FormID
    plugin_name TEXT NOT NULL, -- 拥有插件文件名
    Internal_id INTEGER NOT NULL, -- 插件本地 ID（24 或 12 位）
    record_type TEXT NOT NULL -- 记录类型（'REFR'、'NPC_' 等）
）；
```

---

### 11. Asset Conversion Cache (`conversion_cache`)

Stores plugin file path hashes and timestamps to bypass re-converting unchanged files.

```sql
如果不存在则创建表 conversion_cache (
    plugin_path 文本主键，
    file_hash BLOB NOT NULL,
    last_converted INTEGER NOT NULL
）；
```

---

### 12. Point Light Sources (`lights`)

One row per `LIGH` base record: radius, colour, flags, falloff exponent and the
optional `FNAM` fade, which is what the runtime places a point light from. The
radius is a `DATA` `u32` in Creation units widened to a float, and the row is
written whether or not the record has a `MODL`: an invisible light still lights
the space, and most `LIGH` records in `Skyrim.esm` are invisible. A record whose
`DATA` is missing or holds fewer than the 20 bytes these columns need gets no
row rather than invented values, and a `LIGH` without a `MODL` gets no `statics`
row either - there would be no mesh to draw.

A reference that places a light usually carries its own `XRDS` radius, stored in
`references.radius_override` (10,810 of the 12,148 `LIGH` references in
`Skyrim.esm`), which overrides the base record's radius for that placement.

```sql
如果不存在则创建表灯（
    id INTEGER PRIMARY KEY, -- LIGH FormID
    editor_id 文本，
    radius REAL NOT NULL, -- 创建单位 (DATA u32)
    color_r 整数非空，color_g 整数非空，color_b 整数非空，
    flags INTEGER NOT NULL, -- DATA 标志（动态、可进位、负数、闪烁、默认关闭，...）
    衰减 REAL NOT NULL，
    fade REAL，--FNAM，如果存在
    time INTEGER, -- DATA[0] i32, 灯光持续时间
    field_of_view REAL, -- DATA[20]，聚光锥
    close_clip REAL，--数据[24]
    flicker_period REAL，-- DATA[28]，原始（xEdit 显示它乘以 0.01）
    flicker_intensity_amplitude REAL, -- 数据[32]
    flicker_movement_amplitude REAL, -- 数据[36]
    值 INTEGER, -- DATA[40] u32
    重量实数，--数据[44]
    sound_id INTEGER, -- SNAM -> SNDR
    lens_id INTEGER -- LNAM -> LENS（镜头光晕）
）；
```

The columns from `time` on were added for #269 gap 6. They come from the full 48-byte `DATA`
(xEdit 4.1.5f `wbDefinitionsTES5.pas`, `LIGH`); the in-house schema accepts only that length, so
a shorter `DATA` leaves them NULL for both readers, and floats that are not finite are NULL.
`sound_id` and `lens_id` keep a FormID only when the winning record exists with that type (the
in-house reader clears links that do not resolve and the legacy reader keeps them), and the legacy
reader now remaps `LIGH` `SNAM`/`LNAM` as the in-house schema already did. The columns are
additive and optional, with no `shared::WORLD_DATABASE_SCHEMA_VERSION` bump: `create_tables` adds
them to an older reused `lights` table. The engine reads radius, colour and flags only.

---

### 13. Water Definitions (`waters`)

Stores one row per `WATR` record: Skyrim's per-water colours and reflectivity, decoded from the
record's `DNAM` subrecord (offsets in `docs/research/water.md` section 1.2), plus the raw
subrecords for anything not broken out into a column.

```sql
如果不存在则创建表水域 (
    id INTEGER PRIMARY KEY, -- WATR FormID
    editor_id 文本，
    不透明度 INTEGER, -- ANAM, 0-100
    flags INTEGER NOT NULL, -- 记录头标志
    shallow_color INTEGER, -- DNAM+40: 打包 0x00BBGGRR
    deep_color INTEGER, -- DNAM+44: 打包 0x00BBGGRR
    Reflection_color INTEGER, -- DNAM+48: 打包 0x00BBGGRR
    菲涅耳 REAL, -- DNAM+24: 菲涅耳量 (Schlick F0)
    反射率 REAL, -- DNAM+20: 反射率量
    flow_normal_path TEXT, -- NAM5，规范化（仅限 SSE 流量图水域）
    data BLOB NOT NULL -- 序列化子记录负载
）；
```

`shallow_color`/`deep_color`/`reflection_color`/`fresnel`/`reflectivity` are additive columns: a
`skyrim_world.db` built before they existed has a `waters` table without them, and
`AssetCatalog::water_colors` (`crates/engine/src/world/database.rs`) returns `None` for every
water against such a database rather than failing to open it. The engine falls back to Skyrim's
DefaultWater values (`render::DEFAULT_WATER_FRESNEL` / `render::DEFAULT_WATER_REFLECTIVITY`, and
`streaming.rs`'s deep-colour constant) until a reconversion populates them. Adding them did **not**
bump `shared::WORLD_DATABASE_SCHEMA_VERSION`: nothing that already reads `waters` depends on their
presence, and the fallback exists specifically so a reconversion is not required.

---

### 14. Grass Definitions and Landscape Associations (`grass_types`, `landscape_texture_grasses`)

`grass_types` projects each effective `GRAS` record, including its canonical converted
model path (`MODL` through the existing mesh asset-path mapping). Grass models use the
same NIF-to-GLB converter as other meshes. `flags` below contains the `DATA` grass
flags rather than record header flags. `load_order` identifies the winning
override; `formid_map` retains owning-plugin identity independently.

The layout follows [xEdit's TES5 GRAS and LTEX definitions](https://github.com/TES5Edit/TES5Edit/blob/dev-4.1.5/Core/wbDefinitionsTES5.pas)
and [CommonLibSSE-NG's TESGrass definition](https://github.com/CharmedBaryon/CommonLibSSE-NG/blob/main/include/RE/T/TESGrass.h).
`DATA` is 32 bytes; its padding and unprojected fields stay in `records.data`.
The converter preserves authored values without clamping to vanilla ranges. A field
whose bytes are missing, or a non-finite float, becomes NULL. Missing, empty or unsafe
model paths also become NULL; the original subrecords remain available for diagnostics.
Path normalization does not verify that the referenced asset exists. Even vanilla
records can name an absent mesh, so a non-NULL `model_path` is not an availability
guarantee. Grass consumers must tolerate missing models, skip the unavailable
model with a useful diagnostic, and continue processing other grass types.

```sql
如果不存在则创建表grass_types (
    id INTEGER PRIMARY KEY, -- 已解析的 GRAS FormID
    editor_id 文本，
    model_path TEXT, -- 来自 MODL 的规范网格/...glb
    密度 INTEGER, -- DATA+0: u8
    min_slope INTEGER, -- DATA+1: u8, 度数
    max_slope INTEGER, -- DATA+2: u8, 度数
    units_from_water INTEGER, -- DATA+4: u16，距水位的距离
    water_comparison INTEGER, -- DATA+8: u32，下面的枚举
    位置范围 REAL，-- 数据+12：f32
    height_range REAL, -- DATA+16: f32
    color_range REAL, -- 数据+20: f32
    wave_period REAL, -- DATA+24: f32
    标志 INTEGER, -- DATA+28: u8
    load_order INTEGER NOT NULL
）；

如果不存在则创建表 Landscape_texture_grasses (
    ltex_id INTEGER NOT NULL, -- 已解析的 LTEX FormID
    gras_id INTEGER NOT NULL, -- 已解析 GNAM GRAS FormID
    主键（ltex_id、gras_id）
）；
```

`water_comparison` values 0 through 7 mean Above At Least, Above At Most,
Below At Least, Below At Most, Either At Least, Either At Most,
Either At Most Above and Either At Most Below, respectively. `flags` bits 0,
1 and 2 mean Vertex Lighting, Uniform Scaling and Fit to Slope. These definitions
establish the authored contract, not Skyrim's exact placement equations.

Each non-null repeated `LTEX.GNAM` yields an association; duplicate pairs collapse.
The winning LTEX list replaces the earlier list in full. There is no foreign key on
the grass target: a deleted or unresolved grass can still be named by a texture,
and consumers resolve it against `grass_types`. Both projections refresh atomically
from a complete effective load-order export, removing stale grass and associations
on deletions or later overrides. Deleted GRAS records therefore do not reappear as
grass definitions. The subset export used by movement annotation preserves unrelated
grass rows; if it includes an LTEX, only that texture's association list is replaced.
Movement annotation continues to accept schema 4 through the current schema and
preserves the existing database version; it does not perform a full reconversion.

---

### Other Runtime and In-House Tables

These tables are written by `create_tables` / the export and had no DDL here before #269. Their
DDL below is copied from `crates/converter/src/esm/exporter.rs` and
`crates/converter/src/esm/inhouse*.rs`; `statics` is documented with `references` (§5) by #270.

```sql
如果不存在则创建表 npcs ( -- 每个 NPC_ 一行
    id 整数主键，editor_id 文本，
    全名 TEXT, -- FULL
    race_id INTEGER, -- RNAM
    class_id INTEGER, -- CNAM
    flags INTEGER NOT NULL -- 记录头标志
）；
如果不存在则创建索引 idx_npcs_editor_id ON npcs(editor_id);

如果不存在则创建表 move_types ( -- MOVT 0003580D (NPC_Default_MT) 仅限今天
    id 整数主键，
    editor_id TEXT NOT NULL, -- 写为文字“NPC_Default_MT”
    left_walk REAL NOT NULL, left_run REAL NOT NULL, -- SPED, 每秒创建单位
    right_walk REAL NOT NULL，right_run REAL NOT NULL，
    forward_walk REAL NOT NULL，forward_run REAL NOT NULL，
    back_walk REAL NOT NULL，back_run REAL NOT NULL，
    load_order INTEGER NOT NULL
）；
如果不存在则创建表 race_movement_links ( -- 每场比赛一行
    id 整数主键，
    walk_movt_id INTEGER, -- WKMV
    run_movt_id INTEGER, -- RNMV
    load_order INTEGER NOT NULL
）；
如果不存在则创建表 move_game_settings ( -- 仅 fMoveCharWalkBase 和 fJumpHeightMin
    id 整数主键，editor_id 文本不为空，
    值 REAL NOT NULL，load_order INTEGER NOT NULL
）；

CREATE TABLE IF NOT EXISTStexture_sets ( -- 每个 TXST 一行；序言中的槽角色
    id 整数主键，editor_id 文本，
    扩散路径文本，--TX00
    正常路径文本，--TX01
    发光路径文本，--TX03
    高度路径文本，--TX04
    环境路径文本，--TX05
    mask_path文本，--TX02
    镜面反射路径文本，--TX07
    详细路径文本——TX06
）；
CREATE TABLE IF NOT EXISTS Landscape_textures ( -- 每个 LTEX 一行
    id 整数主键，editor_id 文本，
    texture_set_id INTEGER, -- TNAM -> TXST
    material_type INTEGER, -- MNAM：材料类型的 FormID (MATT)，不是数字
    摩擦 REAL, -- HNAM 字节 0
    恢复 REAL -- HNAM 字节 1
）；
```

The in-house reader also writes these auxiliary tables (diagnostic and provenance data; the
engine does not read them):

```sql
CREATE TABLE IF NOT EXISTS inhouse_source_records ( -- 原始解压源字节
    form_id 整数主键、source_form_id 整数非空、load_order 整数非空、
    record_type TEXT NOT NULL，标志 INTEGER NOT NULL，负载 BLOB NOT NULL
）；
CREATE TABLE IF NOT EXISTS inhouse_localized_fields ( -- 它遇到的每个本地化字符串字段
    form_id INTEGER NOT NULL，field_index INTEGER NOT NULL，
    签名 TEXT NOT NULL，field_name TEXT NOT NULL，
    string_table TEXT NOT NULL, -- 字符串、dlstrings 或 ilstrings
    string_id 整数非空，load_order 整数非空，
    已解决的文本文本，
    状态 TEXT NOT NULL CHECK(status IN ('已解决','null_id','缺失')),
    主键（表单 ID，字段索引）
）；
如果 inhouse_terrain_source_order 不存在则创建表（
    form_id 整数主键，source_record_offset 整数非空
）；
CREATE TABLE inhouse_terrain_layers ( -- 每次完成导出时重新创建
    form_id 整数主键，cell_id 整数非空，
    load_order INTEGER NOT NULL，layer_data BLOB NOT NULL
）；
CREATE TABLE inhouse_unresolved_terrain_textures ( -- 无法解析的 LAND 图层纹理
    form_id 整数非空，texture_form_id 整数非空，
    主键（form_id，texture_form_id）
) 没有 ROWID；
```

---

### 15. Interior Room Graph (`room_bounds`, `room_portals`, `room_links`, `reference_rooms`, `room_graph_stats`)

Skyrim marks the rooms of an interior with `RoomMarker` references (base `0x1F`) and the
doorways between them with `PortalMarker` references (base `0x20`). The converter decodes both
and records which room boxes each placed object belongs to. The tables hold facts only: which
objects to draw is the engine's policy (design: `docs/research/interior-room-graph-converter.md`,
upstream issue #188). Only interior cells and only box rooms (`XPRM` shape 1) are written;
exterior markers are skipped silently, and rooms of other shapes, and rooms or portals with a non-finite size or transform, are skipped and counted in the conversion report.
All FormIDs are in load-order numbering, like `references.id`.
The two marker base forms are the ones `Skyrim.esm` defines (always load-order slot 0, so the numbers do not change with the load order). A plugin that places rooms with a marker base of its own is not recognised and produces no rows; the conversion report does not count that case.

```sql
CREATE TABLE IF NOT EXISTS room_bounds ( -- 每个内部 RoomMarker 一行，带有一个框 XPRM
    ref_id INTEGER PRIMARY KEY, -- 标记的references.id
    cell_id 整数非空，
    pos_x REAL NOT NULL, pos_y REAL NOT NULL, pos_z REAL NOT NULL, -- 创建单位
    rot_x REAL NOT NULL, rot_y REAL NOT NULL, rot_z REAL NOT NULL, -- 弧度，作为参考
    half_x REAL NOT NULL, half_y REAL NOT NULL, half_z REAL NOT NULL, -- XPRM[0..12]
    lighting_template_id INTEGER, -- LNAM, 缺失时为 NULL
    image_space_id INTEGER, -- INAM, 缺失时为 NULL
    标志 INTEGER NOT NULL DEFAULT 0 -- XRMR 标志字节 | (XRMR u16 << 8)，保持原始状态
）；
如果不存在则创建索引 idx_room_bounds_cell ON room_bounds(cell_id);

CREATE TABLE IF NOT EXISTS room_portals ( -- 具有 XPRM 的内部 PortalMarker 的每个 XPOD 对一行
    ref_id INTEGER NOT NULL, -- 标记的references.id；不唯一，请参阅pair_index
    pair_index INTEGER NOT NULL DEFAULT 0, -- XPOD 中（起点、终点）对的位置
    cell_id 整数非空，
    pos_x REAL NOT NULL, pos_y REAL NOT NULL, pos_z REAL NOT NULL,
    rot_x REAL NOT NULL、rot_y REAL NOT NULL、rot_z REAL NOT NULL、
    half_x REAL NOT NULL, half_z REAL NOT NULL, -- 门口半宽半高
    room_a INTEGER, -- 该对的起源； 0 时为 NULL，不存在或不存在该单元格的房间
    room_b INTEGER, -- 配对的目的地；一样
    主键（ref_id，pair_index）
）；
如果不存在则创建索引 idx_room_portals_cell ON room_portals(cell_id);

如果不存在，则创建表 room_links ( -- XLRM：在没有门户的情况下一起看到的房间
    room_ref 整数非空，
    linked_ref 整数非空，
    主键（room_ref、linked_ref）
）；

CREATE TABLE IF NOT EXISTS reference_rooms ( -- 房间成员资格，仅包含房间的室内装饰
    Reference_id INTEGER NOT NULL, --references.id
    room_ref INTEGER NOT NULL, -- room_bounds.ref_id
    包含 INTEGER NOT NULL, -- 1: 在框内旋转； 0：仅边界与其重叠
    主键（reference_id、room_ref）
) 没有 ROWID；
如果不存在则创建索引 idx_reference_rooms_room ON reference_rooms(room_ref);

CREATE TABLE IF NOT EXISTS room_graph_stats ( -- 转换报告计数器，每个名称一行
    名称文本主键，
    值 INTEGER NOT NULL
）；
```

Columns and rules:

- **`room_bounds`**: the box is centred on the marker's position and oriented by its rotation
  (the same Euler convention as any reference, `shared::coordinates`). `half_*` are the `XPRM`
  half-extents in the room's own frame. `lighting_template_id` (`LNAM`) and `image_space_id`
  (`INAM`) are decoded for the lighting work in #129-#135; nothing in the engine uses them yet.
- **`room_portals`**: a portal's doorway is the marker's local X-Z rectangle (normal along local
  +Y); `half_x`/`half_z` are its half-width and half-height. A portal marker without `XPRM`
  has no size and gets no row. `XPOD` is an array of 8-byte (origin, destination) FormID pairs
  (xEdit 4.1.5f defs, `Core/wbDefinitionsTES5.pas`: `wbArray(XPOD, 'Portal Data',
  wbStruct('References', [wbFormIDCk('Origin', [REFR, NULL]), wbFormIDCk('Destination',
  [REFR, NULL])]))`), so a portal gets **one row per pair**, numbered by `pair_index` from 0
  in stored order; the geometry columns repeat on every row. A portal whose `XPOD` is absent,
  empty or shorter than one pair gets one row with `pair_index` 0 and both sides NULL. Bytes
  after the last whole pair are ignored. Every `XPOD` in the official masters is one pair, so
  vanilla output has `pair_index` 0 only. **Readers must not assume `ref_id` is unique**: join
  or group on `(ref_id, pair_index)`, or on `ref_id` and expect several rows. The key changed
  from `ref_id` alone, and `create_tables` migrates an older-shaped `room_portals` (no
  `pair_index`) in place, inside one savepoint: every row is kept with `pair_index` 0 until the
  next full export rebuilds the table.
  In the plugin remap `XPOD` is an array of FormIDs: every four bytes are remapped, and a
  length that is not a multiple of four is dropped with a warning, like any other FormID array.
- **`room_links`**: one row per `XLRM` as stored on the room, with no mirror rows. A link joins
  rooms that may have no portal between them but must be seen together: if one is visible, so is
  the other. The target is stored as written, even if it is not a `room_bounds` row.
- **`reference_rooms`**: written for every placed object (`REFR`, `ACHR`, `ACRE`, `PGRE`,
  `PMIS`) in an interior cell that has at least one room, except the room and portal markers
  themselves. `contained = 1` rows come from the ESM export: the object's pivot (its position)
  lies inside the box, edges included. `contained = 0` rows are added after model bounds are
  known (the integration stage, once `statics.bounds_*` come from the converted GLBs): the
  pivot is outside, but the object's bounds, placed with the reference's scale, rotation and
  position, overlap the box. `statics.bounds_*` are glTF-axis (Y-up) numbers in Creation units,
  with no metre scale. The test is conservative (the placed bounds' corners, taken into the room
  frame, as an axis-aligned box against the half-extents), so it can report an overlap that does
  not exist, which only makes the engine draw more. A `contained = 1` row is never also written
  as `contained = 0`.
  Overlap rows need a base with bounds in `statics`, which holds every base type with a world
  model (`STAT`, `MSTT`, `FURN`, `TREE`, `FLOR`, `CONT`, `DOOR`, `ACTI`, `TACT`, `APPA`, `ADDN`,
  `ARTO`, placed items and armour, and `LIGH` with a model; see §5). Placed references of other base types get pivot rows only: actors
  (`NPC_`, which move and stay ordinary entities), lights without a model (their reach is a
  lighting question, not geometry), decals (`TXST`), and bases with no geometry (`IDLM`, `SOUN`,
  `ASPC`). In the all-plugins conversion these were 22,251 of 364,262 pivot rows (measured before `TACT`, `APPA`, `ADDN` and `ARTO` joined `statics`).
- **`room_graph_stats`**: the counters the conversion report shows in `integration-report.json`
  under `room_graph`: `rooms`, `portals` (portal markers written, not rows), `links`,
  `portals_with_missing_side` (`room_portals` rows with a NULL side, so a portal with several
  pairs can add several), `portals_with_extra_pairs` (markers whose `XPOD` holds more than one
  pair), `portals_with_trailing_xpod_bytes` (markers whose `XPOD` length is not a multiple of 8),
  `non_box_rooms_skipped`, `portals_without_size_skipped`, `invalid_markers_skipped` (rooms and portals with a non-finite size or transform), `objects_in_no_room` and
  `objects_in_several_rooms` (both by pivot), and `overlap_rows`.

The tables are derived data: a full load-order export clears and rebuilds them from the winning
records on every conversion, like the grass tables, and the subset export used by movement
annotation leaves them alone. `create_tables` adds them to an older reused database.

**These tables are optional and did not bump `shared::WORLD_DATABASE_SCHEMA_VERSION`.** No engine code reads
them yet; a reader must probe for them first, as the engine does for `lights` and the additive
`waters` columns, so a database without them behaves as before.

| Case | Converter writes | Engine |
|---|---|---|
| Interior without room markers | no rows | draw everything, as today |
| Old database, tables absent | n/a | feature off, no error |
| Portal side 0, missing or not a room | `room_a`/`room_b` NULL | ignore that edge |
| Object in no room / several | no `contained = 1` row / several | always drawn |
| Camera outside every room | n/a | draw everything |
| Non-box room shape | skipped, counted | its objects are in no room, so drawn |
| Exterior markers | not written | out of scope (#106) |

---

### 16. Reference Emittance Sources (`emittance`)

A NIF shader with `SLSF1_External_Emittance` (bit 29) takes its emission colour from outside the
model: from the light or region the placing reference names in `XEMI`
(`references.emittance_id`). Skyrim keeps building windows dark by day this way. The Sleeping Giant
Inn's reference in Riverwood (`0002D763`) names `FXLightRegionInvertWindowWhiterun` (`0008277B`),
whose one weather `FXWthrInvertWindowsWhiterun` (`0008277A`) has a black Effect Lighting colour by
day and an amber one at night (#242).

The complete load-order export rebuilds the table, writing one row per distinct `XEMI` target it can
resolve, as 8-bit sRGB at sunrise, day, sunset and night:

- `LIGH`: the `DATA` colour (bytes 8-10), the same at every time of day.
- `REGN`: the `RDWT` weather with the highest chance (the first on a tie), and that weather's
  `NAM0` Effect Lighting colour, the tenth colour type. `NAM0` holds 16 bytes per colour type
  (sunrise, day, sunset and night, each RGB and an unused byte); the official plugins store 13,
  14 or 17 types. A region changes weather with its climate, which the runtime does not model, so
  its likeliest weather stands in.

Effect Lighting rather than Sunlight: the inn's window shader has a white tint and a neutral grey
glow map, so the warm colour of Skyrim's lit windows at night can only come from the source. Both
colours are black by day for the window regions.

A target that is missing, deleted, not a `LIGH` or `REGN`, or whose weather has no Effect Lighting
colour gets no row and a warning; the runtime then keeps the model's own emission. The movement
annotator's subset export does not touch this table.

Measured on Skyrim SE with its official plugins: 9,655 references carry `XEMI`, naming 211
sources. 166 lights and 31 regions resolve; 14 regions list no weather at all (the largest,
`RiverwoodMountainsSnow` `0002E2CC`, is named by 479 references), and what Skyrim lends from such
a region is unverified, so their 651 references keep the model's emission.

```sql
如果不存在则创建表发射率 (
    id INTEGER PRIMARY KEY, -- XEMI 目标：LIGH 或 REGN FormID
    source_type TEXT NOT NULL, -- 'LIGH' 或 'REGN'
    Weather_id INTEGER, -- REGN：其颜色来自的 WTHR
    sunrise_r 整数非空，sunrise_g 整数非空，sunrise_b 整数非空，
    day_r 整数非空，day_g 整数非空，day_b 整数非空，
    Sunset_r 整数非空，sunset_g 整数非空，sunset_b 整数非空，
    night_r 整数非空， night_g 整数非空， night_b 整数非空
）；
```

The engine reads the day columns, its one time of day; see
[`color-pipeline.md`](../engine/color-pipeline.md#external-emittance-242).

**The table and `references.emittance_id` are optional and did not bump
`shared::WORLD_DATABASE_SCHEMA_VERSION`**, for the same reason as the room tables: they only add a
table and a column, and the engine probes for both. `create_tables` adds `emittance_id` to an older
reused `references` table. A database without them loads with every reference unlit by `XEMI`, so
`External_Emittance` surfaces keep the model's own emission.

---

### 17. Load Door Links (`door_links`)

A load door is a `REFR` of a `DOOR` base record that carries an `XTEL` subrecord. `XTEL` is 32
bytes in Skyrim SE: the destination door reference (FormID, bytes 0..4), the arrival position
(three `f32`, bytes 4..16), the arrival rotation (three `f32` radians, bytes 16..28) and a `u32`
flags word (bytes 28..32, `1` = no alarm). The arrival frame is the one the player is placed in
on this side's destination, not the position of the destination door. Both record readers keep
the 32 bytes unchanged in `records.data` and remap the destination FormID with the load order
(`form_id_layout` in `crates/converter/src/esm/mod.rs`; the in-house schema describes the
subrecord as `teleport` in `crates/converter/src/records/schema.json`). The `door_links` table is
projected from those canonical `records` and `references` rows, not from either reader, so the
legacy and in-house frontends publish identical rows.

```sql
CREATE TABLE IF NOT EXISTS Door_links ( -- 每个装载门一行可跟随
    ref_id INTEGER PRIMARY KEY, -- 门的references.id
    destination_ref_id INTEGER NOT NULL, -- XTEL[0..4]，在 FormID 重新映射之后
    pos_x REAL NOT NULL, pos_y REAL NOT NULL, pos_z REAL NOT NULL, -- XTEL[4..16], 创建单位
    rot_x REAL NOT NULL, rot_y REAL NOT NULL, rot_z REAL NOT NULL, -- XTEL[16..28]，弧度
    destination_cell_id INTEGER NOT NULL, -- 持有目的地门的单元格的 cells.id
    destination_worldspace_id INTEGER, -- 该单元格的世界空间，对于内部空间为 NULL
    destination_name TEXT -- 内部单元完整（否则 EDID），或世界空间 EDID
）；
```

A door is skipped, with no row, when it has no `XTEL`, when the `XTEL` is not exactly 32 bytes,
when the destination is `0`, when any arrival value is not finite, or when the destination door
is not a placed `DOOR` reference in a stored cell (an exterior cell needs its grid). The winning
record decides: an override that removes `XTEL` removes the link. The `XTEL` flags word is not
projected; it stays in `records.data`.

A full load-order export clears and rebuilds the table on every conversion, like the grass and
room tables. The movement-annotation subset export leaves it alone, and a diagnostic archive
(which defers every runtime view) leaves it empty. `create_tables` adds the table to an older
reused database, and adds `destination_name` to a table that predates it.
`door-links-annotate <skyrim_world.db>` rebuilds it in one transaction from a converted
database's own winning records, with no plugin reparsed, so an existing package gains doors
without a full conversion. It takes the asset lock exclusively (close the engine first), accepts
schemas 4 through the current one, and leaves meshes, terrain, `schema_info`, reader identity and
the other tables unchanged. A corrupt `records` row fails the rebuild and keeps the old table.

**This table is optional and did not bump `shared::WORLD_DATABASE_SCHEMA_VERSION`**, for the same
reason as the room tables: it only adds a table, and the engine probes for it. The engine reads
it through a `LEFT JOIN` on `references`, only when the table exists with every column above.

| Case | Converter writes | Engine |
|---|---|---|
| Old database, table absent | n/a | logs once, doors stay scenery |
| Table of another shape (missing column) | n/a | warns, doors stay scenery |
| Door without `XTEL`, or with a malformed one | no row | scenery |
| Destination cell has no resolvable record | no row | scenery |
| Row with a NULL arrival value | n/a | not a load door |

---

### 18. Actor Model Catalog (`actor_models`, `actor_model_parts`)

Every non-deleted `NPC_` base in the merged record map gets one `actor_models` row describing its
race, sex, effective skin armor, body/head resolution and head path, plus one `actor_model_parts`
row per biped slot the effective skin resolves. The catalog is resolved once, after the record
loop, from the merged record map; the tables are projected from canonical records, not from either
reader's raw `records.data`, so the legacy and in-house frontends publish identical rows.

```sql
如果不存在则创建表 actor_models (
    id INTEGER PRIMARY KEY, -- NPC_ FormID
    race_id 整数非空，
    性别 TEXT NOT NULL, -- '男性' | ‘女’
    Skin_armo_id INTEGER, -- 有效的 NPC WNAM 覆盖，否则有效的 RACE/WNAM
    head_model TEXT, -- FaceGen 或默认头部 NIF 路径
    head_source TEXT, -- 'facegen' | “默认”
    分辨率 TEXT NOT NULL, -- 'body', 'body_head', '未解决'
    Skin_tone_r REAL, -- NPC_/QNAM, 0..1, 编码比例
    肤色_g 真实，
    Skin_tone_b 真实，
    重量实数 -- NPC_/NAM7, 0..100
）；
如果不存在则创建表 actor_model_parts (
    npc_id 整数非空，
    槽 TEXT NOT NULL, -- 'body' | '手' | ‘脚’
    model_path 文本不为空，
    armature_race_id INTEGER, -- ARMA/RNAM：NPC 的种族或 DefaultRace (25)
    Skin_texture_set_id INTEGER，--ARMA/NAM0（男）或 NAM1（女），TXST
    主键（npc_id，插槽）
）；
```

**Traits.** When `NPC_/ACBS` `u16` at offset 18 has bit 0 set, `TPLT` is followed while each
target also has that bit set. The final Traits root supplies race (`RNAM`), sex (`ACBS` bit 0 at
offset 0), and skin override (`WNAM`). A Traits-enabled NPC's own valid `WNAM` is ignored. A
visited set detects cycles; there is no fixed depth limit. If the chain cannot resolve, the
original NPC still gets one unresolved row with `race_id = 0`, placeholder `sex = 'male'`, no
skin, parts, or head. Diagnostics identify missing, deleted, wrong-type, cyclic, or `LVLN`
targets. The detailed source and evidence boundaries are in
[`actor-catalog-inheritance-policy.md`](../../research/actor-catalog-inheritance-policy.md).

**Armature skin texture set.** The game draws a part's `skin_tint` shapes with the armature's skin
`TXST` for the NPC's sex (`NAM0` male, `NAM1` female), not with the part NIF's own texture set:
retail `femalehands_1.nif` has no `_s`, while `NakedHands` (`00000D6C`) gives female hands
`SkinHandFemale_1` (`0003EDE9`), whose `TX07` is `FemaleHands_1_S.dds`. `skin_texture_set_id`
stores that FormID (`NULL` when the armature has none for the sex; the other sex's set is not
used), and `armature_race_id` the armature's own `RNAM`, so `25` marks a part that came from the
shared `DefaultRace` armature. The converter composes the default humanoid body from the
`DefaultRace` sets: per sex and slot, the set most parts with `armature_race_id = 25` use, applied
in NIF slot order (`TX00`→0, `TX01`→1, `TX03`→2, `TX04`→3, `TX05`→4, `TX02`→5, `TX06`→6,
`TX07`→7; slots the set leaves empty keep the NIF's path) when the armature's model file is the
part being composed. Both columns arrived after the first catalog export without a schema bump;
`create_tables` adds them to an older reused `actor_model_parts`, and a database without them
composes the body from the NIFs' own textures.

**Skin tone.** `NPC_/QNAM` is three little-endian `f32` (red, green, blue), stored as written:
0..1 on the encoded scale (retail values are whole steps of 1/255; Alvor `00013475` is
198,176,168). The game passes it to the `skin_tint` shaders of the body, hands and feet as their
tint colour. A base without `QNAM`, or with a short or non-finite one, stores `NULL` in all three. Since
QNAM is not a confirmed Traits inheritance field, a Traits-enabled source NPC stores `NULL` even
when it or its resolved root carries QNAM; this conservative catalog policy avoids inventing root
inheritance.
The columns arrived after the first catalog export without a schema bump: `create_tables` adds
them to an older reused `actor_models`, and the engine probes for them, so a database without them
still loads and its actors stay untinted.

**Body weight.** `NPC_/NAM7` is one little-endian `f32` in 0..100: the weight the game blends
between the thin (`_0`) and heavy (`_1`) body meshes (0 is thinnest, 100 is heaviest). The column
is nullable; a base with no `NAM7`, or with a short, non-finite or out-of-range one, stores `NULL`
and the engine draws the heavy body. The weight comes only from the base's own record: `TPLT`
template inheritance is not resolved for it, matching `QNAM` and `RNAM`. The column arrived after
the first catalog export without a schema bump (`WORLD_DATABASE_SCHEMA_VERSION` stays 9):
`create_tables` adds it to an older reused `actor_models`, and the engine probes for it, so a
database without it still loads.

**Sex.** `NPC_/ACBS` bit 0 (`1` = female). A base with no `ACBS` defaults to male and records a
`missing_acbs` diagnostic.

Without Traits, the NPC supplies its own race and sex. A missing effective-root `ACBS` defaults
to male and records `missing_acbs`.

**Body resolver chain.** Effective Traits-root `NPC_/RNAM` -> the `RACE` record -> effective skin
armor -> every `ARMO/MODL` armature in authored order -> each `ARMA`:

1. Skin is the effective Traits-root NPC's `WNAM` override when it names a live `ARMO`; otherwise
   it falls back to the effective race's `WNAM`. An invalid explicit override records its
   diagnostic and uses that race fallback. This is a conservative catalog policy, not proof of
   Skyrim engine fallback behavior.
2. An armature is compatible when its primary `ARMA/RNAM` equals the effective race, or a
   resolvable additional-race `ARMA/MODL` names that race. These matches share one tier and keep
   authored `ARMO/MODL` order.
3. Compatible armatures are considered before armatures whose primary `RNAM` is `25`
   (`DefaultRace`), whatever order `ARMO/MODL` lists them in. If neither tier matches, no part is
   written and `no_matching_arma` is recorded (the resolver never picks the first `ARMA`).
4. `MOD2` is the male path. Female NPCs use `MOD3`, falling back to `MOD2` when `MOD3` is absent
   or empty.
5. The slot is the armature's biped bit: bit 2 = body, bit 3 = hands, bit 7 = feet (slot number
   minus 30). The resolver reads `ARMA/BOD2` when present and falls back to `ARMA/BODT`, which is
   the field retail Skyrim's `ARMA` records actually carry (the same `u32` slot word at offset 0;
   `ARMO` is the record with `BOD2`). An armature with neither recognized bit is rejected loudly
   with an `unrecognized_slot` diagnostic; a slot is never inferred from the filename.
6. Paths are deduplicated case-insensitively, and a slot already filled keeps the higher-priority
   armature.

**Head.** The `RACE/DATA` `facegen_head` flag (bit 1 of the flags word at offset 32) selects the
FaceGen geometry `meshes/actors/character/facegendata/facegeom/<plugin>/<formid>.nif`, lowercased,
with `<plugin>` and plugin-local FormID from the effective Traits root's defining-owner
`formid_map` identity (`plugin_name`/`internal_id`). The local ID uses eight-digit lower-case hex
without the load-order slot prefix, so the runtime mod index never appears in the file name
(`head_source = 'facegen'`). Using the Traits root is a conservative catalog policy, not proof of
engine copy-body or FaceGen ownership behavior. Otherwise, when the
resolved body is humanoid (a part path contains `malebody` or `femalebody`), the race/sex default
head is used (`head_source = 'default'`):
`meshes/actors/character/character assets/` plus `malehead.nif`/`femalehead.nif`, with the
`maleheadargonian.nif`/`maleheadkhajiit.nif` variants when the race EditorID names Argonian or
Khajiit. A creature body never gets a humanoid default head.

**Path bases.** `head_model` is archive-root-relative: it starts with `meshes/` and uses forward
slashes. `actor_model_parts.model_path` preserves the authored `ARMA/MOD2` or `ARMA/MOD3`
string, including its case and separators. Retail part paths are relative to `meshes/` and
omit that prefix; consumers must normalize these authored paths before asset lookup.

This slice computes the head *path string* only; it does not compose a head NIF and does not treat
`HDPT/PNAM` as the head mesh. FaceGen presence is the race flag, not a per-NPC file probe: the ESM
export path has no asset index, so PR-6 validates the path before loading.

**Reason codes.** A base that fails to resolve records the reason it was rejected: `no_rnam`,
`no_race_record`, `race_no_wnam`, `no_armo_record`, `armo_no_modl`, `no_matching_arma`. Traits
resolution and skin overrides add nine codes: `missing_traits_template`, `deleted_traits_template`,
`wrong_type_traits_template`, `traits_template_cycle`, `leveled_traits_template`,
`zero_npc_skin_override`, `missing_npc_skin_override`, `deleted_npc_skin_override`, and
`wrong_type_npc_skin_override`. A base that reaches head resolution but has no head path also
records `no_facegen_and_no_default`; early returns for unresolved Traits, missing race links, or
missing race records do not. The reasons are not mutually exclusive, and a report that counts
them counts a base once per reason.
`unrecognized_slot` and `missing_acbs` are recorded wherever they occur. Diagnostics are printed
by the exporter and returned by `converter::actor_models::resolve_actor_models`; they are not
stored in the database.

**Resolution.** `body_head` when at least one part and a head path resolve, `body` when parts
resolve without a head, and `unresolved` otherwise. A base with no effective-root `RNAM` is still
written with `race_id = 0` and `resolution = 'unresolved'`. Any unresolved Traits chain also
produces one row for the original NPC with `race_id = 0`, placeholder `sex = 'male'`, and no parts
or head, so it is recorded rather than dropped.

**Ownership.** A complete load-order export drops and rebuilds both tables in the record
transaction. A subset export, such as movement annotation, leaves them alone: a partial record map
cannot resolve an `NPC_` whose race or skin it did not receive, so it must not overwrite a complete
row with an unresolved one. A diagnostic archive (which defers every runtime view) leaves both
tables empty. `NPC_` bases are never written into `statics`.

**These tables are optional and do not bump `shared::WORLD_DATABASE_SCHEMA_VERSION`** (still 9):
they only add tables, and consumers must check that they exist before reading them. No mandatory
join is added to `REFERENCE_JOIN`; a database without them still opens and renders statics.