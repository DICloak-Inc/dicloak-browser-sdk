# WebRTC 指纹全链路文档

> 从用户在 UI 上选择 WebRTC 模式，到最终传递给内核的参数，覆盖完整链路、全部场景分支和外部依赖。

---

## 一、UI 交互层

### 1.1 一级选项：WebRTC 模式

用户在环境编辑页的指纹配置区域选择 WebRTC 模式，共 4 个选项：

| UI 标签 | 枚举值 | 说明 |
|---------|--------|------|
| Substitute（替换） | `'replace'` | 启用 WebRTC，替换公网 IP 为指定值，隐藏本地 IP |
| Forward（转发） | `'forward'` | 将页面指定的 STUN 服务器替换为特定服务器，降低 IP 泄露风险 |
| Real（真实） | `'truth'` | 启用 WebRTC，网站可检测到真实 IP |
| Disable（禁用） | `'disable'` | 禁用 WebRTC，网站无法通过 WebRTC 检测到 IP |

**默认值**：`'disable'`

### 1.2 二级选项：Replace 模式下的子模式

仅当一级选项为 `replace` 时显示，用户通过单选按钮选择：

| UI 标签 | 内部标识 | 对应 flag 组合 | 说明 |
|---------|---------|---------------|------|
| 手动输入 | `'manual'` | `webrtcSyncProxyIpFlag=false, webrtcUseRandomInternalIp=false` | 用户手动填入一个 IP 地址 |
| 使用网络出口 IP | `'proxyIp'` | `webrtcSyncProxyIpFlag=true, webrtcUseRandomInternalIp=false` | 自动使用代理检测得到的出口 IP |
| 使用随机内网 IP | `'randomIp'` | `webrtcSyncProxyIpFlag=false, webrtcUseRandomInternalIp=true` | 自动生成一个仿真内网 IP |

### 1.3 三级选项：随机内网 IP 的保持策略

仅当二级选项为 `randomIp` 时显示：

| UI 标签 | 字段值 | 说明 |
|---------|--------|------|
| 不保持 | `webrtcKeepRandomInternalIp = false` | 每次打开环境都生成新的随机 IP |
| 保持 | `webrtcKeepRandomInternalIp = true` | 跨次打开复用同一个随机 IP（代理 IP 变化时才重新生成） |

### 1.4 输入校验

手动输入模式下，IP 格式校验：
- 正则：`/^((25[0-5]\.|2[0-4]\d\.|1\d{2}\.|[1-9]?\d\.){3}(25[0-5]|2[0-4]\d|1\d{2}|[1-9]?\d))$/`
- 仅 `replace` + `manual` 组合时校验，其他子模式不需要用户输入

---

## 二、数据模型

### 2.1 存储字段（ExtendConfigResp）

| 字段 | 类型 | 默认值 | 说明 |
|------|------|--------|------|
| `webrtcType` | `string` | `'disable'` | 一级模式枚举 |
| `webrtcValue` | `string` | `''` | IP 地址（手动填入 / 运行时填充） |
| `webrtcSyncProxyIpFlag` | `boolean` | `false` | 是否同步代理出口 IP |
| `webrtcUseRandomInternalIp` | `boolean?` | `false` | 是否使用随机内网 IP |
| `webrtcKeepRandomInternalIp` | `boolean?` | `false` | 随机内网 IP 是否保持 |
| `webrtcRandomInternalIpSaved` | `string?` | `undefined` | 已保存的随机内网 IP（保持模式下使用） |

### 2.2 UI flag 与字段的映射关系

UI 使用一个计算属性 `webrtcReplaceType` 来双向转换：

