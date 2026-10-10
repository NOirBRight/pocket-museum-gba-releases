# 更新清单 v1

`updates.json` 为 UTF-8 JSON。此文件是 #51 / #52 的共同契约。客户端拒绝未知的 `manifest_version`。客户端忽略顶层和 `packs` 里不认识的键，所以新增一类数据包不提升 `manifest_version`。现有对象（某一类包、附件、增量）内部的字段集合是固定的；往里面加字段、或改变现有字段含义，都要提升 `manifest_version`。发布工具只发已知键（`validate` 对多出来的键报错）。已写明的 `apk` 从必填对象放宽为对象或 null 不升版本：尚无已发布客户端，`apk` 为 null 仍是 `manifest_version: 1`。不得把解析失败当作“已是最新版”，也不得把解析失败当作“暂时没有可用的应用更新”。契约样例的 hash、证书、大小和版本都是测试值，不可发布。

## 顶层与下载位置

发布工具接受并写出的顶层字段：`manifest_version: 1`、`repository`（公开仓库 `owner/name`）、`release_tag`、`apk`、`packs`。客户端忽略此外不认识的键。公开仓库 `NOirBRight/pocket-museum-gba-releases` 已创建；与履行开源义务的 `pocket-museum-gba-oss` 分开。

客户端手动请求固定公开仓库的 `raw.githubusercontent.com/<owner>/<repo>/main/updates.json`，不需要 token，不查询私有仓库。仓库地址必须由客户端固定，不能因清单而任意切换源。资产对象固定四字段：

| 字段 | 类型与含义 |
|---|---|
| `name` | ASCII 安全文件名，无目录；同一 release 内唯一 |
| `url` | `https://github.com/{repository}/releases/download/{release_tag}/{name}`，无凭据或查询串 |
| `size_bytes` | 整数，`0 < n < 2147483648`，精确附件字节数 |
| `sha256` | 64 位小写十六进制，对下载附件原始字节计算 |

JSON 不含 ROM、用户游戏库、存档、游玩记录、自建金手指、编号登记表、凭据或私有构建路径。hash 是传输完整性，不能单独证明发布者身份。HTTPS 与固定仓库是渠道边界；APK 正式证书还须由客户端以既有安装证书/可信配置校验，不能信任清单自行宣称的新证书。

## APK

`apk` 为对象或 `null`，键必须存在。`null` 表示当前未通过本渠道提供安装包：客户端显示“暂时没有可用的应用更新”。这不是检查失败，也不发起下载。

非 null 时固定字段：`package_name`（`com.pocketmuseum.gba`）、正整数 `version_code`、非空 `version_name`、正整数 `min_sdk`、`certificate_sha256`、`notes`、`supported_pack_schemas`、`bundled_pack_versions`、`artifact`。此时 `artifact` 必填，不能为 null。没有安装包就写顶层 `apk: null`，不要留一个没有附件的 APK 对象。`version_code`、`certificate_sha256` 和 `supported_pack_schemas` 都只描述清单里的这份新 APK；留着这些字段却没有附件，客户端无法区分“有更新但下不了”和“渠道没有提供安装包”。

`notes` 是 `{ "zh": "中文变化说明", "en": "English changes" }`。`certificate_sha256` 为正式 APK 实际单签名证书的 SHA-256，不是 APK 文件 hash。比较更新用 `version_code`；安装前还须比对实际包名、版本、最低系统要求与固定正式签名，最终交系统安装器。

`supported_pack_schemas` 是 `catalog` / `media` / `cheats` 各自支持的结构版本整数数组；它只描述清单内这份新 APK。当前已安装应用认不认某个包的结构，以本地代码为准。`bundled_pack_versions` 同样有三键，值为 `{schema_version, data_version}` 或 `null`（未内置）。三包的数据版本与 APK 版本无关。

## 三类独立包

