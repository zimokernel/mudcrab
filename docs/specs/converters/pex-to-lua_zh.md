# Papyrus (.pex/.psc) 到 Luau 转译规范

本文档详细介绍了将 Bethesda Papyrus 编译的字节码 (`.pex`) 和源脚本 (`.psc`) 转译为 Mudcrab 的现代高性能 **Luau** 脚本的技术规范。

---

## 1. 概述和目标

- **输入：** Skyrim `.pex` 字节码（从 BSA 档案或松散脚本文件中提取）或 `.psc` 源代码。
- **输出：**干净的**Luau**脚本文件（`.lua`）。
- **目标：**
  1. 用闪电般快速的 Luau VM 取代 Skyrim 缓慢的单线程 Papyrus 虚拟机。
  2. 将 Papyrus 语言功能（事件、状态、属性、本机函数、异步等待）映射到惯用的 Luau 表和协程中。

### 在`mlua`中选择最佳Lua引擎：

`mlua` 箱通过 Cargo 功能标志支持多个 Lua 后端：

| `mlua` 功能标志 |后端引擎 | Mudcrab 的性能和适用性 |
| :------------------------ | :------------------------ | :---------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------------- |
| **`luau` / `luau-jit`** | **卢奥（Roblox）** | ⭐ **Mudcrab 的强制引擎。** 专为 AAA 游戏引擎而构建。具有 JIT 编译器、内置沙箱（防止恶意 mod 脚本）、可选静态类型和超快 C/Rust FFI 桥。 |
| **`luajit`** | **LuaJIT 2.1** | 🔥 **快速的原始计算执行。** 非常适合复杂的数学，但缺乏 Luau 的现代沙箱和静态类型功能。                                                                                      |
| **`lua54` / `vendored`** | **标准 PUC Lua 5.4** |高度可移植且兼容的解释器，但缺乏 JIT 编译（比 LuaJIT/Luau 慢 5-7 倍）和沙箱。                                                                                              |

---

## 2. Papyrus 字节码到 Luau 概念映射

|纸莎草纸概念| Papyrus 语法/操作码 |转译的 Luau 代码 |
| :---------------------- | :------------------------------------------------ | :---------------------------------------------------------------- |
| **脚本类** | `Scriptname QuestScript extends Quest` | `local QuestScript = Class("QuestScript", Quest)` |
| **属性** | `Actor Property PlayerRef auto` | `self.PlayerRef = Engine.getForm(0x00000014)` |
| **事件处理程序** | `Event OnInit() ... EndEvent` | `function QuestScript:onInit() ... end` |
| **州** | `State Active ... EndState` | Luau状态表切换（`self:setState("Active")`）|
| **异步等待** | `Utility.Wait(2.5)` |协程产量计时器：`task.wait(2.5)` |
| **全球本地通话** | `Debug.Notification("Hello")` | `Engine.Debug.notification("Hello")` |
| **函数调用** | `TargetRef.Enable()` | `self.TargetRef:enable()` |

---

## 3. 转译工作流程架构

```
┌────────────────────────────────┐
│ Skyrim Papyrus 脚本 (.pex) │
└────────────────┬────────────────┘
                │
                ▼（命名为二进制解析器/Champollion 反编译器）
┌────────────────────────────────────────────────────────────────────────────┐
│ 1. 反编译和解析抽象语法树（AST） │
│ - 读取标头（Magic `0xFAEA4CB5`，主要/次要版本）│
│ - 解析字符串表、结构、变量、属性、函数、状态 │
│ - 构造 Papyrus AST（抽象语法树）节点 │
└────────────────┬────────────────────────────────────────────────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ 2. AST 转换和代码生成器（Rust 编译器通行证） │
┌────────────────────────────────────────────────────────────────────────────┐
│ 3. 输出编译后的 Luau 文件 (.lua) │
│ - 保存到 `transformed_data/scripts/` 准备 `mlua` 运行时执行 │
└────────────────┴────────────────────────────────────────────────────────────┘
```

---

## 4. Complete Code Transpilation Example

### Original Skyrim Papyrus Source (`QF_MQ101_0003372B.psc`):

```papyrus
脚本名 QF_MQ101_0003372B 扩展 Quest

ReferenceAlias 属性 Alias_Ralof auto
ReferenceAlias 属性 Alias_Hadvar auto

事件 OnInit()
    Debug.Notification("Helgen 处决任务开始！")
    实用程序.等待(1.5)
    Alias_Ralof.GetReference().Enable()
结束事件

自动状态等待
    事件 OnTouch（ObjectReference 又名ActionRef）
        转到状态（“活动”）
    结束事件
结束状态

状态活跃
    事件 OnTouch（ObjectReference 又名ActionRef）
        ;什么也不做
    结束事件
结束状态
```

### Transpiled Mudcrab Luau Output (`QF_MQ101_0003372B.lua`):

```lua
本地 QF_MQ101_0003372B = 类（“QF_MQ101_0003372B”，任务）

函数 QF_MQ101_0003372B:initProperties()
    self.Alias_Ralof = self:getAlias("Alias_Ralof")
    self.Alias_Hadvar = self:getAlias("Alias_Hadvar")
结束

函数 QF_MQ101_0003372B:onInit()
    Engine.Debug.notification("Helgen 执行任务开始！")
    coroutine.yield(1.5) -- 异步延迟而不阻塞CPU线程
    本地 ralof_ref = self.Alias_Ralof:getReference()
    如果 ralof_ref 那么
        ralof_ref:启用()
    结束
结束

-- 状态机实现
QF_MQ101_0003372B.States = {
    等待={
        onTouch = 函数(self, akActionRef)
            self:setState("活动")
        结束
    },
    活跃 = {
        onTouch = 函数(self, akActionRef)
            ——什么也不做
        结束
    }
}

返回QF_MQ101_0003372B
```

---

## 5. Runtime Architecture (`crates/scripting`)

Inside Mudcrab, the **`scripting` crate (`crates/scripting`)** encapsulates the `mlua` JIT VM, isolates script execution from main engine recompilations, and generates IDE type annotations:

```rust
// 里面 crates/scripting/src/engine.rs
使用 mlua::{Lua, 结果};
使用 bevy::前奏::*；

pub 结构 LuauScriptEngine {
    卢阿： 卢阿，
}

实现 LuauScriptEngine {
    pub fn new() -> 结果 <Self> {
        让 lua = Lua::new();

// 将原生 Rust 引擎 API 注册到 Luau 命名空间
        让全局变量 = lua.globals();
        让engine_table = lua.create_table()?;

// 绑定Engine.Debug.notification
        engine_table.set("通知", lua.create_function(|_, msg: String| {
            println!("[天际调试]: {}", msg);
            好的(())
        })?)?;

globals.set（“引擎”，engine_table）？；
        好的（自我{lua}）
    }

pub fnexecute_script(&self, script_code: &str) -> 结果 <()> {
        self.lua.load(script_code).exec()
    }
}
```

### 6. Modder Intellisense & Type Safety (`.d.lua` Generator)

`crates/scripting` includes an automated type definition exporter (`mudcrab_scripting::generate_typedefs()`):

- Exposes all `Engine.*` API methods as EmmyLua / LuaLS type definitions (`mudcrab.d.lua`).
- Distributable via **Luarocks** for one-command mod developer setup (`luarocks install mudcrab-types`).
- Gives mod developers instant autocompletion, inline documentation, and static type checking in VS Code, Zed, Neovim, and Cursor.