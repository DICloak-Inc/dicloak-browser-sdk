# DIC Browser SDK

基于 Chromium 的浏览器 SDK，提供指纹管理和进程控制功能，适用于浏览器自动化和反检测场景。

## 🚀 快速开始

### 安装

1.将 **dic-browser-sdk.zip** 解压在项目根目录下（或者你喜欢的其他任何地方），修改 package.json：

```bash
  "dependencies": {
    "dic-browser-sdk": "file:./dic-browser-sdk"
  }
```

2.安装 SDK 所需依赖（在你项目根目录中执行）

```bash
npm install
```

3.完成

### 基础使用

```javascript
const { createSDK } = require('dic-browser-sdk');
const path = require('path');

async function main() {
  // 1. 创建SDK实例
  const sdk = createSDK();

  // 2. 初始化SDK
  await sdk.initialize({
    key: 'Your usage sdk key', // sdk key 必须
    baseDir: path.join(__dirname, 'data'),
    logLevel: 'info',
  });

  // 3. 创建指纹配置
  const { instanceId, fingerprintConfig } = await sdk.createFingerprint({
    userAgent:
      'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36',
    proxy: {
      host: 'proxy.example.com',
      port: 8080,
      type: 'HTTP',
      username: 'user',
      password: 'pass',
    },
    fingerprint: {
      canvas: { type: 'noise' },
      rtc: { type: 'disable' },
    },
  });

  // 4. 启动浏览器实例
  const { id, wsUrl } = await sdk.launch({
    instanceId: instanceId,
    fingerprintConfig: fingerprintConfig.config,
    chromiumPath: '/path/to/chromium.exe', // 本次启动使用的浏览器内核路径，必填
  });

  console.log(`浏览器启动成功: ${id}`);
  console.log(`WebSocket URL: ${wsUrl}`);

  // 5. 关闭实例
  await sdk.close(id);

  // 6. 清理资源
  await sdk.cleanup();
}

main().catch(console.error);
```

## 📚 主要功能

### 1. SDK 初始化

```javascript
await sdk.initialize({
  key: 'Your usage sdk key', // 必填
  baseDir: './browser-data', // 数据目录
  sourceDataDir: './source-data', // 可选，仅用于新实例首次初始化
  chromiumPath: '/path/to/browser', // 可选，历史兼容字段；实际启动请在 launch 中传
  logLevel: 'info', // 日志级别
});
```

> `sourceDataDir` 可以不传。只有 `baseDir/instanceId` 不存在时，SDK 才会尝试复制非空的模板目录；实例目录已经存在时一律直接复用，不执行模板复制。模板未配置、不存在或为空时，由 Chromium 自行生成用户数据。
>
> 推荐在每次 `launch` 时显式传入 `chromiumPath`。初始化阶段的 `chromiumPath` 仅作为历史兼容配置保留，不作为多内核场景下的启动依据。

### 2. 指纹配置

#### 业务方最常用的 3 个参数

- `userAgent`
  建议由业务方传完整字符串，SDK 不会根据 `uaOs` / fonts / WebGL 自动生成匹配 UA，也不会自动修改、拼接或纠正。
- `fingerprint.platformVersion`
  只有目标站点会读取 Client Hint 时再传。SDK 只负责透传给内核参数 `platform.version`。
- `proxy.ipInfo`
  只有你希望语言、时区、地理位置等信息跟随代理环境时再传。

如果你只想尽快跑通，通常先关注 `userAgent` 即可；只有在目标站点会校验更细的环境信息时，再补 `platformVersion` 和 `proxy.ipInfo`。

如果业务不传 `userAgent`，SDK 会使用历史默认随机 UA 兜底。这个兜底 UA 不会跟 `uaOs`、字体、WebGL 或当前内核版本自动对齐，生产环境不建议依赖。

#### 最常见的两种传法

```javascript
// 场景1：只需要控制 UA
const fingerprint = await sdk.createFingerprint({
  userAgent:
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36',
});

// 场景2：目标站点会读取 Client Hint，需要同时传 platformVersion
const fingerprintWithClientHint = await sdk.createFingerprint({
  userAgent:
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36',
  fingerprint: {
    platformVersion: '15.0.0',
  },
});
```

