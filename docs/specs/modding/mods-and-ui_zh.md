# Mudcrab 模组架构和兼容性策略

《Skyrim》拥有 PC 游戏中最大的模组生态系统。本文档概述了 Mudcrab 如何处理旧版 Skyrim 模组（`.esp`、`.esm`、`.esl`、`.bsa`、网格/纹理覆盖、Papyrus 脚本），同时引入现代的原生 Lua 模组。

---

## 1. 双层模组架构

```
┌────────────────────────────────┐
                               │ 模组/用户设置 │
                               │ (VORTEX / MO2 / 数据目录) │
                               └────────────────┬────────────────┘
                                               │
                                               ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ Mudcrab Mod 摄取和沙箱 │
│ │
│ ┌────────────────────────────────┐ ┐────────────────────────────────┐ │
│ │ 第 1 层：旧版兼容性 │ │ 第 2 层：本机现代模组 │ │
│ │ - 将 `.esp`/`.esl` 解压到数据库 │ │ - 直接 glTF / KTX2 资源 │ │
│ │ - 即时 .nif/.dds 转换 │ │ - 原生 Luau 脚本 │ │
│ │ - Papyrus > Lua 转译 │ │ - 热重载和沙盒 │ │
│ └────────────────────────────────┘ └────────────────────────────────┘ │
└──────────────────────────────────────────────┬──────────────────────────────┘
                                               │
                                               ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ 统一的 Mudcrab 引擎运行时 │
│ (虚拟文件系统 + SQLite Mod 分层) │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Handling Legacy Skyrim Mods (`.esp`, `.esl`, `.bsa`, `.nif`, `.dds`)

### A. Mod Plugin Load Order & Database Layering (`.esp` / `.esl`)

In original Skyrim, `.esp` files override records from `Skyrim.esm` based on load order (e.g. `plugins.txt`).

- **SQLite Mod Layering:**
  - Mudcrab stores base game data in `skyrim_world.db`.
  - When a user adds an `.esp` mod, the converter imports the plugin's records into a mod table with a higher **load order priority weight**.
  - Database query:
    ```sql
SELECT * FROM refr_objects WHERE form_id = ? ORDER BY load_order DESC LIMIT 1;
    ```
  - **Benefit:** Instant record resolution with zero engine hacking required!

### B. Dynamic On-the-Fly Asset Conversion (Loose Files & BSA Mods)

- Mods containing loose `.nif` or `.dds` files are detected by the Virtual Filesystem (VFS).
- The converter transpiles mod `.nif` files to `.glb` and `.dds` textures to `.ktx2` in a local cache directory (`mod_cache/`).

### C. Legacy Papyrus Scripts (`.pex` Mods)

- `.pex` bytecode contained within mod `.bsa` archives or `Scripts/` folders is run through the **Papyrus-to-Lua AST Transpiler**.
- Mod functions call the unified Mudcrab Lua API bindings seamlessly.

## 3. Integrated Launcher Mod Manager & Storage Structure

Instead of relying solely on external tools, Mudcrab includes a **Built-in Mod Manager** inside the Launcher.

### Storage Hierarchy (`Mudcrab/`)

```
泥蟹/
├── game_data/ # 解压后的基础游戏和 DLC 资产
├──transformed_data/ # 转换后的基础游戏（glTF、KTX2、SQLite）
├── mods/ # 用户安装的原始mod文件夹（通过Launcher安装）
│ ├── ModA_HighResArmor/
│ └── ModB_CustomQuest/
├──transformed_mods/ # 转换后的mod缓存（glTF、KTX2、Lua）
└── config.json # Mod 加载顺序和活动插件列表
```

### Built-in Mod Manager Features:

- **Drag-and-Drop Installation:** Drop `.zip` / `.7z` / `.rar` mod archives directly into the Launcher.
- **Visual Load Order Manager:** Drag and drop `.esp` / `.esl` load order priorities with conflict detection.
- **One-Click Transformation:** Clicking "Enable Mod" automatically runs the asset converter for that mod into `transformed_mods/`.

---

## 4. UI Modernization (Flash / Scaleform `.gfx` ➔ Bevy Native `bsn!` UI)

### The Skyrim UI Problem:

Original Skyrim uses **Autodesk Scaleform**, which executes Adobe Flash (`.swf` / `.gfx` files with ActionScript 2.0). Flash is obsolete, insecure, and heavily limits UI customization and performance.

### The Mudcrab Native Bevy UI Architecture (`bsn!`):

Mudcrab completely eliminates webviews, embedded browser engines, and Flash runtimes. All UI elements (hud, menus, inventory, dialogue choices) are transpiled directly into native **Bevy 0.19 `bsn!` (Bevy Scene Notation)** declarative widget trees for pure zero-overhead GPU execution.

```
┌────────────────────────────┐
│ 原始天际 Flash UI │
│ (.swf / .gfx 比例尺) │
└──────────────┬──────────────┘
              │
              ▼（离线Flash/Scaleform转译器）
