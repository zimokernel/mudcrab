# Mudcrab Bevy 引擎示例和模式

本文档将官方 **Bevy 0.19 Examples** (`bevy.org/examples`) 直接映射到 Mudcrab 子系统实现中。

---

## 1. 子系统到 Bevy 示例映射

|泥蟹特征 |大量示例类别 |目标群示例 |泥蟹适应 |
| :------------------------------------------------ | :---------------------- | :-------------------------------- | :------------------------------------------------------------------------------------------------------------------------------------------------ |
| **细胞空间流** | `Async Tasks` / `Scene` | `async_channel`、`scene` |后台线程流 SQLite 边界框对象并生成 `bsn!` 场景包，没有 UI 口吃。        |
| **PBR 和环境照明** | `3D Rendering` | `pbr`、`lighting`、`atmosphere` | Skyrim 太阳方向、体积雾和 PBR 金属粗糙度材质。                                    |
| **实例树叶和树木** | `3D Rendering` | `instancing`，`mesh_custom` | Vercidium 风格的 GPU 间接绘制调用用于渲染数百万棵树和草叶。                      |
| **角色动画** | `Animation` | `animated_fox`、`animation_graph` |从转换后的 glTF 动画映射的 NPC 行走、奔跑和剑攻击的骨骼动画播放。 |
| **3D 空间音频和天气声音** | `Audio` | `spatial_audio_3d` |位置 3D 音频发出相对于摄像机位置的地下城回声、河流和龙吼声。      |
| **本机 UI 和 HUD** | `UI` | `ui`、`bsn_ui`、`borders` |使用 Bevy 0.19 `bsn!` 宏渲染的生命条、指南针、对话选择和库存屏幕。         |

---

## 2. 代码片段改编

### A. 3D 空间音频（`spatial_audio_3d.rs` ➔ 地下城回声）

```rust
使用 bevy::audio::{AudioPlugin, SpatialAudioSink}；

fnspawn_dungeon_river_sound(
    mut 命令：命令，
    asset_server：Res<AssetServer>，
）{
    命令.spawn((
        AudioPlayer::new(asset_server.load("sounds/river_stream.ogg")),
        PlaybackSettings::LOOP.with_spatial(true),
        Transform::from_xyz(12.0, 0.0, -5.0), // 从河流坐标发出的声音
    ））；
}
```

### B. Instanced Foliage Rendering (`instancing.rs` ➔ Skyrim Forests)

```rust
// 将数千个 PineTree glTF 实例分组到单个 GPU 绘制调用中
fn setup_instanced_forest(
    mut 命令：命令，
    mut 网格：ResMut<Assets<Mesh>>,
    mut 材料：ResMut<Assets<StandardMaterial>>,
）{
    让tree_mesh = meshes.add(Cuboid::default()); // 替换为 glTF PineTree 句柄
    让tree_material = Materials.add(Color::srgb(0.1, 0.4, 0.1));

// 在 1 个 GPU 绘制调用中生成 5,000 个实例树变换
    命令.spawn_batch((0..5000).map(|i| {
        （
            Mesh3d(tree_mesh.clone()),
            MeshMaterial3d(tree_material.clone()),
            变换::from_xyz((i % 70) as f32 * 4.0, 0.0, (i / 70) as f32 * 4.0),
        ）
    }));
}
```

---

## 3. Benefits of Leveraging Official Bevy Examples

1. **API Compliance:** Using official Bevy 0.19 patterns guarantees compatibility with upcoming engine releases.
2. **Built-in Performance:** Bevy's official examples are written by core engine maintainers, ensuring optimal memory allocations and GPU pipeline usage.