```
读取方向（flag → UI）：
  webrtcSyncProxyIpFlag === true      → 'proxyIp'
  webrtcUseRandomInternalIp === true  → 'randomIp'
  其他                                → 'manual'

写入方向（UI → flag）：
  'proxyIp'  → webrtcSyncProxyIpFlag=true,  webrtcUseRandomInternalIp=false
  'randomIp' → webrtcSyncProxyIpFlag=false, webrtcUseRandomInternalIp=true
  'manual'   → webrtcSyncProxyIpFlag=false, webrtcUseRandomInternalIp=false
```

---

## 三、预处理层（paramsMergeStep）

环境打开流程中，`paramsMergeStep` 会根据 `webrtcType` 和各 flag 对 `webrtcValue` 进行运行时赋值。以下按执行顺序排列。

### 3.1 清理阶段

```
如果 webrtcType === 'truth' 或 'disable'：
  → delete webrtcValue
  → 后续不会再处理 webrtcValue
  → 最终效果：只传 rtc.type，无 rtc.value
```

### 3.2 代理出口 IP 同步（replace + proxyIp）

```
前置条件：webrtcSyncProxyIpFlag === true 且 webrtcType === 'replace'

步骤：
  1. webrtcValue = ''（先清空）
  2. 如果代理检测成功（checkProxyResult.success && checkProxyResult.ip）：
     → webrtcValue = checkProxyResult.ip
     否则 webrtcValue 保持为空字符串
```

**依赖**：代理检测结果 `checkProxyResult`，来自 `proxyCheckStep` 或 `fetchEnvInfoStep` 中的代理检测流程。

### 3.3 转发模式（forward）

```
前置条件：webrtcType === 'forward'

步骤：
  1. webrtcValue = ''（先清空）
  2. 如果代理检测成功：
     → webrtcValue = checkProxyResult.ip
     否则 webrtcValue 保持为空字符串
```

**依赖**：同上，依赖代理检测结果。

### 3.4 随机内网 IP（replace + randomIp）

```
前置条件：webrtcUseRandomInternalIp === true 且 webrtcType === 'replace'

分支 A — 保持模式（webrtcKeepRandomInternalIp === true）：

  获取当前代理出口 IP：currentProxyIp = checkProxyResult.ip || ''
  获取上次保存的代理 IP：savedProxyIp = openEnvDetail.proxyIpInfo.ip
  获取上次保存的随机 IP：savedRandomIp = webrtcRandomInternalIpSaved

  if (savedProxyIp !== currentProxyIp) {
    // 代理 IP 变化 → 重新生成
    webrtcValue = generateWiFiIP(uaOs)
    webrtcRandomInternalIpSaved = webrtcValue
    → 异步调用 saveEnvfingerprint 持久化到后端
  } else if (savedRandomIp 存在) {
    // 代理 IP 未变化 → 复用已保存的
    webrtcValue = savedRandomIp
  } else {
    // 首次（无历史记录）→ 生成并保存
    webrtcValue = generateWiFiIP(uaOs)
    webrtcRandomInternalIpSaved = webrtcValue
    → 异步调用 saveEnvfingerprint 持久化到后端
  }

分支 B — 不保持模式（webrtcKeepRandomInternalIp === false）：
  webrtcValue = generateWiFiIP(uaOs)   // 每次都生成新的
```

**依赖**：
- `generateWiFiIP(uaOs)` — 随机内网 IP 生成函数
- `openEnvDetail.proxyIpInfo.ip` — 上次打开时保存的代理 IP（来自服务端）
- `webrtcRandomInternalIpSaved` — 上次保存的随机 IP（存储在环境指纹配置中）
- `saveEnvfingerprint` — 写回后端的 API（仅非 RPA 模式下调用）

---

## 四、随机内网 IP 生成算法（generateWiFiIP）

根据 `uaOs` 区分桌面端和移动端，使用加权随机分布生成仿真内网 IP。

### 4.1 桌面端（Windows / Mac / Linux）

格式：`192.168.<third>.<fourth>`

第三段（子网段），加权分布：