更多背景和示例见 [docs/UA_AND_CLIENT_HINT_GUIDE.md](./docs/UA_AND_CLIENT_HINT_GUIDE.md)。

#### 完整参数参考

注意：指纹中提到的所有“基于 ip”指的是基于业务创建指纹时透传的 proxy.ipInfo 信息，并不是指本机 ip；目前当设置为基于 ip 的指纹项，会自动使用 proxy.ipInfo 信息作为指纹支撑（lang、acceptlang、timeZone、geo）。
关于 ipInfo 字段透传，[参考这里](http://ip-api.com/json) 检测结果中的字段即可（注意该站 https 会失败，改为 http 访问）。

```typescript
/**
 * 指纹相关类型定义
 */

// 指纹模式类型
// noise = 噪音（SDK内部处理）
// truth = 真实（SDK内部处理）
// custom = 自定义（部分指纹有特殊配置能力）
export type FingerprintMode = 'noise' | 'truth' | 'custom';

// ip指纹类型
// ip = 代表基于ip信息(proxy.ipInfo)
// custom = 代表自定义传入(此时传对应的value)
export type IpFingerprintType = 'ip' | 'custom';

// 界面语言类型
// for-acceptLang = 来自语言项（继承）
// truth = 真实
// custom = 代表自定义传入(此时传对应的value)
export type LangType = 'for-acceptLang' | 'truth' | 'custom';

// 分辨率类型
// truth = 真实
// random = 随机
// custom = 自定义（此时传value具体的分辨率，RatioValue）
export type RatioMode = 'custom' | 'truth' | 'random';

// 分辨率配置
export interface RatioValue {
  width: string;
  height: string;
}

// WebGpu类型
// inWebGL = 基于WebGL
// truth = 真实
// disable = 禁止WebGpu
export type WebGPUType = 'inWebGL' | 'truth' | 'disable';

// 地理位置类型
export type GeoType =
  | 'ask' // 询问
  | 'ask-ip' // 询问-基于ip
  | 'ask-custom' // 询问-自定义（此时建议传具体的地理位置）
  | 'allow' // 允许
  | 'allow-ip' // 允许-基于ip
  | 'allow-custom' // 允许-自定义（此时建议传具体的地理位置）
  | 'block'; // 禁止

// 字体类型
// truth = 真实
// custom = 自定义（此时需要传value为字体列表的数组，如["Abyssinica SIL","AnjaliOldLipi", "Apple Braille Pinpoint 6 Dot"]等）
export type FontType = 'truth' | 'custom';

// WebRTC类型
// disable = 禁止（不向站点暴露 WebRTC IP）
// replace = 替代（向站点暴露你指定的 IP，或由 SDK 回退取 proxy.ipInfo.ip）
// forward = 转发（走 forward 语义，并固定补 rtc.stun）
// truth = 真实（保留真实 WebRTC 行为）
export type RTCType = 'disable' | 'replace' | 'forward' | 'truth';

// 设备内存值
export type DeviceMemoryValue = '2' | '4' | '8';

// 硬件并发数值
export type HardwareConcurrencyValue =
  | '2'
  | '3'
  | '4'
  | '6'
  | '8'
  | '10'
  | '12'
  | '16'
  | '20'
  | '24';

// 额外的业务透传信息，基于用户提供的代理信息检测的结果。（非必填项可不传）
// 目前仅有必填项（acceptLang、lang、geo、timeZone）在满足一定条件下可为对应指纹项提供支撑的。
// 可参考：http://ip-api.com/json
export interface IpInfo {
  acceptLang: string[];
  lang: string;
  geo: {
    longitude: string;
    latitude: string;
    accuracy: string;
  };
  timeZone: string;
  city?: string;
  country?: string;
  countryCode?: string;
  ip?: string;
  region?: string;
  regionCode?: string;
}

// 用户的代理信息（直接交给内核代理转发）
export interface ProxyProps {
  host: string;
  port: number;
  type: 'HTTP' | 'HTTPS' | 'SOCKS5';
  username?: string;
  password?: string;
  ipInfo?: IpInfo;
}

export interface AdvancedConfig {
  /** 是否恢复上次会话 */
  restoreLast?: 'enable' | 'disable';
  /** Google 翻译默认目标语言，对应 profile Preferences.translate_recent_target */
  translateRecentTarget?: string;
  /** 浏览器安全能力配置 */
  security?: {
    /** 禁止管理/移除扩展，以及禁止从本地安装扩展到浏览器 */
    disableExtensionManagement?: boolean;
    /** 禁止访问谷歌扩展商店和扩展设置页面 */
    blockExtensionStoreAndSettings?: boolean;
  };
  /** URL过滤配置 */
  urls?: {
    black?: string; // base64 后的黑名单文件路径
    white?: string; // 白名单
    kyc?: string; // KYC名单
  };
  /** 扩展配置 */
  extension?: {
    disable?: boolean;
    hideIds?: string;
    paths?: string[]; // 加载本地扩展路径
  };
}

/**
 * 操作系统类型 (WINDOWS:Windows MAC:Mac ANDROID:Android IOS:iOS LINUX:Linux)
 */
export enum OsType {
  Windows = 'WINDOWS',
  Mac = 'MAC',
  Android = 'ANDROID',
  Ios = 'IOS',
  Linux = 'LINUX',
}

// 账号信息
export interface Account {
  username: string;
  password: string;
  platform: string;
}

export interface FingerprintParams {
  /** 操作系统类型（默认windows） */
  uaOs?: OsType;
  /** UA */
  userAgent?: string;
  /** 序号 */
  serialNumber?: number;
  /** 图标提示文本 */
  iconHintText?: string;
  /** 代理配置 */
  proxy?: ProxyProps;
  /** 账号配置 */
  accounts?: Account[];
  /** 高级配置 */
  advancedConfig?: AdvancedConfig;
  /** 详细指纹 */
  fingerprint?: FingerprintConfig;
}

// 指纹配置
export interface FingerprintConfig {
  /** 语言配置 */
  acceptLang?: {
    type: IpFingerprintType;
    value?: string[];
  };
  /** 界面语言配置 */
  lang?: {
    type: LangType;
    value?: string;
  };
  /** 时区配置 */
  timeZone?: {
    type: IpFingerprintType;
    value?: string;
  };
  /** 地理位置配置 */
  geo?: {
    type: GeoType;
    value?: string;
  };
  /** 音频配置 */
  audio?: {
    type: FingerprintMode;
    value?: string;
  };
  /** 电池配置 */
  battery?: {
    type: FingerprintMode;
    value?: string;
  };
  /** 客户端矩形配置 */
  clientRects?: {
    type: FingerprintMode;
    value?: string;
  };
  /** 设备内存配置 */
  deviceMemory?: {
    type: FingerprintMode;
    value?: DeviceMemoryValue;
  };
  /** 字体配置 */
  font?: {
    type: FontType;
    value?: string[]; // 当type为custom时的字体标识
  };
  /** WebRTC配置 */
  rtc?: {
    type: RTCType;
    value?: string; // replace / forward 模式下优先使用该值
    useRandomInternalIp?: boolean; // 仅 replace 模式生效，开启后自动生成随机内网 IP
  };
  /** Canvas指纹配置 */
  canvas?: {
    type: FingerprintMode;
    value?: string;
  };
  /** 剪贴板配置 */
  clipboard?: {
    type: 'enable' | 'disable';
  };
  /** 密码提示 */
  passwordHint?: boolean;
  /** 禁止查看网站密码，开启后会关闭保存密码提示和密码管理器 */
  disablePasswordView?: boolean;
  /** WebGL配置 */
  webgl?: {
    type: 'custom' | 'truth';
    vendor?: string; // custom 模式必填，对应内核参数 webgl.vendor
    renderer?: string; // custom 模式必填，对应内核参数 webgl.render
    adapterInfoArchitecture?: string; // WebGPU adapter architecture，对应内核参数 webgpu.arch
    adapterInfoVendor?: string; // WebGPU adapter vendor，对应内核参数 webgpu.vendor
  };
  /** WebGPU配置 */
  webGPU?: {
    type: WebGPUType;
    value?: string;
  };
  webGPUImage?: {
    type: FingerprintMode;
    value?: string;
  };
  /** 硬件并发数 */
  hardwareConcurrency?: {
    type: FingerprintMode;
    value?: HardwareConcurrencyValue;
  };
  /** 媒体设备配置 */
  mediadevice?: {
    type: FingerprintMode;
    value?: string;
  };
  /** 是否开启硬件加速 */
  accelerate?: boolean;
  /** 分辨率 */
  ratio?: {
    type: RatioMode;
    value?: RatioValue;
  };
  /** 窗口配置 */
  windowConfig?: {
    width: number;
    height: number;
  };
  /** 语音指纹 */
  speechvoices?: {
    type: FingerprintMode;
    value?: string;
  };
  /** 追踪 */
  track?: string;
  /** 启动参数 */
  startParams?: string;
  /** 平台版本(Client Hint 高熵字段，对应内核参数 platform.version，由业务方传入) */
  platformVersion?: string;
  /** 端口扫描保护 */
  port?: string;
  /** 启动页面 */
  startUrl?: string[];
}
```

#### 基础指纹配置

```javascript
const fingerprint = await sdk.createFingerprint({
  userAgent: 'Mozilla/5.0 ...',
  iconHintText: '1',
  proxy: {
    host: 'proxy.com',
    port: 8080,
    type: 'HTTP',
    username: 'user',
    password: 'pass',
    ipInfo: {
      acceptLang: ['zh-CN', 'zh'],
      lang: 'zh-CN',
      geo: {
        longitude: '121.4737',
        latitude: '31.2304',
        accuracy: '10',
      },
      timeZone: 'Asia/Shanghai',
      ip: '255.23.41.3',
    },
  },
});
```

#### 高级指纹配置

```javascript
const fingerprint = await sdk.createFingerprint({
  userAgent: 'Mozilla/5.0 ...',
  // 详细指纹
  fingerprint: {
    canvas: { type: 'noise' }, // Canvas指纹
    rtc: { type: 'disable' }, // WebRTC
    audio: { type: 'noise' }, // 音频指纹
    font: { type: 'truth' }, // 字体
    deviceMemory: {
      type: 'custom',
      value: '8',
    },
    hardwareConcurrency: {
      type: 'custom',
      value: '8',
    },
    ratio: {
      type: 'custom',
      value: { width: '1920', height: '1080' },
    },
    platformVersion: '15.0.0',
  },
  advancedConfig: {
    restoreLast: 'enable', // 恢复上次会话
    security: {
      disableExtensionManagement: true, // 禁止管理/移除扩展，以及禁止从本地安装扩展
      blockExtensionStoreAndSettings: true, // 禁止访问谷歌扩展商店和扩展设置页面
    },
  },
  accounts: [
    {
      // 账号信息
      platform: 'example.com',
      username: 'user123',
      password: 'pass123',
    },
  ],
});
```

#### 随机字体工具函数

SDK 提供 `getFonts` / `createRandomFontValue` 工具函数，用于按目标 OS 生成符合客户端规则的随机字体列表。SDK 不会在启动链路中擅自决定业务字体；业务方需要先调用工具函数拿到列表，再通过 `fingerprint.font` 的 `custom` 模式显式传入。

规则：

- 随机抽样池为 `common + 目标平台 unique`
- 不传 `fontCount` 时，默认返回 `目标平台 minimum 字体集 + 随机 40-60 个额外字体`
- 传入 `fontCount` 时，`fontCount` 只表示随机抽样数量，不是最终字体数量
- 抽样后会强制补齐目标平台的 minimum 字体集
- 最终结果会去重、过滤空值，并按客户端规则排序
- `getFonts` 只接受 SDK 支持的 `OsType` 值，非法 OS 会抛错
- `truth` 模式不生成字体列表，不读取内置字体数据，表示使用运行环境真实字体
- 已有 `custom` 字体列表会被原样尊重，SDK 不会自动补齐 minimum set

启动内核时，`custom` 字体列表不会直接塞进 `font.value`。SDK 会把字体列表写入实例目录下的文本文件，将文件绝对路径 base64 后写入 `font.list`；`font.value` 只是字体列表 hash，用于表达列表变化，不能替代 `font.list`。

**建议：Chromium 环境如果业务方需要显式控制随机字体强度，`fontCount` 建议传 `80-100`。注意它不是最终数组长度；SDK 仍会补齐目标 OS 的 minimum 字体集，所以不同 OS 的最终字体数量会不同。**

如果业务方要求所有 OS 的最终字体数量都至少达到 `80`，建议显式传入 `getFonts(os, 80)` 或更高值；默认策略依赖各 OS minimum 字体集大小，移动端平台的默认最终数量可能低于 `80`。

单平台示例：

```javascript
const { createSDK, getFonts, OsType } = require('dic-browser-sdk');

const sdk = createSDK();

// 默认：Windows minimum 字体集 + 随机 40-60 个额外字体
const fonts = getFonts(OsType.Windows);

// 显式控制随机抽样数量：随机抽 90 个，再补齐 Windows minimum 字体集
const strongerFonts = getFonts(OsType.Windows, 90);

const { instanceId, fingerprintConfig } = await sdk.createFingerprint({
  uaOs: OsType.Windows,
  userAgent: 'Mozilla/5.0 (Windows NT 10.0; Win64; x64) ...',
  fingerprint: {
    font: {
      type: 'custom',
      value: strongerFonts,
    },
  },
});
```

多候选 OS 场景可以使用 `createRandomFontValue`：

```javascript
const { createRandomFontValue } = require('dic-browser-sdk');

const fonts = createRandomFontValue({
  os: {
    windows: ['Windows 10', 'Windows 11'],
    linux: ['Ubuntu'],
  },
  sampleSize: 80,
});
```

更多说明见 [docs/font-randomization.md](./docs/font-randomization.md)。

#### UA 与 platformVersion 说明

- `userAgent` 建议由业务方传完整字符串，SDK 不会根据 `uaOs` / fonts / WebGL 自动生成匹配 UA，也不会自动修改、拼接或纠正。
- 如果业务不传 `userAgent`，SDK 会使用历史默认随机 UA 兜底；该兜底 UA 不保证与目标 OS、字体、WebGL、`platformVersion` 或内核版本一致，生产环境不建议依赖。
- `fingerprint.platformVersion` 由业务方按目标系统版本传入，SDK 只负责透传给内核的 `platform.version` 参数。
- 建议 `chromiumPath` 对应的内核版本与 `userAgent` 中的 Chrome 主版本尽量保持一致。
- 建议 `userAgent`、目标系统、`platformVersion` 三者保持一致，否则业务站点可能识别出环境不一致。
- iOS 场景通常重点是传正确的 UA；`platformVersion` 主要用于配合 Client Hint 场景。

#### Google 翻译默认目标语言

业务方可以通过 `advancedConfig.translateRecentTarget` 指定 Chromium 中 Google 翻译功能默认选中的目标语言：

```javascript
const fingerprint = await sdk.createFingerprint({
  advancedConfig: {
    translateRecentTarget: 'ja',
  },
});
```

- SDK 在启动内核前把该值写入实例的 `Default/Preferences` 顶层字段 `translate_recent_target`。
- 新 profile 没有 Preferences 时，这项翻译配置只预置类似 `{"translate_recent_target":"ja"}` 的 32 字节紧凑 JSON，而不是复制 `sourceDataDir` 资料模板；其他既有启动步骤仍可能准备 Cookie DB 等自身所需文件，其余 profile 数据由 Chromium 首次启动生成。
- 每次启动传入该字段都会覆盖 profile 中的已有值；未传时不会调用翻译目标 updater，不会因为这项能力额外读取、创建或修改 Preferences。扩展修补等既有能力仍可能按原逻辑访问 Preferences。
- 空字符串和纯空白字符串会在配置验证阶段被拒绝；写入值会去除首尾空白。
- SDK 不内置容易过期的 Google 翻译语言白名单。业务方应传入目标内核支持的语言代码；不支持的值可能保留在 Preferences 中，但 Chromium 会忽略并使用自己的目标语言回退规则。
- 该字段控制启动默认值，不是锁定策略。用户在运行期间手动修改或 Chrome Sync 更新都可能改变它；业务下次仍传参启动时，SDK 会再次写入业务值。
- 同一 Node.js 进程内对同一 profile 的翻译偏好更新会串行执行；不支持多个进程或正在运行的 Chromium 同时写同一个 profile。
- 它不修改 `fingerprint.acceptLang`（请求语言偏好）或 `fingerprint.lang`（浏览器界面语言），三者互不推导。

#### 禁止查看网站密码

业务方可以通过 `fingerprint.disablePasswordView` 控制是否禁止用户在网站密码相关界面查看密码。

- 默认不禁止，等同于 `disablePasswordView: false`
- 开启 `disablePasswordView: true` 后，保存密码提示和密码查看 / 密码管理入口都会被关闭，SDK 会把以下参数写入加密后的 `launchKey`：
  - `password.hint = "0"`
  - `pwmanager.enable = "0"`
- 关闭或默认状态下，密码查看 / 密码管理入口保持开启，SDK 会把 `pwmanager.enable = "1"` 写入加密后的 `launchKey`
- 默认保存密码提示开启，SDK 会把 `password.hint = "1"` 写入加密后的 `launchKey`
- 旧字段 `fingerprint.passwordHint` 仍然兼容，只控制保存密码提示：
  - `passwordHint: false` 会下发 `password.hint = "0"`，但不会关闭 `pwmanager.enable`
  - 如果同时传 `disablePasswordView: true`，保存密码提示和密码查看 / 密码管理入口都会关闭

示例：

```javascript
const fingerprint = await sdk.createFingerprint({
  userAgent: 'Mozilla/5.0 ...',
  fingerprint: {
    disablePasswordView: true,
  },
});
```

#### 浏览器安全能力

浏览器安全能力放在 `advancedConfig.security` 下，不属于指纹项。当前支持两个开关：

- `disableExtensionManagement`
  - 开启后，用户不能在浏览器里管理/移除扩展，也不能从本地安装扩展
  - SDK 会把 `extension.disable = "1"` 写入加密后的 `launchKey`
- `blockExtensionStoreAndSettings`
  - 开启后，限制访问 Chrome Web Store 和扩展设置页面
  - SDK 会把 `extension.disable = "1"` 写入加密后的 `launchKey`
  - SDK 会在本次实例用户目录下生成黑名单文件，内容包含 `chromewebstore.google.com`
  - SDK 写入 `launchKey` 的 `urls.black` 是黑名单文件路径的 base64，不是域名明文
  - 如果业务已经传了 `advancedConfig.urls.black`，SDK 会解码该文件路径并向原黑名单文件追加 `chromewebstore.google.com`

这两个能力最终都会通过 `--launch-key=<加密后的launchParam>` 传给内核，不会通过明文命令行参数下发。

兼容说明：

- `advancedConfig.extension.disable` 是历史底层开关，仍然兼容。
- `advancedConfig.extension.disable` 和 `advancedConfig.security.disableExtensionManagement` 最终都会映射为 `extension.disable = "1"`。
- 新接入业务建议使用 `advancedConfig.security.disableExtensionManagement`，语义更接近“禁止管理/移除扩展，以及禁止从本地安装扩展”。
- `advancedConfig.extension.disable` 只等价于禁止扩展管理/本地安装，不会自动追加扩展商店黑名单；如需限制 Chrome Web Store 和扩展设置页面，请使用 `advancedConfig.security.blockExtensionStoreAndSettings`。
- `advancedConfig.security.blockExtensionStoreAndSettings: true` 会强制下发 `extension.disable = "1"`；因此即使同时传 `disableExtensionManagement: false`，表现也会和两个字段都为 `true` 一样，扩展管理/本地安装也会被禁用。
- 如果同时传入旧字段和新字段，SDK 会按开启处理，不会产生互相覆盖。

示例：

```javascript
const fingerprint = await sdk.createFingerprint({
  userAgent: 'Mozilla/5.0 ...',
  advancedConfig: {
    security: {
      disableExtensionManagement: true,
      blockExtensionStoreAndSettings: true,
    },
  },
});
```

#### WebRTC 使用说明

可以先按下面的方式理解 `fingerprint.rtc`：

- `disable`
  - 不希望站点通过 WebRTC 看到 IP 时使用
- `truth`
  - 保留真实 WebRTC 行为时使用
- `replace`
  - 希望站点看到一个你控制的 IP 时使用
- `forward`
  - 需要走 forward 语义时使用；SDK 会固定补 `rtc.stun = stun:stun.l.google.com:19302`

`replace` 和 `forward` 的取值规则：

- SDK 不负责做代理检测。
- `rtc.value` 优先使用业务方显式传入的 `fingerprint.rtc.value`
- 如果未传 `fingerprint.rtc.value`，则回退使用业务透传的 `proxy.ipInfo.ip`

`replace` 模式下的额外能力：

- 如果传 `fingerprint.rtc.useRandomInternalIp = true`，SDK 会优先生成随机内网 IP，并忽略 `rtc.value` / `proxy.ipInfo.ip`
- 当前 SDK 只支持“不保持模式”，也就是每次生成新的随机内网 IP；还不支持跨次复用同一个随机 IP

示例：

```javascript
// 1. 手动指定一个 WebRTC IP
rtc: {
  type: 'replace',
  value: '9.9.9.9',
}

// 2. 让 SDK 直接使用业务传入的 proxy.ipInfo.ip
rtc: {
  type: 'replace',
}

// 3. 使用随机内网 IP
rtc: {
  type: 'replace',
  useRandomInternalIp: true,
}

// 4. 使用 forward 模式
rtc: {
  type: 'forward',
  value: '8.8.8.8',
}
```

#### WebGL 匹配信息工具函数

SDK 提供 `getWebGLInfo` / `toWebGLFingerprint` 工具函数，用于按目标 OS 获取一组相互匹配的 WebGL manufacturer、WebGL renderer 和 WebGPU adapter metadata。SDK 不会在启动链路中擅自替业务方决定 WebGL；业务方需要先调用工具函数拿到匹配记录，再通过 `fingerprint.webgl` 的 `custom` 模式显式传入。

规则：

- `getWebGLInfo(os)` 只从目标 OS 的数据中随机返回一条完整记录
- `getWebGLInfo(os, manufacturer)` 会在目标 OS + 指定 manufacturer 的候选记录里随机选择
- manufacturer 使用精确匹配；建议先通过 `getWebGLManufacturers(os)` 查看支持值
- 返回记录会保持 WebGL manufacturer、WebGL renderer、adapter vendor、adapter architecture 的原始匹配关系
- `getWebGLInfo` 只接受 SDK 支持的 `OsType` 值，非法 OS 会抛错
- 当前数据中 Windows / Mac 记录包含 adapter metadata；Linux / Android / iOS 记录没有 adapter metadata，`toWebGLFingerprint` 会自动省略这些空字段
- `truth` 模式不下发 `webgl.vendor`、`webgl.render`、`webgpu.arch`、`webgpu.vendor`，表示使用运行环境真实 WebGL / adapter 行为

正常使用场景下，业务只需要传 OS：

```javascript
const webglInfo = getWebGLInfo(OsType.Windows);
```

只有当业务已经明确希望限制显卡厂商时，才需要传第二个参数 `manufacturer`。比如业务画像已经确定为 Windows + AMD，或业务自己要控制 Intel / NVIDIA / AMD 的分布比例，可以先通过 `getWebGLManufacturers(os)` 查看可选值，再传给 `getWebGLInfo(os, manufacturer)`。

推荐用法：

```javascript
const { createSDK, getWebGLInfo, toWebGLFingerprint, OsType } = require('dic-browser-sdk');

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

按 manufacturer 收窄候选池：

```javascript
const manufacturers = getWebGLManufacturers(OsType.Windows);
// [ 'Google Inc. (AMD)', 'Google Inc. (Intel)', 'Google Inc. (NVIDIA)' ]

const amdWebglInfo = getWebGLInfo(OsType.Windows, 'Google Inc. (AMD)');
```

如果业务方需要手动映射字段，可以在 `webgl.type = 'custom'` 时传：

- `fingerprint.webgl.vendor` -> 内核参数 `webgl.vendor`，custom 模式必填
- `fingerprint.webgl.renderer` -> 内核参数 `webgl.render`，custom 模式必填
- `fingerprint.webgl.adapterInfoArchitecture` -> 内核参数 `webgpu.arch`
- `fingerprint.webgl.adapterInfoVendor` -> 内核参数 `webgpu.vendor`

示例：

```javascript
webgl: {
  type: 'custom',
  vendor: webglInfo.webglManufacturer,
  renderer: webglInfo.webglRender,
  adapterInfoArchitecture: webglInfo.adapterInfoArchitecture,
  adapterInfoVendor: webglInfo.adapterInfoVendor,
}
```

更多说明见 [docs/webgl-info.md](./docs/webgl-info.md)。

### 3. 浏览器实例管理

#### 启动实例

```javascript
const { id, wsUrl } = await sdk.launch({
  instanceId: 'my-instance', // 可选，不提供则自动生成
  fingerprintConfig: fingerprint.config,
  chromiumPath: '/path/to/browser', // 必填，本次启动使用的浏览器内核路径
  useSystemProxy: false, // 可选，默认 false，不读取本机系统代理
});
```

`chromiumPath` 必须在每次启动实例时显式传入。这样可以避免多个环境混用不同内核版本时，某次启动误用其他环境的内核路径。

#### 实例状态查询

```javascript
// 获取单个实例信息
const instance = sdk.getInstance('instance-id');
console.log(instance.status); // running, stopped, error 等

// 获取所有实例
const allInstances = sdk.getAllInstances();

// 检查实例是否存活
const isAlive = sdk.isInstanceAlive('instance-id');

// 获取SDK状态
const status = sdk.getStatus();
console.log(status.totalInstances, status.runningInstances);
```

#### 关闭实例

```javascript
// 普通关闭
await sdk.close('instance-id');
```

## 🔧 高级用法

### 代理配置

SDK 默认不会读取或使用本机系统代理。业务方在 `createFingerprint` 中传入的 `proxy` 会直接作为浏览器实例的一级代理下发给内核。

如果确实需要让某个实例读取本机系统代理，可以在启动实例时显式传 `useSystemProxy: true`：

```javascript
const instance = await sdk.launch({
  fingerprintConfig: fingerprint.config,
  chromiumPath: '/path/to/browser',
  useSystemProxy: true,
});
```

开启 `useSystemProxy` 后，如果同时配置了业务代理，SDK 会将本机系统代理作为一级代理，将业务代理作为二级代理传给内核。未开启时，即使本机存在系统代理，也不会生成系统代理相关的 `--proxy-server` 参数。

```javascript
// HTTP代理
proxy: {
  host: 'proxy.com',
  port: 8080,
  type: 'HTTP',
  username: 'user',
  password: 'pass'
}

// SOCKS5代理
proxy: {
  host: 'socks.com',
  port: 1080,
  type: 'SOCKS5',
  username: 'user',
  password: 'pass'
}
```

### 事件监听

```javascript
sdk.on('instanceLaunched', (instance) => {
  console.log('实例启动:', instance.id);
});

sdk.on('instanceClosed', ({ instanceId }) => {
  console.log('实例关闭:', instanceId);
});

sdk.on('processExit', (data) => {
  console.log('进程退出:', data.pid);
});
```

### 批量管理

```javascript
// 启动多个实例
const instances = [];
for (let i = 0; i < 5; i++) {
  const fingerprint = await sdk.createFingerprint({
    userAgent: `UserAgent-${i}`,
    // ... 其他配置
  });

  const instance = await sdk.launch({
    fingerprintConfig: fingerprint.config,
    chromiumPath: '/path/to/browser',
  });

  instances.push(instance);
}

// 批量关闭
for (const instance of instances) {
  await sdk.close(instance.id);
}
```

### 环境 Cookie

```javascript
// 获取标准格式 Cookie
const cookies = await sdk.getStandardCookies(instanceId);
// 设置标准格式 Cookie（会暂存，在下次启动实例时生效）
await sdk.setStandardCookies(instanceId, cookies);
```

注意事项：

- 实例必须为关闭状态时操作。
- 当前 SDK 支持多个内核版本（如 134 / 142 / 143）。跨内核版本迁移时，Cookie 存在兼容风险。
- SDK 会在启动前根据本次 `launch.chromiumPath` 识别目标内核版本，并自动检查 Cookie DB 表结构；不兼容时会替换为对应版本的空模板，再写入业务侧通过 SDK 暂存的 Cookie。
- 对业务侧来说，这意味着旧版本内核下生成的本地 Cookie，在切换到新版本内核后，不建议默认认为一定还能原样延续。

## 🛠️ 你需要准备

- Node.js >= 18.0.0
- Windows/macOS
- 可用的 Chromium 内核

## 📖 错误处理

```javascript
try {
  await sdk.launch(options);
} catch (error) {
  console.error('启动失败:', error.message);
  console.error('错误码:', error.code);
}
```

常见错误码参考：[ERROR_CODES.md](./docs/ERROR_CODES.md)`<br>`

## 🔍 调试

启用详细日志：

```javascript
await sdk.initialize({
  baseDir: './data',
  logLevel: 'debug', // error, warn, info, debug
});
```

## 📄 许可证

MIT License
