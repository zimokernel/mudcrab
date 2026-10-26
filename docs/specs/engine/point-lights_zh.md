# `LIGH` 参考文献中的泥蟹点光源

运行时如何将流式传输的 Skyrim 灯光参考转换为 Bevy 点光源、它的亮度以及
一次启用了多少个。代码是 `crates/engine/src/lights.rs`（转换、
规模和预算），`crates/engine/src/world/database.rs`（构建的行）和
`crates/engine/src/streaming.rs`（产生灯光的地方）。

灯光是**选择加入**：除非引擎以 `--lights` 运行，否则本文档中不会发生任何情况。

---

## 1. 读到的内容

两个转换器表携带数据（`docs/specs/converters/db-schema.md`，§5 和§12）。发动机
通过它已经运行的单元格查询读取它们，因此单元格的灯与单元格的灯一起到达
参考文献：

|来源 |专栏 |用作 |
| :--- | :--- | :--- |
| `lights`（加入参考文献的基本记录）| `radius` | `PointLight::range` 和强度的参考距离 |
| `lights` | `color_r`、`color_g`、`color_b` | `PointLight::color`，作为 sRGB 字节 |
| `lights` | `flags` |是否放置了灯（两个标志位；参见§3）|
| `"references"` | `radius_override` |参考的 `XRDS` 半径，胜过记录的 |

参考的 `radius_override` 是布局本身的大小，12,148 `LIGH` 中的 10,810
`Skyrim.esm` 中的参考文献带有一个。非正有限数的覆盖将被忽略：
`XRDS` 已被视为负数，半径是尺寸，而不是开关 - 标志带有开/关
状态 - 因此无意义的覆盖将记录的半径保留在适当的位置，而不是让空间保持黑暗。
`MAX_RADIUS_OVERRIDE`（8,192 个单位，两个外部单元）之上的覆盖也被忽略：基础
游戏的正覆盖值高达 6,919，然后跳至 9 个异常值（15,967 至 3,736,737），
将到达许多细胞。当引用没有可用的时候，使用记录自己的半径
覆盖。

当数据库工作人员打开表和列时，每个连接都会**探测**一次（`has_lights`、`has_radius_override`），
因此，在导出灯光之前转换的数据库仍然会加载：每个引用都会读取为未点亮
并且没有查询失败。参考行的 `light` 和 `light_radius_override` 字段是 `None`
未点亮的参考。

## 2. 一行如何变成一盏灯

`point_light(row, radius_override)` 返回 `Some(PointLight)` 以获取发动机可能点亮的记录
世界，否则为 `None`：

- `range` 是以创建单位为单位的有效半径。该引擎将一个创造单元渲染为一个
  Bevy 世界单位，因此不应用任何转换。
- `color` 是唱片的颜色，与游戏中的所有其他颜色一样读取为 sRGB。
- `intensity` 来自有效半径 (§4)。
- `shadow_maps_enabled` 是 **错误**：每个灯一个立方体贴图在 64 个灯下是无法承受的。天际
  确实有阴影投射灯（唱片的阴影标志）；向他们致敬是留到以后再说。
- `radius` 区域保持为 0，因此不存在过大的镜面高光。

`streaming::spawn_cell` 将灯光放置为**其参考的子级**，这就是让它坐下的原因
在引用所在的位置，按照渲染原点重新设置单元格的其余部分，然后消失
当电池卸载时。基本记录为 `LIGH` 的参考通常根本没有模型，因此
参考还获得一个 `Visibility` 组件：如果没有它，Bevy 警告（B0004）并且光孩子可以
永远不会变得可见。

## 3. 不发光的旗帜

`LIGH` `DATA` 携带标志字。两位使引擎不放置任何内容（UESP，“Skyrim Mod：Mod File
格式/轻”）：

|位|名称 |为什么会被跳过 |
| :--- | :--- | :--- |
| `0x0000_0004` |负|光消除光；该引擎无法减去聚集光 |
| `0x0000_0020` |默认关闭 |记录在脚本将其打开之前是黑暗的，运行时不会对此进行建模 |

这两个位都不是由附带插件中的任何 `LIGH` 记录设置的，因此跳过它们不能轻松
脱离游戏自己的世界；它们的存在是为了模组和脚本。

## 4. 亮度

Bevy 的点光源在距离 `d` 处贡献，

```text
Lout = 反照率 * NdotL * 颜色 * 强度 * 窗口(d) / (4 * PI^2 * d^2)
窗口(d) = (1 - (d / 范围)^4)^2
```

The `4*PI^2` is Bevy's, not this engine's: `bevy_pbr/src/render/light.rs` divides a point light's
intensity by `4*PI` on the CPU (`ExtractedPointLight`) and `Fd_Burley` in
`bevy_pbr/src/render/pbr_lighting.wgsl` supplies the Lambert `1/PI`. The ambient light contributes
`albedo * ambient_colour * brightness`, with neither term (`bevy_pbr/src/render/light.rs`,
`ambient_color: ... * ambient_light.brightness`, and `pbr_ambient.wgsl`).