| 值 | 权重 | 对应路由器品牌 |
|----|------|--------------|
| 1 | 45% | TP-Link / 华为 / 中兴（192.168.1.x） |
| 0 | 25% | 腾达 / D-Link（192.168.0.x） |
| 31 | 12% | 小米（192.168.31.x） |
| 50 | 8% | 华硕（192.168.50.x） |
| 10 | 5% | 192.168.10.x |
| 3 | 3% | 192.168.3.x |
| 100 | 2% | 192.168.100.x |

第四段（主机段），加权分布：

| 范围 | 权重 | 说明 |
|------|------|------|
| 100 ~ 200 | 75% | DHCP 动态分配范围 |
| 2 ~ 50 | 20% | 静态保留地址 |
| 254 | 4% | 备用网关 |
| 1 | 1% | 偶尔分配为 .1 |

### 4.2 移动端（iOS / Android）

50% 概率使用 `10.x.x.x` 格式，50% 概率走桌面端逻辑。

`10.x.x.x` 格式时：

第二段，加权分布：

| 范围 | 权重 | 说明 |
|------|------|------|
| 0 ~ 20 | 40% | 服务器/核心网络 |
| 50 ~ 100 | 30% | 办公区 A |
| 101 ~ 150 | 20% | 办公区 B |
| 200 ~ 220 | 10% | 访客/无线网络 |

第三段：1 ~ 254 均匀随机

第四段，加权分布：

| 范围 | 权重 |
|------|------|
| 1 ~ 10 | 15% |
| 11 ~ 253 | 85% |

---

## 五、内核传参层

经过预处理后，`webrtcType` 和 `webrtcValue` 被转换为 launchParam 传给内核。

### 5.1 launchParam 初始赋值

```
rtc.type = config.webrtcType   // 始终传递，值为 'replace' | 'forward' | 'truth' | 'disable'
```

### 5.2 rtc.value 赋值逻辑

```
// 通用赋值（优先级低）
if (webrtcValue 存在 且 webrtcType !== 'noise') {
  rtc.value = String(webrtcValue)
}

// replace + randomIp 覆盖（优先级高，覆盖上面的值）
if (webrtcType === 'replace' 且 webrtcUseRandomInternalIp) {
  rtc.value = webrtcValue    // 此时 webrtcValue 已被预处理为随机 IP
}
```

### 5.3 forward 模式特殊处理

```
if (webrtcType === 'forward') {
  rtc.type = 'forward'                              // 再次赋值（确保覆盖）
  rtc.stun = 'stun:stun.l.google.com:19302'         // 硬编码 STUN 服务器地址
  if (webrtcValue 存在) {
    rtc.value = String(webrtcValue)                  // 代理出口 IP
  }
}
```

> 注意：桌面端存在一个不一致 — 普通打开环境 `rtc.stun = 'stun:stun.l.google.com:19302'`（带 `stun:` 前缀），RPA 打开环境 `rtc.stun = 'stun.l.google.com:19302'`（无前缀）。

### 5.4 完整参数映射总览

| 场景 | rtc.type | rtc.value | rtc.stun | 说明 |
|------|----------|-----------|----------|------|
| **truth** | `'truth'` | 不传 | 不传 | 使用真实 WebRTC |
| **disable** | `'disable'` | 不传 | 不传 | 禁用 WebRTC |
| **replace + 手动输入** | `'replace'` | 用户输入的 IP | 不传 | 手动指定替换 IP |
| **replace + 代理出口 IP** | `'replace'` | `checkProxyResult.ip` 或空 | 不传 | 自动同步代理 IP |
| **replace + 随机内网 IP** | `'replace'` | `generateWiFiIP()` 结果 | 不传 | 自动生成仿真内网 IP |
| **forward** | `'forward'` | `checkProxyResult.ip` 或空 | `'stun:stun.l.google.com:19302'` | 转发模式 + STUN 服务器 |

---

## 六、外部依赖关系

### 6.1 代理检测结果（checkProxyResult）

