# 数据格式和规格参考

本文档概述了 Mudcrab 使用的 Bethesda Creation Engine 文件的二进制结构。

---

## 1. `.bsa` / `.ba2` 存档格式

Skyrim 特别版使用两种 BSA 存档格式：

1. **经典天际 BSA（标题魔法：`BSA\0` / 版本 `105`）**
2. **特别版 BA2（标头魔术：`BTDX` / 版本 `1`）**

### 经典 BSA 二进制标头（36 字节）

|偏移|类型 |字段名称 |描述 |
| :----- | :-------- | :------------------------------------- | :------------------------------------------------------------------------------------------------ |
| `0x00` | `[u8; 4]` | `file_id` |魔法标识符：`"BSA\0"` (0x00415342) |
| `0x04` | `u32` | `version` |格式版本（《湮灭》为 `104`，《天际》为 `105`）|
| `0x08` | `u32` | `offset` |到第一个文件记录头的偏移量（通常为 36）|
| `0x0C` | `u32` | `archive_flags` |位标志（0x1 = 包括目录名称，0x2 = 包括文件名称，0x4 = 压缩）|
| `0x10` | `u32` | `folder_count` |文件夹记录块总数|
| `0x14` | `u32` | `file_count` |所有文件夹中的文件总数 |
| `0x18` | `u32` | `total_folder_name_length` |所有文件夹字符串路径的总长度 |
| `0x1C` | `u32` | `total_file_name_length` |所有文件字符串名称的总长度|
| `0x20` | `u32` | `file_flags` |指定包含的文件类型（网格、纹理、声音、脚本等）的标志 |

---

## 2. `.esm` 记录格式

Skyrim 主文件（`Skyrim.esm`、`Dawnguard.esm`）由一系列**记录**和**组**组成。

###记录头结构（24字节）

|领域|尺寸|描述 |
| :---------- | :------------ | ：-------------------------------------------------------------------------------- |
| `type` | 4 字节 ASCII |记录类型标识符（例如 `TES4`、`CELL`、`LAND`、`NPC_`、`REFR`）|
| `data_size` | `u32` |此标头之后的有效负载数据的大小 |
| `flags` | `u32` |记录标志（例如压缩、删除、忽略）|
| `form_id` | `u32` |此游戏记录的唯一32位ID |
| `revision` | `u32` |修订次数 |
| `version` | `u16` |录音版|
| `unknown` | `u16` |内部填充/标志 |

### Mudcrab 的关键记录类型

- **`TES4`**：主文件头（包含作者、描述、主依赖项）。
- **`CELL`**：内部或外部单元定义（世界坐标、照明参数）。
- **`LAND`**：地形的高度图、顶点法线、顶点颜色和纹理层数据。
- **`REFR`**：世界放置参考（将 FormID 网格/对象链接到 3D 坐标 `X, Y, Z, RotX, RotY, RotZ`）。
- **`NPC_`**：Actor 定义（统计数据、网格路径、属性）。

---

## 3. `.nif` 模型格式

NIF（NetImmerse 格式）文件存储 3D 几何体、骨架、动画通道和材质属性。

###《天际》中的主要 NIF 块：

1. **`NiHeader`**：文件签名（`NetImmerse File Format Header`）、版本字符串、块数。
2. **`BSFadeNode` / `NiNode`**：包含变换矩阵的场景图节点（`translation`、`rotation`、`scale`）。
3. **`BSTriShape`**：Skyrim SE 中引入的紧凑几何缓冲区，包含顶点位置 (`Vec3`)、UV 坐标 (`Vec2`)、法线 (`Vec3`) 和索引缓冲区。
4. **`BSLightingShaderProperty`**：包含纹理路径的材质（`base_color`、`normal_map`、`specular`）。

---

## 4. `.lip` FaceFX 唇形同步曲线

Skyrim `.lip` 文件存储以每秒 30 帧采样的 FaceFX 动画曲线。转换器将它们解码为`crates/converter/src/lip.rs`；这是一个只读解析器，而不是编码器或字符动画集成。

该文件以 24 字节小端字节序标头开头。其 RLE 有效负载扩展到每帧 33 个 float32 曲线槽的窗口。 16 个 FaceGen 视位曲线位于插槽 6–21，按此顺序：`Aah`、`BigAah`、`BMP`、`ChJSh`、`DST`、`Eee`、`Eh`、 `FV`、`I`、`K`、`N`、`Oh`、`OohQ`、`R`、`Th`、`W`。剩余的槽是其他 FaceFX 曲线，并保留在解码的网格中。

一些生成的文件具有额外的记录头字节或一字节定时字段移位。解码器根据声明的帧几何形状检查有效负载偏移，并在无法自信地修复标头时标记定时不可用。
帧时间为`(frame + timing_first) / 30`；模糊的计时不会妨碍曲线检查。

使用转换器诊断来验证或检查文件：

```sh
货物运行-p转换器--bin lip-check--path/to/lips
货物运行-p转换器--bin检查-lip--path/to/file.lip
货物运行-p转换器--bin检查-lip--path/to/file.lip 12
```

`lip-check` recursively decodes `.lip` files and validates the translated 16 viseme values. Directory scans report progress every 500 files and print failures without flooding the console; checking one file prints its details. `inspect-lip` prints header/payload details and active visemes; with a frame index it prints that frame's viseme values.