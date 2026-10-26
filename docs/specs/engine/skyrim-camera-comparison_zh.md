# 匹配《Skyrim》和《Mudcrab》相机拍摄的照片

Mudcrab 的 `--shots` 文件指定绝对创建单元摄像机位置、偏航、俯仰、水平 FOV 和像素尺寸。 Riverwood 灯具为 `crates/engine/tests/fixtures/color-grade-riverwood-shots.json`。 `scripts/capture_color_grade.py` 捕捉相同的三个姿势，其中包括分级、仅饱和度和完整实验分级。它会拒绝不稳定的捕捉、不匹配的姿势或尺寸以及失败的引擎运行。对于这些静态镜头，水动画固定为 1.0 秒。

## 基于时间轴的相机框架

[FreeCamera框架](https://www.nexusmods.com/skyrimspecialedition/mods/174046)有[公共源代码和文档](https://github.com/staalo18/FreeCameraFramework)。其[时间轴格式](https://github.com/staalo18/FreeCameraFramework/blob/main/Documentation/TimelineFileExample/TIMELINE_FORMAT.md)接受YAML世界坐标转换、世界旋转`[pitch, roll, yaw]`和FOV点。设置 `followGround: false` 以防止自动更改请求的 Z 坐标。设置 `allowUserRotation: false`、`useDegrees: true` 和 `playbackMode: wait` 以保持最终姿势。

源检查确认 `TimelineManager::PlayTimeline` 将时间线位置写入自由相机平移，将俯仰/偏航写入其旋转，并将 FOV 写入 `PlayerCamera::worldFOV`。这建立了声明性输入路径。它不会在《Skyrim》和《Bevy》之间建立像素相同的投影。该框架还需要一个客户端模组或其示例来导入和播放时间线；它不是一个独立的照片模式界面。

露台姿势的草案是：

```yaml
格式版本：1
播放模式：等待
在播放期间显示菜单： false
允许用户旋转：假
跟随地面：假
使用度：true
翻译要点：
  - 时间：0.0
    类型：世界
    位置：[23400.0，-47100.0，400.0]
旋转点：
  - 时间：0.0
    类型：世界
    旋转：[7.0，0.0，315.0]
视点：
  - 时间：0.0
    视野：75.0
```

These angle and FOV values are calibration candidates, not verified native mappings. Before treating the resulting screenshots as references, check heading zero, yaw direction, pitch sign, camera position readback, and Skyrim's FOV convention at the chosen aspect ratio. Capture from the camera's actual eye, rather than assuming the player's position equals it.

Load the correct worldspace/cell separately and allow streaming and LOD to finish. Match weather, time of day, resolution, UI visibility, and the retail rendering configuration. The framework's verified camera input path does not automate these controls or screenshot saving. Its documented dependencies include SKSE64, Address Library, and powerofthree's Tweaks; verify runtime compatibility against the installed Skyrim version before deployment.

## Photo Mode JSON presets: a simpler route for stills

Authenticated Nexus metadata identifies [Photo Mode](https://www.nexusmods.com/skyrimspecialedition/mods/91701) as version 3.0.4. Its matching [public source](https://github.com/powerof3/PhotoMode/blob/bf50c35/src/PhotoMode/Tabs/CameraPositions.cpp) reads and writes `CameraPosition_<timestamp>.json` presets in the game's Documents folder under `Saves\PhotoMode\CameraPositions`. The [serialized fields](https://github.com/powerof3/PhotoMode/blob/bf50c35/src/PhotoMode/Tabs/CameraPositions.h) include `position`, `rotation`, `fov`, `viewRoll`, `pcPosition`, `pcRotation`, and `pcCell`.

This provides a declarative camera input without building a separate framework client. Save one native preset first, preserve its metadata and cell/player fields, and edit the camera pose fields for each Mudcrab shot. `rotation` is the free-camera's two-component pitch/yaw, stored directly in native radians; it is not the framework's three-component degree array. `ApplyToCamera` restores translation, nonzero rotation, world FOV, and view roll. A rotation array of `[0, 0]` is skipped by the loader, so it cannot reliably restore that orientation from an arbitrary prior pose. Loading a preset also does not automatically load or teleport into an unattached cell. Load the scene first and calibrate the camera's angle signs and FOV against Mudcrab before accepting a reference.

The same mod exposes time freezing, time-of-day, and weather controls. For matched stills, start with Photo Mode's JSON presets and these controls; use FreeCamera Framework when timeline playback or a custom API client is needed. Source and metadata establish the supported input format; the isolated session subsequently verified Photo Mode loading and native camera readback, while cross-engine calibration remains open.

## Other candidates

| Tool | Useful role | Current limit |
| --- | --- | --- |
| [AutoTest](https://www.nexusmods.com/skyrimspecialedition/mods/42520), [source](https://github.com/Muriel-Salvan/AutoTest) | JSON test suites, location traversal, startup/exit automation | Default worldspace/grid or cell locations and camera panning do not specify an exact eye pose. A custom camera test adapter is needed. |
| [Photo Mode](https://www.nexusmods.com/skyrimspecialedition/mods/91701), [source](https://github.com/powerof3/PhotoMode) | Manual camera framing, saved camera state, time/weather controls, and screenshots | Editable JSON camera presets are supported in 3.0.4; native projection calibration and screenshot automation remain unverified. |
| [Screenshot Location and Metadata](https://github.com/maxkagamine/ScreenshotLocationAndMetadata) | Candidate metadata recorder for capture readback | Recording metadata does not establish camera input control. Verify the fields and runtime compatibility before relying on it. |
| [Camera Scripter](https://www.nexusmods.com/skyrim/mods/79041) | Legacy `.scs` camera scripting | A 32-bit Skyrim mod; the author's SE port restriction makes it unsuitable for this SE workflow. |

The present evidence includes public documentation/source, Mudcrab captures, and the [isolated Skyrim session](../../../tools/skyrim-comparison/README.md): SKSE and five native plugins loaded, and Photo Mode wrote a native camera readback for the Riverwood overlook. No matched retail capture or retail color-grading parity was verified.

Authenticated Nexus metadata checked on 2026-10-12 confirms FreeCamera Framework 1.1.0 (174046), AutoTest 1.2.0 (42520), and Photo Mode 3.0.4 (91701). Photo Mode also exposes time freezing, time-of-day, and weather controls; preserve the target game state and avoid its image filters when collecting vanilla references.