发布工具要求 `packs` 正好有 `catalog`、`media`、`cheats` 三键，可为 null（当前未提供/已撤下，不删除设备现有包）。客户端忽略此外不认识的键，不因此拒绝整份清单。`apk` 为 null 时，已列出的数据包仍按本节选择；结构是否可读仍由本地代码判断，不从缺失的 APK 对象推断。非 null 对象的字段集合固定：`schema_version`、`data_version`（正整数）、`notes`、`catalog_base`、`full`（资产对象）、`deltas`（数组）。附件和增量的字段集合同样固定。这些对象内部多出来的字段，客户端判不合格。

目录完整包是原样 SQLite，媒体完整包是现有格式的 ZIP（根目录 `manifest.json` 与 `说明.txt`），金手指完整包是原样 SQLite。清单版本必须等于包内 `pack_info`，不能只改清单来增加数据版本。

目录的 `catalog_base` 为 null；媒体与金手指为构建来源 `{schema_version, data_version}`，对应包内 `catalog_schema_version` / `catalog_data_version`。它不是最低目录版本或相等要求：较新目录按累计 ID 变更表映射旧 ID，较旧目录忽略找不到的 ID，沿用 ADR0009。游戏/版本 ID 永不复用，目录包含累计 `id_redirect`；发行工具不重发号或生成用户库。

结构不支持时不换入，提示升级 APK。数据版本每次该包发布递增；目录、媒体和金手指可以独立发布，新 release 清单可以指向复制进该 release 的未变包。无更新/当前更高版本时不降级。跳过中间版本可直接选择目标全量，或使用明示的直达增量，不自动串联多个差分。

## 增量传输契约

每个 delta 固定字段：

| 字段 | 含义 |
|---|---|
| `format` | 差分格式标识；#52 定义并实现内部结构/算法，包括新增、变化和移除。客户端只选择已支持格式 |
| `from_schema_version`, `from_data_version` | 必须精确匹配当前包的结构/数据版本 |
| `from_sha256` | 基准完整发行附件的 hash；本地导入或前次增量得到的包必须保留已验证发行身份才能匹配 |
| `to_schema_version`, `to_data_version`, `to_sha256` | 必须等于目标包与 `full.sha256` |
| `artifact` | 差分附件本身的大小、hash、下载地址 |

样例 `example-delta-v1` 只是演示字段，不表示格式已实现。#52 确定格式后填写实际标识，不凭此文件新增视频格式。`from_data_version < to_data_version`；可跳版本。不同基准、schema、hash 或不支持的 format 不可应用。支持跨结构迁移的算法须由 #52 明确验证，不能仅凭目标结构可读就假定能迁移。

附件 hash 校验与应用结果校验是两步：先校验差分字节，再由对应格式验证重建目标与目标发行身份一致。SQLite 可用目标字节 hash；ZIP 媒体解包后的结果不能直接拿文件夹 hash 与 ZIP hash 比较，#52 的格式须提供确定性的重建或文件内容清单校验。未实现这一检查的格式不能投入发布。

选择顺序：目标结构可读 → 当前数据版本较旧 → 精确匹配已安装发行身份且支持格式的增量（多个候选选最小字节数）→ 否则展示 `full` 大小供用户选择。`select_pack` 返回 `delta_choice` / `full_choice` 候选，不发网络请求、不自动转全量。没有媒体初始包同样走全量选择。

下载/应用在暂存完成，校验与切换全部成功才记录目标发行身份；失败保留旧包。续传需同时确认服务端支持 Range 与文件身份（包括 URL/hash/长度及服务端校验标识），不能因相同文件名就拼接。`to_sha256` 不是 HTTP ETag。网络恢复不触发自动更新。

## 发布顺序

公开仓库已创建。每次发行仍先在本地准备；清单含 APK 时先完成正式 APK 审计。评审通过并取得发布授权后，先上传并核验该 tag 的全部资产，再发布 release，最后提交/推送对应 `updates.json`。已公开 tag 的资产不可覆盖；修正使用新 tag。只增加清单引用不会授权视频、自动联网或私有源码发布。`apk` 为 null 的清单不附带安装包，也不代替 APK 审计。
