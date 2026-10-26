# 景观纹理比例

默认 LAND 纹理频率为 **每轴每个完整单元 24 次重复**，
或 12 个象限。这仅缩放漫反射/法线采样；它没有
重新缩放 17x17 象限油漆重量网格。

＃＃ 证据

- Sheson 2020 年 8 月 23 日 [xLODGen 解释](https://stepmodifications.org/forum/topic/15184-xlodgen-terrain-settings-compare/page/4/)
  明确给出每个 Skyrim 单元格 24x24 块和每个四单元格宽 96x96
  4 级地形 LOD 纹理。
- DoubleYou的[Skyrim SE默认设置调查](https://stepmodifications.org/forum/topic/11523-skyrim-special-edition-default-values-for-all-valid-ini-settings/)
  在景观下记录 `fLandTextureTilingMult=3.0000`。这些是默认值
  当 INI 中缺少该设置时，不建议自定义设置。
- 2026年9月30日的只读静态检查独立恢复了
  以下计算来自 SkyrimSE.exe，文件版本 1.7.104.0，SHA-256
  `846efccf0c1374d71f892907f46549560f2fcb0a75cb87a3eed438baa0f1402f`。

确切的命名设置 `fLandTextureTilingMult:Landscape` 具有静态浮动
RVA `0x207b868` 的值为 3.0。在以 RVA `0x2b3ce0` 开头的函数中，
该设置在 `0x2b4052` 处读取；代码计算 `4 / setting`，然后是
倒数为 `0x2b41d1`。边界为 17 的两个循环将索引 0..16 乘以
增加并存储 `0x2b4252` 和 `0x2b4257` 处的浮点对。
函数边界取自 PE 运行时函数表；没有符号
名称是假设的。对于非零默认值，这相当于：

```text
步骤 = 1 / (4 / 设置) = 3 / 4
Quadrant_uv(x, y) = (x * 步长, y * 步长), x,y 为 0..16
```

Sixteen intervals give 12 repeats across a quadrant, and 32 intervals give 24
across a cell. Thus `8 * 3` is the equivalent full-cell formula `(32 / 4) * 3`;
eight is not an already-applied effective default.

## Engine mapping and limits

The shared crate defines [`LAND_TEXTURE_REPEATS_PER_CELL`](../../../crates/shared/src/lib.rs)
as `32.0 * (3.0 / 4.0)`, exactly `24.0`. The engine imports this constant;
converter terrain LOD baking can use the same value without depending on the engine.

`build_terrain_quadrant_mesh` uses full-cell UVs `x / 32, y / 32`.
Multiplying by 24 produces the same 0.75 UV increment per interval. Adjacent
quadrants differ by 12 whole repeats, preserving texture phase under repeat
wrapping. The renderer regression test checks all four quadrant spans and
that paint-weight coordinates still cover samples 0..16.

This establishes the default sampling frequency, not exact lighting or visual
parity. User/mod overrides of the original game's tiling setting are not
imported. Arbitrary future tiling values also require consistent phase at
quadrant/cell boundaries and agreement with terrain LOD. Shared glTF/LAND
image sampler ordering is a separate issue tracked in
[#101](https://github.com/Mudcrab-Team/mudcrab/issues/101), with follow-up
[#102](https://github.com/Mudcrab-Team/mudcrab/pull/102).