# Mudcrab 跨平台目标架构和策略

本文档概述了 Mudcrab 的多平台目标策略，确保跨 **桌面（Windows、Linux、macOS）** 和 **移动/ARM（Android、iOS、iPadOS）** 的本机性能。

---

## 1. 支持的目标矩阵

|平台|主要图形 API | CPU架构|构建目标|输入法 |
| ：-------------- | :--------------------- | :------------------------ | :---------------------- | :--------------------------------- |
| **Windows** | Vulkan/DirectX 12 | `x86_64` |原生 `.exe` |键盘/鼠标、游戏手柄 |
| **Linux** |伏尔甘 | `x86_64` / `aarch64` |原生二进制/Flatpak |键盘/鼠标、游戏手柄 |
| **macOS** |金属（通过 WGPU）| `aarch64`（苹果芯片）|原生 `.app` 捆绑包 |键盘/鼠标、游戏手柄 |
| **安卓** | Vulkan / OpenGL ES 3.2 | `aarch64` (ARM64) |原生 `.apk` / `.aab` |屏幕触摸、蓝牙游戏手柄 |
| **iOS / iPadOS** |金属| `aarch64` |原生 `.ipa` 捆绑包 |屏幕触摸、MFi 游戏手柄 |

---

## 2. 跨平台成功的架构支柱

### A. Bevy 和 WGPU（图形抽象）

- Bevy 使用 **`wgpu`** 作为其渲染引擎后端。
- `wgpu` 自动瞄准 **Vulkan** (Windows/Linux/Android)、**Metal** (macOS/iOS) 和 **DirectX 12** (Windows)，无需为每个操作系统编写单独的渲染代码。

### B. 资产管道一致性（glTF 2.0 和 KTX2）

- 通过将 `.dds` 转换为 **KTX2 Basis Universal** 并将 `.nif` 转换为 **glTF 2.0 (`.glb`)**，资产 **100% 与平台无关**。
- 移动 GPU（Adreno、Mali、Apple GPU）将 KTX2 无缝解包为 **ASTC / ETC2** 格式，而桌面 GPU 则解包为 **BC1 / BC7**。

### C. 触摸和游戏手柄响应控制

- Mudcrab 的 UI 引擎包含适用于 Android 和 iOS 的 **自适应触摸覆盖系统**。
- 自动输入设备切换：
  - 鼠标/键盘 ➔ Xbox/PlayStation 控制器 ➔ 屏幕虚拟操纵杆。

---

## 3. 特定于平台的注意事项

### 📱 安卓 (`aarch64-linux-android`)

- 使用 `cargo-apk` / `ndk-build` 构建。
- 使用 Android 存储访问框架 (SAF) 在启动器设置期间选择 Skyrim 安装文件。
- 内存管理：KTX2 纹理流可防止具有 4GB - 6GB RAM 的移动设备上出现内存不足 (OOM) 崩溃。

### 🍏 macOS 和 iOS (`aarch64-apple-darwin` / `aarch64-apple-ios`)

- 通过 Metal 在 Apple Silicon (M1/M2/M3/M4) 芯片上完全原生。
- iOS 构建编译具有触摸 UI 覆盖层和 MFi 游戏手柄支持。

### 🐧 Linux (`x86_64-unknown-linux-gnu`)

- 对 Wine/Proton 的零依赖。使用 Vulkan 支持编译的本机 Linux 二进制文件。
- 通过 AppImage 和 Flatpak 进行便携式分发。