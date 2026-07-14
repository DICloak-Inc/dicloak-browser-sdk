# WebGL 匹配信息工具函数

SDK 提供 WebGL 匹配信息工具函数，用于按目标 OS 获取一组相互匹配的 WebGL manufacturer、WebGL renderer 和 WebGPU adapter metadata。SDK 不会在启动链路中擅自替业务方决定 WebGL；业务方需要显式调用工具函数，并把返回结果通过 `webgl: { type: 'custom', ... }` 传入。

## 函数定位

- `getWebGLInfo(os, manufacturer?, options?)`：业务侧主入口。按 OS 和可选 manufacturer 随机返回一条匹配记录。
- `toWebGLFingerprint(info)`：把 `getWebGLInfo` 的返回结果转换成可直接传入 `fingerprint.webgl` 的 `custom` 配置。
- `getWebGLManufacturers(os?)`：查看指定 OS 支持的 manufacturer 值。
- `getSelectableWebGLInfoList(os, manufacturer?)`：查看指定 OS / manufacturer 下的候选记录，主要用于调试和排查数据。

## 数据来源

WebGL 数据随 SDK 内置在 `assets/webgl-info/`。当前主数据源是 `webgl_info_version.json`，每一条记录都是一个完整组合：

```json
{
  "webgl_manufacturer": "Google Inc. (AMD)",
  "webgl_render": "ANGLE (AMD, AMD Radeon Series (0x00001636) Direct3D11 vs_5_0 ps_5_0, D3D11-25.20.15003.5010)",
  "os": "Windows",
  "adapterinfo_vendor": "amd",
  "adapterinfo_architecture": "gcn-5"
}
```

SDK 会把整条记录作为原子单元返回，不会把一个 manufacturer 和另一条记录的 renderer / adapter metadata 拼接到一起。这样可以避免业务手动组合出不匹配的 WebGL 指纹。

SDK 不会扫描运行机器硬件，也不会联网获取 WebGL 数据。

当前 SDK 工具函数只使用 `webgl_info_version.json` 作为主数据源。它包含 `webgl_manufacturer`、`webgl_render`、`os`、`adapterinfo_vendor`、`adapterinfo_architecture`，能满足 `webgl.vendor`、`webgl.render`、`webgpu.arch`、`webgpu.vendor` 这一组参数的匹配需求。

业务正常调用 `getWebGLInfo` 时不需要关心 JSON 文件名，也不需要自己选择数据源。

## OS 维度用法

```javascript
const { createSDK, getWebGLInfo, toWebGLFingerprint, OsType } = require('general-sdk');

const sdk = createSDK();

const webglInfo = getWebGLInfo(OsType.Windows);

const { instanceId, fingerprintConfig } = await sdk.createFingerprint({
  uaOs: OsType.Windows,
  userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...',
  fingerprint: {
    webgl: toWebGLFingerprint(webglInfo),
  },
});
```

`getWebGLInfo(OsType.Windows)` 会只从 Windows 数据中随机选择一条记录。返回结构示例：

```javascript
{
  os: 'Windows',
  webglManufacturer: 'Google Inc. (Intel)',
  webglRender: 'ANGLE (Intel, Intel(R) HD Graphics 4600 (0x00000416) Direct3D11 vs_5_0 ps_5_0, D3D11-10.18.14.5067)',
  adapterInfoVendor: 'intel',
  adapterInfoArchitecture: 'gen-8',
}
```

## OS + manufacturer 用法

正常业务只需要传 OS。第二个参数 `manufacturer` 是可选收窄条件，不是必填项；如果业务不知道该传什么，就不要传。

可以考虑传 `manufacturer` 的场景：

- 业务画像已经明确了显卡厂商，比如 Windows + AMD、Windows + NVIDIA。
- 业务希望自己控制显卡厂商分布，比如批量环境里按比例分配 Intel / NVIDIA / AMD。
- 上游系统已有硬件画像字段，需要 WebGL 与这些字段保持一致。
- 需要排查或复现某个厂商组合的问题，比如固定 AMD 池做测试。