A converted light is given the intensity that makes the two equal at **half its own radius**, times
`LIGHT_EXPOSURE`:

```text
强度 = 4 * PI^2 * AMBIENT_ILLUMINANCE * LIGHT_EXPOSURE * (半径/2)^2 / 窗口(半径/2)
窗口(半径/2) = (1 - 1/16)^2 = 225/256
```

with the constants of `crates/engine/src/lights.rs`:

| Constant | Value | Meaning |
| :--- | :--- | :--- |
| `AMBIENT_ILLUMINANCE` | `160.0` | the brightness the engine's world path inserts into `GlobalAmbientLight` (`crates/engine/src/app.rs`) |
| `LIGHT_EXPOSURE` | `10.0` | how many times the ambient a light delivers at half its radius |
| `HALF_RADIUS_ILLUMINANCE` | `4*PI^2 * 160 * 10` = 63165.6 | the delivered illuminance the intensity is solved for |
| `HALF_RADIUS_WINDOW` | `225/256` | Bevy's range window at half a range (`getRangeFalloff`) |

**Ten is a round default, not a fitted number.** At half its radius a light delivers ten times the
ambient illuminance, an order of magnitude, so a lamp's pool reads clearly against the room around it.
It is not fitted to any image, and tuning it is a later pass (§7). The scale is per radius rather than
per record, so one constant serves a 75-unit Dwarven lamp and a 3300-unit water light alike: every
light delivers the same surface brightness at half its own reach. A 512-unit torch gets about
`4.7e9`.

## 5. The 256-unit intensity reference cap

Sizing a light's intensity from its own `radius/2` treats the radius as brightness as well as reach,
and a `LIGH` radius is only reach. A shipped record shows the size of the error: a radius of 4334 is
71 times a 512-unit torch's reference radius, so a light sized from its own radius comes out 71 times
a torch and washes out the space it hangs in.

`INTENSITY_REFERENCE_RADIUS = 256.0` caps the reference distance: a light whose radius is up to
`2 * 256 = 512` is lit at its own `radius/2`, byte for byte as the formula above, and a bigger light
is lit at 256 units whatever its radius. The big light keeps its whole `range`, so it still *reaches*
as far as the record says; what it loses is a brightness that grows with the square of that reach.
512 units is the widest common `LIGH` radius, which is what makes the cap invisible to every ordinary
torch, lamp and brazier.

## 6. The budget: 64 enabled lights

`LightsPlugin` runs one system, `budget_lights`, in `PostUpdate` after transforms are propagated. It
enables the `ENABLED_LIGHT_BUDGET = 64` lights nearest the camera and sets every other `SkyrimLight`
to `Visibility::Hidden`. Hidden is the switch that matters: Bevy extracts a light to the render world
only while it is visible. A `PointLight` without the `SkyrimLight` marker is never ranked, so the
budget cannot touch a light that a fixture or a future engine feature spawns.

The choice is cached in a resource and re-made when:

- the camera has moved more than `BUDGET_RECHOOSE_DISTANCE = 256.0` units, or
- the **set** of spawned lights has changed - a light that is not in the set the last choice was made
  from, or a count that no longer matches, which covers a cell streaming in or out.

The set is what is remembered, not the count: a cell that unloads as another streams in with the same
number of lights is invisible to a count, and a room walked into and stopped in would stay dark until
the player moved another 256 units. Re-choosing is a sort over every spawned light, so it is not done
every frame.

## 7. The `--lights` flag

`EngineConfig::lights` (`crates/engine/src/config.rs`) is false by default and set by `--lights`.
`LightsPlugin` is registered for every run - it owns the budget, not the spawning - but
`streaming::spawn_cell` places a light only while the flag is on. A default run therefore renders
exactly as it did before this feature, and the acceptance, profiling and benchmark baselines that
`scripts/phase2-*.ps1` take do not move.

## 8. What is left for later

- **Tuning.** `LIGHT_EXPOSURE` is a documented round default; the exact value belongs with a
  comparison of rendered frames, as does any per-space ambient.
- **Skyrim's own attenuation.** The record's exponent column is published but not read, and Bevy's
  inverse-square curve is used instead of the game's own falloff curve.
- **Fade and flicker.** The converter publishes the `FNAM` fade and, since #269, the `DATA` time
  field, flicker period and amplitudes, the `FOV` cone and the near clip in `lights`
  (`db-schema.md` §12). The engine reads none of them: a converted torch burns steadily and lights
  in every direction.
- **Shadows.** Off for every converted light; a light therefore also lights the far side of the wall
  it is mounted on.