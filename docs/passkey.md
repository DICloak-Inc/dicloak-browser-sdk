# DIC Browser SDK Passkey 参数指南

业务方可以通过 `advancedConfig.passkey` 将自建 Passkey 服务配置传给浏览器内核。SDK 负责参数透传，Passkey 的注册、使用及同步由配套内核与业务方的 Passkey 服务处理。

## 使用条件

- 使用已支持自建 Passkey 功能的配套浏览器内核。仅内核主版本相同不能保证包含此功能，请使用交付时配套的内核。
- 业务方准备自己的 Passkey 服务，并提供对应的服务地址、用户 ID 和有效的 session token。
- SDK 的初始化与安装步骤见 [主文档](../README.md)。

## 参数位置与字段

在 `sdk.createFingerprint()` 的 `advancedConfig.passkey` 中传入对象，并在 `sdk.launch()` 时传入返回的 `fingerprintConfig.config`。

```typescript
interface PasskeyConfig {
  server: string;
  uid: string;
  token: string;
  headers?: Record<string, string>;
  passphrase?: string;
}
```

| 字段 | 类型 | 必填 | 说明 |
| --- | --- | --- | --- |
| `server` | `string` | 是 | 自建 Passkey 服务器地址，不能为空，例如 `https://passkey.example.com:8443`。 |
| `uid` | `string` | 是 | Passkey 服务对应的用户 ID，不能为空。 |
| `token` | `string` | 是 | 内核向 Passkey 服务发起请求时使用的 session token，不能为空。该值由业务方提供，与 SDK 初始化使用的 SDK Key 分开配置。 |
| `headers` | `Record<string, string>` | 否 | 内核请求 Passkey 服务时携带的自定义请求头，名称和值均为字符串。 |
| `passphrase` | `string` | 否 | 指定的主密码，由业务方按其 Passkey 服务的约定提供。 |

SDK 按原值序列化这些字段，不补默认值、不修改地址或凭证、不验证凭证有效性。业务方需提供上述必填字段的非空值；可选字段不使用时直接省略。

## 完整调用示例

下面的地址与凭证均为示例值，请替换为业务方实际提供的配置。`headers` 和 `passphrase` 展示可选字段的传法，按需保留。

```javascript
const { createSDK } = require('dic-browser-sdk');
const path = require('path');

async function main() {
  const sdk = createSDK();
  const { initialized } = await sdk.initialize({
    key: 'Your usage sdk key',
    baseDir: path.join(__dirname, 'browser-data'),
    logLevel: 'info',
  });
  if (!initialized) throw new Error('SDK 初始化失败，请检查 SDK Key 和日志');

  try {
    const { instanceId, fingerprintConfig } = await sdk.createFingerprint({
      userAgent: 'Your full browser user agent',
      advancedConfig: {
        passkey: {
          server: 'https://passkey.example.com:8443',
          uid: 'your-user-id',
          token: 'your-session-token',
          headers: {
            'X-Api-Key': 'your-api-key',
            'X-Tenant-Id': 'your-tenant-id',
          },
          passphrase: 'your-master-passphrase',
        },
      },
    });

    const { id } = await sdk.launch({
      instanceId,
      fingerprintConfig: fingerprintConfig.config,
      chromiumPath: '/path/to/browser',
    });

    // 在这里执行网页上的 Passkey 注册或登录流程，完成后关闭实例。
    await sdk.close(id);
  } finally {
    await sdk.cleanup();
  }
}

main().catch(console.error);
```

最小配置只需 `server`、`uid` 和 `token`：

```javascript
advancedConfig: {
  passkey: {
    server: 'https://passkey.example.com',
    uid: 'your-user-id',
    token: 'your-session-token',
  },
},
```

## 内核参数格式

SDK 将 `advancedConfig.passkey` 对象序列化为 UTF-8 JSON，再进行标准 Base64 编码，写入内核启动参数 `passkey`：

```javascript
launchParams.passkey = Buffer.from(
  JSON.stringify(advancedConfig.passkey),
  'utf8',
).toString('base64');
```

整个启动参数对象继续通过 SDK 现有加密流程生成 `--launch-key`。业务方传入普通对象即可，无需自行进行 JSON/Base64 编码，也无需添加独立的 `--passkey` 命令行参数。

未配置 `advancedConfig.passkey` 时，SDK 不下发内核参数 `passkey`，保持原有启动方式。需要更新 token、请求头或主密码时，业务方应在下次启动前传入更新后的配置；SDK 不负责运行期间的凭证刷新或 Passkey 服务请求。
