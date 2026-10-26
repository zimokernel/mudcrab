# 错误报告

现状：本地实施；本机平台和版本检查在下面仍然开放。

跟踪：[#88，开发者控制台和命令奇偶校验](https://github.com/Mudcrab-Team/mudcrab/issues/88)。
该问题已分配给BimingtonBill，其后续范围包括`bugreport`。
与该所有者协调实施；此规范不会更改分配。

## 目的和首次发布

玩家应该能够报告他们正在查看的场景，而无需查找日志或复制
手动启动设置。第一个版本保存了屏幕截图和一个小的诊断包，
然后打开一个预先填写的 GitHub 问题表单。播放器提供症状和再现步骤，
检查收集到的信息，附加捆绑包，然后自行提交问题。

第一个入口点是现有的开发人员控制台。设置菜单操作可以调用
当该菜单可用时提供相同的服务；它不是此版本的依赖项。自动
崩溃恢复、报告后端、Mudcrab 内的 GitHub 身份验证、自动上传、
视频捕捉和游戏回放是稍后的工作。实时报告无法恢复以下过程
已经崩溃或渲染器无法提供图像。转换器故障和发动机
启动失败需要稍后的启动器或命令行报告路径。

## 源基线

2026年10月13日检查`268f9884f7acaaa0302425331064f7268aac9d3c`，与遥控器匹配
检查时为 `main`。这些是现有的钩子，不能证明报告有效：

- [console.rs](../../../crates/engine/src/console.rs) 提供 `ConsoleRegistry`,
  `AppConsoleExt::add_console_command`，同步命令调度和控制台暂停/输入所有权。
  在检查的基线上，它没有 `bugreport` 命令。
- [app.rs](../../../crates/engine/src/app.rs) 配置 `LogPlugin` 和验收截图
  路径。该屏幕截图路径等待成功的资产准备就绪；错误报告不得使用其门。
- [shots.rs](../../../crates/engine/src/shots.rs) 使用 `ScreenshotCaptured` 并有 PNG 书写，
  超时和原子发布助手值得分享。它的结算循环和退出行为是特定的
  自动拍摄，不应复制到交互式报告中。
- [指标.rs](../../../crates/engine/src/metrics.rs),
  [streaming.rs](../../../crates/engine/src/streaming.rs)，以及
  [profiling.rs](../../../crates/engine/src/profiling.rs) 公开有用的诊断信息。一些收藏
  仅在基准测试期间运行；不能假设它们的输出在普通游戏中存在。
- [01-bug-report.yml](../../../.github/ISSUE_TEMPLATE/01-bug-report.yml) 已经要求
  描述、复制、日志、系统信息和严重性。

引擎需要自己的有界日志缓冲区，因此报告在直接从
终端。启动器捕获的输出是可选的支持证据，而不是必需的输入。

## 播放器工作流程

1. 打开控制台并运行`BugReport`。命令匹配不区分大小写，因此
   `bugreport` 有效。帮助说明它会保存诊断信息并打开 GitHub 进行审查。
2.立即显示`Collecting bug report…`。记录运行时上下文和最近的日志，然后请求
   当前的游戏窗口图像。不要等待加载完成或错误消失。
3. 将可读文件和 ZIP 存档保存在本地。显示实际的存档路径以及
   `Report saved` 或 `Report saved with missing diagnostics`，命名任何缺失的部件。
4. 在预先填写的错误报告表单上打开系统浏览器。该表格标识了报告并
   告诉玩家检查并附加 ZIP。玩家写下标题、症状、再现
   步骤和严重性，然后单击 GitHub 的提交按钮。
5. 关闭控制台将根据现有的暂停规则恢复播放。报告完成情况及
   浏览器启动永远不会恢复虚拟时间或自行捕获光标。

`BugReport local` 无需打开浏览器即可执行相同的收集。额外参数返回
`usage: BugReport [local]` 未开始工作。这也是离线和自动化的入口点。
一次只能有一份报告处于活动状态；另一个请求返回 `A bug report is already being collected`。

打开浏览器是一种交接，而不是证明表单已加载或报告已提交。
Mudcrab 不会收集 GitHub 令牌或进行问题创建 API 调用。

## 截图和暂停行为

在通常的后处理后，以当前的物理像素捕获主游戏窗口
尺寸。不捕获桌面或其他应用程序窗口。保留HUD、碰撞轮廓、
以及玩家看到的其他游戏诊断。仅隐藏控制台和报告叠加层
对于图像，然后恢复其先前的可见性。

编码前回读完成、跳过或超时后立即恢复覆盖可见性
和档案工作。当工作继续进行时，控制台可以显示进度。仅图像捕捉
阶段拥有临时叠加抑制；包装不得抑制该抑制 30 秒。

图像请求不得更改相机姿势、FOV、曝光、图形设置或流媒体
预算。在临时覆盖隐藏期间，游戏输入保持被阻止。关闭的用户
收集期间的控制台可以控制该选择；恢复不得重新打开它。如果有菜单
稍后添加条目，对该菜单的可见性应用相同的规则并暂停所有权。

分别记录上下文快照时间/帧和图像请求时间/帧。渲染和
当虚拟时间暂停时，流媒体可以继续。将相机和设置元数据与
提交捕获的帧，并且仅当渲染器提供时才识别渲染帧
那个身份。单独的回调时间戳并不是渲染帧的时间戳。不描述
捆绑作为确定性再现或精确姿势重播。

使用实时作为最后期限。允许 GPU 读回最多 10 秒，恢复每个
退出路径，如果回读失败则保留文本报告。无头会话，最小化/不可渲染
窗口、缺少摄像头或渲染器故障会产生指定的不可用组件，而不是挂起。
延迟回调带有报告 ID，并且不能覆盖另一个报告或恢复过期的请求。

## 捆绑内容

根据 UTC 时间戳和随机后缀创建报告 ID。发布一个以此命名的目录
ID和兄弟姐妹`<report-id>.zip`。 ZIP 包含一个包含以下相关文件的目录：

- `manifest.json`：`schema_version: 1`，报告ID，UTC创建时间，收集持续时间，总体
  `complete` 或 `partial` 状态，以及每个组件的状态、大小、SHA-256 和遗漏原因。
  清单本身不会散列。没有可发布清单的失败收集是一个错误，
  不是保存的报告。
- `report.json`：如下所述的结构化运行时快照。未知值为 `null`
  组件元数据中的原因；缺失值不得成为合理的默认值。
- `logs.txt`：限制最近的引擎诊断，包括时间戳、级别、目标和净化文本。
  剥离 ANSI/控制序列。包括代表的时间范围和掉落的次数，
  被截断和省略的事件。
- `screenshot.png`：捕获的游戏窗口图像（如果可用）。
- `report.md`：带有描述和复制占位符的可编辑问题草案，紧凑
  系统摘要、报告 ID、收集状态以及附加 ZIP 的说明。
- `README.txt`：内容、缺少的组件以及查看日志和图像的简短提醒
  在分享之前。屏幕截图可能包含玩家可见的名称或无法编辑的文本
  通过文本清理器可靠地进行。

每个引用的文件在发布之前都必须存在并具有声明的大小和哈希值。邮政编码条目
名称是固定的相对路径；存档生成不得遍历资产目录或遵循
用于收集额外文件的符号链接。准备并验证任务所属的目录和 ZIP
输出文件系统上的临时路径，然后使用单独的重命名进行发布。切勿覆盖
较早的报告。提交点是两个工件的成功发布；然后才显示
保存结果或打开浏览器。如果目录发布成功但ZIP发布失败，
保留有效目录并报告 `Report files preserved; ZIP could not be saved` 及其
本地路径。将其视为存档失败，并且不要自动打开 GitHub。仅清理
任务拥有的临时剩余物。保存的报告是不可变的。浏览器打开失败显示在
控制台并且不要重写其内容。

### 运行时快照

导出指定的允许列表，而不是序列化整个 ECS 世界或 `EngineConfig`：

- 构建：Mudcrab 版本、构建提供的 Git 提交、已知的脏状态、目标三重、实际
  货物简介和 Bevy 版本。释放包装提供了这些价值；源档案或
  构建时没有出处报告未知。用户提供的分析标签不是构建身份。
- 平台：操作系统/版本、架构、CPU 型号/核心数、总内存、可用进程内存
  暴露时的测量、GPU 适配器、驱动程序和渲染后端。不收集主机名，
  帐户名、序列号或网络地址。
- 会话：正常运行时间、交互/无头/固定模式、WALK/NOCLIP 状态、暂停状态、当前
  活性空间和细胞身份。可用时使用运行时活动空间状态；否则说明
  该限制不是声称发射世界空间在转换后是当前的。
- 姿势：玩家在场时的位置、摄像机位置/方向、渲染原点网格、外部
  世界空间或内部单元、投影/FOV 和显式单位/轴约定。转换外观
  通过渲染原点的渲染本地位置；分别标记渲染本地 Bevy 坐标
  来自绝对创建坐标。 NOCLIP 中缺席的球员并不是零位置球员。
- 渲染：实际窗口大小/比例、呈现模式、MSAA、色调映射/曝光和有效
  支持光照、雾、水、剔除和 LOD 设置。区分配置值
  有效的运行时值，包括游戏期间更改的设置。
- 流媒体：有效半径和预算、驻留/加载/失败单元计数、待处理资产/LOD
  工作、验证/回退计数器以及最多 32 个最近的失败记录。不要克隆无限的
  限制之前的故障向量；包括遗漏记录计数。
- 性能：在普通游戏期间测量的最近帧时间摘要和内存值，
  包括样本数量、样本间隔和测量范围。一种新的轻量级帧时间环可以
  覆盖最后 10 秒，上限为 2,048 个样本。请勿启用基准测试活动或
  自动进行昂贵的分析，并且不调用这些样本性能验收证据。
- 内容：只读数据库/转换器架构版本、转换完成状态和现有
  LOD 构建身份或转换器配置哈希（如果可用）。识别每个哈希值的含义；
  配置哈希不会对每个资产进行指纹识别。导出选定的缓存元数据，而不是
完整的清单、插件清单、世界数据库或整个包的哈希扫描。

当存在选择时，可以添加所选对象参考ID。它不需要实施
在错误报告发布之前，单击选择或其他剩余的控制台功能。

Treat non-finite numeric measurements as unavailable with a reason, preserving valid JSON.坐标
tests must cover negative exterior grids, render-origin rebasing, and the Creation/Bevy axis conversion.

### 收集限制和存储

使用这些首次发布的限制并报告每个截断或遗漏：

- 最近日志：内存中最多 2,048 个事件和 1 MiB；每个事件最多 4 KiB 的 UTF-8。
  首先逐出最旧的事件并保留有效的 UTF-8。已启用导出信息/警告/错误事件
  由当前记录器；不要启用详细/调试日志记录或更改终端过滤。保留
  缓冲区与控制台是否打开无关。
- 结构化文本：总计最多 2 MiB。序列化前绑定字段长度和失败记录。
- 屏幕截图：最多 1600 万像素的读回和 16 MiB 的编码 PNG。如果窗口超过
  回读限制或编码超出字节限制，省略图像并给出原因。请勿调整大小
  游戏窗口或默默降低渲染质量以适应。
- ZIP：最多 20 MiB。如果需要，删除可选的有效负载并记录原因；保留最少的
  有效的清单、运行时快照和问题草稿。如果无法写入，则报告失败。
- 收集：30 秒实时发布截止日期，包括 10 秒读回截止日期。
  到期时，停止等待，恢复捕获覆盖并报告超时。放弃等待浏览器
  5秒后单独发射。这些束缚了玩家的等待；阻塞操作系统调用不能
  保证在截止日期前终止。保留其工作插槽，直到它实际退出。

默认输出是 `reports/` 下特定于操作系统的每用户 Mudcrab 应用程序数据目录：
Windows 上为 `%LOCALAPPDATA%/Mudcrab/reports`，Windows 上为 `~/Library/Application Support/Mudcrab/reports`
macOS 和 Linux 上的 `$XDG_DATA_HOME/mudcrab/reports`，回退到 `~/.local/share/mudcrab/reports`。
公开 `--bug-report-dir <path>` 以进行显式覆盖和自动化测试。不要回落到
权限失败时的工作目录或资产包。仅显示已解析的本地输出路径
在本地控制台中；切勿将其嵌入问题 URL 中。保留已完成的报告直至用户删除
它们，并仅清理失败或取消的请求所拥有的临时文件。

## 隐私和导出规则

在浏览器切换之前，集合是本地的。打开 GitHub 会发送预先填写的小摘要
在网址中；它不会上传诊断档案。玩家审核控制附件和提交。

在编写任何导出之前，请用稳定的标记替换已知的家庭、资产和报告根，例如
`~`、`<assets>` 和 `<reports>`。匹配 Windows 驱动器/UNC 路径和两个路径分隔符以及
Unix 路径。屏蔽其他绝对机器路径。保留有用的相关资产名称和数字
表单 ID。通过同一导出器清理错误字符串、嵌套 JSON、问题文本和日志消息。

切勿收集环境变量、原始命令行、凭据、cookie、令牌、保存文件、
插件二进制文件、网格、纹理、数据库转储或不相关的应用程序日志。从一个开始
引擎和相关渲染器/资产加载器日志目标和已审核字段的白名单。面具已知
凭证模式，省略无法安全导出的事件，并对这些省略进行计数。不
声称模式匹配证明所有任意文本都是无秘密的；可读包仍然存在
分享前可供玩家检查。敏感的可选数据没有自动回退。

## GitHub 移交

目标 `https://github.com/Mudcrab-Team/mudcrab/issues/new` 与
`template=01-bug-report.yml`。预填充现有文本字段 ID：

- `description`：描述症状、报告ID、完整/部分状态和提醒的提示
  查看并附加 ZIP。
- `reproduction`：提示步骤和预期/实际行为。不要发明复制品
  相机坐标或控制台历史记录。
- `system-info`：清理版本/提交、操作系统、GPU/后端、模式和紧凑场景标识符。

为玩家留下标题和严重程度。不要发送日志、屏幕截图、本地路径、插件列表、
或 URL 中的完整快照。不要求标签或受让人；现有模板拥有
它的错误标签。使用 URL 库将每个参数编码为数据，而不是 shell 插值。

[GitHub 文档问题表单文本字段预填充和 URL 长度失败](https://docs.github.com/en/issues/tracking-your-work-with-issues/using-issues/creating-an-issue#creating-an-issue-from-a-url-query)。
将完整编码的 URL 限制为 4,096 字节作为产品限制，而不是声称的 GitHub 服务器限制。
如果太长，保留模板和报告ID，省略可选的汇总字段，并直接
播放器为 `report.md`。无论浏览器是否可用，都存储该可读草稿。

使用操作系统的浏览器打开 API 和固定的 HTTPS 目标。切勿使用
任意日志值或 shell 命令作为 URL。如果打开失败，显示 URL 和保存的草稿
路径，以便玩家稍后可以归档。如果需要登录，则模板已更改，或者 GitHub 无法登录
达成后，玩家可以使用本地选秀和捆绑包；无需网络可用性
收集它们。成功启动浏览器不会验证当前模板字段。

## 实现边界

添加 `BugReportPlugin` 和接受来自控制台或未来菜单的请求的服务。
控制台处理程序将请求排队并立即返回。它不得编码 PNG、创建 ZIP、
同步探测 Git，或在拥有 `&mut World` 时执行文件系统/网络工作。

将工作分为三个部分：

1. 在游戏线程上，拍摄有界允许列表快照，建立覆盖/输入所有权，
   并提交截图。实时投票结果和截止日期，即使在暂停时也是如此。
   当捕获完成、跳过或超时时恢复临时叠加抑制。
2. 关闭游戏线程，清理文本，对接收到的图像进行编码，写入/散列/存档文件，以及
   发布捆绑包。让工作人员远离 ECS 引用。绑定优秀作品，原始图像记忆，
   和排队结果；检查超时不得留下无限的孤儿工作路径。
3.发布后，报告本地结果并打开浏览器，除非请求为`local`。
   将每个完成与其请求 ID 相匹配并忽略过时的完成。

使用显式 `capturing`、`packaging`、`publishing`、`saved`、`failed` 和 `retiring` 状态。超时了
工作线程仍在运行的请求输入 `retiring`。使用 `The previous bug
report is still finishing` 拒绝新请求，直到该工作线程退出，而不是创建另一个工作线程。检查
取消以及各阶段之间以及每个发布步骤之前的截止日期。安
过期的请求无权开始另一个重命名、确认已保存的报告或打开
浏览器。已经在进行中的操作系统调用在过期后仍然可以完成；保留任何有效的任务拥有的
文件，报告其本地恢复路径（如果可用），并在工作进程结束后清理临时文件
返回。浏览器打开超时不得触发打开重复选项卡的自动重试。

通过集中共享功能重用屏幕截图助手和诊断结构。请勿重复使用
接受成功门、改变基准报告、更改记录器过滤器或引入第二个
与 Bevy 的 `LogPlugin` 冲突的日志订阅者。有界跟踪层应该提供
最近的日志环，同时保留现有的终端日志记录。报告生成必须避免记录
它自己的导出器失败回到无限递归。

安装对交互式引擎运行的支持。无头/测试路径可以通过相同的方式导出文本
屏幕截图标记为不可用且浏览器打开被禁用的服务。自动拍摄和
基准测试启动决不能作为副作用而打开浏览器。

## 验收和验证

当所有这些都满足时，第一个版本就完成了：

- [ ] `Help bugreport` 文档 `BugReport [local]`；两种拼写都通过现有的解决
  不区分大小写的注册表和无效参数不会创建文件或浏览器启动。
- [ ] 一个命令保存一个可检查的目录和有效的 ZIP，打印实际的输出路径，
  并在发布后立即打开正确的预填 GitHub 表单一次。
- [ ] 玩家可以查看/编辑表格并手动附加 ZIP。引擎从不提交
  问题或上传附件。
- [ ] Capture 适用于待处理的负载、失败的资产和诊断回退。它保留了
  可见的缺陷，而不是等待验收准备或更改渲染设置。
- [ ] PNG 中缺少控制台文本；先前的控制台状态、光标所有权和虚拟时间
  暂停状态可以在成功、失败、用户关闭和浏览器焦点更改中幸存。
- [ ] 构建、内容、场景、相机和有效设置元数据标识其来源；失踪
  测量和部分捕获仍然明确。窗口大小和坐标使用实际值。
- [ ] 在复制/分配输入之前、记录、读回、序列化和 ZIP 期间保持边界
  一代。截断计数和组件错误可以在清单中检查。
- [ ] 重复请求、延迟回调、捕获超时、无头模式、只读存储、满磁盘、
  序列化/PNG/ZIP 故障和浏览器打开故障会产生有限的、可恢复的行为。
- [ ] 隐私装置涵盖 Unix 和 Windows/UNC 路径、外部资产根、敏感嵌套
  字段、凭证模式、Unicode 和问题 URL 编码。排除的哨兵数据永远不会出现
  在导出的文件或浏览器 URL 中；允许的相对路径仍然有用。
- [ ] 离线采集成功。取消或关闭 GitHub 表单会留下本地包
  完好无损，后续报告无法覆盖它。

实施测试应涵盖导出器/清单合同、存档内容和哈希值，
限制、路径清理、URL 字段映射以及注入故障下的请求状态机。
使用无头计划控制台测试来检查暂停的进度、重复的请求、过时的结果、
和输入恢复。纯序列化测试不能证明 GPU 读回或浏览器行为。

发布前，在 macOS/Metal、Linux/Vulkan、Windows/DX12 上运行交互路径，包括
每个后端至少有一个故意损坏的资产场景。检查实际的 PNG 和存档，
确认浏览器中预填充的字段而不提交测试问题，并练习浏览器/焦点
恢复。测量空闲环缓冲区成本和报告收集对同一会话的影响
禁用报告，保留存储库现有的性能验收规则。这些是
所需的发布检查；以下本地验证未建立完整平台或
绩效验收。

## 本地实现及验证

实现是[bug_report.rs](../../../crates/engine/src/bug_report.rs)，具有单独的
快照、日志记录、隐私和导出模块。从控制台运行`BugReport`进行收集和
浏览器切换，或 `BugReport local` 用于离线报告。 `--bug-report-dir <path>` 覆盖
正常的每用户输出目录。报告还支持直接启动引擎。

对于无资产本机检查，请运行：

```sh
货物运行--锁定--配置文件快速-p引擎--示例bug_report_smoke--/tmp/mudcrab-bug-report-evidence
```

The smoke example opens its own window, deliberately requests a missing GLB, keeps failure/fallback
counters nonzero, pauses virtual time, collects a local report, verifies image/archive existence
and pause preservation, then exits. It never opens a browser or submits an issue. Its procedural
scene checks native capture mechanics; it is not a full-pack visual or performance campaign.

Historical evidence collected at original PR head
`76048172882c837d0f7ad4ba211ce7c106cd6f7b` on macOS / Apple M1 Pro / Metal; these results were
not rerun at later heads:

- Engine library tests: 463 passed and 1 failed out of 464. The failure was
  `physics::simulation_tests::jump_launches_and_returns_to_ground`, with
  `jump apex 56.491875 too low above rest 50.825207`. An untouched archive of baseline
  `268f9884f7acaaa0302425331064f7268aac9d3c` failed the same test with those values.
- Native smoke capture produced an opaque game-window PNG with the console absent, despite the
  deliberate missing asset. Virtual time remained paused, console state stayed open, and capture
  suppression was restored after image collection. Actual physical dimensions were 1280 by 720
  at scale factor 2.
- The local ZIP passed CRC checks. Every included file matched its manifest size/hash and the
  corresponding file in the readable report directory. Path-redaction sentinels were absent from
  the text exports. The snapshot retained nonzero failed-cell, asset-failure, and fallback counters.
- Navigating the prefilled GitHub route reached GitHub sign-in with the intended destination and
  query preserved. The preview profile was not signed in, so authenticated field rendering and
  manual attachment/submission were not verified.

CPU validation at commit `52dffdd64d2b187e07d8cfd02962095965ef9a09` on the merged PR head:

- Locked engine all-targets/all-features check passed.
- Bug-report library tests: 53 passed.
- Engine library CPU tests: 979 passed and 1 ignored; the physics jump test passed.
- Strict engine Clippy (`--all-targets --all-features -- -D warnings`) and formatter check passed.
- This validation did not rerun the historical native smoke, Metal capture, archive inspection, or
  GitHub sign-in flow. Those historical results are not current-head runtime acceptance evidence.

Release checks still required: real-pack reporting, signed-in GitHub form prefilling, actual browser
focus recovery, Linux/Vulkan and Windows/DX12 capture, and idle/collection performance measurements.
The current engine has no public active-space state, so that identity is explicitly unavailable;
the configured launch worldspace and exterior coordinate mapping have separate labels. Unknown
essential diagnostics make the report partial rather than silently supplying guessed values.

## Delivery order

1. Add build provenance, bounded log capture, allowlisted snapshots, sanitizer, archive publication,
   and `BugReport local`, including failure-injection coverage.
2. Add asynchronous screenshot capture and overlay restoration; verify live success and partial
   reports without acceptance readiness gates.
3. Add the reviewed GitHub-form handoff and validate existing template IDs, URL encoding, length,
   browser failure, and sign-in/offline fallback.
4. Complete native backend, privacy, input, and performance checks. Coordinate a menu action with
   the settings owner if desired; console command parity and object selection remain independent.