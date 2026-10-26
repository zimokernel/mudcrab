# NIF 到 glTF 2.0 / GLB 转换规范

本文档详细介绍了将 Bethesda NetImmerse (`.nif`) 3D 网格文件转换为现代 GPU 就绪 **glTF 2.0 (`.glb`)** 二进制文件的技术规范。

---

## 1. 概述和目标

- **输入：** Skyrim `.nif` 文件（NiHeader、BSTriShape / BSDynamicTriShape / NiTriShape、BSLightingShaderProperty）。
- **输出：** 独立 glTF 2.0 二进制文件 (`.glb`)。
- **目标：** 将传统专有 3D 几何体转换为标准 PBR 兼容的 glTF 基元，在 Bevy 内呈现零拷贝。

---

## 2. 块映射参考表

| Skyrim NIF 块类型 | glTF 2.0 等效项 |转换逻辑 |
| ：---------------------------------------------------------- | :---------------------- | :-------------------------------------------------------------------------------- |
| **`NiHeader`** | `asset` 元数据 |复制生成器和版本标签 |
| **`NiNode` / `BSFadeNode`** | `nodes` |转换局部变换矩阵（`translation`、`rotation` 四元数、`scale`）|
| **`BSTriShape` / `BSDynamicTriShape` / `NiTriShape`** | `meshes` + `primitives` |提取顶点位置、法线、UV、切线和索引缓冲区； `NiTriShape` 仅是静态几何体（第 7.2 节）|
| **`BSLightingShaderProperty`** | `materials` |将 Bethesda 着色器标志映射到 glTF PBR 金属粗糙度属性 |
| **`BSShaderTextureSet`** | `textures` + `images` |将《Skyrim》纹理槽（`_d.dds`、`_n.dds`、`_s.dds`）映射到 glTF URI/KTX2 手柄 |
| **`NiSkinInstance` / `BSDismemberSkinInstance`** | `skins` |地图骨骼索引（`JOINTS_0`）和顶点权重（`WEIGHTS_0`）；参见§7 |

`BSMultiBoundNode`、`BSOrderedNode`、`NiBillboardNode` 和 `BSValueNode` 是 `NiNode`，带有一些
尾随字段，并像 `NiNode` 一样进行转换：它们的变换和子项加入场景，并且
额外的字段会被读取但不会导出。广告牌保持其创作方向而不是
转身面对镜头。在解析这些节点类型之前，它们下面的形状丢失了所有
父级变换并坐在场景根部（#288）：锻造厂的光芒、火焰和淬火水
漂浮在距熔炉约 170 个单位的地方。
没有转换器架构碰撞：在此更改之前进行的转换保留其缓存的 GLB
直到他们重新皈依。

---

## 3.详细数据提取步骤

