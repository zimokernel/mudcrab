# 资产和数据现代化管道策略

本文档概述了摄取旧版 Skyrim 格式（`.bsa`、`.dds`、`.nif`、`.hkx`、`.esm`）并输出适合现代 Web 和本机运行时（Bevy Engine、glTF 2.0、KTX2、WebGPU）的现代资产标准的转换管道。

转换器默认使用内部读卡器。 `--record-reader legacy`
选择手写阅读器； `--record-reader inhouse` 选择默认
明确地。两条路线使用相同的插件顺序、运行时间表和地形
缓存写入器。内部路由扫描内存映射记录并解码编写的内容
构建规范投影输入之前的模式条目；它不运行
支持记录的传统记录解码器。未知的记录类型仍然不透明。
通过有界的每个插件诊断，可以忽略格式错误的字段和记录；
有效的邻近记录将在发布过程中继续存在。语义地形和
在候选者替换较早的记录之前，会捕获每条记录的移动失败
可用的获胜者。传统解码在检查的记录边界或末尾恢复
当前有界组的，并且从不搜索损坏的有效负载字节
替换标头。格式错误的可选本机链接已被清除或其
省略的事件；格式错误的必需链接仅省略其记录候选者。
玩家列表、MO2 配置文件和自动发现忽略不可用的 TES4/MAST 标头及其依赖闭包，同时可用的邻居继续；非空选择减少到没有可用的插件显然会失败。选择恢复还忽略依赖循环、剩余依赖倒置以及与其依赖相关的槽溢出，如下所述。不可用的选择权限仍然是一个转换错误。
真正的删除仍然适用。创作高度被拒绝的土地
或跳过不完整的帧；真正无高度的纹理/色彩记录
保留现有的默认高度缓存行为。

两个阅读器都保留其终端诊断并返回每个插件的限制
按严重程度进行总结。 `PipelineReport.warnings` 包含数据丢失摘要：
格式错误的记录/框架、不可用的候选者、清除的可选链接、被截断
灯光 ID 以及内部解码器/适配器问题（例如省略字段或缺失）
本地化文本。 `PipelineReport.notices` 包含成功兼容
重新映射：过去插件列表的主索引被视为插件自己的列表，并且
内部读者的供应商自我兼容线。计数和第一个示例
保持每个插件和种类的限制。

现有的完整性规则要求零跳过的资产输入，并且没有
警告。可用的相邻记录仍然发布，但数据丢失警告标记
报告和转换清单都不完整，并使 CLI 退出
代码 1. 兼容性声明保持可见，不会影响完整性或
退出代码；混合运行保留了两种严重性并且保持不完整。
`--report-json`、CLI 和启动器显示警告和通知。读者
恢复不会增加单独的跳过资产计数器。元数据重建
和 `--reuse-assets` 使用相同的严重性和完整性规则
再导出记录。仅具有兼容性重新映射的官方输入仍然完整。

`record-reader.json` 标记所选前端。旧版 `projection_version=3`
保留 WRLD RNAM 有效负载和 TNAM 字符串并恢复记录感知 VMAD
所有权兼容性。版本2添加了修正的派系链接，严格原生
将所有权和本地恢复链接到版本 1 更正的 TX06/TX07 和 ARMO
世界模型预测。这些元数据合约独立于资产
制作人 27. 内部适配器版本 11 增加了投影感知获胜者入场资格
版本 10 的更正架构布局、GMST 获胜者准入、联合本地化
和文件名命名空间。所需的运行时投影检查解析本地化
录取前的字段，因此后来不可用的候选者保留其可用的
前任没有跨越承认的删除。
版本 12 保持零 FormID GMST 设置不同并防止拒绝
身份冲突会阻止稍后可用的定义。
内部身份包括
编写了架构 SHA-256、适配器版本、字符串表语言和编码，以及
`inhouse-reader-diagnostics.json` 记录有界解码器和适配器问题。
清单的 `configuration_hash` 保留元数据的完整读者身份
出处。数据库、地形缓存和 LOD 工件在每个
转换，包括恢复分期。恢复命令保留读者的选择。
转换后的资产使用单独的制作者、设置和源证明，如下所述
下面；仅进行完整的读者身份更改并不会放弃这些候选人或
存档提取清单。

