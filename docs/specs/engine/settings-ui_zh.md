# 交互设置和控制

打开右上角的 **设置/控制**，或按 **Escape**。简单的 Bevy UI 是暂时的；渲染偏好及其消费者与其外观无关。

菜单暂停虚拟时间，释放光标，并清除待处理的移动输入；空中玩家保持其速度和跳跃状态，并在恢复后恢复弧线。 Escape 或 Resume 将其关闭。游戏过程中，F3释放光标而不打开菜单；左键单击再次捕获它。光标捕获会在下一帧返回，因此关闭单击或按键无法进入游戏。打开菜单之前存在的暂停将保持暂停状态。开发者控制台和菜单不能相互打开。

按钮支持鼠标单击和 Tab / Shift+Tab 导航（通过 Enter / Space 激活）。左/右调整焦点设置，或在选项卡具有焦点时切换选项卡。仅当前页面参与键盘导航；聚焦按钮滚动到视图中。使用滚轮滚动、Page Up / Page Down、或 Home / End；滚动条显示页面溢出时的位置。

设置按标题分组（例如点光源、太阳阴影、水、每帧预算）。切换读取为开/关开关。值与默认值不同的行在其开头显示一个强调栏，并且控制选项关闭的设置（例如点光源关闭时的灯光预算）会变暗，但保持可调。页脚显示状态行，其中“保存”作为主要操作。

## 图形

- 点光源、启用的光源预算、点光源阴影和阴影贴图分辨率。启用灯光也会在已加载的单元格中创建它们；禁用它们会隐藏这些灯。创作的无效、负面和最初关闭的灯保留其现有的排除规则。
- 太阳光照、照度和 RGB 颜色；环境亮度和 RGB 颜色。
- 太阳阴影、阴影贴图分辨率、级联计数和距离乘数。底层阴影范围遵循完整细节卸载网格。
- 天空穹顶、天空亮度乘数、外部距离雾和雾密度乘数。内部单元保留其现有的天空/雾规则。
- 水反射和反射倍增器；波浪、流动法线、波浪尺度、速度和强度。反射乘数叠加在每个水体转换后的菲涅耳值和反射率值上。
- CPU 截头体剔除、GPU 间接绘图、GPU 遮挡剔除和深度预通道。启用遮挡可启用其深度预通道和间接绘制依赖性；禁用任一依赖项都会禁用遮挡。
- MSAA、曝光、色调映射、垂直 FOV、垂直同步、窗口模式和窗口分辨率。步行冲刺将其现有的 FOV 加值添加到所选的基本 FOV 中。
- 颜色分级：关闭、饱和度、完整（传统）、批准的 sRGB、线性快照和记录 sRGB，以及实时饱和度、亮度、对比度、枢轴、色调量和色调 RGB。默认值保留批准的沉睡巨人 sRGB 外观和冷灰色天空/雾。与阶段相关的行在不活动时变暗。反射保持未分级。控制台命令同步菜单。有关域、持久性、启动优先级和捕获隔离，请参阅下面的集成合同。
- 后续加载的状态 HUD 和诊断资产回退。现有的失败实例保持其当前的回退状态。碰撞轮廓仍然是“图形”选项卡和控制台中可用的仅会话切换。

该小组将房间剔除列为不可用，直到 #188 提供实现为止。 HDR 场景合成保持启用状态。显示渲染器后端和适配器信息；后端/设备功能需要渲染器初始化。编写的材质颜色、纹理平铺和每个水的因子仍然是资产数据。

## 流媒体

用户界面显示完整的细节和卸载半径；所有四种受支持的地形 LOD 距离设置；单元提交计数/时间预算；单元卸载、模型生成和渲染资产上传预算；和 IO 线程数。 LOD 距离更改会刷新当前摄像机位置处的查询。卸载半径必须覆盖完整细节半径。

IO 线程在启动时创建。更改该值需要保存并重新启动；其他设置在会话期间应用，诊断回退在后续加载时生效。数字 UI 限制了绑定的交互式编辑。只有在标签上注明的情况下，零预算才是无限的。

## 保存的首选项和自动化

Save 通过临时文件写入经过验证的 JSON 文件并重命名。重置默认值将默认值应用于会话；保存重置后仍然存在。菜单中出现读取或写入失败。无效的保存文件将被忽略，而不更改启动配置。

显式启动选项设置的字段不会从运行值写入。 “保存”保留运行之前保存的文件值，或者在不存在有效文件时保留菜单默认值。运行设置保留启动值，并且文件在以后运行时保持有效（无需该选项）。显式选项在下次发布时仍然会胜出。其他字段保存如菜单中所示。

|平台|文件 |
| ---| ---|
| macOS | `~/Library/Application Support/Mudcrab/settings.json` |
|窗户| `%APPDATA%/Mudcrab/settings.json` |
| Linux | `$XDG_CONFIG_HOME/Mudcrab/settings.json` 或 `~/.config/Mudcrab/settings.json` |

仅针对交互式世界游戏加载首选项。显式 CLI 选项和有效的受支持的 INI 键会覆盖这些设置的已保存值。基准测试、无头、夹具、自动飞行、验收屏幕截图和镜头运行保持现有配置，不安装菜单或加载首选项。它们的启动选项对于自动化仍然有用。

