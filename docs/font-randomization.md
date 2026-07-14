# 随机字体工具函数

SDK 提供随机字体工具函数，用于按目标 OS 生成可传给 `fingerprint.font` 的自定义字体列表。SDK 不会在启动链路中擅自替业务方决定字体；业务方必须显式调用工具函数，并把返回结果通过 `font: { type: 'custom', value }` 传入。

## 函数定位

- `getFonts(os, fontCount?, options?)`：业务侧主入口。已确定目标 OS 时优先使用这个函数。
- `createRandomFontValue(options?)`：高级入口。适合业务先有多个候选 OS、暂未决定最终 UA 的场景。
- `resolveFontPlatforms(context)`：平台解析辅助函数，主要用于调试或业务自检，普通接入通常不需要调用。
- `getSelectableFontList(context)`：查看随机抽样池，主要用于调试和排查数据。

## 数据来源

字体数据随 SDK 内置在 `assets/font-data/`：

- `common.txt`：多平台共享的随机抽样池
- `win_full_fonts_unique.txt`：Windows unique 字体
- `mac_full_fonts_unique.txt`：macOS unique 字体
- `linux_full_fonts_unique.txt`：Linux unique 字体
- `ios_full_fonts_unique.txt`：iOS unique 字体
- `android_full_fonts_unique.txt`：Android unique 字体
- `minimum_sets.json`：各平台必须补齐的 minimum 字体集

SDK 不会扫描运行机器系统字体，也不会联网获取字体数据。

## 单平台用法

```javascript
const { createSDK, getFonts, OsType } = require('general-sdk');

const sdk = createSDK();

// 默认：Windows minimum 字体集 + 随机 40-60 个额外字体
const fonts = getFonts(OsType.Windows);

const { instanceId, fingerprintConfig } = await sdk.createFingerprint({
  uaOs: OsType.Windows,
  userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...',
  fingerprint: {
    font: {
      type: 'custom',
      value: fonts,
    },
  },
});
```

## fontCount 参数含义

`fontCount` 是随机抽样数量，不是最终结果数量。SDK 会在随机抽样后补齐目标平台的 minimum 字体集，所以最终结果数量由两部分共同决定：

```text
最终字体列表 = 随机抽样字体 + 目标 OS minimum 字体集 - 去重
```

不传 `fontCount` 时：

```javascript
const fonts = getFonts(OsType.Windows);
```

SDK 返回 `目标平台 minimum 字体集 + 随机 40-60 个额外字体`。这里的额外字体会从抽样池里排除 minimum 字体后再抽取，所以业务更容易理解最终数量范围。

传入 `fontCount` 时：

```javascript
const fonts = getFonts(OsType.Windows, 90);
```

SDK 会先从 `common + Windows unique` 中随机抽取 `90` 个字体，再补齐 Windows minimum 字体集，最后去重和排序。最终数量通常会大于 `90`，但具体数量取决于随机抽样结果与 minimum 字体集的重叠情况。

**建议：Chromium 环境如果业务方需要显式控制随机字体强度，`fontCount` 建议传 `80-100`。注意它不是最终数组长度；SDK 仍会补齐目标 OS 的 minimum 字体集，所以不同 OS 的最终字体数量会不同。不要为了把最终数量强行压到 80-100 而删除 minimum 字体集，否则可能破坏目标 OS 的字体一致性。**

如果业务方要求所有 OS 的最终字体数量都至少达到 `80`，建议显式传入 `getFonts(os, 80)` 或更高值。默认策略依赖各 OS minimum 字体集大小，移动端平台的 minimum 字体集较小，因此默认最终数量可能低于 `80`。

`getFonts` 只接受 SDK 支持的 `OsType` 值；如果传入未知 OS，会直接抛错，避免静默生成缺少 minimum 字体集的 common-only 列表。

## 多候选 OS 用法

如果业务先给出多个候选 OS，再决定 UA，可以使用 `createRandomFontValue`，此时 SDK 会使用候选 OS 的平台并集：

```javascript
const { createRandomFontValue } = require('general-sdk');

const fonts = createRandomFontValue({
  os: {
    windows: ['Windows 10', 'Windows 11'],
    linux: ['Ubuntu'],
  },
  sampleSize: 80,
});
```

当 `os` 存在非空数组时，`os` 优先级高于 `uaOs`。如果没有传入任何目标 OS，SDK 只能从 common 池抽样，并会打印 warning，因为它无法补齐具体系统的 minimum 字体集。

## 生成规则

1. 解析目标平台：优先使用 `os` 的非空候选平台；否则使用 `uaOs`。
2. 构造随机抽样池：`common + 目标平台 unique`。
3. 如果未传 `sampleSize`，从非 minimum 抽样池中随机抽取 `40-60` 个额外字体。
4. 如果传入 `sampleSize`，从完整抽样池中随机抽取 `sampleSize` 个字体。
5. 补齐目标平台的 minimum 字体集。
6. 对最终结果去重、过滤空值，并按客户端规则排序。

## 启动参数传递

业务传入 `fingerprint.font.type = 'custom'` 且 `value` 非空时，SDK 启动内核前会：

1. 将最终字体列表写入实例用户目录下的字体文本文件。
2. 文件内容使用英文逗号分隔字体名。
3. 将字体文件绝对路径做 base64。
4. 在加密 launch key 中写入 `font.list = <base64 file path>`。
5. 同时写入 `font.value = <字体列表 hash>`，用于表达字体列表变化。

`font.value` 不是字体列表本身，也不能替代 `font.list`。内核需要通过 `font.list` 找到字体文件并读取完整列表。

`truth` 模式不会写字体文件，也不会传 `font.list`。当前 SDK 为兼容旧参数结构，`truth` 下会把 `font.value` 置为空字符串。

## 边界

- `truth` 模式：SDK 不生成字体列表，不读取字体数据文件，表示使用运行环境真实字体。
- `custom` 模式：SDK 尊重业务方传入的字体列表，不自动补齐 minimum set。
- 历史环境、导入环境、已有 custom 字体列表不会被 SDK 自动修复。
- 随机字体能力只由 `getFonts` / `createRandomFontValue` 显式触发。
