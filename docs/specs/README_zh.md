# 泥蟹技术规格 (`docs/specs/`)

**Mudcrab** 的全面技术架构规范、资产转换器管道、渲染优化和模组子系统设计。

---

## 📁 规格目录结构

```
文档/规格/
├── README.md # 规格目录索引
├──转换器/ # 离线资产转换规范
│ ├── pipeline.md # 资产和现代化管道概述
│ ├── esm-to-sqlite.md # 主数据库（SQLite 3、R-Tree、rkyv 缓存）
│ ├── nif-to-gltf.md # 3D 网格导出器（BSTriShape、蒙皮、PBR）
│ ├── dds-to-ktx2.md # 纹理转码器（基础通用KTX2）
│ ├── pex-to-lua.md # Papyrus 字节码到 Luau 转译器
│ └── data-formats.md # 旧版天际 vs 现代格式矩阵
├── engine/ # Bevy 引擎核心和渲染规格
│ ├── Architecture.md # 工作空间架构 & Bevy ECS 核心
│ ├── vercidium-optimizations.md # 实例化间接绘制池和 HZB 剔除
│ └── bevy-examples.md # Bevy 0.19 模式映射
├── modding/ # Mod 管理器、脚本和 UI 规范
│ ├── launcher.md # 设置向导和 GUI 启动器规范
│ ├── mods-and-ui.md # 内置 Mod 管理器和 Flash-to-Bevy bsn！用户界面
│ └── features.md # 解锁功能（AI对话、合作、零加载）
└── meta/ # 域和系统上下文
    ├── context.md # 领域术语表和规范定义
    ├──platforms.md # 平台兼容性目标矩阵
    └──requirements.md # 系统硬件要求对比
```

---

## 🛠️ Category Summary & Quick Links

### 🔄 1. Asset Converters (`converters/`)

Offline asset transpilation specs handled by the `converter` crate:

- 📄 **[`pipeline.md`](converters/pipeline.md)** — Master offline conversion architecture and pipeline matrix.
- 📄 **[`esm-to-sqlite.md`](converters/esm-to-sqlite.md)** — `Skyrim.esm` parsing strategy, libSQL sync, and `rkyv` zero-copy terrain cache.
- 📄 **[`db-schema.md`](converters/db-schema.md)** — Complete SQLite 3 database DDL schema (`skyrim_world.db`), table constraints, and indices.
- 📄 **[`nif-to-gltf.md`](converters/nif-to-gltf.md)** — Converting `.nif` geometry, material shaders, and skin weights to `glTF 2.0`.
- 📄 **[`dds-to-ktx2.md`](converters/dds-to-ktx2.md)** — DirectDraw surface transcoding to Basis Universal `KTX2`.
- 📄 **[`pex-to-lua.md`](converters/pex-to-lua.md)** — Papyrus `.pex` bytecode decompilation to sandboxed `Luau`.
- 📄 **[`data-formats.md`](converters/data-formats.md)** — Binary comparison matrix between legacy and modern asset formats.

### ⚙️ 2. Engine, Scripting & Rendering (`engine/`)

Core runtime specifications handled by the `engine` and `scripting` crates:

- **[`bug-reporting.md`](engine/bug-reporting.md)** — Local diagnostic bundles and reviewed GitHub bug-report workflow for #88.
- **[`landscape-texture-scale.md`](engine/landscape-texture-scale.md)** — Shared default LAND texture frequency, evidence, and terrain LOD agreement.
- **[`settings-ui.md`](engine/settings-ui.md)** — Native Bevy settings menu, rendering inventory, saved preferences, and controls reference.
- 📄 **[`architecture.md`](engine/architecture.md)** — 4-crate Cargo workspace structure, Bevy ECS setup, and `crates/scripting` isolation.
- 📄 **[`vercidium-optimizations.md`](engine/vercidium-optimizations.md)** — GPU instanced indirect rendering (`DrawMeshInstancedIndirect`) and GPU culling.
- 📄 **[`bevy-examples.md`](engine/bevy-examples.md)** — Code examples mapping Bevy 0.19 patterns to engine subsystems.

### 🎮 3. Modding, UI & Launcher (`modding/`)

Scripting, type definitions, UI, and launcher specifications:

- 📄 **[`launcher.md`](modding/launcher.md)** — First-run setup wizard and client launcher.
- 📄 **[`mods-and-ui.md`](modding/mods-and-ui.md)** — Integrated mod manager and Flash `.gfx` to Bevy `bsn!` UI node transpilation.
- 📄 **[`features.md`](modding/features.md)** — Unlocked engine innovations (Ollama LLM dialogue, zero loading screens, native co-op).

### 📐 4. System & Domain Metadata (`meta/`)

Project domain glossary and system benchmarks:

- 📄 **[`context.md`](meta/context.md)** — Mudcrab canonical terms and domain glossary.
- 📄 **[`platforms.md`](meta/platforms.md)** — Cross-platform target matrix (Windows, Linux, macOS, Android ARM64, WASM).
- 📄 **[`requirements.md`](meta/requirements.md)** — Official Skyrim SE vs Mudcrab hardware system specs comparison.