# mod 记录的架构附加组件

内部转换器可以加载 mod 作者提供的 JSON 模式进行解码
新记录类型或现有类型的新子记录。一个模式使得
声明的值可供工具使用，并在这些值中启用 FormID 重新映射
价值观。引擎游戏支持、运行时表、UI 和脚本执行仍然
需要自己实现。

如果没有附加组件，阅读器可以安全地将未知子记录保留为不透明的
`records.data` 中的规范字段，按本机出现顺序。重复
未知的签名仍然单独出现。原来解压后的
有效负载也保留在 `inhouse_source_records` 中。不搜索不透明字节
对于可能的 FormID：此类单词保留其本机文件相关值。
具有显式内置通用挂钩的字段，目前 VMAD 处于其他状态
不支持的记录类型，仍然使用该钩子及其类型化链接处理。

## 选择模式

使用内部读取器并按所需顺序重复 `--schema-addon`：

```text
转换器数据modern_assets --record-reader inhouse --schema-addon schemas/mod-links.json --no-lod
```

Library callers set `PipelineConfig.schema_addons`, an ordered `Vec<PathBuf>`,
and `record_reader = RecordReader::Inhouse`. The corresponding serialized
configuration fields are:

```json
{
  "record_reader": "内部",
  “schema_addons”：[“schemas/mod-links.json”]
}
```

This fragment supplements the other required configuration fields. An omitted
`schema_addons` list defaults to empty. The CLI uses the same ordered loader;
selecting add-ons with the legacy reader is rejected. Conversion validates and
freezes the complete batch before creating or recovering output. A bad add-on
causes one configuration error without applying any portion of the batch or
replacing an existing pack.

## Format and ownership

The top-level envelope permits only `version`, `name`, `definitions`, and
`records`. `version` must be `1`; `definitions` is optional. `name` contains
1-128 ASCII letters, digits, `.`, `_`, or `-`, excluding the standalone names
`.` and `..`. Names must be unique across the ordered batch. At most 64 add-ons,
8 MiB per file, and 32 MiB of total source JSON are accepted.

Each record entry accepts `signature`, optional `source`, `fields`, optional
`allow_unordered`, and optional `extends`. A new type uses the same field format,
kind validation, definition resolution, nesting bounds, and supported deciders
as the [compiled schema](../../../crates/converter/src/records/SCHEMA-FORMAT.md).
Schemas supply data; a JSON file cannot add executable decider code.

Built-in records and definitions retain ownership. Declaring an existing record
without `extends: true` is an error. With `extends`, only new standalone
subrecords with explicit signatures can be added. They may contain byte structs,
arrays, and unions, but
cannot introduce a subrecord group, reuse an existing subrecord signature or
decoded role name, or change the record's ordering policy. Multiple add-ons may
extend the same type when their additions are disjoint. Duplicate record
declarations within one file, duplicate definitions, and conflicting additions
are rejected, including definitions with identical replacement bytes.

Extension fields match before, between, or after ordinary fields without moving
the ordinary schema cursor. They retain their physical position in canonical
data. `repeat: true` allows repeated occurrences; a second occurrence of a
singleton extension is a rejected known field. New record types use the normal
ordered matcher unless they explicitly set `allow_unordered: true`.

## Worked example

Save this synthetic format as `schemas/mod-links.json`:

```json
{
  “版本”：1，
  “名称”：“合成模组链接”，
  “定义”：{}，
  “记录”：[
    {
      “签名”：“ZNW1”，
      “字段”：[
        {"signature": "EDID", "name": "editor_id", "kind": "zstring"},
        {“签名”：“SCAL”，“名称”：“custom_value”，“种类”：“u16”}，
        {
          “签名”：“XLNK”，
          “名称”：“自定义链接”，
          “种类”：“form_id”，
          “目标”：[“KYWD”]
        }
      ]
    },
    {
      “签名”：“KYWD”，
      “延伸”：正确，
      “字段”：[
        {
          “签名”：“XLNK”，
          “名称”：“自定义链接”，
          “种类”：“form_id”，
          “目标”：[“KYWD”]
        }
      ]
    }
  ]
}
```

`ZNW1` is a new type with a string, a little-endian 16-bit unsigned value, and a
four-byte keyword reference. The existing `KYWD` schema keeps its ordinary
fields while gaining the independent `XLNK` reference. A source `SCAL` containing
`7E A1` becomes unsigned `0xA17E` with this add-on. Before the add-on it remains
the opaque byte sequence `7E A1`.

Suppose the plugin's native master index zero names a master assigned global
slot two, and `XLNK` contains the native keyword link `0x00001801`. Without the
add-on, both new-record and keyword-extension links remain the bytes
`01 18 00 00`. With it, both links become `Value::FormId(0x02001801)` and canonical
bytes `01 18 00 02`, provided the winning target is a `KYWD`. The record header's
identity uses the usual full/light load-order rules independently of whether
its payload is opaque or schema-decoded.

Any additional safely framed `XEND` field remains opaque and in place in both
conversions. A malformed declared `XLNK` is omitted from canonical data with a
bounded field diagnostic; an invalid optional link is cleared to zero with a
bounded link diagnostic. Recognized fields rejected by ordered matching also
remain omitted. Opaque preservation does not reintroduce those rejected bytes.
Valid neighboring fields and records continue through publication.

If `ZNW1` uses VMAD, declare the shared `vmad` definition explicitly at its
appropriate ordered position, for example
`{"signature":"VMAD","definition":"vmad"}`. The common-fields VMAD fallback
applies to unsupported record types; it does not implicitly add undeclared
fields to a new supported schema.

## Reproducing output

The conversion freezes the exact add-on documents for its run. The pack's
`conversion-manifest.json` and `record-reader.json` record the ordered add-on
names and exact-byte SHA-256 hashes. The reader's schema fingerprint includes
the built-in schema and ordered add-on identities. Changing source whitespace
or add-on order changes this fingerprint; relocating an identical file does
not. An empty list retains the embedded schema's original hash. Mesh, texture,
and script asset-settings hashes remain independent of these record schemas.

The database stores every original document in
`inhouse_schema_addons(ordinal, name, sha256, schema_json)`, with `schema_json` as
exact bytes. Header provenance in `record_overrides` retains the producing
schema fingerprint. History replay checks the stored documents' order, names,
hashes, and combined fingerprint before preparing their decoder. It uses these
retained documents rather than the original schema files, so changing or moving
those files after conversion does not change a saved history result. A damaged
or incompatible schema snapshot prevents typed replay; it never silently
falls back to a different schema.

Add-ons define decoding and preservation contracts. They do not create a new
runtime gameplay system, guarantee support for every mod, or infer semantics
inside bytes that remain opaque. New typed record families can require separate
runtime consumers and compatibility work.