```
┌──────────────────────┐
│ 天际 NIF 文件 │
└──────────┬────────────┘
           │
           ▼（二进制阅读器/名义）
┌────────────────────────────────────────────────────────────────────────────┐
│ 1. 提取几何缓冲区 (BSTriShape) │
│ - 位置：Vec3<f32> ➔ glTF 访问器“位置” │
│ - UV 贴图：Vec2<f32> ➔ glTF 访问器“TEXCOORD_0” │
│ - 法线：Vec3<f32> ➔ glTF 访问器“NORMAL” │
│ - 切线：Vec4<f32> ➔ glTF 访问器“TANGENT” │
│ - 索引：u16 / u32 ➔ glTF 访问器“ELEMENT_ARRAY_BUFFER” │
└──────────────────────────┬────────────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ 2. 贴图材质和纹理（BSLightingShaderProperty） │
│ - 插槽 0（漫反射）➔ 基本颜色纹理 │
│ - 插槽 1（法线贴图）➔ 法线纹理 │
│ - 插槽 2（地下/环境）➔ 金属粗糙度纹理 │
│ - Alpha 标志 ➔ alphaMode ("OPAQUE" / "MASK" / "BLEND") │
└──────────────────────────┬────────────────────────────────────────────────┘
                           │
                           ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ 3. 构建并编写 glTF 2.0 二进制文件 (.glb) │
│ - 编写 JSON 块（节点、网格、材质、访问器、视图） │
│ - 写入 BIN 块（交错顶点和索引缓冲区） │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 4. Material Parameter Conversion Matrix

Before glTF publication, Mudcrab builds a validated material contract for every reachable shape.
The contract follows the shape's explicit shader, texture-set and alpha-property block references;
block order and filename suffixes are not used to associate or classify materials. Unsupported
properties are recorded as explicit exclusions, while invalid references and non-finite values fail
conversion with the source file, shape block and shader block in the diagnostic.

| Skyrim Shader Feature    | Skyrim Flag / Value                           | glTF PBR Property                                                                                       |
| :----------------------- | :-------------------------------------------- | :------------------------------------------------------------------------------------------------------ |
| **Base Color**           | Diffuse texture (`Slot 0`) + material alpha   | `pbrMetallicRoughness.baseColorTexture` + `baseColorFactor`, interpreted by glTF as sRGB color + alpha |
| **Normal Map**           | Normal texture (`Slot 1`)                     | `normalTexture`, interpreted by glTF as linear data                                                     |
| **Roughness / Specular** | Glossiness value + specular texture (`Slot 7`)| `roughnessFactor = 1.0 - clamp(glossiness / 100.0)` + `KHR_materials_specular`                          |
| **Metallic Factor**      | No validated metalness input in the SSE IR    | Fixed to `0.0`; environment mapping is not misclassified as metalness                                   |
| **Emissive / Glow**      | Glow map (`Slot 2`) or emissive color/strength| `emissiveTexture`, `emissiveFactor` and `KHR_materials_emissive_strength`                               |
| **Two-Sided Rendering**  | `SLSF2_Double_Sided` flag                     | `doubleSided: true` only when the flag is set                                                            |
| **Alpha Transparency**   | `NiAlphaProperty` and alpha-related flags     | `alphaMode: "MASK"` with normalized threshold, or `"BLEND"`                                           |

Height/detail, environment, environment-mask, inner-layer and greyscale slots remain in the
`OPEN_SKYRIM_material` extension because core glTF has no equivalent Skyrim shader semantics.
The extension also records premultiplied-alpha and screen-door-alpha requirements. Texture URIs
always target the canonical KTX2 hierarchy; the semantic DDS-to-KTX2 encoding itself is closed by
the following conversion stage. Once that stage has published the hierarchy, the pipeline prunes
from every GLB the texture URIs whose source texture is absent from the installed game data -
including a base-color URI - together with every core or `OPEN_SKYRIM_material` slot that referenced
them, so a NIF naming a texture Bethesda never shipped still publishes and renders with its
remaining maps. The dropped paths are recorded per mesh under `pruned_texture_references` in
`conversion-manifest.json` and do not make the conversion incomplete.

---

## 5. Rust Implementation (`mesh_tools` Builder Architecture)

We use the **`mesh_tools`** crate (`GltfBuilder`), which provides an incredibly clean, ergonomic API for assembling vertices, normals, UVs, and PBR materials into binary `.glb` files.

```rust
使用 mesh_tools::GltfBuilder；

pub 结构 NifToGltfConverter；