仅启动输入（例如资源路径、世界空间/启动单元、后端选择、夹具选择和捕获/分析输出）不是运行时渲染首选项。该菜单不添加引擎启动标志。

## 控制

“控制”选项卡列出了菜单/光标键、控制台输入和完成、鼠标外观、移动、NOCLIP/WALK 切换、飞行/跳跃、慢走/快速飞行、冲刺、碰撞轮廓、调试酒杯和控制台帮助。 HUD 报告运动、冲刺、速度、碰撞状态和阻塞行走原因，而无需重复绑定。

## 原生预览

`python3 scripts/workspace.py cargo run --profile quick -p engine --example settings_menu` 使用真实的天空、雾、水和渲染器系统在无资产场景上打开实际菜单。它不会加载或保存个人偏好。

对于所有三个选项卡的有界捕获：

```sh
MUDCRAB_UI_PREVIEW_DIR=/tmp/mudcrab-ui python3 脚本/workspace.py 货物运行 --profile 快速 -p 引擎 --example settings_menu
```

The preview saves `graphics.png`, `streaming.png`, and `controls.png`, then exits after the final screenshot completes. This verifies the native UI/render path; it is not a gameplay or performance acceptance run.

## Approved grading integration

PR #262 incorporates the complete #257 head (including review fix 5db0b05), and
also requires the settings foundation in #229. Keep #257 as the grading/capture
owner; merge both foundations before integrating the menu. No grading code needs
to be copied back into #257.

| Mode | Saturation | Brightness, contrast and tint |
| --- | --- | --- |
| Off | Identity | Disabled |
| Saturation only | Bevy linear post-tonemap | Disabled |
| Full (legacy) | Bevy linear post-tonemap | sRGB post pass |
| Approved sRGB | sRGB post pass | sRGB post pass |
| Snapshot linear | Linear post pass | Linear post pass |
| Record sRGB | Loaded IMGS CNAM/TNAM in sRGB post pass | sRGB post pass |

Fresh interactive preferences and Reset defaults select Approved sRGB and Cool
Gray: saturation 1.66, brightness 1.38, contrast 1.50, tint amount 0.56 toward RGB
(0.88, 0.81, 0.72), pivot 0.18. Cool Gray retains #257's quarter-density blue-white
fog. The menu's sky/fog preset does not discard custom grading values.

All six modes, preset, tuning and pivot persist in the existing validated JSON.
The override mask records only fields explicitly changed by the user. Unedited
rows display the resolved source values, including authored values outside the
menu's edit range; loading those values does not turn them into overrides.
Console SceneGrade and ScenePreset select the canonical mode/preset grade and
synchronize the menu and tuning. Rejected commands leave source and settings
unchanged. Reset clears the override mask and restores the approved sRGB / Cool
Gray defaults.

Startup precedence is field-specific. Valid INI values and explicit CLI options
overlay saved preferences for their corresponding fields. `--color-grade`
owns the active mode and clears saved numeric overrides for that launch, so the
selected source supplies canonical values. `--visual-preset` owns only the
preset and keeps saved mode and tuning. Explicit `--grade-preset`, `--tone-map`
and `--tone-map-params` sources protect the active mode and numeric overrides
from stale saved values. An image-space selector pins authored source data but
does not select Record sRGB by itself. A previously saved Record sRGB mode uses
that source only when selection is explicit; without a selection it reports
unavailability and uses the approved default. The grade source and tuning
resolve before the first visible camera frame.

Save preserves the stored grade mode and override values for fields owned by
launch sources, while saving unrelated menu changes. It does not write the
current CLI/source-derived grade into preferences. Legacy Vivid is available
with `--visual-preset vivid --color-grade full`.

The Bevy tone-map menu selection flows through the shared resolver. Explicit
`--tone-map retail-curve` owns camera tonemapping (`None`) and keeps its loaded
HNAM White and curve parameters across unrelated menu edits; the Bevy tone-map
choice remains available on the Tony path. Exposure still applies to scene and
reflection cameras, while reflections remain ungraded.

Interactive tuning exists only in interactive world runs (and the asset-free
preview). Hidden-window, shots, acceptance, benchmark and fixture runs do not load
preferences or create tuning. They retain #257's mode/preset resolution and receipt
contracts, including the original 1.15 brightness for snapshot-linear. The
cinematic shader, capture script, comparison fixtures and review corrections are
unchanged from #257. Reflections remain HDR and ungraded; tuning affects scene
views only.

Default sun and ambient colors now derive from WeatherLighting instead of the
settings foundation's earlier white/blue approximations. The fog-density slider
scales atmospheric extinction and inscattering while retaining the preset,
including its quarter-density Cool Gray baseline.

At commit `52266688`, 1,054 engine-library CPU tests passed with 1 ignored,
strict engine/shared Clippy passed for all targets and features with `-D warnings`,
and workspace formatting passed. Native UI, GPU behavior, visual parity, gameplay
and performance were not exercised. CI on the published head remains a merge gate.