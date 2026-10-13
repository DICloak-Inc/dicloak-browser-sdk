# DIC Browser SDK Passkey 参数指南

使用自建 Passkey 服务时，在创建浏览器配置的 `advancedConfig.passkey` 中填写服务地址、用户信息和凭证，然后使用该配置启动浏览器。

## 使用条件

- 使用支持自建 Passkey 功能的配套浏览器。
- 准备可用的 Passkey 服务地址、用户 ID 和有效的 session token。
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
| `token` | `string` | 是 | 访问 Passkey 服务时使用的 session token，不能为空。该值与 SDK 初始化使用的 SDK Key 分开配置。 |
| `headers` | `Record<string, string>` | 否 | 访问 Passkey 服务时需要携带的自定义请求头，名称和值均为字符串。 |
| `passphrase` | `string` | 否 | 指定的主密码，由业务方按其 Passkey 服务的约定提供。 |

`server`、`uid` 和 `token` 都需要填写；`headers` 和 `passphrase` 按 Passkey 服务的要求配置，不需要时直接省略。

## 最小配置

如果 Passkey 服务不要求额外请求头或主密码，只需填写三个必填字段：

```javascript
const { instanceId, fingerprintConfig } = await sdk.createFingerprint({
  advancedConfig: {
    passkey: {
      server: 'https://passkey.example.com',
      uid: 'your-user-id',
      token: 'your-session-token',
    },
  },
});

await sdk.launch({
  instanceId,
  fingerprintConfig: fingerprintConfig.config,
  chromiumPath: '/path/to/browser',
});
```

## 完整调用示例

下面展示从初始化到启动、关闭浏览器的完整用法。请将 SDK Key、服务地址、用户信息、凭证、UA 和浏览器路径替换为实际值。示例中的 `headers` 和 `passphrase` 按需保留。

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

## 配置调整

需要更换用户、token、请求头或主密码时，使用更新后的参数重新创建浏览器配置，并在下一次启动时使用该配置。

不需要使用自建 Passkey 服务时，省略 `advancedConfig.passkey` 即可。
