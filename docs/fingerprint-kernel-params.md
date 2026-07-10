# SDK 指纹参数与内核对齐说明

> 本文档是 **SDK 内部对齐文档**，用于判断当前 SDK 与客户端 / 内核最新指纹参数能力是否一致。
> 
> 重点不是“业务方新增了哪些指纹”，而是：
> - SDK 当前负责生成哪些启动参数
> - 哪些字段只是业务侧传入，SDK 负责透传或消费
> - 客户端 / 内核已有但 SDK 还未补齐哪些兼容能力

---

## 1. 文档适用范围

当前只聚焦这 6 类能力：

- Canvas
- WebRTC
- WebGL
- UA
- 窗口大小
- 分辨率

本文档站在 SDK 视角，不讨论客户端私有页面展示、桌面端持久化状态、客户端额外预处理等非 SDK 职责。

---

## 2. SDK 当前的两条传参通道

内核当前通过两条通道接收参数：

| 通道 | 形式 | SDK 当前状态 |
|------|------|--------------|
| `launchParam` | `--launch-key=<encrypted_json>` | 已实现 |
| 命令行参数 | 直接拼接 `--user-agent=...`、`--window-size=...` 等 | 已实现 |

补充说明：

- `launchParam` 在 SDK 内部表现为 `Record<string, string>`。
- 所有 launchParam 的 value 最终都会被序列化成字符串。
- `expirets.value` 由 SDK 在临近启动内核时重新计算，避免参数过期。

相关实现：

- [src/fingerprint/fingerprint-builder.ts](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts)
- [src/core/chromium-launcher.ts](/Users/lllyee/codes/dic-browser-sdk/src/core/chromium-launcher.ts)

---

## 3. 先区分三类责任

在做客户端对齐前，先区分责任边界。

### 3.1 SDK 负责生成

这类参数由 SDK 根据业务入参和默认规则生成：

- `expirets.value`
- `acceptlang`
- `timezone.value`
- `geo.type` / `geo.value`
- `canvas.type` / `canvas.value`
- `rtc.type` / `rtc.value`
- `webgl2.value`
- `width.value` / `height.value`

### 3.2 业务侧负责传入，SDK 只透传或消费

这类不是 SDK 自己“生产”的：

- `userAgent`
- `fingerprint.platformVersion`
- `proxy.ipInfo`
- `fingerprint.windowConfig`
- `fingerprint.ratio.value`
- `fingerprint.webgl.vendor` / `fingerprint.webgl.renderer`

补充说明：`userAgent` 仍有历史默认随机 UA 兜底，但这个兜底不会根据 `uaOs` / fonts / WebGL / `platformVersion` 自动生成匹配组合。生产接入仍应由业务方显式传完整 UA。

### 3.3 客户端 / 内核可能已支持，但 SDK 未必已支持

这类是本次升级最需要重点核对的内容：

- Canvas Rev2
- WebGL Image Rev2
- WebRTC 新枚举或额外联动逻辑
- WebGPU metadata 相关 launchParam
- 与内核版本相关的兼容分支

---

## 4. 六类能力对齐结论

### 4.1 Canvas

#### SDK 当前已支持

- 支持 `canvas.type`
- 支持 `canvas.value`
- 支持内部自动生成 `canvas_rev2.value`
- 默认会生成 `canvas.value`
- 业务侧当前可传的枚举以 SDK 类型定义为准：`noise`、`truth`、`custom`

相关实现：