如果业务已经确定显卡厂商，可以把 `getWebGLManufacturers(os)` 返回的 manufacturer 原样传入：

```javascript
const { getWebGLInfo, toWebGLFingerprint, OsType } = require('general-sdk');

const webglInfo = getWebGLInfo(OsType.Windows, 'Google Inc. (AMD)');

const webgl = toWebGLFingerprint(webglInfo);
```

这会只从 `Windows + Google Inc. (AMD)` 的候选记录里随机选择一条，不会返回 Intel / NVIDIA 的 renderer。

如果需要先查看某个 OS 支持哪些 manufacturer：

```javascript
const manufacturers = getWebGLManufacturers(OsType.Windows);
// [ 'Google Inc. (AMD)', 'Google Inc. (Intel)', 'Google Inc. (NVIDIA)' ]
```

`manufacturer` 使用精确匹配；传入当前 OS 下不存在的值会抛出 `RangeError`，避免静默降级到其他显卡组合。

## 手动映射

推荐使用 `toWebGLFingerprint(info)`，也可以手动映射：

```javascript
const webglInfo = getWebGLInfo(OsType.Windows, 'Google Inc. (AMD)');

const webgl = {
  type: 'custom',
  vendor: webglInfo.webglManufacturer,
  renderer: webglInfo.webglRender,
  adapterInfoVendor: webglInfo.adapterInfoVendor,
  adapterInfoArchitecture: webglInfo.adapterInfoArchitecture,
};
```

字段对应关系：

- `webglInfo.webglManufacturer` -> `fingerprint.webgl.vendor` -> 内核参数 `webgl.vendor`，custom 模式必填
- `webglInfo.webglRender` -> `fingerprint.webgl.renderer` -> 内核参数 `webgl.render`，custom 模式必填
- `webglInfo.adapterInfoArchitecture` -> `fingerprint.webgl.adapterInfoArchitecture` -> 内核参数 `webgpu.arch`
- `webglInfo.adapterInfoVendor` -> `fingerprint.webgl.adapterInfoVendor` -> 内核参数 `webgpu.vendor`

当前导出数据中，Windows / Mac 记录包含 adapter metadata；Linux / Android / iOS 记录没有 adapter metadata。`toWebGLFingerprint` 会在这些字段不存在时自动省略，SDK 不会向内核下发空的 `webgpu.arch` / `webgpu.vendor`。

## truth 与 custom

- `custom`：业务方显式传入 `vendor` / `renderer` / 可选 adapter metadata，SDK 负责透传给内核。
- `truth`：表示使用运行环境真实 WebGL 行为。SDK 不会下发 `webgl.vendor`、`webgl.render`、`webgpu.arch`、`webgpu.vendor`。即使业务对象里带了这些字段，`truth` 模式下也不会交给内核。

`custom` 模式下 `vendor` 和 `renderer` 必须同时存在。推荐使用 `toWebGLFingerprint(info)` 生成这两个字段，避免业务手动漏传或拼错。

示例：

```javascript
fingerprint: {
  webgl: {
    type: 'truth',
    vendor: 'Google Inc. (Intel)', // 会被忽略，不会下发给内核
  },
}
```

`webGPU.type` 仍然只控制 `webgpu.value`，不等价于 WebGPU adapter metadata。adapter metadata 目前跟随 `fingerprint.webgl` 的 `custom` 模式透传。

## 边界

- `getWebGLInfo` 只接受 SDK 支持的 `OsType` 值；传入未知 OS 会直接抛错。
- `manufacturer` 必须使用 `getWebGLManufacturers(os)` 返回的原始值，SDK 不做大小写、厂商别名或模糊匹配。
- `webgl.type = 'custom'` 时，`vendor` 和 `renderer` 是必填项；缺少任一字段会在参数校验阶段报错。
- 工具函数只在业务显式调用时读取 WebGL 数据；SDK 启动链路不会自动替业务选择 WebGL。
- 返回记录来自内置数据文件，后续如果业务侧数据源更新，需要同步更新 `assets/webgl-info/` 后重新构建 SDK。