实现 NifToGltfConverter {
    /// 将解析后的 Skyrim NIF 结构转换为二进制 GLB 文件
    pub fn Convert_and_export(nif: &SkyrimNif, output_path: &str) -> Result<(), Box<dyn std::error::Error>> {
        让 mut builder = GltfBuilder::new();

// 1. 创建PBR材质
        让材质 = builder.add_pbr_material(
            一些（“SkyrimMaterial”.to_string（）），
            Some([1.0, 1.0, 1.0, 1.0]), // 基色 (RGBA)
            一些（nif.材料.粗糙度），
            一些（nif.材料.金属），
        ）；

// 2. 添加网格基元（位置、法线、UV、索引）
        让 mesh_index = builder.add_custom_mesh(
            一些（“SkyrimMesh”。to_string（）），
            &nif.positions, // Vec<[f32; 3]>
            &nif.normals, // Vec<[f32; 3]>
            &nif.uvs, // Vec<[f32; 2]>
            &nif.indices, // Vec<u32>
            一些（材料），
        ）；

// 3. 创建带有变换的场景节点
        让node_index = builder.add_node(
            一些（“RootNode”.to_string（）），
            一些（网格索引），
            Some(nif.translation), // [x, y, z]
            Some(nif.rotation), // 四元数 [x, y, z, w]
            Some(nif.scale), // [sx, sy, sz]
        ）；

生成器.add_scene(
            一些（“SkyrimScene”.to_string（）），
            一些（vec！[node_index]），
        ）；

// 4. 将二进制GLB直接导出到磁盘
        builder.export_glb(output_path)?;

好的（（））
    }
}
```

## 6. Collision extras and rigid-body dynamics

Scene extras carry `mudcrabCollision` (`shared::collision::CollisionAsset`): the authored
collision `shapes`, the `skipped` blocks, and, since version 2, a `bodies` array (#104 phase a).
A version 1 asset has no `bodies`; readers treat every shape in it as fixed and ignore body
fields they do not know.

Each `bhkCollisionObject` whose rigid body (`bhkRigidBody` or `bhkRigidBodyT`) yields shapes
produces one `CollisionBody`:

| Field | Source (byte offset in the Skyrim SE body block) | Conversion |
| --- | --- | --- |
| `node`, `target` | the collision object's target `NiNode` | glTF node index and NIF node name |
| `shapes` | the shapes extracted for this body | indices into `shapes` |
| `havok.collision_layer` | havok filter layer (4) | raw |
| `havok.motion_system`, `deactivator_type`, `quality_type` | 224, 225, 227 | raw (`hkMotionType`, `hkDeactivatorType`, `hkQualityType`) |
| `mass` | 180 | unchanged (kg) |
| `inertia` | `hkMatrix3`, three rows of four floats (116) | R I R^T for the shapes' transform, x 70^2, then the Creation-to-runtime basis |
| `center_of_mass` | `Vector4` (164) | x 70, same transform as the shapes, then the runtime basis |
| `linear_damping`, `angular_damping` | 184, 188 | unchanged |
| `friction`, `restitution` | 200, 208 | unchanged |
| `max_linear_velocity` | 212 | x 70 |
| `max_angular_velocity` | 216 | unchanged (rad/s) |

Shapes are read from the body's shape block, through `bhkMoppBvTreeShape`,
`bhkTransformShape`/`bhkConvexTransformShape` and `bhkListShape` wrappers, and map onto
`shared::collision::CollisionShape`:

| NIF block | Shape | Conversion |
| --- | --- | --- |
| `bhkCompressedMeshShape` | `Mesh` | decompressed vertices and triangles |
| `bhkNiTriStripsShape` | `Mesh` | strip data as triangles |
| `bhkBoxShape` | `Hull` | the eight transformed half-extent corners (16) |
| `bhkConvexVerticesShape` | `Hull` | vertices from 36 |
| `bhkCylinderShape` | `Hull` | two 16-point rings around the A-B axis (A @16, B @32), radius @48 |
| `bhkCapsuleShape` | `Capsule` | A @16 and B @32, radius max(Radius 1 @28, Radius 2 @44) |
| `bhkSphereShape` | `Capsule` | zero-length capsule (a = b = the shape origin), radius @4 |
| `bhkMultiSphereShape` | `Capsule` per sphere | count @16 (1..=8), `NiBound {centre, radius}` from @20 |

Points are Havok units scaled by 70 before the Creation-to-runtime basis; radii are scaled by
70 and the transform scale, like the capsule arm. A sphere or multi-sphere under a non-uniform
or sheared transform has no single radius and is skipped. A degenerate cylinder (A = B, radius
below or equal to zero, non-finite), an out-of-range multi-sphere count and any malformed or
unsupported shape land in `skipped` instead of becoming collision.

Only bodies on physical Skyrim layers are read (`SkyrimLayer` in nif.xml): 0 UNIDENTIFIED,
1 STATIC, 2 ANIMSTATIC, 3 TRANSPARENT, 4 CLUTTER, 5 WEAPON, 9 TREES, 10 PROPS, 13 TERRAIN,
17 GROUND, 26 TRANSPARENT_SMALL, 27 INVISIBLE_WALL, 28 TRANSPARENT_SMALL_ANIM, 31 STAIRHELPER
and 35 COLLISIONBOX. Layer 15 NONCOLLIDABLE (harvestable flora and other nonphysical objects)
is dropped silently; layer 12 TRIGGER is skipped with the reason "trigger volume (layer 12) is
not physical"; every other layer is reported as unsupported.

`kind` is `dynamic` when the motion system is a simulated one (dynamic, sphere or box inertia,
plain or stabilized, thin box: 1, 2, 3, 4, 5, 8), the quality type is a moving one (debris,
moving, critical, bullet: 3, 4, 5, 6) and the mass is finite and above zero; `keyframed` for
`MO_SYS_KEYFRAMED` (6) with mass 0; otherwise `fixed`. `convex` is true when every shape of the
body is a box, capsule or hull (a compressed or strip mesh makes it false).

The `node` index and `target` name are checked against the GLB that is actually written, not the
source NIF's static node list (skeletal and effect NIFs lay their nodes out differently): a body
is kept only when `target` is non-empty and the GLB's node at `node` has that name. A name
used by more than one node is trusted at its index only when the GLB's nodes start in the
same order as the NIF's static scene, which the index was predicted from; otherwise, and
always when the collision is added to a GLB built elsewhere (`annotate`), the name must be
unique.

A body on an unsupported layer, a trigger, a non-colliding body (a `HavokFilter` with the
"No Collision" flag, or a collision response of 2 RESPONSE_REPORTING or 3 RESPONSE_NONE, in
either of the two copies a body stores) or a body whose target node cannot be resolved is listed
in `skipped` and contributes no collision. A body whose shapes cannot all be read is listed in
`skipped`; the shapes it did read stay as fixed collision. A body whose dynamics cannot be read,
or whose glTF node fails the check above, is listed in `skipped` but its shapes stay as fixed
collision.

A cylinder's 16-point rings are inscribed in the true circle (about 2% inside at the chord
midpoints); like the box and hull arms, the Havok convex radius (a thin shell) is not added.

Collision coverage (sphere, multi-sphere and cylinder shapes) moved the converter to schema 27:
schema 24 and 26 outputs keep their verified scripts and archive ingestion and regenerate GLBs, textures
and world data, so no cached GLB keeps the older collision extras.

## 7. Skinning support boundary

### 7.1 Supported

- **Legacy skin instances:** `NiSkinInstance` and `BSDismemberSkinInstance`. Joint names come from
  `instance.bones` (block references resolved to `NiNode` names); inverse bind matrices come from
  the referenced `NiSkinData`, one per joint in the same order. Per-vertex joints and weights come
  from the shape's vertex buffer, or from the skin partition when the buffer carries none.
- **`BSSkin::Instance`** (Fallout 4) behaviour is unchanged. It does not occur in Skyrim meshes.
- **Geometry:** `BSTriShape` and `BSDynamicTriShape`. A dynamic shape keeps its live positions in
  the trailing dynamic `Vec4` array while `data_size` is zero; UVs, triangles and per-vertex
  weights come from the referenced `NiSkinPartition`. Missing vertex channels are filled from the
  partition channel by channel, not all-or-nothing.
- **Partition layouts:** both the SSE layout (inline vertex buffer plus per-partition records) and
  the legacy Bethesda layout. Multi-partition blocks are merged into one bone palette, and a
  partition whose palette differs is remapped into the merged one. The concatenated vertex map
  ties each weight row to its shape vertex, including when the row count equals the vertex count
  (the map can be a permutation) and when partitions repeat shared vertices. Rows are read in
  shape-vertex order only when the partition has no map.
- Bone indices in a shared vertex buffer (the shape's, or the partition's SSE copy) address the
  skin instance's bone list directly. Only the per-partition `bone_indices` rows are palette-local
  and are remapped through `bone_palette`. Only influences with a non-zero weight are validated,
  and weights are normalised to sum to 1.
- Inverse binds come from each `NiSkinData` bone transform as translation · rotation · scale.
- A skeleton that repeats a node name keeps the first occurrence; a skin that binds a repeated
  name is rejected, because either copy could be the bone it was authored against.
- Each skinned shape becomes one `SkeletalMesh` that records its source block, so material and
  vertex-colour contracts associate by block instead of by shape-type enumeration order.

### 7.2 Rejected

- `BSDismember` body-part flags are retained for diagnostics only; they do not split meshes.
- Legacy `NiTriShape` / `NiTriShapeData` geometry is not skinned (no such file exists in the
  Skyrim SE actor meshes).
- `BSSubIndexTriShape` skinning is not handled (same dataset: no such block).

### 7.3 Failure contract

A NIF that declares a skin and cannot produce one fails conversion. It does not publish a static
substitute and does not emit an empty scene. The diagnostic names the NIF path and the block
indices, lists every skin instance that no exported shape claimed together with the block type that
references it, and reports all unresolved joint names at once rather than the first one.

Classes that still fail:

- a rig that some skeleton covers at 50% or more, while no skeleton covers it fully and no path
  candidate covers it at 50%: the resolver does not guess between partial matches;
- a skin that binds a name its resolved skeleton repeats (§7.1).

Not failures, but approximations with their own follow-ups: a rig no skeleton covers at 50% converts
self-contained from its own hierarchy (FX meshes skinned to their own controller bones are the
retail case), and a skeleton only a RACE record identifies (for example Alduin) is chosen by joint
coverage instead, which can pick a near-identical rig.