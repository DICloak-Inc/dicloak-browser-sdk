# UA 与 Client Hint 传参说明

这份文档面向使用 SDK 的业务方，重点说明 `userAgent` 和 `platformVersion` 该怎么传，以及传错后可能带来的影响。

## 先记结论

- `userAgent` 由业务方传完整字符串，SDK 不会自动生成，也不会帮你修正。
- `fingerprint.platformVersion` 也由业务方传，SDK 只负责透传给内核参数 `platform.version`。
- 如果业务站点会读取 Client Hint，仅修改 `userAgent` 可能不够，还需要同时关注 `platformVersion`。
- 如果是 iOS 场景，重点通常是把 UA 传正确；iOS 上 Client Hint 能力有限，不要假设它和桌面端一致。

## SDK 做什么

SDK 只做两件事：

- 把 `userAgent` 原样透传到浏览器启动参数
- 把 `fingerprint.platformVersion` 透传到内核启动参数 `platform.version`

SDK 不做这些事情：

- 不自动推导系统版本
- 不根据 UA 自动生成 `platformVersion`
- 不自动检查 UA 和 `platformVersion` 是否匹配

## 推荐传法

### 1. 只需要控制 UA

适用于主要依赖传统 UA 的场景，尤其是 iOS。

```javascript
const fingerprint = await sdk.createFingerprint({
  userAgent:
    'Mozilla/5.0 (iPhone; CPU iPhone OS 17_4_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) CriOS/142.0.7444.46 Mobile/15E148 Safari/604.1',
});
```

### 2. 需要同时兼顾 Client Hint

适用于业务站点会读取 `Sec-CH-UA-Platform-Version` 或 `navigator.userAgentData` 的场景。

```javascript
const fingerprint = await sdk.createFingerprint({
  userAgent:
    'Mozilla/5.0 (Windows NT 10.0; Win64; x64) AppleWebKit/537.36 (KHTML, like Gecko) Chrome/142.0.0.0 Safari/537.36',
  fingerprint: {
    platformVersion: '15.0.0',
  },
});
```

上面这个例子里：

- `userAgent` 由业务方自己完整传入
- `platformVersion: '15.0.0'` 会被 SDK 透传成内核参数 `platform.version`

## `platformVersion` 常见映射

业务方如果需要传 `fingerprint.platformVersion`，可以优先参考下面这张表。

| 目标系统 | 推荐值 |
| --- | --- |
| Windows 7 | `0.1.0` |
| Windows 8 | `0.3.0` |
| Windows 10 | `10.0.0` |
| Windows 11 | `15.0.0` / `19.0.0` |
| macOS 15 | `15.3.1` / `15.7.4` |
| macOS 14 | `14.1.1` / `14.8.0` |
| macOS 13 | `13.0.0` / `13.7.0` |
| macOS 12 | `12.1.0` / `12.6.0` / `12.7.0` |
| macOS 11 | `11.2.2` / `11.2.3` / `11.7.3` |
| macOS 10 | `10.12.0` / `10.13.0` / `10.14.0` |
| Android 16 | `16.0.0` |
| Android 15 | `15.0.0` |
| Android 14 | `14.0.0` |
| Android 13 | `13.0.0` |
| Android 12 | `12.0.0` |
| Android 11 | `11.0.0` |
| Android 10 | `10.0.0` |
| Android 9 | `9.0.0` |
| Linux | 空值 |
| iOS | 一般不传 |

说明：

- 如果同一系统有多个可选值，业务方按目标版本自行选择即可。
- SDK 不会帮你推导或纠正 `platformVersion`。

## iOS 场景提醒

如果你在业务侧维护 iPhone 或 iPad 的 UA，建议重点关注这些点：

- iOS Chrome 场景应使用 `CriOS`，不要继续沿用 `Chrome`
- `AppleWebKit` 应使用 iOS 对应值，不要混用 Android 常见值
- `Mobile/15E148` 和 `Safari/604.1` 一般需要保持正确
- `iPhone` / `iPad` 设备标识要与目标设备一致

对 SDK 来说，这些内容都属于业务方的 UA 组装责任，SDK 不会代替你修正。

### iOS UA 关键字段参考

| 项目 | 推荐写法 |
| --- | --- |
| 浏览器标识 | `CriOS` |
| WebKit | `AppleWebKit/605.1.15` |
| 构建号 | `Mobile/15E148` |
| Safari | `Safari/604.1` |
| 设备标识 | `iPhone` 或 `iPad` |
| 系统版本写法 | `iPhone OS 17_4_1` 这类格式 |

版本号写法规则：

- 把系统版本中的 `.` 替换成 `_`
- iPhone 常见片段：`CPU iPhone OS 17_4_1 like Mac OS X`
- iPad 常见片段：`CPU OS 17_4_1 like Mac OS X`

常见例子：

- `iOS 17.4.1` -> `CPU iPhone OS 17_4_1 like Mac OS X`
- `iOS 17.4` -> `CPU iPhone OS 17_4 like Mac OS X`
- `iOS 16.7.7` -> `CPU iPhone OS 16_7_7 like Mac OS X`
- `iPadOS 18.2.1` -> `CPU OS 18_2_1 like Mac OS X`

### iOS 系统版本列表示例

业务方如果需要直接挑选常见版本，可以优先参考下面这些值。
下面列举的是常见示例，不是完整列表；如果你需要其他版本，按同样的写法规则扩展即可。

常见版本示例：

- iOS 13：`iPhone OS 13_1_3`、`iPhone OS 13_5_1`、`iPhone OS 13_7`
- iOS 14：`iPhone OS 14_1`、`iPhone OS 14_4`、`iPhone OS 14_8`
- iOS 15：`iPhone OS 15_0_2`、`iPhone OS 15_3_1`、`iPhone OS 15_6_1`
- iOS 16：`iPhone OS 16_1_1`、`iPhone OS 16_4_1`、`iPhone OS 16_7_7`
- iOS 17：`iPhone OS 17_1_2`、`iPhone OS 17_4_1`、`iPhone OS 17_7_10`
- iOS 18：`iPhone OS 18_0_1`、`iPhone OS 18_2_1`、`iPhone OS 18_3_1`

参考示例：

```javascript
Mozilla/5.0 (iPhone; CPU iPhone OS 17_4_1 like Mac OS X) AppleWebKit/605.1.15 (KHTML, like Gecko) CriOS/142.0.7444.46 Mobile/15E148 Safari/604.1
```

## 常见问题

### 1. 只传 UA，不传 `platformVersion` 可以吗？

可以。  
如果目标站点不依赖 Client Hint，通常只传 UA 就够了。

### 2. 什么情况下建议补 `platformVersion`？

当目标站点会读取这些信息时建议补：

- `Sec-CH-UA-Platform-Version`
- `navigator.userAgentData.getHighEntropyValues(['platformVersion'])`

### 3. 最容易出什么问题？

最常见的是参数不一致，例如：

- UA 写的是 iOS，但 `platformVersion` 传的是 Windows 风格版本
- UA 已经改成新版本，但 `platformVersion` 还是旧值
- iOS UA 里仍然使用 `Chrome/...` 而不是 `CriOS/...`

如果这些信息互相对不上，业务站点可能会识别为异常环境。