内部模式下的本地化字段使用其声明的 STRINGS、DLSTRINGS 或
ILSTRINGS 银行，`Strings/<plugin>_<language>.<bank>`，来自有效存档 VFS，
松散文件优先。 `--language`（`PipelineConfig.language`，默认
`english`) 选择语言与《天际》的 `sLanguage` 一样；一定是字母，
数字和下划线，因为它是文件名的一部分。恢复命令保留它。每个经过验证的事件都保留在规范记录中。空或
丢失的 ID 会产生空的规范文本并保留其原始 ID、银行、
获胜插件和状态在`inhouse_localized_fields`；可为空的运行时名称
列保持为 NULL。可读表中缺少非零键会产生计数
通知；丢失或不可读的表格以及引用的格式错误的条目会产生
警告。空 ID 不需要其声明的银行。明确解决
空文本仍然可以与未解析的文本区分开。选定的联合或定义叶的显式
银行优先于其母行银行；未指定的叶子继承父银行，
然后默认为 STRINGS。
普通插件文本和英文字符串库使用Windows-1252，包括字节
恰好形成有效的 UTF-8。另一种语言的库被读取为 UTF-8，如
Skyrim SE 的非英语银行是（西里尔字母、波兰语），回落到 Windows-1252
对于无效 UTF-8 的文本；非英文包记录的读者身份
这是 `string_table_encoding`，而英文包的标识没有改变。
VMAD 字符串使用其独立的 UTF-8 格式。
需要归一化的文本解码后编码为UTF-8；已经准确了
ASCII 代码保留其原始宽度和终止符。原始源字节
在出处仍然可用。读者身份记录了两个编码合同。
元数据重建使用选定的读取器，同时验证保留的资产
他们原始记录的阅读器配置。较旧的未盖章包装和确切的
`{"mode":"legacy"}` 印章保留历史未版本遗留合约；
它们不被识别为版本化投影。确切的版本 1 和版本 2
遗留邮票和完整哈希值仍然被视为其原始保留资产
合同，并且不会升级到当前的预测。
Adapter-12 元数据可准确识别适配器 10 和适配器 11 保留资产合同
具有匹配的原始模式/附加字节、读取器字段和组合设置
哈希。原始读者和资产制作者在重复中保留记录
元数据重建。字符串表语言仅更改数据库文本，因此包
用一种语言阅读是用另一种语言重建的有效来源；它的证明是
检查它记录的语言。这些显式的兼容性对并不授权任意
旧的内部身份或自动扩展到未来的适配器。
新的完整配置哈希
包括版本化的旧身份，而 `configuration_hash_for_schema`
保留历史遗留哈希复制。元数据保留验证每个
针对其记录的读者合约的哈希值；版本化源不能省略其
保留资产读取器的出处并从当前的元数据读取器推断出来。
出处保留了读取器、生产者模式和组合设置哈希
跨越重复的元数据重建，以及任何每种资产设置
证明。保留的模型/纹理保留其经过验证的字节，并且可能携带较旧的字节
TXST 角色编码。明确保留不会升级他们的生产者或补助金
正常缓存资格：稍后的正常转换会检查原始生产者
以及当前设置、源依赖性和解析的纹理编码。
组合的默认主体和生成的默认步态剪辑携带单独的
`derived_outputs` 样张。元数据重用仅复制受支持的派生文件
记录配方、逻辑输入哈希值、输出大小和最终 SHA-256。它
在重复重建中保留这些原始证明，包括运行时包
没有原始 VFS 输入。它不会从未经验证的电流中重新生成它们
数据。历史派生档案无证明，无法取得个人派生
输出（包括读取、复制或哈希失败）和未知的杂散运行时文件
被遗漏并附有一份有限的通知；已核实的邻近资产仍然存在
不变。尽可能删除部分派生副本；清理失败
计入同一个通知中，而不会中止重建。这些可选的省略
不要使元数据重建不完整。使用正常转换重新生成
派生输出。记录的普通资产保留其校验和要求。
未知的来源生产者身份将被保留资产路由拒绝。
插件文件名键在本机 Windows-1252 解码后使用 Unicode 小写，
包括带重音的大小写对。相同的共享功能提供播放器和 MO2 选择、合并的 MO2 数据索引、主和身份查找以及存档附件密钥。记录签名和 GMST EditorID 保留其
单独的 ASCII 约定。文件名拼写在源证明中仍然可用；
Unicode 规范化和通用 Unicode 大小写折叠均未实现。
完整的加载顺序数据库导出替换规范/运行时行、身份
以事务方式映射和导出空间/LOD 状态。公共插件转换
替换同一导出事务中的插件元数据。子集导出保留
不相关的行并删除更新的引用的过时的外部空间行。
调用者表和资产转换证明在完全快照替换后仍然存在。
该路由不复制存档提取缓存并清除其清单条目；
下一次正常运行会在记录新的可重用文件之前提取这些存档一次
来源/完整性证明。

显式保留资产路由接受确切的 schema-15/16 生产者
记录固定 Zstd 级别 6 的配置变体。受保护的 schema-15
源出处哈希值默认质量 192、UASTC 级别 2 和脚本 ABI 1
加上这个固定的压缩级别
`b23881a864cdcfe1609779a9a27fefab2de629b50ec339101d4e0ad592e7e435`。
纹理质量和脚本 ABI 仍必须与记录的配置相匹配；
每个保留的资产都会被复制并根据其输出哈希进行验证。这个
兼容性证明不会改变正常转换或暂存缓存标识。

对于仅记录诊断，`inhouse-records <Data> <plugins.txt> <new directory>`
写入数据库、单元缓存、读取器身份和诊断报告，无需
运行资产阶段或发布运行时包。新目录还保留
选择`.strings-input`下的英文存档STRINGS输入进行比较；确实如此
不取`--language`。

`--typed-signatures TAG1,TAG2` 选择已编写的打印信封进行保留；
`--database-work-dir <new directory>` 在单独的本地磁盘上构建 SQLite 并
将关闭的数据库和诊断复制到输出中，并保留两个副本。
输出和工作目录必须是新鲜的并且与输入分开。

