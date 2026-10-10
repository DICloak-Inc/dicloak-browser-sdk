# Linux 接入指南

本指南说明在 Linux 服务器或桌面上安装 SDK、启动浏览器和配置代理的用法。

## 1. 准备 SDK 和浏览器

1. 按 README 的安装步骤解压 SDK，并执行 `npm install`。运行环境需要 Node.js 18 或更高版本。
2. 将配套的 Linux 浏览器内核完整解压到固定目录，保留包内的文件和子目录。
3. 后续启动时，将 `chromiumPath` 设置为实际浏览器可执行文件的路径，例如 `/opt/browser/browser`（请替换为实际路径），不要只传目录或压缩包路径。

如果提示没有执行权限，可以为可执行文件增加权限：

```bash
chmod +x /opt/browser/browser
```

访问中文网页前，请按 README 的字体安装步骤准备中文字体。

## 2. 在无桌面的服务器上启动

启动时传入 `headless: true`，可以在没有桌面和显示服务的服务器上使用浏览器。
下面的示例以 Linux 指纹为目标；将 SDK Key、浏览器路径和 UA 版本替换为实际使用的值：

```javascript
const { createSDK, OsType } = require('dic-browser-sdk');
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
      uaOs: OsType.Linux,
      userAgent:
        'Mozilla/5.0 (X11; Linux x86_64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/147.0.7727.149 Safari/537.36',
    });

    const { id, wsUrl } = await sdk.launch({
      instanceId,
      fingerprintConfig: fingerprintConfig.config,
      chromiumPath: '/opt/browser/browser',
      headless: true,
    });

    console.log('浏览器已启动：', id);
    console.log('自动化连接地址：', wsUrl);

    // 在这里接入自动化流程；流程结束后关闭实例。
    await sdk.close(id);
  } finally {
    await sdk.cleanup();
  }
}

main().catch(console.error);
```

需要显示浏览器窗口时，省略 `headless` 或设置 `headless: false`，并在有桌面或显示服务的环境中运行。

## 3. 使用代理和设置指纹

需要代理时，在 `createFingerprint` 的配置中加入 `proxy`：

```javascript
proxy: {
  type: 'HTTP',
  host: 'proxy.example.com',
  port: 8080,
  username: 'your-proxy-user', // 代理不需要认证时可省略
  password: 'your-proxy-password',
},
```

Linux 上请显式配置代理，`useSystemProxy: true` 不会自动读取系统代理。

运行机器使用 Linux，不会自动把指纹配置改成 Linux。希望呈现 Linux 指纹时，显式设置 `uaOs: OsType.Linux` 和匹配的 `userAgent`；需要其他指纹环境时，按业务目标设置。UA 中的 Chrome 主版本建议与所用浏览器内核一致，更多说明见 [UA 与 Client Hint 传参说明](./UA_AND_CLIENT_HINT_GUIDE.md)。
