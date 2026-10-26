# Mudcrab 域上下文

## 术语表和规范术语

### 1. 离线编译管道（`pipeline`）

提前批量转换系统，摄取原始 Bethesda 资产（`.esm`、`.esp`、`.bsa`、`.nif`、`.dds`、`.pex`）并将其编译为现代标准运行时资产（`SQLite 3`、`rkyv`、`glTF 2.0`、`KTX2`、`Luau`）。

### 2.统一主数据库（`skyrim_world.db`）

通过编译 `Skyrim.esm` 和所有活动插件覆盖 (`.esp`/`.esl`) 创建的单个 SQLite 3 数据库，按 `plugins.txt` 确定的优先顺序排列。具有匹配 `FormID` 的记录已预先合并，因此运行时读取为 `O(1)`，无需运行时插件冲突解决。

### 3.空间R树索引（`refr_spatial`）

SQLite 3 虚拟 R-Tree 表索引通过 3D 轴对齐边界框 (`minX, maxX, minY, maxY, minZ, maxZ`) 放置世界引用 (`REFR`)。用于超快的亚毫秒级单元相机可见性查询。

### 4. 零复制地形缓存 (`cell_cache.rkyv`)

内存映射 (`mmap`) 二进制缓冲区使用 `rkyv` 零拷贝序列化处理高密度空间数据（例如来自 `LAND` 记录的 33x33 高度图网格和顶点法线），以允许直接从 RAM 进行 0-CPU 成本读取。