显式诊断选项 `--record-archive-only` 保留相同类型
选择，规范 `records` 和 `formid_map`，原始源有效负载，物理
覆盖历史记录、本地化键/文本/状态、地形图层/源顺序以及两者
地形缓存。它推迟了运行时物化表、脚本索引和
外部空间索引。脚本字节和类型化 VMAD 字段保留在记录中。
`record-bundle-scope.json` 以 `INCOMPLETE` 开头，仅声明完成
所有保留的工件和任何数据库复制完成后。它列出了保留的和
延期表格和邮票 `runtime_ready=false`，`record-reader.json` 也是如此。
此选项不会生成可运行包或声明整个运行时 SQL
等价性。默认诊断导出和普通转换器仍然构建所有
运行时视图。没有诊断模式可以转换网格、纹理、脚本或 LOD。

固定无链接标量数组使用带有共享元素的紧凑解码存储
布局。序列化仍会发出现有标记 `Array`、`Struct` 和
`Bytes` 值按源顺序排列，数值和规范字段未更改
字节。包含链接、字符串、选择器或延迟字段的布局保留
普通代表。 Rust 消费者可以使用 `Value::array_element` 或
`Value::materialized_array` 为任一表示形式；详尽的枚举匹配
必须处理 `Value::CompactArray`。流式编写器避免具体化
数组。序列化数据和运行时存档布局不会获得新的变体。

导出缓冲区在最后一个消费者之后被释放。接受的原始记录，
物理源历史记录和未解析的本地化字节仍然可以通过
他们的发布步骤。运行时投影仅克隆保留的子记录，
在过滤省略的字段之前避免冗余的全记录克隆。

在启动进程之前设置 `MUDCRAB_RECORDS_PROFILE=1` 以发出阶段时序
对 stderr 的观察。每个观察结果都会命名阶段、经过的墙秒数
和记录计数；这些计时需要一台安静的机器来进行有意义的比较。
这是单一配置文件开关。它涵盖了粗略的导出阶段，读取没有
时钟禁用时，并忽略观察写入失败。每条记录的详细信息
SQL/绑定工具不是生产导出的一部分。

内部地形出版物按插件优先级选择最新接受的 LAND
以及多个身份共享一个单元时的物理记录顺序。它保留了
标准化原始纹理 ID、插槽和混合权重
`inhouse_terrain_layers` 和 `cell_cache_preserved.rkyv`，包括每个活动
超出运行时材质容量的层。运行时 `cell_cache.rkyv`
保留其现有的每象限一碱基加五重叠限制。它删除了
以最小的创作不透明度总和进行叠加，通过删除来打破联系
首先是最高的槽位，并对每个受影响的单元格发出一个有界警告
并记录四个象限各自的丢弃数量。保留的槽位和权重会为缺失、已删除或类型不匹配的纹理目标使用显式的零 ID 占位符；已清除的 ID 与其来源赋值一起参与相同的排序。原生空值赋值会在此替换之前解析，因此不会挤掉非空赋值，也不会恢复已丢弃的权重。运行时材质构建接受任意槽位中的占位符；如果非零纹理缺少已转换图像，现有查找检查仍会报错。原生字段和填充数据会独立于这些派生缓存保留在源表中，包括已被替代的槽位赋值、原生 NULL 及其权重。保留缓存仅作辅助用途；引擎仍只加载运行时缓存。缓存布局和运行时选择保持不变；常规转换和元数据重建会重新生成这两种缓存及图层表。

可通过 `--plugins-file FILE` 进行显式玩家选择。可选的 `--skyrim-ccc-file FILE` 会覆盖游戏文件夹创建列表，并且 `--skyrim-ini-file FILE` 会提供常规测试文件设置，而无需读取环境配置文件。这些选项也以 `PipelineConfig.plugins_file`、`skyrim_ccc_file` 和 `skyrim_ini_file` 形式存在。选择规则和可恢复的缺失主行为记录在[数据库规范](db-schema.md)中。自动发现仍然是默认设置。

MO2 配置文件使用相同的官方/创建激活、Windows-1252 解码、最后重复选择和丢失主恢复。 `--mo2-instance` 和 `--mo2-profile` 在隐式活动插件后保留配置文件的文件覆盖和 `loadorder.txt` 顺序。 `loadorder.txt` 中省略的活动插件按 `plugins.txt` 顺序附加在有序插件之后，并带有警告。丢失数据条目和丢失/禁用主关闭会在可用邻居发布时发出警告并跳过。 `Skyrim.ccc` 来自物理数据旁边的游戏文件夹。当它丢失或不可读时，播放器和 MO2 选择激活已安装的 `ccBGSSSE001-Fish.esm`、`ccQDRSSE001-SurvivalMode.esl`、`ccBGSSSE037-Curios.esl`、`ccBGSSSE025-AdvDSGS.esm` 和 `_ResourcePack.esl` 的子集，按顺序在五个官方大师之后。可读的 CCC（包括空文件）仍然具有权威性。 MO2 选择和 `--plugins-file` 是互斥的。内部本地化字段使用来自存档库之前的 MO2 数据覆盖的有效获胜松散英文字符串库；畸形的获胜银行仍未通过计数诊断得到解决，而不会恢复为失败的银行。通过 `--reuse-assets` 重建元数据需要经过身份验证的物理数据和记录的插件顺序；它在源/输出工作之前拒绝 MO2 配置文件选择。对 MO2 配置文件使用正常转换。