┌────────────────────────────────────────────────────────────────────────────┐
│ Bevy 0.19 `bsn!` 声明式 UI 树 │
│ ┌────────────────────────┬────────────────────┬────────────────────┐ │
│ │ `bsn!` 节点结构 │ Bevy UI 样式 │ Luau UI 处理程序 │ │
│ │ (Native Bevy Widgets) │ (颜色/版式)│ (ActionScript > Luau) │ │
│ └────────────────────────┴────────────────────┴────────────────────┘ │
└────────────────────────────┬────────────────────────────────────────────────┘
                              │
                              ▼（纯GPU本机执行）
┌────────────────────────────────────────────────────────────────────────────┐
│ Bevy 引擎渲染管线 │
│ │
│ Native Bevy UI 系统（零 Webview / 零嵌入式引擎开销） │
│ - Direct Bevy ECS 查询和 Mutator 系统集成 │
│ - 亚毫秒级渲染和帧速率无限制的 GPU 绘图 │
│ │
│ ┌──────────────────────────────────────────────────────────────────┐ │
│ │ Bevy ECS Bridge：UI 事件 < 直接 ECS 事件 > Luau / 引擎 │ │
│ └────────────────────────────────────────────────────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

#### How it works step-by-step:

1. **Flash Extraction & AST Transpilation (Offline Phase):**
   - **Flash Layouts ➔ `bsn!` Widgets:** Vector shapes, buttons, text fields, and container layouts inside `.swf`/`.gfx` are exported (via AST extraction in `mudcrab_converter`) and transpiled into native Bevy UI nodes using `bsn!` declarative syntax.
   - **ActionScript 2.0 ➔ Unified Luau Handlers:** ActionScript UI scripts (e.g. inventory filtering, stats calculations, menu navigation) are transpiled into standard **Luau** functions. Mudcrab uses Luau as the **single, unified scripting engine** for both UI handlers and quest/game logic—eliminating secondary UI scripting runtimes.

2. **Rendering & Execution inside Bevy (Runtime Phase):**
   - **Bevy 0.19 `bsn!` Native UI:**
     - UI layout structures load directly into native **Bevy 0.19 `bsn!` widget trees**.
     - `bsn!` provides clean, patchable UI widget trees with automatic asset dependency management directly in Rust/Bevy without requiring any browser runtimes, webview overhead, or HTML rendering libraries.

3. **Direct ECS Event Integration (UI ↔ Bevy ECS ↔ Luau Engine):**
   - Interacting with UI widgets triggers standard Bevy ECS events directly in Rust.
   - Game state updates (e.g. player health changes, item acquisitions) instantly mutate Bevy UI components without IPC serialization overhead.

---

### Code Architecture Example: Transpiling Flash UI to `bsn!`

#### Original ActionScript 2.0 Event (inside Skyrim's `InventoryMenu.gfx`):
```actionscript
上（释放）{
    _root.EquipItem(this.itemID);
    this.gotoAndStop("已装备");
}
```

#### Transpiled Unified Luau UI Handler (`modern_assets/scripts/ui/inventory.lua`):
```lua
本地 InventoryUI = {}

函数 InventoryUI:onItemSlotReleased(slot)
    Engine.Player:equipItem(slot.itemID)
    slot:setState("已装备")
结束

返回库存UI
```

#### Generated Bevy 0.19 `bsn!` Native Rust Component (`src/ui/inventory_menu.rs`):
```rust
使用 bevy::前奏::*；

// 从 InventoryMenu.gfx 转换为 Bevy 0.19 bsn！
pub fnspawn_inventory_menu（mut命令：命令，资产：Res<AssetServer>）{
    命令.spawn(bsn! {
        节点{
            宽度：Val::百分比(100.0)，
            高度：Val::百分比(100.0)，
            justify_content：JustifyContent::SpaceBetween，
            ..默认()
        } [
            // 左侧：库存物品列表容器
            节点{
                宽度：Val::Px(400.0)，
                flex_direction: FlexDirection::列，
                背景颜色: 颜色::srgb(0.05, 0.05, 0.05).into(),
                ..默认()
            } [
                文本::新(“库存”),
                按钮节点{
                    边框： UiRect::all(Val::Px(1.0)),
                    ..默认()
                } [
                    Text::new("铁长剑"),
                ]
            ],
            // 右侧：项目 3D 预览框架
            节点{
                宽度：Val::Px(500.0)，
                高度：Val::Px(500.0)，
                ..默认()
            }
        ]
    });
}
```