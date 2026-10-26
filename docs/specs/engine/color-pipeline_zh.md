# L1材质和颜色管道

轨道 [L1 #131](https://github.com/Mudcrab-Team/mudcrab/issues/131)，位于 [vanilla lighting parity #129](https://github.com/Mudcrab-Team/mudcrab/issues/129) 下。 L0 参考捕获和 L1 实现可以同时进行。 L0 证据关乎零售平价接受，而不是综合测试或实施。

## 范围和证据

默认目标：未修改的 Skyrim SE。内部和外部案例具有同等优先级。 Mod 研究揭示了原生行为；附加照明仍然超出范围。 L1 通过受控探针修复了输出域不一致和 NIF 表面输入。他们不建立《天际》的图像空间方程或选择其最终曝光/色调曲线。

代码基线：`a9f2310ccfc691eebb97fde18df1e8d334b7d744`；贝维 0.19。下面的源跟踪描述了实际的运行时行为。旧天空笔记中关于编码天气插值和雾方程的主张仍然需要 L0/L2 零售证据；此更改保留了这些输入。

## 管道追踪和所有权

|舞台/业主|输入→输出|当前限制/下一个所有者 |
|---|---|---|
| `converter/src/texture.rs`、`material.rs` | DDS通道→语义KTX2传输格式；漫反射/发光使用 sRGB 别名，法线/数据使用线性视图 |共享源字节可能有不同的视图；保留 alpha 作为数据。现有的往返测试涵盖编码通道。 |
| `converter/src/material.rs` |验证的每个形状 NIF 值 → glTF 因子、扩展、源附加 | #81：镜面启用、法线 alpha 蒙版和光泽指数近似。 #82：静态辉光槽资格和发射能量已修正；动画发射仍然不受支持。 ＃83：阿尔法/紫外线。 #84：效果/编辑器表面。其余问题仍悬而未决；发射动画在#82 之外。 |
|贝维 glTF / `StandardMaterial` | sRGB 纹理解码一次；材料因素和数据纹理保持线性→材料响应| Bevy 的 PBR BRDF 是近似值，未恢复 Skyrim 着色。切线/模型空间法线需要不同的处理。 |
| `shaders/terrain.wgsl` |线性层样本和法线数据→加权材质响应→PBR照明|编写的法线约定、层语义和镜面反射响应仍然是单独的材质探针。 |
|贝维 `pbr_functions.wgsl` |灯光 + 材质 → 曝光缩放线性 RGB | `Exposure.ev100` 使用 Bevy 的 `2^-EV100 / 1.2`。当前的 9.7 是根据之前的默认值固定的，而不是《Skyrim》值。 Emission的暴露权重是材料自有的；默认发射与曝光无关。 |
| `sky.rs`、`shaders/sky.wgsl` |编码天气行混合 → 一个 sRGB 解码 → 线性调色板亮度 |现有调色板是固定的晴天近似值。天空/雾调色板值已经占据了合成域；不要将它们再次乘以相机曝光。 L2 拥有授权状态及其单位。 |
| `DistanceFog` |裸露表面RGB + 线性雾调色板→ 雾化线性RGB |现有的基于 Bevy 能见度的朝向 Fog Far 的大气雾仍然是明确的。内部雾去除是当前行为，而不是完整的《天际》内部语义。 L2 拥有修正权。 |
| `render.rs`反射相机|主视图曝光时的场景 → `Rgba16Float`，色调映射已禁用 |浮动线性目标保留高于 1 的值。即使在不活动时，反射门也会复制曝光。现有的反射层和几何体选择保持不变。 |
| `shaders/water.wgsl` |暴露的水光+暴露的线性反射→混合→表面雾|反射既不接受第二次照明评估，也不接受显示变换。现有的菲涅尔/波和反射覆盖范围是近似值。 |
| `color_pipeline.rs` / 场景摄像机 | HDR 合成 → 选择 Bevy 变换（默认为 Tony）或显式零售曲线 → 选择后色调映射等级 → 显示编码 |运行时从冷灰色天空和下面描述的沉睡巨人 sRGB 快照开始。设置菜单可以选择 Tony 路径上的 Bevy 变换。显式零售曲线选择拥有相机色调映射，并在不相关的设置更改中保持活动状态。保存的 `full` 等级使用 Bevy 线性 `post_saturation`，然后使用报告的 `ISSkyrimClearDAY`（0x00012F88，`Update.esm` 获胜者）CNAM 1.5/1.135/1.4 和 TNAM 0.42 的 sRGB 亮度/对比度/色调0.894/0.839/0.773。对比度使用 0.18 的艺术枢轴。除非在交互设置中进行更改，否则 EV100 仍保持固定的 Bevy 基线。 `HNAM` 在托尼路径上保持未应用状态。下面的单独零售曲线实验仅使用白色。 L4拥有适应和有效的原生状态。 |

每个材质着色器中之前的非 HDR 网格路径色调映射。自定义天空着色器绕过了该操作，反射 RGB 通过反射和水通道中的色调映射。 HDR 合成会在合成后移动色调变换。实验性电影级添加了全屏着色器通道。它会更改图像并消耗更多渲染目标内存；目标硬件性能仍然是一个可接受的要求。

## 实验性电影等级限制

sRGB 域亮度/对比度/色调通道保留了 Riverwood 外观
在本地测试期间获得批准。这是一种艺术解读，未经证实
《天际》着色器奇偶校验。原生 CPU 曲线、参数打包和最终指数
现已追踪到生产者，但该等级仍缺乏原生适配，其采样
旋转、绽放并完成输出传输。一张捕获的《天际》帧现在可以识别
第 26 组及其输入，针对 GPU 回放进行五像素算术检查；
这种状态尚未在 Mudcrab 中重现，完整的零售通行证订单仍然存在
未经验证。请参阅[捕获的 GPU 状态](../../../research_notes/Skyrim%20native%20tone%20mapping/gpu_state.md)
和[本机包装跟踪](../../../research_notes/Skyrim%20native%20tone%20mapping/gpu_binding_trace.md)。
群饱和度
以线性 RGB 运行，亮度权重与自定义色调略有不同
通过。负输入和色域外输出被剪裁； alpha 被保留并且
零售阿尔法/开花饲料分级被省略。 `HNAM` 已记录但未应用
默认托尼变换。下面的显式零售曲线实验使用
仅限白色。

场景摄像机共享选定的等级，包括内饰。实验的
记录模式可以加载直接图像空间或明确选择的创作空间
天气槽；它不能解析有效的运行时天气或内部状态。所有应用程序都使用
这些相机设置必须注册`SkyrimCinematicPostPlugin`。相机必须保留
`CompositingSpace::Linear`;不支持使用 sRGB 或 Oklab 覆盖它。订购
相对于 AA 和 CAS 尚未得到验证。反射当前导致禁用
传递副本。颜色成分探针的 `--legacy-output` 路径禁用
两个等级均保留其原始未分级基线。调查报告确定了
等级和领域；材料探针现在记录完整的后通道。

## A/B 控制和匹配的捕获

### 沉睡巨人旅馆快照实验

`SceneGrade snapshot-srgb` 和 `SceneGrade snapshot-linear` 从四舍五入开始
2026年10月13日提供的照片模式UI值：饱和度1.66，亮度1.15，
对比度 1.50，色调量 0.56，色调 RGB `(0.88, 0.81, 0.72)`。照片模式
将色调滑块列出为红色、蓝色、绿色；实验按正常顺序存储 RGB。
这些是一个场景的当前基本等级值，而不是通用的《天际》默认值。

sRGB 模式现在应用用户请求的 20% 亮度增加，从 1.15 到
1.38。通过此调整，饱和度、对比度、色调、曝光度和雾度保持不变。
原始快照值和线性模式保留亮度1.15。
批准的外观名为 `approved-sleeping-giant` 并存储在
`crates/engine/src/presets/approved-snapshot.json`：原始圆形快照，
最终分级值、天空颜色和雾覆盖是单独的数据字段。
`--grade-preset <json>` 在启动时加载相同的格式。其等级仅适用
至 `snapshot-srgb`；它的天空/雾仅适用于冷灰色。它不会取代
加载了 Skyrim 记录、Vivid 或 `full` 模式。亮度1.38是最终的，不是
再次乘以1.2。

两种模式都会禁用 Bevy 后饱和度并评估饱和度，然后评估色调
在电影后期通行证中。原始亮度提供给这两个操作；消极的
饱和度产生的通道被保留，直到色调、亮度和对比度
已经运行，然后输出被剪裁。 `snapshot-srgb` 在 sRGB 编码上运行
色调图后值； `snapshot-linear` 直接在线性后色调图上运行
价值观。 Alpha 穿过并且反射保持未分级。

这些模式保留选定的天空/雾预设、曝光、TonyMcMapface 和固定 0.18
对比枢轴。它们不再现《天际》的原生色调曲线、改编、
绽放、最终伽玛或修饰剂组合物。 UI 值经过四舍五入，并且
正确的本机采样域仍然是捕获门。这些是受控域
实验而不是完整的零售渲染器端口。

启动时使用 `--color-grade snapshot-srgb|snapshot-linear`。返回
`SceneGrade full` 恢复鲜艳或冷灰色的现有分级。 `ScenePreset`
仍然是一个独立的天空/预设选择器。两个保存的预设都保留其现有的
价值观。发动机以冷灰色天空和 `snapshot-srgb` 分级启动。这个
使用快照的饱和度1.66；冷灰色保存的饱和度 1.25 适用于以下情况
选择 `SceneGrade full`。恢复原来的生动外观
`--visual-preset vivid --color-grade full`，或运行 `ScenePreset vivid`，然后运行
控制台中的 `SceneGrade full`。拍摄收据包括已解析的分级值，
哪个饱和阶段处于活动状态，以及后通过域标志。对于小型 GPU
成分检查，运行 `color_pipeline_probe --gray --color-grade snapshot-srgb`
或 `snapshot-linear` 与新的 `--output` 目录。

### 创作图像空间记录实验

`--color-grade record-srgb --image-space 0x12F88` 读取 IMGS 获奖记录
来自转换后的 `skyrim_world.db`。或者选择天气的创作
带 `--image-space-weather 0x81A --image-space-slot day` 的插槽；可用插槽
是 `sunrise`、`day`、`sunset` 和 `night`。 FormID 接受十进制或 `0x` 十六进制。
直接选择器和天气选择器不能组合使用，并且天气/时隙标志必须
一起供应。从比较屏幕截图中没有猜测出选择器。
较旧的存档可能会保留插件相关的 IMSP 链接而无需标准化
元数据。因此，天气查找仅接受经过验证的身份情况：
零槽基本插件形式，具有匹配的原始元数据和天气获胜者
优先级 0 或 1。后来的覆盖和 mod 拥有的天气链接明显失败；使用一个
直接为这些档案选择规范的 IMGS。

此模式将原始 CNAM 饱和度/亮度/对比度和 TNAM 色调发送到
现有的 sRGB 后通道，没有批准的快照覆盖。使用默认的
色调映射选择保留了 TonyMcMapface、固定 EV100 9.7、艺术 0.18 枢轴、当前照明和
独立选择天空/雾。这是一个校准实验，而不是天际的实验
本机色调映射器。所有九个 HNAM 值都保留在加载的源中，并且
收据。托尼被选中后，他们仍然明确未申请；适应、本土绽放、本土色调
曲线、最终转移和改性剂成分尚未解决。

控制台 A/B 支持在不选择 `record-srgb` 的情况下加载记录：
`SceneGrade record-srgb`，则`SceneGrade snapshot-srgb`恢复批准
等级。在没有加载记录的情况下切换到记录模式会失败并保留
之前的模式。为了恢复完整批准的外观曲线实验，
选择 `SceneToneMap tony` 和 `SceneGrade snapshot-srgb`。丢失/格式错误的选定记录会导致启动失败，而不是
替换预设。赛程和综合运行不需要零售数据
拒绝记录选择器而不是打开世界数据库。

`SceneState` 和自动 `shots.log` 收据包括来源选择、IMGS
以及可选的 WTHR 身份、获胜插件/加载订单、原始 CNAM/TNAM/HNAM 值，
批准的覆盖来源、有效等级和 `hnam_applied: false`
托尼选择了。零售曲线收据将 White 标识为唯一适用的 HNAM 字段。
创作的天气时段是记录数据的证据，而不是参考系的证据
有效天气、混合修改器或 GPU 常量。成分探针
没有世界数据库并明确拒绝 `record-srgb`。

2026年10月15日的本地验证使用了已批准的本地金属捕获物
默认、直接 ClearDAY IMGS 和多云天气的创作日时段。来源，
等级、相机、准备情况和原始 HNAM 收据均被忽略
`data/visual-comparisons/record-loader-20261015/` 目录。默认的
解析的分级值与批准的捕获完全匹配。 PNG身份是
单独报告：未更改的可执行文件的重复捕获也有所不同，
因此这些结果并不能建立按位图像的再现性。其原因是
捕获变异仍未得到验证。

### 显式零售曲线实验

`--tone-map retail-curve --tone-map-params <json> --image-space <FormID>`
在现有全屏通道中选择单独的标量亮度曲线。
批准的默认值仍为 `--tone-map tony`。评分保持独立
已选择；本实验支持`snapshot-srgb`、`record-srgb`和`off`。
此路径将拒绝旧版 `full`/`saturation` 和 `snapshot-linear`。

参数文件需要`name`、`source`、`curve`（`reinhard`或`filmic`），
`reinhard_white_scale`、`filmic_white_scale`、`sampled_luminance` 和
`target_luminance`。每个数值都必须是有限且正的。它没有
隐式默认值。 HNAM.White来自选定的记录，恢复的
CPU公式推导出其曲线系数。本机静态 PE 初始值设定项显示
白色刻度 1.1/1.4 和胶片禁用。后来的一个孤立的《天际》会议也
观察进程内存和控制台证据中的这些值。参数文件
仍然明确选择它们；它不会继承正在运行的 Skyrim 会话或
建立历史参考屏幕截图的设置。参见
[CPU证据](../../../research_notes/Skyrim%20native%20tone%20mapping/next_step.md)
和[运行时观察](../../../research_notes/Skyrim%20native%20tone%20mapping/runtime_state.md)。

场景相机禁用 Bevy 色调映射，因此标量曲线会暴露
HDR 恰好一次。 Bevy 的色调阶段颜色分级和解带抖动被跳过。
现有的 sRGB 等级、艺术对比度枢轴和显示传输仍然保留。
`SceneGrade off` 禁用分级，同时保留所选曲线。反思
保持其未分级的 HDR 输出，并且从不独立接收曲线。

只有 HNAM.White 对这个实验做出了贡献。其他 HDR 字段保持不应用状态。
固定样本/目标亮度值代表 P.x/P.y，无需调整
历史；他们不会改变等级枢轴。现在原生参数打包
经追踪，审核后的最终指数为`1 / fGamma`；这个实验不
应用该指数。布鲁姆、适应/历史、有效修饰符、采样
数值域和完整的输出传输仍不在此重建范围内。
历史参考的选定着色器和输入保持不变。这个
路径是带有 Bevy 输入和 sRGB 分级的零售曲线，而不是完整的《Skyrim》
色调映射器。见【包装】(../../../research_notes/Skyrim%20native%20tone%20mapping/gpu_binding_trace.md)
以及【输出证据】(../../../research_notes/Skyrim%20native%20tone%20mapping/output_transfer_trace.md)。

预加载参数文件并在保持 Tony 选择的情况下进行记录，然后使用
`SceneToneMap retail-curve` 和 `SceneToneMap tony` 用于控制台 A/B。无效
选择将保持先前的状态不变。自动和实时截图
收据绑定音调选择、系数、记录源和实际摄像机
对捕获的渲染帧进行色调映射/曝光。

每个调整步骤都使用冻结的批准和天际参考
【比较工作流程】(tonemap-tuning.md)。保留原始 PNG 和收据
与视觉判断分开。屏幕截图字节标识不是保真度门。

### 预设和诊断阶段

对于可见的调整会话，`--start-shot <file>` 从
包含一个外部姿势的经过验证的文件。它设置流单元、世界空间、
旋转和水平视野，无需选择自动镜头运行器。添加
`--stdin-console` 使用相同的控制台通过其标准输入控制该进程
命令作为反引号 UI。该桥是选择性加入的并且没有网络侦听器。
`SceneState` 报告分级和流媒体/上传准备情况。 `SceneShot <png-path>`
仅在就绪检查通过后捕获可见窗口并写入 JSON
在图像旁边注明收据。它拒绝现有的输出路径。收据
根据要求记录分级状态并将相机/准备情况证据绑定到捕获的内容
渲染帧；使用 `--shots` 进行严格的自动化姿势和分级证据。现有的交互式鼠标和移动控件
除非有自动选项（例如 `--acceptance-screenshot`），否则保持可用
选择固定相机会话。

`--no-render-diagnostics` 禁用 Bevy 的 GPU 计时和管道统计查询
用于排除 Metal 计数器缓冲区分配故障。它保留了
场景设置、CPU 帧计时、流测量和捕捉准备检查。
在使用此选项的运行中，GPU 计时证据不可用。

两个命名的视觉预设保留了批准的外观，同时允许进行实验：

|预设| `full` 模式下的 Bevy 后饱和 |天空上/下/地平线（sRGB 字节）|
|---|---|---|
|生动| 1.5 | 1.5 `(21,77,117)` / `(60,135,183)` / `(125,163,183)` |
|冷灰色| 1.25 | 1.25 `(178,187,198)` / `(194,202,209)` / `(207,214,220)` |

Vivid 完全保留了现有的晴天配置。冷灰色使用灯光，
略带蓝灰色渐变，饱和度较低。两者都保留亮度1.135，
对比度 1.4，色调量 0.42 及其现有 RGB、EV100 9.7 和 TonyMcMapface。
冷灰使用原始雾密度的四分之一，相当于能见度
创建单位从 48,300 增加到 193,200。它的雾远颜色是柔和的
雪白蓝白色：`(210,222,235)`，而不是 sRGB 字节中的 `(116,168,203)`。生动保留了原汁原味
雾的颜色和密度。灯光、阴影和相机姿势保持不变。天空
颜色是在色调映射和分级之前创作的输入，而不是最终显示 RGB。

在以下位置运行 `ScenePreset vivid`、`ScenePreset cool-gray` 或 `ScenePreset toggle`
反引号控制台。如果没有参数，`ScenePreset` 报告活动预设并
诊断级模式。切换是立即且可逆的。使用
`--visual-preset vivid|cool-gray` 启动时；冷灰色是运行时默认值。预设
选择不会改变下面的独立 `SceneGrade` 模式。使用完整
分级以查看每个完整的外观。预设已内置于引擎中；控制台
选择持续当前进程。拍摄收据记录`visual_preset`
和 `preset_saturation` 以及诊断级模式。

打开反引号控制台并运行 `SceneGrade off`、`SceneGrade saturation` 或
`SceneGrade full`。如果没有参数，`SceneGrade` 报告当前模式。 `off`
禁用两个评分阶段； `saturation` 仅启用 Bevy 后饱和；
`full` 还可以启用实验性 sRGB 亮度/对比度/色调通道。曝光，
色调映射、相机姿势、照明和雾保持固定。反射保持未分级，
新生成的场景摄像机继承当前模式。运行时默认为
`snapshot-srgb` 冷灰色。

使用`--color-grade off|saturation|full|snapshot-srgb|snapshot-linear|record-srgb`
用于启动选择。设置菜单保留了这六种模式。其可选的
字段掩码将用户调整与显示的解析库区分开：未编辑
字段保留有效的已批准或已加载的记录值以及一个已编辑字段
仅覆盖该字段。源证据记录基础身份、原始记录
和出处、应用的覆盖掩码/值以及有效等级。相机
应用程序和捕获收据使用相同的中央解析器。对于可重复的
三模式捕获，运行：

```bash
python3脚本/capture_color_grade.py \
  --引擎目标/调试/引擎\
  --资产/绝对/路径/到/转换/资产\
  --输出/绝对/路径/到/新/捕获目录
```

The script uses `crates/engine/tests/fixtures/color-grade-riverwood-shots.json`:
three exact 1920×1080 Riverwood poses covering facades, roofs, shaded surfaces and
distant geometry. It runs each mode sequentially with the same assets, lighting,
fog, stream radius and poses. `--shots` pins water animation to 1.0 seconds; normal
interactive runs and matched movement routes keep animated water. Capture receipts
record grade selection and fixed water time. The script rejects failed runs,
timeout captures, missing receipts and inconsistent poses before accepting a set.
PNG files and logs stay outside the repository; this is Mudcrab A/B evidence,
not matched retail Skyrim or a performance benchmark.

## Invariants

- V1: Production scene cameras and visual fixtures use explicit `SceneColorPipeline`: HDR, EV100 9.7, TonyMcMapface, linear compositing, Bevy post-saturation and the registered cinematic post plugin. Reflection disables both grades. These settings are experimental; defaults are not evidence of vanilla parity.
- V2: Reflection storage preserves linear values above 1; no tone map or sRGB target view before water sampling. Reflection exposure matches the main camera before rendering, including after exposure changes while water is invisible. Missing reflection exposure ! restore before rendering; pose/visibility updates continue.
- V3: Sky, unlit mesh, fully fogged mesh, terrain emission and unit-reflecting water given equal composition-domain RGB produce matching output within 2/255 per channel. Exterior includes sky; interior has a black background. Test neutral gray and saturated HDR inputs.
- V4: Diagnostic inputs, camera and output settings, samples and verdict are recorded. Probe failure returns a nonzero status; stale reports are removed at startup. Synthetic consistency is not retail parity.
- V5: Preserve NIF source values and declared unsupported families. Do not compensate for pending material errors with global tint, exposure, ambient or emission changes.
- V6: Static emission → authored linear tint × multiplier × eligible glow sample; preserve zero, dim, HDR and black tint. Slot 2 glow ! Glow shader or Glow_Map; Own_Emit alone ≠ texture eligibility. Non-finite source emission or overflowing energy → contextual conversion error before clamping.
- V7: Converter schema 17 → rebuild GLBs from schemas 12–16; reuse verified unchanged texture/script/archive outputs only with matching source/configuration. Historical emission output retained its producer 17 identity; current compatibility is recorded in V11.

- V8: Finite signed NIF tint & static lighting multiplier ! convertible; glTF nonnegative projection retains raw signed color/multiplier and names lower-clamp approximation. Positive channel energy follows V6; signed shader parity remains gap.

- V9: NIF glossiness exponent → bounded monotonic `(2 / (n + 2))^0.25` perceptual roughness; GGX lobe approximation, not exact Skyrim BRDF.
- V10: Specular flag off or strength zero → explicit zero glTF factor. Enabled tangent normals → shared linear normal-alpha mask; model-space normals excluded. glTF specular factor ∈ [0,1]; only tagged loaded masks receive Bevy 0.19 compensation; generic glTF unchanged. F0 still squares scalar/mask inputs; native Skyrim intensity/BRDF parity remains gap.
- V11: Pruning/remapping ! both specular extension textures; removed mask → unchanged bounded factor and no native compensation. Current producer 27 combines native BC textures, emission/specular, source-surface fixes, corrected TXST slots and collision coverage. Rebuild all legacy GLB/KTX2/world outputs from schemas 12–24 and 26; producer 24 GLB compatibility ended with the collision data change. Verified unchanged scripts/archive ingestion remain reusable. Exact producer 27 required for staged meshes/textures; encoder mode and GPU quality participate in configuration identity. Runtime/launcher accept complete converter schemas 15–27 and world schemas 3–9. Schema 5 grass worlds need no LOD tables. Metadata-only retained meshes/textures preserve original producer/configuration and require normal conversion to repair old encoding and collision data.
- V12: Scene-only glTF loads ! reach and retain recursively loaded state after unused subassets release. Native material override retains stock hook's recorded source dependency; scene cloning preserves source and native handles. Probe ! no root-glTF or explicit material loads that mask dependency lifetime failures.

- V13: Converted tangent-space NIF normals ! DirectX Y convention exactly once at native material construction; generic glTF/model-space maps unchanged. Preserve linear RGB & source alpha. ±X/±Y/asymmetric GPU swatches ! match independent geometric normals ≤2/255; legacy no-flip control ! fail.
- V14: NiAlphaProperty owns test/blend enable; vertex-alpha/premultiplied/screendoor flags alone ! opaque. Preserve shader-enabled vertex alpha except native tree/LOD exclusions; prepass & shadow ! same discard as color pass.
- V15: Source UV offset/scale & four S/T clamp modes ! survive conversion. Conflicting wrap or transfer uses ! distinct asset paths; load order cannot change sampler. Alias source mapping ! cache/pruning/restoration consistency.
- V16: Declared EditorMarker & unsupported Fire_Refraction ! excluded with reason; ordinary visible controls remain. Unsupported shader features ! explicit compatibility inventory, no silent parity claim.

## Supported response and remaining material work

| Family / path | Current representation | L1 acceptance status |
|---|---|---|
| Ordinary lighting / opaque | glTF `StandardMaterial`, PBR lighting | Output consistency can be tested now; Specular enable, tangent mask and roughness corrected by T8; native BRDF/model-space parity remains open. Static emission corrected by T6. |
| Alpha-tested / blended | NiAlphaProperty mode, shader-enabled vertex channels, PR #99 prepass | Synthetic color/depth/shadow silhouettes verified; additive and other nonstandard blend factors still approximated. |
| Tangent-space normal maps | Linear RGB/alpha; native material applies DirectX Y convention once | Asymmetric mesh directions match geometric-normal references; NIF authored tangents still regenerated. Terrain owns a separate tangent frame and needs separate native-direction evidence. |
| Model-space normals | No established compatibility path in this slice | Named L1 gap; generic tangent interpretation cannot count as acceptance. |
| Environment-map / parallax / skin / hair / other NIF lighting variants | Raw source contract/extras plus generic approximation | [Compatibility inventory](../../lighting-l1-compatibility.md) names per-family gaps; retained metadata alone is not shader support. |
| Effect shader surfaces | Generic approximation with explicit exclusions; palette effects drawn through their palette (#289) | Declared EditorMarker and Fire_Refraction excluded with reasons; no general effect, refraction, particle or animation parity claim. |
| Water / sky | Custom Bevy shader paths | Output-domain test only; authored behavior and full-scene parity remain open. |

## Verification

- `cargo test -p engine --lib`: explicit scene settings, production reflection format, reflection exposure lifecycle, existing renderer/sky regressions.
- `cargo run -p engine --example color_pipeline_probe -- --output <dir> [--interior] [--gray]`: real GPU shader path and pixel comparison; uses headless GPU readback and requires a Vulkan adapter with at least 32 sampled-texture and sampler slots. The probe requests WebGPU features with terrain limits explicitly raised; it does not benchmark the full production device feature set. Software Vulkan is sufficient for functional validation, not performance acceptance.
- Run all four combinations: exterior/interior × gray/HDR. Each writes `probe.png` and `probe.json`. The probe fixes 800×600, orthographic camera `(0,0,10)`, EV100 9.7, TonyMcMapface, no dither/MSAA, full diagnostic fog on one swatch, unit reflectivity on water, and constant sky rows. All are isolated diagnostic settings, not shipping values.
- The probe uses the production reflection allocation and camera setup; its unlit source geometry, render layer and visibility are diagnostic. A clear-only reflection would not exercise material tone mapping. The HDR case includes blue = 2.0 to detect clipping and duplicate display transforms.
- `--legacy-output` is a negative control: restore the previous non-HDR cameras and 8-bit reflection target inside the probe. It must fail the consistency check, with a nonzero status and saved pixel differences. This option does not exist on the game CLI.
- Complete L1 acceptance additionally needs NIF-to-runtime material probes, integrated dependency fixes and matched vanilla neutral/material captures from L0. Leave #131 open until those gates pass.

## Static emission publication (historical schema-17 slice)

Historical packages and probes retain their original producer identities. Current producer 27 regeneration and compatibility are recorded in V11.

[Emission issue #82](https://github.com/Mudcrab-Team/mudcrab/issues/82): `Own_Emit` declares own emittance; `Glow_Map` declares third-slot glow (`vendor/project-wormhole-nif/src/nif_flags.rs`). Glow shader type also permits slot 2. Own_Emit alone retains slot 2 as unclassified source data; no emissive texture sampling.

Publication: `peak = max(1, max(emissive_color))`; `emissiveFactor = max(0, emissive_color) / peak`; `emissiveStrength = max(0, emissive_multiple) * peak`. glTF factor remains in [0,1]; reconstructed linear RGB retains nonnegative authored energy. Extension emitted whenever strength ≠ 1, including 0 and values below 1. Black tint stays black with glow present. Bevy 0.19 glTF loader multiplies factor by strength into `StandardMaterial.emissive`; glow uses sRGB decode once, alpha does not scale opaque emission. No camera/ambient compensation.

Signed NIF tints remain valid. Negative channels retain previous glTF lower-clamp approximation; raw color/multiplier retained under `extras.openSkyrim.sourceEmission`, with explicit representation label. Signed-emission shader behavior remains unsupported; this projection does not claim Skyrim parity. Strength overflow → contextual conversion error. Source contract retains original valid values. Static lighting multipliers retain their raw sign in the contract and `sourceEmission`, including with nonnegative/black tint; only the glTF projection clamps negative values to zero. These are static shader-property values, not proven controller endpoints. NaN/±infinity rejected before publication; animation and shader-family compatibility remain L1 gaps. Contributor reference inspected at `BimingtonBill/mudcrab:ff96ac2b91bec0765d9ce59b2890449923f9d3ed`; emission publisher there retains old defects, so this slice uses current material owner directly.

`material_emission_probe --output <dir> [--interior] [--legacy-emission]`: converter-published synthetic NIF contracts → glTF/KTX2 → Bevy loader → GPU swatches. Nine cases: zero, dim, unit, HDR, dim HDR, black glow, untextured HDR, Own_Emit atlas, signed tint. Loaded factors checked against authored energy; glow view ! `Rgba8UnormSrgb`. Converted swatches compared with independently computed material RGB at tolerance 2/255; zero cases ! black, other references ! visible. 800×900, orthographic camera `(0,0,10)`, no lights/ambient/fog/dither/MSAA, pinned scene tone map/exposure. Interior/exterior here change diagnostic background only; no authored scene parity claim. Synthetic contract publication ≠ full NIF-file parse coverage. `--legacy-emission` restores old energy/eligibility defects inside probe and ! fail with exit 1. Every run records PNG, JSON, generated glTF/KTX2; removes stale verdicts before startup.

Historical emission migration (schema 17): schemas 12–16 retain verified non-GLB entries/archive ingestion only when source and original configuration hash match; GLBs/world data rebuilt. Configuration changes still invalidate cache. Stage journal schema check rejects old staged GLBs. Current runtime/launcher accept complete converter schemas 15–27 and world database schemas 3–9; cell-cache version unchanged.

For testing, reconvert to separate output directory with converter built from this branch, then run matching engine against that directory. Existing packs remain valid in engine but retain old emission until reconverted. Preserve old pack for rollback; older #137 engine rejects schema 17, so use new engine for new pack. Retail reconversion ! isolated matching package; delivery evidence recorded after successful conversion/startup.

Verification: `v6_emission_preserves_zero_dim_hdr_and_black_glow_energy`, `v6_own_emit_does_not_enable_slot_two_glow`, `v6_rejects_overflowing_emission_with_context`, `v8_signed_tint_keeps_source_and_clamps_only_gltf_negative_channels`, `recent_schema_migrations_reuse_only_unchanged_asset_kinds`, `v15_old_material_schemas_rebuild_meshes_and_reuse_compatible_assets`; engine runtime-schema acceptance tests; both probe backgrounds plus legacy negative control. Full converter/engine library suites, formatting and Clippy required before publication.

## External emittance (#242)

`SLSF1_External_Emittance` (bit 29) says a surface's emission colour comes from outside the model:
from the light or region its reference names in `XEMI`. Building windows use it. The Sleeping
Giant Inn's window (`architecture\farmhouse\inn01.nif`, shape `Inn01:14`) is a Glow shader with a
white tint, multiplier 3 and a neutral grey glow map; its Riverwood reference names
`FXLightRegionInvertWindowWhiterun`, whose weather's Effect Lighting colour is black by day and
amber at night. Publishing the static emission alone left the windows lit in daylight.

The GLB keeps the flag in `extras.openSkyrim.shaderFlags1`, so no mesh is reconverted. The
converter resolves each `XEMI` source's colour at every time of day into the `emittance` table
([db-schema §16](../converters/db-schema.md#16-reference-emittance-sources-emittance), an
optional table the engine probes for). At runtime:

- `NifMaterialPlugin` marks each primitive whose material has bit 29 with `ExternalEmittance`,
  carried with the scene into every instance.
- A reference whose source resolved gets `ReferenceEmittance`: the day colour (the engine's one
  time of day, `sky.rs`) decoded from 8-bit sRGB to linear, as light colours are.
- When the model is instantiated below the reference, each marked surface draws with a material
  whose emission is the model's own emission times that colour. Materials are shared per model
  material and colour, and the model's material stays unchanged for other references. The cache
  of shared materials holds ids, not handles, so a material is freed with the last surface that
  draws with it when its cell unloads.

Approximations: the colour multiplies the published emission instead of replacing the NIF tint,
because the glTF cannot separate tint from multiplier; the two agree for the white-tinted window
shaders. A region's likeliest weather stands in for its climate. A surface whose reference names
no source, or one the converter could not resolve, keeps the model's emission; Skyrim's behaviour
there is unverified. A database converted without the `emittance` table has no sources and keeps the
previous always-lit result until reconverted.

Verification: `resolves_reference_emittance_from_regions_and_lights`,
`the_sleeping_giant_inn_windows_are_dark_by_day` (opt-in, reads `Skyrim.esm`),
`returns_the_day_colour_of_a_reference_emittance_source`,
`external_emittance_needs_bit_29_of_shader_flags_1`,
`a_black_day_colour_darkens_only_the_external_emittance_surface` and the other `emittance` tests.

## Effect palette (#289)

A `BSEffectShaderProperty` with `Greyscale_To_PaletteColor` (flags 1, bit 4) or
`Greyscale_To_PaletteAlpha` (bit 5) does not draw its source texture. The texture's grey level
picks a colour, and its alpha an opacity, from the greyscale (palette) texture. Forge fires and
coals, braziers and many magic effects use it. Drawn as an ordinary material, the blacksmith
forge's coals (`furniture\blacksmithforgemarker.nif`, `BlacksmithForge01:2`, palette
`gradhotcoals`) came out as a flat white disc.

The converter already publishes the palette as the `greyscale` slot of `OPEN_SKYRIM_material` and
keeps the flags, so no mesh is reconverted. At runtime `NifMaterialPlugin` gives each such effect
an `EffectPaletteMaterial` (`StandardMaterial` extended by `shaders/effect_palette.wgsl`) in place
of its standard material. The lookup follows the retail effect pixel shader as reconstructed by
Community Shaders, revision `4d6ac751a05f2e0a10f8add4cc1ba5a0b80ea9e5`,
[`Effect.hlsl`](https://github.com/doodlum/skyrim-community-shaders/blob/4d6ac751a05f2e0a10f8add4cc1ba5a0b80ea9e5/package/Shaders/Effect.hlsl):

- colour = base colour scale × palette(texture.g, base colour.r × vertex.r).rgb;
- alpha = palette(texture.a, base colour.a × vertex.a).a;
- otherwise texture × base colour × vertex colour, as before. The result is unlit, like the game's
  effect shader, and goes through the usual fog and output transform.

Colour spaces: the game looks up the stored grey level, so the shader re-encodes the
sRGB-decoded source sample. Bevy loads a texture with no standard glTF slot names through an sRGB
view, so the palette colour is decoded once on sampling; spawn validation requires that view. The
base colour scale is recovered from the published emission (`emissive / base colour`, see the
emission publication above); a black base colour publishes none and falls back to 1.

The depth prepass and shadow passes use `shaders/effect_palette_prepass.wgsl`, which tests the
same alpha as the colour pass (`shaders/effect_palette_functions.wgsl` holds the shared lookup).
With Bevy's standard prepass, an alpha-tested effect whose palette alpha disagrees with its
source alpha would write depth and cast shadow where it draws nothing, or the reverse.

`ExtendedMaterial` reports none of its fields' handles as asset dependencies. The handler
therefore adds each palette image under a label of its own (`…/nif_palette_image`, an asset whose
image is a `#[dependency]`), and the palette material depends on that label. A scene then counts
as loaded only once its palettes have loaded; without this, spawn validation could run before a
palette arrived and reject the model for good.

A palette surface that also has an authored depth override (`crate::nif_depth`: decals, surfaces
that test but do not write depth, and surfaces with depth testing off) draws through
`EffectPaletteDepthMaterial`, the depth extension wrapped around the palette material
(`…/nif_palette_depth`). The wrapper keeps the palette shaders and adds the authored depth state,
the sorted colour pass, no prepass and the authored `Cast_Shadows` flag, exactly as
`NifDepthMaterial` does for standard surfaces. The surface keeps its palette material as
`EffectPaletteDepthSource`, which spawn validation and the door crossing's readiness check read.

Not covered: effect lighting influence, falloff, soft depth, membrane shaders, `IgnoreTexAlpha`,
nonstandard blend factors and every animation controller.

`material_palette_probe --output <dir> [--interior] [--standard-material]`: converter-published
synthetic effect contracts → glTF/KTX2 → production palette material → GPU swatches beside unlit
references computed on the CPU (bilinear filtering of the decoded 4×4 palette), with a depth
prepass as the engine's cameras have. Nine cases: mid grey, HDR scale with the row from base red
only, clamped corner, colour and alpha blended, alpha only, a depth-tested decal drawn
through the depth wrapper, and three alpha-tested cases over an unlit plane: a decal through the
wrapper whose palette alpha must discard, opaque source alpha with transparent palette alpha (must
leave the plane visible, not a prepass hole) and the reverse. A second glTF names a palette file that does not exist; its
palette material's recursive load must fail rather than finish. The palette view must be
`Rgba8UnormSrgb`; tolerance 2/255; references must be visible. `--standard-material` draws the
stock material the engine used before and must fail with exit 1.

Local Apple M4 Pro, Metal: both backgrounds match exactly (0/255). Negative controls fail:
`--standard-material` at up to 112/255; the standard prepass in place of the palette prepass
leaves a 104/255 hole in the discarded masked case; without the palette image label the missing
palette's material reports itself loaded. In the full Riverwood pack, the forge's coals render as
embers instead of the white disc. Unit regressions:
`palette_lookup_reads_effect_flags_and_the_greyscale_slot`,
`handler_hands_out_the_palette_by_gltf_texture_index`,
`base_color_scale_is_recovered_from_the_published_emission`,
`effect_palette_material_needs_its_palette_loaded_through_an_srgb_view`,
`the_fade_in_waits_for_an_effect_palette_materials_source_texture_and_palette`.

## Local verification, 2026-10-02

Headless Vulkan on llvmpipe / Mesa 26.2.2, LLVM 21.1.8. No renderer errors in the six final runs. These are functional shader checks, not target-hardware performance or Skyrim visual acceptance.

| Case | Largest channel difference (8-bit) | Expected verdict |
|---|---:|---|
| Exterior, HDR `(0.18, 0.4, 2.0)` | 1 | Pass |
| Interior, HDR | 1 | Pass |
| Exterior, gray `(0.18, 0.18, 0.18)` | 0 | Pass |
| Interior, gray | 0 | Pass |
| Previous output path, HDR | 39 | Fail (negative control) |
| Previous output path, gray | 3 | Fail (negative control) |

HDR mesh/sky/fog/water samples: `(114,151,239)`; terrain: `(114,152,239)`. Previous-path sky: `(118,170,255)`; water: `(107,136,200)`. All new-path gray samples: `(115,115,115)`. Interior background: black. Pixel tolerance was fixed at 2/255 before running; the negative controls returned exit code 1.

Static emission probe: exterior and interior both pass with maximum difference 1/255; all eight loaded material checks pass. Legacy negative control returns exit 1, maximum error 230/255, seven loaded-energy/eligibility checks fail. Zero/black/Own_Emit cases render `(0,0,0)`; dim `(68,21,73)`, unit `(123,49,130)`, textured HDR `(237,145,203)`. 353 converter tests pass (13 existing ignores); 230 engine library tests pass; formatting and converter/engine Clippy libraries/tests/examples pass with warnings denied. Same llvmpipe adapter as above. Evidence: `/home/dev/Projects/mudcrab-lighting-emission-evidence/{exterior,interior,legacy}`; functional synthetic proof, not Fiji performance or vanilla retail acceptance.

Signed-tint correction: 18 failing retail NIFs → 18 converted, zero skips; isolated private-fixture run. Nine-case exterior/interior GPU probes pass ≤1/255; raw signed metadata and nonnegative glTF projection checked. Full converter suite: 354 passed, 13 existing ignores; formatting and Clippy pass. Delivery/full-pack integration passed; T7 complete.

## Fiji delivery, 2026-10-02

Package: `/home/taylor/mudcrab-pr139-8db2a3d`; binaries ! commit `8db2a3d3ef07ef6dd92ad19f0c30d0c4c065da2e` (later documentation-only commits do not change deployed binaries). Separate schema-17 pack: complete; 18 converted, 257849 cache hits, zero skipped; inputs DDS 35663 / NIF 25388 / PEX 15162. World schema 4 integration passes; missing/invalid models 0. Converter quick check passes: 76213 files, 13.1 GB. Package hashes and bundled runtime dependency resolution pass.

RX 6700 XT / RADV NAVI22 / Mesa 26.2.2: nine-case exterior/interior probes both pass, maximum difference 0/255. Legacy control fails with exit 1, maximum error 231/255. Exact packaged `run-riverwood.sh --headless` exits 0 and records `Mudcrab runtime initialized`. Existing #137 package and original schema-16 asset pack preserved; original manifest/database/cell-cache hashes unchanged.

Evidence: package `DEPLOYMENT.json`, `conversion-report.json`, `material-probe/{exterior,interior,legacy}/probe.json`, `asset-check.log`, `smoke-new-assets.log`; local copies under `/home/dev/Projects/mudcrab-lighting-emission-evidence/signed-fix/`. Installed source NIFs stay private. Synthetic GPU/startup proof ≠ matched Skyrim scene acceptance or release performance; L0 comparison and remaining L1 families remain open.

## Specular integration plan

Sources: Bill's `5a116e79c1bc6327b4bd6bdf0c65635464701236` (exponent mapping) & `92d60dc64bcfa0cb210ec0fe4349668ec7c0ce97` (normal-alpha mask/pruning). Adapt owners; do not copy unrelated fork changes. NIF Specular bit 0 and Model_Space_Normals bit 12 traced in vendored flag definitions. Bevy 0.19 glTF loader scales factor by 0.5; fragment multiplies reflectance by mask alpha × 0.5; F0 ! `0.16 * reflectance^2`. Scalar/native response remains declared approximation.

Review: fork factor doubling exceeds glTF domain → reject; native glTF extension handler owns compensation. Tagged masks ! private `/nif` material label & scene binding; no duplicate standard label. Missing/pruned mask ! no compensation. Independent converter publication/pruning tests + runtime loader/GPU comparisons ! pass; legacy behavior ! fail. Gate: GO with V9–V11. No unrelated prepass/sampler integration in this slice.

cmd: `material_specular_probe --output <dir> [--interior] [--legacy-specular]` → PNG/JSON, deterministic paired PBR samples, nonzero failure. Existing camera/exposure settings unchanged; no retail parity claim. Converter marker: `extras.openSkyrim.specularMask = "normal_alpha"`; native handler consumes only marker and actual loaded mask. Native factor compensation ! 2× only at material construction; reloads do not compound.

## Specular local verification, 2026-10-03

358 converter tests pass (13 existing ignores); 231 engine library tests pass; formatting and converter/engine Clippy libraries/tests/examples pass with warnings denied. Nine-case exterior/interior `material_specular_probe`: loaded scene bindings, native factors and shared linear textures pass; maximum paired difference 0/255 on Vulkan llvmpipe / Mesa 26.2.2. Legacy control ! exit 1; maximum difference 248/255. Cases: disabled flag, zero strength, unmasked, full/thatch/zero mask, exponents 100/200, dim strength. Emission regression probe passes ≤1/255.

Evidence: `/home/dev/Projects/mudcrab-lighting-specular-evidence/{exterior,interior,legacy,emission-regression}`. Probe holds camera/exposure/tone map, directional illuminance 5000, zero ambient and no shadows; background varies only exterior/interior. Both native scene binding and independently computed scalar-response references exercised. Retail roof appearance and full-pack delivery remain separate gates; no Skyrim BRDF/parity claim.

## Specular target-hardware verification, 2026-10-03

Full Fiji schema-18 reconversion: 25388 NIFs converted, 232479 cache hits, zero skipped; inputs DDS 35663 / NIF 25388 / PEX 15162. Schema-4 integration passes with missing/invalid models 0; quick asset check passes (76213 files, 13.1 GB). Original schema-16 and prior #139 manifest/database/cell-cache hashes unchanged. Farmhouse audit: 24 thatch materials; `farmlonghouse01.glb` material 1 roughness 0.20 → 0.395188; normal-alpha mask now shares normal texture index 3.

Initial engine ! 224 pending Riverwood instances, no screenshot. Retained source-material dependency fixes scene readiness: 0 pending, 0 load/material validation failures, screenshot captured. Small synthetic fixtures passed even without retention; full Riverwood run supplies failing-first lifetime evidence. `v12_scene_cloning_retains_source_and_native_material_handles` checks reflected scene cloning; probe loads only Scene0, includes inverted-scale variant, and checks recursive readiness again after 90 rendered frames. Engine library 232 passed; updated Clippy/formatting/build pass.

RX 6700 XT / RADV NAVI22 / Mesa 26.2.2: final exterior/interior probes pass, paired difference 0/255; legacy specular control exits 1 with difference 248/255. Matched Riverwood captures: 2540×1375; camera `(2048,1176,452)`, target `(2048,-24,-2048)`, worldspace 60, grid (5,-12), stream radius 2, EV100 9.7, TonyMcMapface. Foreground thatch ROI `[110,785,430,1000]`: encoded-RGB display brightness 161.15 → 140.64 (12.73% decrease). Right thatch `[2090,632,2230,770]`: 120.00 → 112.05 (6.63%). Stable terrain control: mean channel difference 0.154/255; 94.92% pixels identical. Geometry/camera alignment inspected; animated water excluded.

Evidence: `/home/dev/Projects/mudcrab-lighting-specular-evidence/riverwood-comparison/{before-matched.png,after-candidate.png,roof-comparison.png,metrics.json}`, `roof-material-audit.json`, final scene-only probe JSON. Before/after compare #139 versus #141 Mudcrab; blue wash, remaining lighting work, native BRDF/model-space gaps remain. Test-profile 20-second smoke benchmarks pass their configured gates; no release performance or vanilla Skyrim parity acceptance.

## Source material completion (historical schema 19; current producer 27)

`NifMaterialPlugin` extends existing native glTF hook: retain source dependency,
apply tagged normal Y once, preserve mask compensation, apply common native UV
transform even without diffuse. Generic glTF remains unchanged. Alias identity
includes transfer space and S/T clamp mode so shared image loads cannot overwrite
another material's sampler. UV transform remains per material. Historical schema 19 rebuilt GLBs and reused
then-compatible nonmesh outputs. Current producer 27 rebuilds legacy meshes and
textures from known schemas 12–24 and 26, including the formerly compatible producer 24 GLBs, and
retains only verified scripts/archive ingestion.

Riverwood gate source basis: packed NIF tangent follows texture V; split tangent
components follow U. Bevy-generated tangents negate Mikk handedness. Native shader
samples `2 * rgb - 1` in source frame. Converted mesh frame therefore requires
`flip_normal_map_y`; no pixel rewrite, sun reversal or ambient adjustment.
Community Shaders source revision `2f2919a71bed6132b125e41781304c8f6f73d002`,
[`Lighting.hlsl`](https://github.com/doodlum/skyrim-community-shaders/blob/2f2919a71bed6132b125e41781304c8f6f73d002/package/Shaders/Lighting.hlsl),
TBN construction, `TransformNormal`, vertex color and tree/LOD alpha branches.
Direct retail shader/oracle acceptance remains L0-dependent.

`material_normal_probe`: six asymmetric linear DDS normal swatches → converter
material publication → KTX2/glTF → production scene-only native loading → rendered
comparison with geometric normals. Interior/exterior ≤2/255, source alpha preserved,
inverted-scale variant, scene lifetime, all four wrap modes and no-diffuse UV transforms checked. `--legacy-normal` ! fail.
`material_alpha_probe`: fading quad vs explicit half-quad geometry; discarded and
visible color, depth, cast shadow and lit receiver samples. Both backgrounds
≤2/255; `--legacy-prepass` ! fail. Interior diagnostic background ≠ authored
interior lighting verification.

Local llvmpipe Mesa 26.2.2: normal six cases and alpha five cases both backgrounds
match exactly (0/255). Negative controls fail: normal max 61/255, prepass max
154/255. Unit regressions cover modern NIF RGBA through GLB accessors, source
channel matrix, NiAlphaProperty modes, UV/wrap identity, alias pruning/restoration,
cache migration, and exclusion positive/negative controls.

## Historical Fiji GPU verification, 2026-10-03

RX 6700 XT, RADV, Mesa 26.2.2. Both diagnostic backgrounds: normal, alpha,
specular and emission maximum error 0/255; composition maximum 1/255. All ten
positive runs pass. Legacy controls fail with exit 1: normal 61/255, alpha
154/255, specular 248/255, emission 231/255, composition 39/255.
Recorded settings, loader assertions, samples and verdicts:
[`lighting-l1-20261003.json`](../../evidence/lighting-l1-20261003.json).
Synthetic [normal](../../images/l1-normal-comparison.png) and
[alpha](../../images/l1-alpha-comparison.png) comparisons contain no retail assets.

Final workspace verification: 844 passed, 0 failed, 18 existing ignores across
`cargo test --workspace --all-targets` plus `cargo test --workspace --doc`.
Strict workspace Clippy and formatting pass. Independent [CI run 37094801783](https://github.com/Mudcrab-Team/mudcrab/actions/runs/37094801783) passes format, Clippy, tests, security and performance for source commit `dbf048e`. Added sampler/UV assertions in normal
probe subsequently pass targeted Clippy/build and local/Fiji GPU runs.

## Historical L1 delivery, 2026-10-03

[PR #142](https://github.com/Mudcrab-Team/mudcrab/pull/142), source `dbf048e`;
Fiji launcher `/home/taylor/mudcrab-l1-20261003/run-riverwood.sh`.
Optimized test-profile engine/converter and probes, bundled runtime libraries.
Previous #141 pack's manifest/database/cell-cache hashes unchanged.

Schema 19 conversion complete: 25,402 converted, 232,465 cache hits, zero skipped.
25,388 GLBs; 527 absent-source texture references pruned and recorded. World
schema 4 integration passes: missing/invalid models 0, missing textures 0.
Existing source-coverage gaps remain: unavailable model sources 128, unbounded
models 28, unavailable texture sources 16. No claim that conversion creates
content absent from installed Data.

Trusted transferred ingestion/output cache reused with `--no-verify-cache`;
source/configuration checks remain. Post-publication Fiji full size/hash check:
76,213 primary files, 13.5 GB, all pass. Separately verify all 3,998 generated
texture aliases by hard-link identity or source hash; the converter manifest's
primary-file check does not enumerate those aliases. Six differing primary
textures and affected aliases copied privately after the first hash check caught
them. Read-only 1,121,325,056-byte SquashFS stores new meshes/world data inside
package; immutable unchanged textures/scripts share old files by hard link.
Mount helper remounts after reboot and does not pass its mount lock to the daemon.

Matched Fiji captures: six views of three Riverwood gates, before/after exact
1280×800 poses; all settle, none time out. Same engine, camera, exposure and sun;
old schema-18 vs corrected schema-19 material data. Wall normal highlights change
orientation without changing global light direction. Overview checks roofs and
foliage. Retail images stay in private evidence, not the repository.

20-second Riverwood smoke runs after 1,200 warmup frames, RX 6700 XT / RADV,
private Weston GL compositor, 1280×800: before 76.38 FPS / 15.18 ms p95; after
74.38 FPS / 15.56 ms p95. Both pass configured 60 FPS / 16.67 ms gates. New run:
25 resident cells, 2,029 ready assets, 3,872 validated materials; pending assets,
load/material/terrain/water failures and diagnostic fallbacks all 0. Single short
pair, not release performance acceptance or a causal speed comparison.

Private evidence root `/home/dev/Projects/mudcrab-lighting-l1-evidence`:
`retail/conversion-report.json`, `retail/material-inventory.json`,
`fiji/{wall-before,wall-after}`, `fiji/wall-comparison.png`,
`fiji/wall-comparison-metrics.json`, `fiji/riverwood-{before,after}.json`.
[Compatibility inventory](../../lighting-l1-compatibility.md) lists shader gaps.
Implementation T4 delivered; T5 and #131 remain open for L0 matched-reference
acceptance and required unsupported material cases. L2 owns remaining hardcoded
sun/ambient/weather behavior. Lighting addons remain out of scope.

## Emission review audit (2026-10-03)

[Aggregate report](../../evidence/lighting-l1-emission-impact-20261003.json): full schema-19
conversion, 25,388 GLBs; 58,133 published lighting-material instances. Parse winning
NIF bytes by manifest source hash (first hash precedes skeleton dependencies), join
source shader block to material extras, compare pre-#139 and corrected publication
on identical inputs. Reconstructed corrected energy matches all 58,133 published
factors/strengths. Effect shaders and excluded shapes outside count; this corpus
includes installed official DLC/Creation Club content, not a base-game-only sample.

- 3,599 instances lose slot-2 glow eligibility; 0 gain it. Eligibility precedes
  missing-texture pruning, so this is not a count of visible glowing surfaces.
- 41 retained glow-eligible instances have black tint; old white fallback removed.
- 4,177 change energy factor; 4,447 change factor or eligibility across 2,512 files.
- 3,631 lose nonzero emission factor; 0 gain it. Texture samples not evaluated.
- No non-finite source emission or strength overflow in counted instances; two
  finite negative static lighting multipliers keep a zero glTF projection.

Malformed emission still fails its asset with source/shape/shader context. Pipeline
records that failure; no successful complete publication with a skipped bad asset.
No clamp of non-finite/overflowing energy introduced: audited data does not justify
one. Finite signed tints and static lighting multipliers retain V8; signed native
emission remains unsupported. Earlier endpoint attribution was unverified. Review regression tests cover non-finite
multipliers before clamping, all non-finite tint channels and invalid negative
effect-material strength. Converter suite: 357 passed, 13 existing ignores; strict Clippy
and formatting pass. L0 acceptance remains open.

Signed-static follow-up: two Volendrung source meshes reconvert successfully in an
isolated mesh-only fixture. Both retain raw multiplier `-0.6013296842575073` and
publish zero glTF strength; 14 absent fixture texture references are pruned and
recorded. This checks source parsing/publication, not rendering or world coverage.
Synthetic regression covers positive, signed and black tint with and without glow.
Existing prereview packs retain their old metadata until rebuilt; use a separate
output or `--invalidate-cache` to refresh it. No new runtime shader behavior or
asset-format version is introduced by this metadata correction.

## Review follow-up (2026-10-03)

Reflection recovery: optional component query restores missing `Exposure` before
rendering; pose/visibility continue. Shared `DEFAULT_SCENE_EV100` owns baseline.
V1 tests execute five world/visual setup systems plus physics fixture startup;
V2 regression fails before fix, then passes with observer exposure and fallback.
232 engine tests pass; strict engine Clippy and formatting pass.

[Existing terrain/water fixture comparison](../../images/l1-color-fixture-comparison.png)
and [capture conditions](../../evidence/lighting-l1-color-fixture-20261003.json):
identical camera, materials, lighting, fixed water time; previous output path vs
HDR/linear reflection. Fixture has no sky; existing composition probe covers sky.
Private capture-only patch uses WebGPU features and 32 texture/sampler limits on
llvmpipe, omits production renderer gate from screenshot readiness. Both runs emit
Xvfb/Vulkan swapchain diagnostics and fail software performance gates. Images are
review evidence, not production-device or retail-parity acceptance. Capture-only
changes reverted; shipping renderer requirements unchanged.

## Tasks

id|status|task|cites
T1|x|Trace existing color/material owners and name unsupported paths|V5
T2|x|Use explicit HDR scene composition and linear reflection storage; synchronize exposure|V1,V2
T3|x|Render paired synthetic probes and record pixel evidence|V3,V4
T8|x|Adapt Bill roughness/mask corrections; verify native loader, pruning, cache, scene-only lifetime and GPU response|V5,V9,V10,V11,V12
T4|x|Integrate existing material/prepass/sampler fixes, add converted-NIF response probes and retail deployment|V5,V13,V14,V15,V16
T7|x|Restore signed-tint NIF compatibility; verify retail reconversion and delivered pack|V5,V6,V8
T6|x|Fix static emission publication; load converted materials and compare GPU swatches; migrate cache|V5,V6,V7
T5|.|Compare both scene types against L0 references; accept declared tolerances|V3,V5

## Bugs

id|date|cause|fix
B1|2026-10-02|Non-HDR mesh shaders tone-map while sky bypasses transform|V1,V3
B2|2026-10-02|Reflection is display-mapped before water applies another transform|V2,V3
B3|2026-10-02|Xvfb launch lacked `libxkbcommon-x11` runtime path; local software Vulkan rejected optional features; baseline WebGPU limits excluded terrain bindings|Headless readback; explicit 32 texture/sampler slots; no game renderer fallback change
B4|2026-10-02|Probe used unboxed `WgpuSettings` for Bevy 0.19 `RenderCreation::Automatic`|Box settings; compile-only correction, no new invariant
B5|2026-10-02|Own_Emit enables slot-2 glow; multiplier floor, HDR clipping and white fallback alter authored emission|V6,V7
B6|2026-10-02|Probe assumed glTF material handles were StandardMaterial in Bevy 0.19|Load production PBR /std labels; compiler catches type mismatch; no new invariant
B7|2026-10-02|Schema bump changes pinned manifest configuration hash|Snapshot diff reviewed: schema 17 and corresponding configuration hash only; V7
B8|2026-10-02|Rejecting signed NIF tint applies glTF domain to source data; Fiji reconversion skips 18 base/DLC/CC assets|V8; retain source tint; glTF lower clamp remains named approximation
B9|2026-10-02|Narrowed overflow regression left single-element test loop|Clippy catches mechanical shape; remove loop; no new invariant
B10|2026-10-02|Roof specularity ignores source enable/mask; glossiness treated as percentage|V9,V10,V11
B11|2026-10-02|Fork mask compensation publishes glTF factor >1; pruning can retain invalid texture indices|V10,V11; bounded glTF plus native loader compensation
B12|2026-10-02|Pinned snapshots predate schema 18 and specular response|Reviewed diff: schema/hash, flag-disabled factor 0, exponent-derived roughness; V9–V11
B13|2026-10-02|Bevy standard-material conversion helper public fn resides in private module|Clone stock `/std` labeled asset through public LoadContext API; preserve full field mapping; compiler/probe catch wiring, no new invariant
B14|2026-10-02|Native helper retained GltfMaterial parameter after switching to StandardMaterial clone|Correct parameter type; compiler catches mechanical mismatch, no new invariant
B15|2026-10-02|Probe used pre-0.19 scene/shadow names|Use WorldAsset and shadow_maps_enabled; compiler catches API mismatch, no new invariant
B16|2026-10-02|Probe held asset borrow while inserting reference and queried immutable scene guard|Clone normal handle before insertion; mutable WorldAsset guard; compiler catches borrowing, no new invariant
B17|2026-10-02|Adding scene-binding verification exceeded probe system argument lint|Group related glTF/WorldAsset resources; Clippy covers mechanical shape, no new invariant
B18|2026-10-03|Native scene override drops stock material handle while stock hook retains dependency ID; 224 Riverwood instances stay pending; small fixture missed streaming lifetime failure|V12; retain source handle in reflected scene component; scene-only probe, scene-clone regression and failing-first Riverwood gate
B19|2026-10-03|NIF texture Y follows UV V; omitted tangents make Bevy generate opposite bitangent, reversing vertical normal relief|V13; source gate/stonewall frame audit plus independent geometric-normal probe
B20|2026-10-03|Modern BSTriShape export drops COLOR_0; blanket cutout normalization also discards ordinary authored edge alpha|V14; restore RGBA then gate channels by shader flags/tree/LOD semantics
B21|2026-10-03|Normal probe's procedural rectangle lacks tangents; all six samples render flat|V13; generate tangent frame before GPU evaluation; geometric references expose omission
B22|2026-10-03|Exclusion test names Skyrim flags wrapper but parsed source property uses historical Fallout4-named wrapper|Use actual property type; compiler catches mismatch, no new invariant
B23|2026-10-03|Pinned snapshots and pruning test paths predate per-wrap aliases and schema 19|Reviewed schema/hash, GLB size, metadata, samplers and alias diff; V15 covers behavior, no new invariant
B24|2026-10-03|Alias helper appended after test module violates strict Clippy item ordering|Move helper before tests; structural lint, no new invariant
B25|2026-10-03|Desktop compositor changes capture frame then closes window; pixman compositor lacks Vulkan surface support|Private GL headless compositor; six baseline shots settle at exact 1280×800; capture environment only, no new invariant
B26|2026-10-03|Deployment assumes every old texture is reusable; full hash check finds six changed outputs; scp drops mount-helper executable mode; FUSE daemon inherits mount lock|Replace six files and affected aliases privately; chmod helper; close lock FD in daemon; repeat full check and mount invocation. Deployment-only corrections; V4/V15 verification catches failures
B27|2026-10-03|Required reflection Exposure query silently drops camera after component removal|V2; restore missing component; regression checks pose, activation and observer/default exposure
B28|2026-10-03|Review test helper followed test module; strict Clippy rejects item order|Move helper before module; mechanical, no new invariant
B29|2026-10-03|Lighting multiplier max(0) hid NaN and negative infinity before material validation|V6; reject non-finite source before publication
B30|2026-10-03|Static lighting multiplier clamped before metadata; mistaken attribution to controller endpoints|V8; preserve raw signed multiplier; clamp only glTF projection; metadata also for nonnegative tint
B31|2026-10-03|Reference-shot prose omits existing failure counters from settle predicate|Describe zero new failures and failed timeout capture; existing shot-settling tests, no new invariant
B32|2026-10-03|Emission migration prose retained latest-only launcher requirement after shared compatibility range landed|Align prose with existing 15–19 readiness gate and schema-range tests; documentation-only, no new invariant

Camera-mod research and remaining native calibration gates are recorded in [Skyrim camera comparison](skyrim-camera-comparison.md).