所有插件名称键共享 Unicode 小写规范化，包括 TES4 主查找、直接 MO2 插件覆盖和插件到存档匹配。 Player、MO2 和自动发现省略了不可用的标头、缺少主闭包、依赖循环及其传递依赖。 Player 和 MO2 订单还省略了仍然放置在其主控之一及其依赖项之前的插件，而不是移动它。每条路径建立其顺序后，它保留前 254 个完整插件和 4,096 个轻插件，并忽略溢出及其依赖项，而不进行回填。有界通知到达进度和转换报告。 `skipped_plugins` 列出通过恢复删除的每个原始文件名，每个文件名都按恢复前的选择顺序列出，包括尝试模糊身份的每个文件；故意未经检查的插件不属于恢复排除项。 `load_order` 按已发布的数据库槽顺序列出幸存的原始文件名。幸存路径决定插槽和所有后续转换工作。

播放器选择和自动发现拒绝所有共享标准化文件名的已安装文件，每个不明确的身份都会收到一个有界通知。 MO2 同样拒绝获胜覆盖层中模糊的直接插件身份；通常，较高层的独特插件仍然会替换较低层。其他资产冲突保留解析器的现有验证。标头验证和主恢复仅使用剩余的文件，因此在可用的邻居继续时，模糊名称的依赖项也会被忽略。

一个共享匹配器定义存档所有权、MO2 准入和优先级：规范化存档主干等于插件主干或在其后跟一个空格、`-` 或 `_`，例如 ` - Textures`、`-Textures` 或 `_Textures`。最长的匹配安装阀杆决定所有权。仅当具有该所有者主干的插件在选择中幸存下来时，才会提取同伴；残疾和康复的同伴不能覆盖幸存的资产。没有安装直接数据所有者的存档在 MO2 之外保留独立/基本准入，包括仅存档转换。被忽略的嵌套插件不会声明所有权。

MO2 还会在恢复之前使用解析的虚拟文件系统中的所有根插件身份来检查最长安装的所有者。此清单包括未经检查和恢复的插件，其物理源文件位于启用的模组或覆盖中，而不是数据中。较短的幸存前缀无法激活更具体的被拒绝所有者的同伴。具有不同扩展名的插件可能共享一个主干；任何幸存的同根插件都会使其同伴保持活跃。活动插件的 `_` 和 `-` 同伴在 MO2 中被允许在共享匹配器下。 MO2 保留其主动同伴要求并拒绝无主档案。两个路径都保留插件加载顺序排序和松散文件优先级。每次完整转换都会重建存档输入并仅发布当前资产，因此恢复还会删除以前发布的热转换中拒绝的同伴。

元数据重建使用源包的校验和匹配插件顺序来实现存档优先级。在存档提供当前 LOD 设置或地形漫反射输入之前，安装时间最长的直接数据所有者必须按该顺序出现。已验证的源资产仍然保留。

恢复所选插件后，资产修复会选择具有相同安装所有者过滤器的替换源。它保留记录的数据库插件来源和修复目标之外的现有资产。它保留非 MO2 独立准入和 MO2 无主存档排除。存档排序使用相同的共享匹配器和最长匹配的活动词干，为相等的词干保留第一个加载顺序槽。

创建回退警告是在报告的 `selection_warnings` 中键入的条目，命名 CCC 路径、读取失败和安装的内置回退。 CLI 摘要、进度和启动器将它们显示为警告。它们描述了输入选择策略，并且不被视为失败的资产转换或使健康的已发布输出不完整。 `warnings`字段描述转换失败和读卡器数据丢失；成功的兼容性重新映射仍为 `notices`。空的 `skipped_plugins`、`load_order` 和 `selection_warnings` 字段被省略；旧报告使用空向量进行反序列化。

---

## 1. 管道概览图

```
┌────────────────────────────────┐
│ 旧天际数据 │
│ (.bsa, .dds, .nif, .hkx) │
└────────────────┬────────────────┘
                │
                ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ 泥蟹转炉管道 │
│ │
│ ┌────────────────┐ ┌──────────────┐ ┐────────────────────┐ │
│ │ 档案解包器 │ ──► │ 网格与纹理 │ ──► │ 记录/世界数据 │ │
│ │ (BSA / BA2) │ │ 转换器 │ │ 串行器 │ │
│ └────────────────-┘ └──────────────┘ └────────────────────┘ │
└────────────────────────────────┬──────────────────────────────────────────────┘
                                │
                                ▼
┌────────────────────────────────────────────────────────────────────────────┐
│ 现代资产输出格式 │
│ ┌────────────────────────┬────────────────────┬────────────────────┐ │
│ │ glTF 2.0 / glb │ KTX2 / 基础 │ JSON / BSON / Ron │ │
│ │（网格、PBR、骨架）│（GPU 纹理）│（单元和场景图）│ │
│ └────────────────────────┴────────────────────┴────────────────────┘ │
└──────────────────────────────────────────────────────────────────────────────┘
```

---

## 2. Format Conversion Matrix