**影响场景**：replace + proxyIp、forward、replace + randomIp（保持模式下用于判断 IP 是否变化）

```typescript
interface CheckProxyResult {
  success: boolean;      // 代理检测是否成功
  ip: string;            // 代理出口 IP
  country: string;       // 国家
  countryCode: string;   // 国家代码
  timezone: string;      // 时区
  // ...
}
```

**来源**：`proxyCheckStep` / `fetchEnvInfoStep` 中调用 `checkProxyFromApp` 获取。

**异常行为**：
- 代理检测失败时（`success === false`），proxyIp 和 forward 场景下 `webrtcValue` 为空字符串，`rtc.value` 不会被设置
- 代理未配置（`proxyType === 'NON_USE'`）时同理

### 6.2 环境历史数据（openEnvDetail）

**影响场景**：replace + randomIp + 保持模式

| 字段 | 用途 |
|------|------|
| `openEnvDetail.proxyIpInfo.ip` | 上次打开时的代理出口 IP，用于比较是否变化 |
| `config.webrtcRandomInternalIpSaved` | 上次保存的随机内网 IP |

### 6.3 指纹持久化 API（saveEnvfingerprint）

**影响场景**：replace + randomIp + 保持模式，且代理 IP 变化或首次生成

调用 `envBatchFingerprintApi` 将 `webrtcRandomInternalIpSaved` 写回服务端。仅非 RPA 模式下调用。

### 6.4 uaOs（操作系统类型）

**影响场景**：replace + randomIp

`generateWiFiIP(uaOs)` 根据 OS 类型决定生成 `192.168.x.x`（桌面端）还是 `10.x.x.x`（移动端）格式。

---

## 七、SDK 侧对齐差异

| 能力 | 桌面端（electron/main.ts） | SDK 侧（packages/core/launch-config-builder.ts） |
|------|--------------------------|------------------------------------------------|
| rtc.type 赋值 | 支持 | 支持 |
| rtc.value 赋值 | 支持 | 支持 |
| forward 模式 rtc.stun | 支持（硬编码 STUN 地址） | **缺失** |
| forward 模式 rtc.type 覆盖 | 支持 | **缺失** |
| replace + randomIp → rtc.value 覆盖 | 支持 | **缺失** |
| 随机内网 IP 生成 | 在 paramsMergeStep 中处理 | 在 params-processing.ts 中**缺失**此分支 |

SDK 侧需补充：
1. `buildLaunchParams` 中增加 forward 模式的 `rtc.stun` 和 `rtc.type` 覆盖
2. `buildLaunchParams` 中增加 replace + randomIp 的 `rtc.value` 覆盖
3. `processEnvParams` 中增加随机内网 IP 生成逻辑（或由调用方在传入前处理）

---

## 八、源码参考

| 文件 | 关键位置 | 职责 |
|------|---------|------|
| `src/views/environment/components/envEditExtendForm.vue` | L155~211, L396~422, L474~497 | UI 交互、子模式切换、输入校验 |
| `src/hook/useEnvEdit.ts` | L273~336, L503~513 | 默认值定义、一级选项列表 |
| `src/types/entry/env.d.ts` | L1283~1289, L1362~1368 | 类型定义、ConfigItemType 枚举 |
| `src/core/envOpener/steps.ts` → `paramsMergeStep` | L628~723 | 预处理：清理/代理同步/forward/随机IP |
| `src/common/utils/envUtil.ts` → `generateWiFiIP` | L873~916 | 随机内网 IP 生成算法 |
| `electron/main.ts` → `runEnvCallback` | L3495, L3561~3575 | 桌面端 launchParam 生成 |
| `packages/core/src/logic/launch-config-builder.ts` | L33, L87~89 | SDK 侧 launchParam 生成 |
| `packages/core/src/logic/params-processing.ts` | L219~226 | SDK 侧预处理（仅代理同步） |
