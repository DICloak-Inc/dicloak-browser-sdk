# SDK 升级发布说明

本次 SDK 升级，主要面向业务侧补齐了多版本内核支持和新内核兼容能力。

## 这次更新了什么

- 内核支持从原来的单一 `134`，扩展为 `134 / 142 / 143`
- 补齐了新内核相关的部分指纹兼容能力
- 补齐了 Cookie 在新内核与不同平台布局下的兼容处理
- 补充了 UA / Client Hint 相关说明文档
- 新增随机字体工具函数 `getFonts` / `createRandomFontValue`，并在启动时通过 `font.list` 传递完整字体列表
- 新增 WebGL 匹配信息工具函数 `getWebGLInfo` / `toWebGLFingerprint`，帮助业务获取匹配的 WebGL renderer、manufacturer 和 WebGPU adapter metadata

## 业务方需要重点关注什么

- 现在可以按场景选择不同内核版本，不再只有 `134`
- 建议 `chromiumPath` 对应的内核版本，与 `userAgent` 中的 Chrome 主版本尽量保持一致
- 如果目标站点会读取 Client Hint，除了 `userAgent`，还需要按需传入 `fingerprint.platformVersion`
- 如果业务要显式控制字体，建议使用 `getFonts(OsType.xxx, 80-100)` 生成列表后通过 `fingerprint.font` 的 `custom` 模式传入
- 如果业务要显式控制 WebGL，建议先调用 `getWebGLInfo(OsType.xxx)`，再用 `toWebGLFingerprint` 转成 `fingerprint.webgl` 的 `custom` 配置
- 切换内核版本或运行平台时，Cookie 与本地用户数据都存在兼容风险，不建议默认认为一定还能完全原样延续

## 迁移建议

- 如果业务当前只跑 `134` 且希望先平稳升级，可以先继续使用 `134` 验证业务流程
- 在业务流程稳定后，再逐步灰度 `142 / 143`
- 对登录态敏感的场景，建议在切换内核版本时重点关注 Cookie 延续情况

## 一句话总结

这次升级的核心不是单纯增加两个内核包，而是让 SDK 具备了多版本内核使用能力，并补齐了新内核下的指纹和 Cookie 兼容支持。