| Legacy Format                 | Target Modern Format                         | Purpose                                                            | Conversion Tool / Library              |
| :---------------------------- | :------------------------------------------- | :----------------------------------------------------------------- | :------------------------------------- |
| **`.bsa` / `.ba2`**           | Directory Hierarchy / VFS                    | Extracted compressed virtual filesystem                            | Rust (`flate2`, `lz4_flex`)            |
| **`.nif`**                    | **`glTF 2.0` (`.glb`)**                      | Standard 3D geometry, mesh hierarchies, PBR materials, skinning    | `nif-parser` ➔ `gltf` crate            |
| **`.dds` (BC1-BC7)**          | **`KTX2` / Supercompressed Basis Universal** | Fast GPU texture streaming & compressed VRAM footprints            | `image` crate / `basis-universal`      |
| **`.hkx` (Havok)**            | **glTF Animations & Rapier 3D Colliders**    | Skeletal animation clips & physics collision meshes                | Havok XML exporter / custom parser     |
| **`.esm` / `.esp`**           | **`SQLite3` / Binary (Bincode / rkyv)**      | High-performance spatial indexing, memory-mapped zero-copy queries | `rusqlite` / `rkyv` / `zerocopy`       |
| **`.pex` / `.psc` (Papyrus)** | **`Luau` Scripts**                           | High-performance, sandboxed scripting engine                       | `mlua` / Papyrus-to-Luau AST Transpiler |

---

## 3. Detailed Conversion Modules

### A. Mesh Converter (`.nif` ➔ `.gltf` / `.glb`)

- **Geometry Conversion:**
  - Extract `BSTriShape` vertex buffers (Positions, Normals, UVs, Tangents).
  - Combine multi-part meshes into single glTF primitives.
- **Material Mapping:**
  - Map Skyrim `BSLightingShaderProperty` texture slots:
    - Diffuse Map ➔ glTF `baseColorTexture`
    - Normal Map ➔ glTF `normalTexture`
    - Specular Map / Environment ➔ glTF `metallicRoughnessTexture`
- **Skinning & Rigging:**
  - Map `NiSkinInstance` & `NiSkinData` bone weights directly to glTF `skins` and `joints`.

### B. Texture Transcoder (`.dds` ➔ `.ktx2`)

- Converts DirectDraw Surface files into supercompressed **Basis Universal KTX2** textures.
- Derives sRGB, linear-normal, or linear-data encoding from the texture slot recorded by the
  published GLB/world database; filenames and extensions do not determine color space.
- Preserves source alpha and authored mip levels, validates the published KTX2 metadata/hash, and
  aborts the transactional publication when any required texture fails.
- Asset paths use one lowercase canonical key. Archive overlays follow plugin load order, loose
  files have highest priority, and normalized collisions inside one source layer are fatal.
  The shared matcher attaches an archive to its longest matching active plugin stem, using
  exact matches or a following space, dash or underscore and retaining the first slot for
  equal stems. Archives with the same owner use ASCII-lowercase stem order; unattached archives
  follow attached ones. This deterministic tie policy has not been established as equivalent
  to every mod manager's archive insertion order. BSA header file-type categories are advisory;
  unknown category bits do not relax layout, count, offset or compression validation.
- Supports runtime transcoding into WebGPU/Vulkan native compressed formats (BC1/BC7 for Desktop, ASTC/ETC2 for Mobile/Web).

### C. World & Scene Database (`.esm` ➔ `SQLite3` + `rkyv` Zero-Copy Cache)

- **SQLite 3 Database (`skyrim_world.db`):**
  - **Spatial R-Tree Indexing (`rtree` module):** Query cell objects instantly using 3D Bounding Boxes (`X, Y, Z` coordinates) based on player camera position.
  - **Relational FormID Lookup:** Fast `O(1)` index table mapping 32-bit `FormID` ➔ Record payload.
  - **Tables:** `cells` (Grid coords, lighting, worldspace), `references` (Placed glTF model URIs, transforms, flags), `npcs` (Stats, race, dialogue trees).

- **Zero-Copy Hot Storage (`rkyv` / `zerocopy`):**
  - Frequently requested terrain meshes and cell data can be serialized into binary files mapped directly into RAM with `mmap`.
  - Eliminates deserialization CPU overhead during fast player movement across Skyrim's terrain.

### D. Scripting Engine (`.pex` Papyrus Bytecode ➔ `Luau` Scripts)

- **Papyrus Decompiler / Transpiler:**
  - Parse `.pex` binary bytecode or decompiled `.psc` source code into an Abstract Syntax Tree (AST).
  - Translate Papyrus features (Events, States, Properties, Native functions) directly to Luau equivalent tables and functions.
- **Luau Runtime Integration (`mlua` crate):**
  - Luau provides extreme execution speed (near C/Rust speed with JIT or optimized bytecode) and native sandboxing.
  - Expose game engine API bindings (e.g. `Game.getPlayer()`, `Actor.addItem()`, `ObjectReference.enable()`) to Luau via `mlua`.
- **State Preservation & Save Games:**
  - Modern Luau state serialization enables light, fast save-game state snapshots without Papyrus VM thread corruption.

---

## 4. Unified Async Pipeline Orchestration (`tokio`)

The converter pipeline is orchestrated asynchronously using **`tokio`** task concurrency and channels (`tokio::sync::mpsc`):

