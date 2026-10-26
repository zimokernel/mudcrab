# Mudcrab 架构和路线图

使用 **Bevy 引擎** 在 **Rust** 中构建的 **上古卷轴 V：天际（特别版）** 的开源引擎重新实现（并且可扩展用于 Creation Engine / Gamebryo 游戏，如 Morrowind / Oblivion）。

受到 OpenMW (OpenMorrowind) 等项目计划的启发，Mudcrab 旨在为 Bethesda 引擎资产提供现代化、多线程、高性能的运行时。

---

## 1. 核心架构概述

```
┌────────────────────────────┐
                          │ 原始天际资产 │
                          │ (.esm, .esp, .bsa, .nif) │
                          └──────────────┬──────────────┘
                                        │
                                        ▼（离线翻译管道）
┌────────────────────────────────────────────────────────────────────────────┐
│ 现代转换资产存储 │
│ ┌────────────────────────┬────────────────────┬────────────────────┐ │
│ │ glTF 2.0 (.glb) │ KTX2 纹理 │ SQLite 3 数据库 │ │
│ │ (mesh_tools Exporter) │ (基础通用) │ (R-Tree + rkyv) │ │
│ └────────────────────────┴────────────────────┴────────────────────┘ │
└──────────────────────────────────────┬────────────────────────────────────┘
                                         │ (即时内存映射/多线程流)
                                         ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Mudcrab Bevy 引擎运行时 (ECS) │
│ ┌────────────────────────┬────────────────────┬────────────────────┐ │
│ │ Vercidium 实例 │ WebGPU / Vulkan │ Luau (mlua) JIT │ │
│ │ 间接渲染池 │ Bevy 0.19 bsn！用户界面 │ （Web API + 沙箱）│ │
│ └────────────────────────┴────────────────────┴────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Workspace Crate Architecture

During Phase 2, Mudcrab is organized into a **4-crate Cargo Workspace**. Phase 3 adds the isolated `scripting` crate as a fifth member:

```
泥蟹/
├── Cargo.toml # 工作空间根目录清单
├── 板条箱/
│ ├── 启动器/ # UI 应用程序（设置向导、模组管理器、启动器）
│ ├──转换器/ # 转换器管道 (.nif ➔ .glb, .dds ➔ KTX2, .esm ➔ libSQL, .pex ➔ Luau)
│ ├── shared/ # 转换器和引擎共享的版本化数据库/缓存合约
│ └── engine/ # Bevy 游戏引擎运行时（渲染管道、物理、音频、ECS 系统）
```

| Crate Name       | Binary / Crate Type | Key Responsibilities                                                                                                            |
| :--------------- | :------------------ | :------------------------------------------------------------------------------------------------------------------------------ |
| **`launcher`**   | GUI Executable      | First-run setup wizard, game path detection, built-in mod manager UI, triggers converter progress bar, and launches the engine. |
| **`converter`**  | Library & CLI       | Heavy transformation logic (`mesh_tools`, `basis-universal`, `ddsfile`, `nom` parsers). Invoked directly by `launcher`.         |
| **`shared`**     | Library             | Stable `rkyv` cache structures, schema versions, and converter/runtime data contracts. |
| **`engine`**     | Game Executable     | Light, hyper-fast game binary (Bevy 0.19+, `wgpu`, `libsql`). Orchestrates transformed game assets and Bevy ECS systems at 60+ FPS.     |

---

## 3. Modder Developer Experience (DX) & Type Safety

Mudcrab prioritizes a modern, developer-friendly modding ecosystem:

- **Isolated Scripting Crate (`crates/scripting`):** Prevents script runtime changes from forcing full game engine recompilations.
- **Phase boundary:** `crates/scripting` is introduced in Phase 3; the Phase 2 Papyrus compatibility module remains inside `engine` until that migration.
- **Type Definitions (`.d.lua` / EmmyLua / LuaLS):** Generates static type definitions for IDEs (VS Code, Zed, Neovim, Cursor), providing instant autocompletion, inline API docs, and static type checking.
- **Luarocks Package Distribution:** Enables distribution of type defs and community mod libraries via standard Lua package management tools.

---

## 4. Technology Stack & Tools

- **Language:** Rust (2024 Edition)
- **Engine Framework:** [Bevy](https://bevyengine.org/) (v0.19+)
- **Graphics Backend:** `wgpu` (Vulkan / Metal / DirectX 12 / WebGPU)
- **Scripting Engine:** Dedicated `scripting` crate (Luau via `mlua` crate with `luau-jit` feature)
- **Modder Tooling:** EmmyLua / LuaLS `.d.lua` type definitions + Luarocks package manager
- **3D Exporter:** `mesh_tools` (`GltfBuilder`)
- **Texture Encoder:** `basis-universal` + `ddsfile`
- **Database & Cache:** SQLite 3 (`rusqlite` + `rtree`) + `rkyv` zero-copy memory mapping
- **UI Engine:** Bevy 0.19 `bsn!` (Bevy Scene Notation) declarative macro trees