- [src/types/fingerprint.ts#L170](/Users/lllyee/codes/dic-browser-sdk/src/types/fingerprint.ts#L170)
- [src/fingerprint/fingerprint-builder.ts#L127](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts#L127)

#### SDK 当前未对齐

- 不支持业务侧单独传入 `canvasRev2Value`

#### 小结

Canvas Rev2 已经可以在 SDK 内部自动对齐：SDK 会复用现有 `canvas.value` 生成 `canvas_rev2.value`。这次没有新增业务侧字段，但如果后续需要让业务侧单独控制 Rev2 值，仍然需要再补可选字段。

---

### 4.2 WebRTC

#### SDK 当前已支持

- 支持 `rtc.type`
- 支持 `rtc.value`
- 支持 `rtc.stun`
- 当前类型仅支持：
  - `disable`
  - `replace`
  - `forward`
  - `truth`

相关实现：

- [src/types/fingerprint.ts#L31](/Users/lllyee/codes/dic-browser-sdk/src/types/fingerprint.ts#L31)
- [src/fingerprint/fingerprint-builder.ts#L127](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts#L127)
- [src/fingerprint/fingerprint-builder.ts#L228](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts#L228)

#### SDK 当前未对齐

客户端文档中提到的以下能力，SDK 目前没有类型和实现：

- `noise`
- `custom`
- `webrtcSyncProxyIpFlag`
- `webrtcKeepRandomInternalIp`
- `webrtcRandomInternalIpSaved`

#### 小结

WebRTC 是当前差异较大的部分。当前 SDK 已支持 `forward` 的最小语义：

- `replace` 模式下，支持 `useRandomInternalIp = true` 时自动生成随机内网 IP
- `replace` 模式下，`rtc.value` 优先使用业务传入的 `rtc.value`，未传时回退使用 `proxy.ipInfo.ip`
- `rtc.type = forward`
- `rtc.stun = stun:stun.l.google.com:19302`
- `rtc.value` 优先使用业务传入的 `rtc.value`，未传时回退使用 `proxy.ipInfo.ip`

剩余差异主要集中在客户端特有的代理联动和随机内网 IP 逻辑。这里要先区分：

- 哪些只是客户端本地预处理逻辑
- 哪些真的是 SDK 后续也要承接的对内核兼容能力

如果确定要补齐，后续可能不仅是内部实现升级，还会涉及对业务方新增枚举或新增字段。

---

### 4.3 WebGL

WebGL 这里要拆成两个层面看。

#### A. WebGL Metadata

##### SDK 当前已支持

- 支持自定义 `webgl.vendor`
- 支持自定义 `webgl.render`
- 支持透传 `webgpu.arch`
- 支持透传 `webgpu.vendor`
- 支持 `getWebGLInfo` / `toWebGLFingerprint` 工具函数，按 OS 和可选 manufacturer 返回匹配 WebGL 记录
- `webgl.type = truth` 时不下发 `webgl.vendor` / `webgl.render` / `webgpu.arch` / `webgpu.vendor`

相关实现：

- [src/types/fingerprint.ts#L184](/Users/lllyee/codes/dic-browser-sdk/src/types/fingerprint.ts#L184)
- [src/fingerprint/fingerprint-builder.ts#L334](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts#L334)
- [src/fingerprint/webgl-utils.ts](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/webgl-utils.ts)

##### SDK 当前未对齐

- 暂未发现这一项还有明确缺口

#### B. WebGL Image

##### SDK 当前已支持

- 支持 `webgl2.value`
- 支持内部自动生成 `webgl2_rev2.value`
- 当前代码中通过 `fingerprint.webGPUImage` 控制该值

相关实现：

- [src/types/fingerprint.ts#L191](/Users/lllyee/codes/dic-browser-sdk/src/types/fingerprint.ts#L191)
- [src/fingerprint/fingerprint-builder.ts#L256](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts#L256)

##### SDK 当前未对齐

- 不支持业务侧单独传入 `webGLImageRev2Value`

#### 小结

WebGL Image Rev2 已经可以在 SDK 内部自动对齐：SDK 会复用现有 `webGPUImage.value` 生成 `webgl2_rev2.value`，并根据当前内核版本决定是否将 `webgl2.value` 置为 `0`。这次没有新增业务侧字段，但如果后续需要让业务侧单独控制 Rev2 值，仍然需要再补可选字段。

---

### 4.4 UA

#### SDK 当前已支持

- 支持命令行参数 `--user-agent=<ua>`
- `userAgent` 由业务侧传完整字符串，SDK 不改写

相关实现：

- [src/fingerprint/fingerprint-builder.ts#L496](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts#L496)

#### SDK 当前已支持的相关透传

- 支持 `fingerprint.platformVersion -> platform.version`

相关实现：

- [src/types/fingerprint.ts#L243](/Users/lllyee/codes/dic-browser-sdk/src/types/fingerprint.ts#L243)
- [src/fingerprint/fingerprint-builder.ts#L161](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts#L161)

#### 小结

UA 本身不是这次“客户端对齐”的主要缺口。SDK 在这部分的核心职责仍然是：

- 接收业务方传入的完整 UA
- 在需要时透传 `platform.version`

也就是说，这部分更偏“透传能力”和“文档说明”，而不是新增内部指纹算法。

如果业务未传 UA，SDK 仅使用历史默认随机 UA 兜底，不保证与其他指纹项一致。

---

### 4.5 窗口大小

#### SDK 当前已支持

- 支持通过命令行参数 `--window-size=<W>,<H>` 传入窗口大小
- 但前提是业务侧显式传了 `fingerprint.windowConfig`
- 支持 `ratio.type = custom` 时自动生成 `--window-size`
- 当 `windowConfig` 存在时，会覆盖 `ratio` 生成的 `--window-size`

相关实现：

- [src/types/fingerprint.ts#L208](/Users/lllyee/codes/dic-browser-sdk/src/types/fingerprint.ts#L208)
- [src/fingerprint/fingerprint-builder.ts#L515](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts#L515)

#### SDK 当前未对齐

- 暂未发现这一项还有明确缺口

#### 小结

窗口大小 / 分辨率联动规则已经对齐：`ratio.type = custom` 会自动生成 `--window-size`，而显式传入的 `windowConfig` 会作为更高优先级覆盖值。

---

### 4.6 分辨率

#### SDK 当前已支持

- 支持 launchParam：
  - `width.value`
  - `height.value`
- 支持 `ratio.type = custom`
- 支持 `ratio.type = random`
- 在 Android / iOS 下，即使 `ratio.type = truth`，也会下发分辨率值

相关实现：

- [src/types/fingerprint.ts#L15](/Users/lllyee/codes/dic-browser-sdk/src/types/fingerprint.ts#L15)
- [src/fingerprint/fingerprint-builder.ts#L239](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts#L239)

#### SDK 当前未对齐

- 分辨率相关能力暂未发现这一项还有明确缺口

#### 小结

分辨率相关的 launchParam 能力已经存在，并且 `ratio.type = custom` 与命令行窗口大小的联动也已对齐。

---

## 5. 当前对齐进展

如果目标是“升级 SDK，并支持多版本内核”，目前已经完成以下能力：

1. Canvas Rev2 兼容
- 已增加 `canvas_rev2.value`
- 已按内核版本控制旧 `canvas.value` 是否置零
- 未新增业务侧字段，复用现有 `canvas.value`

2. WebGL Image Rev2 兼容
- 已增加 `webgl2_rev2.value`
- 已复用 Canvas 相同的内核版本判断逻辑
- 未新增业务侧字段，复用现有 `webGPUImage.value`

3. WebRTC SDK 职责内收口
- 已支持 `forward`
- 已支持 `rtc.stun`
- 已支持 `replace` / `forward` 从 `proxy.ipInfo.ip` 回退取值
- 已支持 `replace + useRandomInternalIp` 的不保持模式

剩余明确未做项：

- WebRTC 客户端状态型能力，如随机内网 IP 保持模式，暂不建议纳入 SDK 职责

---

## 6. 哪些升级会影响业务侧入参

不是所有内部兼容升级都会影响业务方。可以先按下面这张表判断。

| 升级项 | 是否影响业务侧入参 | 说明 |
|------|------------------|------|
| `expirets.value` 计算时机优化 | 否 | SDK 内部生成逻辑，业务侧无感知 |
| Canvas Rev2 | 否，当前无新增字段 | SDK 内部自动复用现有 `canvas.value` |
| WebGL Image Rev2 | 否，当前无新增字段 | SDK 内部自动复用现有 `webGPUImage.value` |
| `platform.version` 透传 | 已影响，已完成 | 业务侧可选传 `fingerprint.platformVersion` |
| 窗口大小 / 分辨率联动 | 否，当前无新增字段 | 已按内部联动方案实现 |
| WebRTC 新枚举 | 已影响，已落地 | `forward` 已新增，需要同步 README |
| WebRTC 随机内网 IP | 已影响，已落地 | `useRandomInternalIp` 已新增，README 已更新 |
| WebGPU metadata 字段 | 已影响，已完成 | 业务侧可选传 `webgl.adapterInfoArchitecture` / `webgl.adapterInfoVendor`，也可通过 `getWebGLInfo` 获取匹配组合 |
| 字体列表文件参数 | 否，内部 launch key 参数 | `font.type = custom` 且字体列表非空时写入 `font.list`，`font.value` 仅作为列表 hash |
| 禁止查看网站密码 | 已影响，已完成 | 业务侧可选传 `fingerprint.disablePasswordView` |
| 浏览器扩展安全能力 | 已影响，已完成 | 业务侧可选传 `advancedConfig.security.disableExtensionManagement` / `advancedConfig.security.blockExtensionStoreAndSettings` |

判断原则：

- 只改 SDK 内部生成逻辑，不增加业务入参时：
  - 不需要改对外参数表
  - 但建议在升级说明中提一句“内部兼容增强”
- 新增业务可传字段、枚举或行为开关时：
  - 必须同步 README
  - 必须补充示例
  - 如涉及 UA / Client Hint 联动，也要同步更新 [docs/UA_AND_CLIENT_HINT_GUIDE.md](/Users/lllyee/codes/dic-browser-sdk/docs/UA_AND_CLIENT_HINT_GUIDE.md)

---

## 7. 对业务方可能产生的影响

大多数升级对业务方未必是“新指纹”，更可能只是以下两种体现：

- SDK 内部自动兼容了新内核，业务侧不用改
- 某个已有指纹项新增了可选枚举或可选字段

因此后续对外文档建议遵循这个原则：

- 不讲“新增了哪些指纹”
- 只讲“业务方需要传什么”
- 如果新增了字段或枚举，再单独列出

---

## 8. 对外文档更新建议

如果后续实现中出现以下变化，需要同步更新对外文档：

1. README 参数说明
- `FingerprintConfig` 类型片段
- 常用示例代码
- 新增字段 / 新增枚举的使用说明

2. UA 与 Client Hint 文档
- 仅在改动涉及 `userAgent`、`platformVersion`、平台识别、系统版本传参时更新
- 对业务方只写“该怎么传”，不要展开内核实现细节

3. 是否需要单独新增一份业务文档
- 如果 WebRTC、Canvas Rev2、WebGL Rev2 最终真的暴露给业务侧
- 且 README 中一两段文字说不清楚
- 再考虑新增独立说明文档

### 8.1 禁止查看网站密码

`fingerprint.disablePasswordView` 是业务侧显式开关，开启后保存密码提示和密码查看 / 密码管理入口都会被关闭。SDK 会把它转换成加密 `launchKey` 中的两个内核参数：

| 业务入参 | `password.hint` | `pwmanager.enable` | 说明 |
|---------|-----------------|--------------------|------|
| `disablePasswordView: true` | `"0"` | `"0"` | 禁止查看网站密码，并关闭密码管理器能力 |
| `disablePasswordView: false` 或未传，且未关闭 `passwordHint` | `"1"` | `"1"` | 不禁止查看网站密码，保持默认密码管理器能力和保存密码提示 |
| `passwordHint: false` 且未开启 `disablePasswordView` | `"0"` | `"1"` | 只关闭保存密码提示，不关闭密码查看 / 密码管理入口 |

兼容说明：

- 旧字段 `fingerprint.passwordHint` 仍保留，只控制保存密码提示。
- 如果传入 `disablePasswordView: true`，`password.hint` 和 `pwmanager.enable` 都会下发 `"0"`。
- 这两个内核参数必须写入 `launchKey` 加密 JSON，不通过明文命令行参数下发。

### 8.2 浏览器扩展安全能力

浏览器安全能力不属于指纹项，对外入口放在 `advancedConfig.security` 下。

| 业务入参 | `extension.disable` | `urls.black` | 说明 |
|---------|---------------------|--------------|------|
| `disableExtensionManagement: true` | `"1"` | 保持原值或空字符串 | 禁止管理/移除扩展，以及禁止从本地安装扩展到浏览器 |
| `blockExtensionStoreAndSettings: true` | `"1"` | `base64(黑名单文件路径)` | 禁止访问 Chrome Web Store 和扩展设置页面 |

实现说明：

- 两个能力最终都依赖同一个内核开关 `extension.disable = "1"`。
- `blockExtensionStoreAndSettings` 会在本次实例用户目录下生成黑名单文件，文件内容追加 `chromewebstore.google.com`。
- 如果业务已传 `advancedConfig.urls.black`，SDK 会解码出原黑名单文件路径，并向该文件追加 `chromewebstore.google.com`。
- `urls.black` 传给内核的是黑名单文件路径的 base64，不是域名明文。
- `extension.disable` 和 `urls.black` 都必须写入 `launchKey` 加密 JSON，不通过明文命令行参数下发。

兼容说明：

- `advancedConfig.extension.disable` 是历史底层开关，仍然兼容。
- `advancedConfig.extension.disable` 与 `advancedConfig.security.disableExtensionManagement` 都会映射为 `extension.disable = "1"`。
- 新接入业务建议使用 `advancedConfig.security.disableExtensionManagement`；旧字段只作为兼容入口保留。
- `advancedConfig.extension.disable` 只等价于禁止扩展管理/本地安装，不会自动追加扩展商店黑名单；限制 Chrome Web Store 和扩展设置页面需要使用 `advancedConfig.security.blockExtensionStoreAndSettings`。
- `advancedConfig.security.blockExtensionStoreAndSettings: true` 会强制下发 `extension.disable = "1"`；因此即使同时传 `disableExtensionManagement: false`，表现也会和两个字段都为 `true` 一样，扩展管理/本地安装也会被禁用。
- 同时传入旧字段和新字段时，SDK 按开启处理，不存在互相覆盖。

建议执行规则：

- 先完成 SDK 内部兼容
- 再判断是否新增了业务侧能力面
- 只要业务侧能力面发生变化，就在同一轮提交里更新 README 或对应对外文档

---

## 9. 升级执行总表

下面这张表用于后续排期时快速判断优先级和文档动作。

| 能力项 | SDK 当前状态 | 是否影响业务入参 | 需要更新的对外文档 | 当前建议优先级 |
|------|--------------|------------------|-------------------|----------------|
| `expirets.value` | 已支持 | 否 | 无 | 已完成 |
| `platform.version` 透传 | 已支持 | 已影响，已落地 | README、UA 文档已覆盖 | 已完成 |
| Canvas Rev2 | 已支持内部自动对齐 | 否，当前无新增字段 | 无 | 已完成 |
| WebGL Image Rev2 | 已支持内部自动对齐 | 否，当前无新增字段 | 无 | 已完成 |
| 内核版本分支兼容 | 已支持 | 通常否 | 无 | 已完成 |
| WebRTC 新枚举 | 已支持 `forward` | 已影响，README 已更新 | README 已覆盖 | 已完成 |
| WebRTC 随机内网 IP | 已支持不保持模式 | 已影响，README 已更新 | README 已覆盖 | 已完成 |
| WebRTC 状态保持能力 | 暂不纳入 SDK | 会增加状态负担 | 暂不更新 | 暂不做 |
| 窗口大小 / 分辨率联动 | 已支持 | 否，当前无新增字段 | 无 | 已完成 |
| WebGPU metadata | 已支持 | 已影响，README 已更新 | README 已覆盖 | 已完成 |
| UA 原样透传 | 已支持 | 已影响，已落地 | README、UA 文档已覆盖 | 已完成 |

建议使用方式：

1. 先做不会引起业务侧接口震荡的内部兼容
- 例如 Rev2 的底层生成逻辑
- 例如内核版本判断逻辑

2. 再做会新增业务侧能力面的项目
- 例如 WebRTC 新枚举
- 例如 Rev2 可选字段对外暴露

3. 每做完一项都回看对外文档
- 只要业务可传参数发生变化，就在同一轮补 README
- 如果变化涉及 UA / 平台识别，再补 `UA_AND_CLIENT_HINT_GUIDE.md`

---

## 10. 本文档对应的当前代码事实

本结论基于当前仓库代码整理，核心参考如下：

- [src/fingerprint/fingerprint-builder.ts](/Users/lllyee/codes/dic-browser-sdk/src/fingerprint/fingerprint-builder.ts)
- [src/types/fingerprint.ts](/Users/lllyee/codes/dic-browser-sdk/src/types/fingerprint.ts)
- [src/core/chromium-launcher.ts](/Users/lllyee/codes/dic-browser-sdk/src/core/chromium-launcher.ts)
- [README.md](/Users/lllyee/codes/dic-browser-sdk/README.md)

如果后续开始实现 Rev2 或多版本内核兼容，建议直接在本文档追加“已对齐版本记录”，避免再次把客户端文档原样映射成 SDK 现状。