- **Non-blocking Concurrency:** Heavy I/O decompression and file transformations execute concurrently using `tokio::spawn` and `tokio::task::spawn_blocking` for CPU-bound transcode tasks (`dds` ➔ `ktx2` and `nif` ➔ `glb`).
- **Async Progress Reporting:** Sends `ProgressEvent` updates across a `tokio::sync::mpsc::Sender` to the launcher UI or CLI without thread blocking.
- **Unified Interface:** Exposes `AssetPipeline::run_async(config, progress_tx).await` as the single high-leverage entry point for modernizing game assets, and `run_async_with_cancel(config, progress_tx, cancellation)` for a caller that can stop it (the command-line converter's Ctrl+C handler).

### Reuse proofs

Current manifests record `asset_configuration_hashes` for each source kind,
independently of the full reader identity. Each hash contains only the settings
that affect that kind's bytes:

| Kind | Settings proof | Source proof |
| :--- | :--- | :--- |
| NIF -> GLB | Mesh producer contract | NIF and available skeleton bytes, plus scoped material-repair revision and source-texture presence |
| DDS -> KTX2 | Texture producer, fallback quality, UASTC level, Zstd level and CPU/GPU encoder quality | DDS bytes, resolved `TextureEncoding` from the freshly rebuilt database and accepted GLBs, plus the applicable GPU encoding label |
| PEX -> Luau | Script ABI | PEX bytes; known producer eligibility is checked separately |

`derived_outputs` is an optional map keyed by canonical output path, separate
from single-source `entries`. Each proof records a versioned recipe, a map of
canonical logical input paths to winning raw SHA-256 hashes, any recipe-owned
context digest, and the final output's size and SHA-256. `body-compose-v1`
records that sex's heavy body, hands and feet NIFs plus the shared skeleton NIF.
Current producers emit `body-compose-v2`, adding
`recipes/body-compose-v2/context.json`: the SHA-256 of compact JSON containing
the ordered heavy skin choices and thin preparation inputs. Each heavy entry
is `[slot, skin]`; each thin entry is `[slot, winning_raw_sha256, skin]`, in
body/hands/feet order. A skin is null or `[TXST_FormID, eight_NIF_texture_slots]`.
The outer JSON is `[heavy_entries, thin_entries]`. Thin entries are empty when
any thin part is missing, matching the existing preparation gate. These choices
describe main's existing catalog skin selection and thin morph generation;
physical source paths and unrelated database rows are excluded. Historical v1
proofs remain eligible for package-snapshot retention.
`default-gait-v1` consumes the body NIF,
skeleton NIF, skeleton HKX and the sex-specific gait HKX; it builds its bone tree
from the NIF inputs. Original asset producer/settings provenance remains on the
manifest. Material repair and texture pruning refresh the derived output hash
to describe the published bytes.

Normal conversion continues to regenerate these optional derived outputs and
records a proof only on success; these records do not introduce derived warm
cache hits. Absent or malformed individual proofs establish no eligibility.
Adding this provenance does not change asset bytes or the existing global cache
identities; future recipe byte changes require their own identity change.

Worker counts, GPU batch size and `--no-lod` do not affect these settings proofs.
A reader/schema edit that leaves the actual asset dependencies and encoding
unchanged therefore permits reuse. A role edit that changes one DDS's resolved
encoding changes that DDS's source proof and rebuilds it. Reuse still requires
the published output to exist with its recorded size and SHA-256.
Staging journal entries carry the same per-kind settings proof and also require
the current producer contract and matching source/output bytes; older journal
entries carrying a full configuration hash do not acquire a new proof implicitly.

Normal producer eligibility remains independent of explicit metadata retention.
The only cross-producer mesh allowance is producer 24 -> 26; a future producer
does not inherit that allowance, so producer 27 (which changes the GLB collision
data) reuses only its own meshes. Older textures regenerate, and scripts require
both a supported producer and matching ABI settings. A retained producer remains
the producer of its original bytes after any number of metadata rebuilds.

Older manifests without per-kind settings hashes must match an exact known
combined configuration hash for their original producer and a known reader
identity. An opaque historical in-house hash whose schema/adapter identity is
unavailable cannot establish asset settings, so its converted assets may rebuild
once. Archive candidates survive this conservative rejection.

Archive extraction reuse is independent of output producer, reader identity and
encoder settings. The archive source SHA-256 must match, and every recorded
content-addressed blob must exist with the recorded size. With cache verification
enabled (the default), each blob must also match its recorded SHA-256. A failed
source or blob integrity check causes that archive to be extracted again;
unaffected archives remain candidates. `--invalidate-cache` explicitly discards
both converted-asset and archive candidates.

### Failed-asset repair

`converter repair-failed <Data> <output> [--apply]` retries only the published manifest's
failed inputs using the original conversion configuration, including reader,
schema add-ons, encoding and MO2 source/profile options. Without `--apply`, the
repair remains in a separate directory for inspection. Applying a repair holds
the output locks, verifies existing outputs, updates the manifest and database,
and publishes transactionally with recovery of the last good output after an
interrupted swap. Missing winning sources remain failures; known authoring resources and geometryless overlay templates are recorded as excluded.
Repaired outputs retain per-kind asset settings and source/integrity proofs.
The manifest records record-data completeness separately from asset failures.
Only normal conversion or a metadata rebuild can establish that proof from a
fresh record read. Asset repair preserves unresolved record loss and missing
historical proof: corrected assets may publish, but the pack remains incomplete
and the repair CLI warns and exits `1`. Run normal conversion to recheck record
data; verified assets remain eligible for reuse. A healthy record source with
only asset failures can become complete after repair. Preview remains separate
from publication and succeeds when its assets and recorded record proof are valid.
Archive extraction candidates stay independent of converted-asset eligibility;
old failed entries cannot override current repaired or excluded decisions.

### Checking a converted output

`converter check <output directory> [--full]` answers "is this converted folder still good?" without
converting anything. It reads `conversion-manifest.json` in the folder and compares every artifact
it lists with what is on disk:

- **Quick** (the default, seconds): each artifact exists and has the recorded size. Only file
  metadata is read.
- **`--full`**: also re-hashes each artifact (SHA-256, in parallel) against the recorded hash.

It also reports a manifest written by another converter schema, a conversion not marked complete,
recorded input failures, and a missing `skyrim_world.db`. It prints `All good` or the problems (up to
20 lines, then a count), and exits `0` when all is good, `1` when problems were found and `2` when the
manifest is missing or unreadable. The check never writes to the folder, and a manifest path that is
absolute or leaves the folder (`..`) is reported instead of read.

A damaged artifact does not need a full reconversion: running the converter again on the same `Data`
folder and output re-hashes each published artifact before reusing it, so only the missing or
changed files are converted again. The library entry point is `converter::check_output`, with a
progress callback for front ends. `converter::check_output_with_cancel` takes the same arguments and
an `&AtomicBool` stop flag: once the flag is set no new artifact is started, the call returns after
the reads already in flight, and the result is an `Err` holding `converter::CheckCancelled` (test it
with `error.is::<CheckCancelled>()`), never a partial report.

---

## 5. Runtime Pack vs Build Workspace

Staging is a build workspace; the published output is a runtime pack. They
are not the same directory.

- Extraction writes originals to `staging/vfs/`. Archive-ingestion blobs
  persist to `<output>.assets-cache/.ingestion-cache/` outside the pack
  (overridable with `PipelineConfig::cache_dir`). Neither ships.
- Conversion writes runtime artifacts (`textures/`, `meshes/`, `scripts/`,
  `skyrim_world.db`, `cell_cache.rkyv`, `integration-report.json`,
  `conversion-manifest.json`) inside staging.
- Publication copies only `report.artifacts` plus the manifest into the
  output directory, then removes non-resumed staging. Hard links are
  preferred with a copy fallback; size audits must deduplicate inodes.
- Resume staging keeps the same layout; retained GLB invalidation still
  excludes `vfs/`.

## 6. Asset Layout & Data Integrity Invariants

1. **VFS Path Normalization (`strip_leading_kind`):**
   - BSA archives and loose mod files use mixed-case conventions (`Textures/`, `Meshes/`, `Scripts/`).
   - The asset pipeline normalizes the leading folder component case-insensitively to ensure flat target mappings (`textures/`, `meshes/`, `scripts/`) without double-nested subfolders (e.g. preventing `textures/Textures/...`).
2. **ESM FormID Remapping Isolation (`is_form_id_subrecord`):**
   - Subrecord remapping during multi-plugin merges is record-type and payload-length aware (`len == 4`).
   - Prevents unintended integer remapping on text strings (`TES4` `CNAM`/`SNAM`), physics parameters (`TREE` `CNAM`), and RGBA color structs (`CLFM`/`AACT`).
3. **Strict Little-Endian ESM Binary Parsing:**
   - All Bethesda ESM multi-byte numeric primitives (integers, floats, FormIDs, and subrecord payloads such as `ACHR` `PDTO`) are parsed as little-endian bytes (`from_le_bytes`).
4. **One File per Extracted Entry (`link_or_copy`):**
   - In the staging workspace, every extracted archive entry is stored twice, as `vfs/<path>` and as the content-addressed blob in `.ingestion-cache/sha256/<xx>/<hash>`, and on a fresh install and on a cache hit alike those two names are a hard link on one file rather than two copies (a cross-volume or linkless filesystem falls back to a copy). Neither ships in the published runtime pack (§5).
   - Every writer into `vfs/` or the cache replaces the path with a new file (renamed over it, or unlink, then copy) instead of writing through it, so a write under one name never changes the other.
   - The exception is a loose-asset override: `overlay_loose_assets` replaces the `vfs` entry with the loose file's own bytes, so that path is no longer a link to the archive entry's blob.
   - Resume and re-conversion writers follow the same replace-not-write rule: pack publication links staging files into the output, so a resumed run must unlink before rewriting any staged artifact it shares with a previous pack.

---

## 7. Command-Line Progress, Interruption and Resuming

`converter <Data> [output]` writes progress to **stderr** and the final summary to stdout, so a
pipeline can keep the outcome while the status goes to the terminal.

`--no-lod` skips generated terrain LOD in normal conversion and `--reuse-assets`.
Full-detail terrain, meshes, textures, scripts and world metadata are still produced.
The resulting database has empty LOD tables and no LOD manifest or payload directory;
an existing package's chunks are not carried forward. Omit the flag on a later run to
build LOD. Toggling it does not change ordinary asset byte identity, so verified current
mesh/texture outputs remain reusable. Resume rebuilds generated database/LOD outputs
before publishing, including when this flag changes.

Conversion owns an exclusive sibling `<output>.lock` for the entire run, including
reads of the previous package. Engine and inspector readers hold shared locks. An
engine using that output prevents conversion from starting, and conversion prevents
new readers from starting. Convert into a separate output directory to keep another
package running. Lock acquisition fails immediately rather than waiting.

Readers open an existing lock file read-only. A read-only installation therefore needs
its owner to provision a readable sibling lock before making the parent read-only.
If that file is missing and cannot be created, startup fails; reading without a lock
would permit replacement of a package while it is in use.

Publication recovery requires the destination's sibling publication record and verified
manifest/generated-file seals. An unowned `<output>.backup-*` directory stops recovery
and remains untouched. Preserve the backup and record for inspection; do not fabricate
a record, delete the backup, or rename it over an existing output. A conversion to a
different new output directory can proceed independently while the old transaction is
investigated. This applies even when the leftover backup predates this recovery protocol.

Every conversion exports remapped `XESP` enable-parent FormIDs, reference `header_flags`,
`enable_parent_id` and `enable_parent_flags`. These database changes also apply to
packages built with `--no-lod`; they are not limited to distant rendering.

### Progress events

`ProgressEvent` (`crates/converter/src/progress.rs`) carries what a status line or a GUI needs:

| Field | Meaning |
| :--- | :--- |
| `stage` | `Discovering`, `Extracting`, `Database`, `Meshes`, `Textures`, `Scripts`, `LodChunks`, `Validating`, `Publishing`, `Complete` |
| `completed`, `total` | Items done and expected for the stage; `fraction()` is `completed / total` |
| `current_file` | The asset or archive in flight |
| `bytes_completed`, `bytes_total` | Bytes done and expected, where the stage knows them cheaply: an archive's file table, or the source sizes a conversion batch sums before it starts |
| `stage_fraction` | The stage's completion when the counts understate it (extraction counts archives but works file by file) |
| `overall` | Whole-run completion in `0.0..=1.0`, from the per-stage weights in `STAGE_WEIGHTS` |

The weights are the measured shares of a fresh Skyrim SE conversion (242,969 assets in 5 h 7 m:
textures about 4 h 30 m of it, extraction 24.2 GB written, meshes about 3 minutes, validation
re-reading every artifact), which is what makes `overall` a time-based bar rather than a
stage-count one. A reconversion that reuses most outputs moves through the early stages faster
than the weights assume, so the bar runs ahead of the wall clock; that is expected.

LOD has no calibrated share of total run time. During that stage the launcher bar
shows completed/total worldspaces and is labeled `LOD <percent>%`; after the stage it
returns to the overall estimate. The underlying overall estimate remains monotonic.
No LOD ETA is shown. A long first world can keep stage completion at zero while work
continues; completed-world counts do not describe work inside that world.

Extraction reports every 512 files (its file table gives both the count and the bytes), each
conversion batch reports every asset with its source size, and validation reports per artifact.

### The status line

On a terminal, the converter keeps one line on screen and redraws it at most four times a second —
on every batch of progress events and on a 250 ms timer, so the elapsed time and the estimate keep
moving while one large asset is being converted — printing a finished line when the stage changes:

```
纹理 61% [总体 72%] 412 项/秒 61.2 MB/秒 00:12:31 已过去 ~00:04:50 剩余
```

`~… left` is an EWMA estimate over `overall` and appears only once the rate has settled (three
samples and five seconds). The shown `overall` never moves backwards, even when a stage finishes
short of its total. A redraw puts the cursor back at the start of the row and pads the line with
spaces over the one it replaces: no escape sequences, which legacy Windows consoles print
literally.

When stderr is not a terminal (a log file, CI), one plain line per stage is printed every few
seconds, and on every stage change, each stamped with the run's elapsed time. `--verbose` restores
the old behaviour: one line per converted asset.

A warning about one asset (`ProgressEvent::notice`, sent when a dangling texture reference is
pruned) is printed as its own line: on a terminal the renderer ends the open status line first, so
a warning never splices into a redrawn line.

### The summary

A finished run prints the converted, reused and failed counts, the total time, the size of the
converted artifacts, the manifest and `--report-json` paths, and when each stage ran (first event
to last, so overlapping stages are still readable). An incomplete run names the first few skipped
inputs and exits non-zero.

`--report-json` includes `lod_elapsed_ms` and `publication_elapsed_ms` measured with a
monotonic clock. LOD time covers compilation; publication time covers runtime pack
construction, seals, verification and directory publication in normal conversion, or
the new-directory rename in a metadata rebuild. It excludes later ingestion-cache
persistence/pruning. Old reports without these fields or `lod_chunks` deserialize with
zero values; zero in an old report is not a recorded measurement.

### Interruption and failure

The command-line converter installs a Ctrl+C handler (`tokio::signal::ctrl_c`). The first Ctrl+C
sets the pipeline's cancellation flag: the asset in flight finishes, no new asset is started, and
the run stops with the staging folder kept. The flag is checked between archives, between stages,
in front of every asset and once more after the last stage, so an interrupt that lands while the
run is packing up still stops it before the publish rename. A second Ctrl+C exits immediately.

**A failed run keeps its staging folder too.** The folder is everything the run has done, and
either way the converter prints what went wrong, the first few assets that failed, where the folder
is, and the exact command that resumes from it:

```
纹理期间 0:12:31 后转换失败：无法转换纹理\rock.dds：...
  失败的资产（前 1 个）：
    - 纹理\rock.dds
  保留暂存文件夹：<output>.staging-<pid>-<stamp>
  从停止处继续：
    转换器“<数据>”“<输出>”--resume-staging“<输出>.staging-<pid>-<stamp>”
  如果您想重新开始，请删除该文件夹以释放空间： <output>.staging-<pid>-<stamp>
```

`--resume-staging` accepts the folder and re-verifies the work already in it, so a stop costs the
asset in flight rather than the run. The manifest is written once, at the end of a run, so a
stopped run has recorded nothing as converted: the resume re-checks each staged file instead, and
deleting the folder only costs the work the next run has to redo.
