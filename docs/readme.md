# DIC Browser SDK

基于 Chromium(DIC) 的浏览器 SDK，提供指纹管理和进程控制功能，适用于浏览器自动化和反检测场景。

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
    sourceDataDir: path.join(__dirname, 'source-data'), // 内核源数据模板路径
    chromiumPath: '/path/to/chromium.exe', // 指定浏览器内核路径
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
  sourceDataDir: './source-data', // 内核源数据模板路径，必填
  chromiumPath: '/path/to/browser', // 浏览器路径
  logLevel: 'info', // 日志级别
});
```

### 2. 指纹配置

#### 业务方最常用的 3 个参数

- `userAgent`
  由业务方传完整字符串，SDK 不会自动修改、拼接或纠正。
- `fingerprint.platformVersion`
  只有目标站点会读取 Client Hint 时再传。SDK 只负责透传给内核参数 `platform.version`。
- `proxy.ipInfo`
  只有你希望语言、时区、地理位置等信息跟随代理环境时再传。

如果你只想尽快跑通，通常先关注 `userAgent` 即可；只有在目标站点会校验更细的环境信息时，再补 `platformVersion` 和 `proxy.ipInfo`。

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
  /** URL过滤配置 */
  urls?: {
    black?: string; // 黑名单
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
  /** WebGL配置 */
  webgl?: {
    type: 'custom' | 'truth';
    vendor?: string;
    renderer?: string;
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

#### UA 与 platformVersion 说明

- `userAgent` 由业务方传完整字符串，SDK 不会自动修改、拼接或纠正。
- `fingerprint.platformVersion` 由业务方按目标系统版本传入，SDK 只负责透传给内核的 `platform.version` 参数。
- 建议 `chromiumPath` 对应的内核版本与 `userAgent` 中的 Chrome 主版本尽量保持一致。
- 建议 `userAgent`、目标系统、`platformVersion` 三者保持一致，否则业务站点可能识别出环境不一致。
- iOS 场景通常重点是传正确的 UA；`platformVersion` 主要用于配合 Client Hint 场景。

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

#### WebGPU metadata 说明

如果业务方需要同时控制 WebGL 和 WebGPU adapter 信息，可以在 `webgl.type = 'custom'` 时传：

- `fingerprint.webgl.vendor` -> 内核参数 `webgl.vendor`
- `fingerprint.webgl.renderer` -> 内核参数 `webgl.render`
- `fingerprint.webgl.adapterInfoArchitecture` -> 内核参数 `webgpu.arch`
- `fingerprint.webgl.adapterInfoVendor` -> 内核参数 `webgpu.vendor`

这些字段由业务方按目标设备信息传入，SDK 只负责透传，不会自动推导。

示例：

```javascript
webgl: {
  type: 'custom',
  vendor: 'Google Inc. (Intel Inc.)',
  renderer: 'ANGLE (Intel, Intel(R) UHD Graphics 630 Direct3D11 vs_5_0 ps_5_0)',
  adapterInfoArchitecture: 'gen-9',
  adapterInfoVendor: 'intel',
}
```

### 3. 浏览器实例管理

#### 启动实例

```javascript
const { id, wsUrl } = await sdk.launch({
  instanceId: 'my-instance', // 可选，不提供则自动生成
  fingerprintConfig: fingerprint.config,
});
```

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
