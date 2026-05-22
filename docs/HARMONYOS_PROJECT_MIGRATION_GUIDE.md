# HarmonyOS 开源项目拉取后编译修复指南

> **目标受众**：AI 辅助开发工具 / 开发者
> **适用场景**：从远程仓库拉取 HarmonyOS / OpenHarmony 项目到本地后，首次编译构建报错
> **项目参考**：`nanoclaw-on-openharmony`（SDK 6.0.1 / API 21，Stage 模型）

---

## 一、排查流程总览

当拉取开源项目后编译报错，按以下优先级逐项检查：

```
┌──────────────────────────────────────────────────────┐
│ ① 设备类型一致性检查（最常见）                         │
│    → build-profile.json5 targets vs module.json5      │
├──────────────────────────────────────────────────────┤
│ ② 签名配置检查                                        │
│    → 证书路径、IDE自动签名                            │
├──────────────────────────────────────────────────────┤
│ ③ SDK 版本兼容性检查                                  │
│    → compileSdkVersion / compatibleSdkVersion         │
├──────────────────────────────────────────────────────┤
│ ④ 依赖完整性检查                                      │
│    → oh_modules、oh-package.json5                    │
├──────────────────────────────────────────────────────┤
│ ⑤ hvigorfile.ts 特殊逻辑检查                          │
│    → hook、自定义插件                                 │
├──────────────────────────────────────────────────────┤
│ ⑥ 其他配置文件检查                                    │
│    → app.json5 / main_pages.json / 资源文件           │
└──────────────────────────────────────────────────────┘
```

---

## 二、检查项详解

### 检查项 ①：设备类型一致性（本次实际遇到的根因）

#### 问题描述

错误信息：`00303214 Configuration Error: The type of target device does not match the device type configured by module: entry`

高版本 SDK（≥5.0.5 / API 17）对 `module.json5` 中的 `deviceTypes` 与 `build-profile.json5` 中 `targets` 的 `deviceType` 做严格一致性校验。

#### 检查步骤

**Step 1：读取 `module.json5`**

路径：`entry/src/main/module.json5`

```json5
// 重点关注 deviceTypes 字段
{
  "module": {
    "name": "entry",
    "type": "entry",
    "deviceTypes": ["phone"], // ← 这里定义了什么设备类型？
    // ...
  }
}
```

**Step 2：读取模块级 `build-profile.json5`**

路径：`entry/build-profile.json5`

```json5
// 重点关注 targets 数组
{
  "targets": [
    {
      "name": "default",
      // ← 检查是否有 config.deviceType？
      // ← 检查是否有 runtimeOS？
    },
    {
      "name": "ohosTest",
      // ← 同上
    }
  ]
}
```

#### 判断逻辑

| 场景 | module.json5 deviceTypes | targets 的 deviceType | 结果 |
|------|--------------------------|----------------------|------|
| ✅ 正常 | `["phone"]` | 未定义（默认继承 module） | 旧版 SDK 可编译 |
| ✅ 正常 | `["phone"]` | `config.deviceType: ["phone"]` | 所有版本可编译 |
| ❌ 报错 00303214 | `["phone"]` | 未定义 | **新版 SDK 严格校验时** |
| ❌ 报错 | `["phone"]` | `config.deviceType: ["tablet"]` | 类型不匹配 |

#### 修复方式

**❌ 错误写法**（会导致 Schema 校验失败 `00303038`）：

```json5
// 不要直接把 deviceType 放在 target 根级别！
"targets": [
  {
    "name": "default",
    "deviceType": "phone"  // ← Schema 不支持此位置
  }
]
```

**✅ 正确写法**：

```json5
"targets": [
  {
    "name": "default",
    "runtimeOS": "HarmonyOS",
    "config": {
      "deviceType": [
        "phone"    // ← 必须放在 config 对象内，且值为数组
      ]
    }
  },
  {
    "name": "ohosTest",
    "runtimeOS": "HarmonyOS",
    "config": {
      "deviceType": [
        "phone"
      ]
    }
  }
]
```

**关键规则**：
- `deviceType` 必须嵌套在 `config` 对象中
- `deviceType` 的值必须是**字符串数组**，不是单个字符串
- `runtimeOS` 在 targets 根级别，取值 `"HarmonyOS"` 或 `"OpenHarmony"`
- targets 中声明的 deviceType 必须是 module.json5 的 `deviceTypes` 的子集

---

### 检查项 ②：签名配置

#### 问题描述

开源项目的 `build-profile.json5` 中签名证书通常使用**原作者机器上的绝对路径**（如 `C:\Users\原作者\.ohos\config\...`），拉取后这些路径不存在，导致签名失败。

#### 检查步骤

读取工程级 `build-profile.json5`：

```json5
{
  "app": {
    "signingConfigs": [
      {
        "name": "default",
        "type": "HarmonyOS",
        "material": {
          "certpath": "C:\\Users\\s\\.ohos\\config\\...",    // ← 检查路径是否存在
          "storeFile": "C:\\Users\\s\\.ohos\\config\\...",   // ← 同上
          "storePassword": "...",
          "keyPassword": "...",
          "profile": "C:\\Users\\s\\.ohos\\config\\...",
          "signAlg": "SHA256withECDSA",
          "keyAlias": "debugKey"
        }
      }
    ],
    "products": [
      {
        "name": "default",
        "signingConfig": "default"   // ← 确认引用的签名方案名称存在
      }
    ]
  }
}
```

同时检查 `.hvigor/filecache/build-profile.json5`，DevEco Studio 的自动签名可能已经更新了缓存中的路径。

#### 修复方式

1. **推荐**：在 DevEco Studio 中通过 `File → Project Structure → Signing Configs` 勾选 `Automatically generate signature`，IDE 会自动更新签名
2. **手动**：如果无法使用自动签名，需要在 AppGallery Connect 申请调试证书并手动配置签名信息

---

### 检查项 ③：SDK 版本兼容性

#### 问题描述

项目配置的 `compileSdkVersion` / `compatibleSdkVersion` 可能高于本机安装的 SDK 版本。

#### 检查步骤

读取工程级 `build-profile.json5` 中 `products` 的 SDK 版本：

```json5
{
  "app": {
    "products": [
      {
        "name": "default",
        "targetSdkVersion": "6.0.1(21)",      // ← 目标 SDK
        "compatibleSdkVersion": "6.0.0(20)",   // ← 最低兼容 SDK
        "runtimeOS": "HarmonyOS"               // ← 运行环境
      }
    ]
  }
}
```

#### 判断逻辑

- `targetSdkVersion` ≤ 本机安装的 SDK 版本 → ✅
- `targetSdkVersion` > 本机安装的 SDK 版本 → ❌ 需降级或升级 SDK
- `runtimeOS: "HarmonyOS"` 表示该产物仅用于 HarmonyOS 设备
- `runtimeOS: "OpenHarmony"` 表示用于 OpenHarmony 设备

#### 修复方式

- 降级：将 `targetSdkVersion` 改为本机支持的最高版本
- 升级：通过 DevEco Studio → SDK Manager 安装对应的 SDK 版本

---

### 检查项 ④：依赖完整性

#### 检查步骤

读取模块级 `oh-package.json5`：

```json5
// entry/oh-package.json5
{
  "name": "entry",
  "version": "1.0.0",
  "dependencies": {}     // ← 检查是否有未安装的依赖
}
```

#### 常见问题

1. **`oh_modules` 目录缺失或不完整**：执行 `File → Sync and Refresh Project` 或在终端执行 `ohpm install`
2. **远程依赖版本不存在**：版本号前有 `^` 前缀的依赖可能拉取到不兼容的最新版，建议去掉 `^` 锁定版本
3. **本地 HAR/HSP 依赖路径错误**：检查相对路径是否正确

#### 修复方式

```bash
# 在项目根目录执行
ohpm install
```

或删除 `oh_modules` 和 `oh-package-lock.json5` 后重新同步。

---

### 检查项 ⑤：hvigorfile.ts 特殊逻辑

#### 检查步骤

读取工程级和模块级的 `hvigorfile.ts`：

**工程级**（`/hvigorfile.ts`）：
```ts
import { appTasks } from '@ohos/hvigor-ohos-plugin';

export default {
  system: appTasks,
  plugins: []  // ← 检查是否有自定义插件
}
```

**模块级**（`/entry/hvigorfile.ts`）：
```ts
import { hapTasks } from '@ohos/hvigor-ohos-plugin';

export default {
  system: hapTasks,
  plugins: []  // ← 同上
}
```

#### 需要关注的情况

- 如果有 `afterNodeEvaluate` hook，可能动态修改了 `module.json5`、`build-profile.json5` 等配置
- 如果有自定义插件，可能与当前 SDK 版本不兼容
- 检查是否有 `getNode(__filename)` 等动态配置逻辑

---

### 检查项 ⑥：其他配置文件

#### app.json5

路径：`AppScope/app.json5`

```json5
{
  "app": {
    "bundleName": "com.example.openclaw_on_openharmony",  // ← 确认包名格式正确
    "vendor": "example",
    "versionCode": 1000000,
    "versionName": "1.0.0",
    "icon": "$media:layered_image",
    "label": "$string:app_name"
  }
}
```

- `bundleName` 必须与 `build-profile.json5` 中 `signingConfigs` 签名的 bundleName 一致
- 真机调试时 `bundleName` 必须在 AppGallery Connect 中注册

#### main_pages.json

路径：`entry/src/main/resources/base/profile/main_pages.json`

```json5
{
  "src": [
    "pages/Index",
    "pages/SettingsPage"
  ]
}
```

- 新增页面后必须在此注册，否则页面路由无法找到

#### local.properties（可选）

路径：项目根目录 `local.properties`

```
nodejs.dir=C:/Users/xxx/AppData/Local/Huawei/DevEcoStudio...
hwsdk.dir=C:/Users/xxx/AppData/Local/Huawei/DevEcoStudio/sdk
```

- 如果 IDE 提示 SDK 路径错误，检查此文件或通过 IDE 设置重新指定 SDK 路径

---

## 三、快速修复脚本（AI 执行流程）

当遇到 HarmonyOS 项目编译错误时，AI 应按以下顺序执行：

```
1. 读取编译报错信息，提取 error code 和 message
2. 根据错误码定位问题类型：

   ┌──────────────────────────────────────────────────┐
   │ 00303214 → 检查项①：设备类型一致性              │
   │ 00303038 → Schema 校验失败，检查 JSON5 格式和字段│
   │ 签名相关 → 检查项②：签名配置                    │
   │ SDK 相关 → 检查项③：SDK 版本兼容性              │
   │ 依赖相关 → 检查项④：依赖完整性                  │
   └──────────────────────────────────────────────────┘

3. 读取对应的配置文件，定位差异
4. 应用修复
5. 重新编译验证
```

---

## 四、本次修复实录

### 项目信息

| 项目属性 | 值 |
|----------|-----|
| 项目名 | nanoclaw-on-openharmony |
| SDK 版本 | targetSdkVersion: 6.0.1(21), compatibleSdkVersion: 6.0.0(20) |
| 运行环境 | HarmonyOS |
| 模型 | Stage |
| 模块 | entry（单模块） |
| 设备类型 | phone |

### 原始错误

```
00303214 Configuration Error: The type of target device does not match the device type configured by module: entry.
```

### 问题分析

| 文件 | 字段 | 值 | 状态 |
|------|------|-----|------|
| `module.json5` | `deviceTypes` | `["phone"]` | ✅ 已配置 |
| `entry/build-profile.json5` | `targets[0].config.deviceType` | 未定义 | ❌ **缺失** |
| `entry/build-profile.json5` | `targets[1].config.deviceType` | 未定义 | ❌ **缺失** |
| `entry/build-profile.json5` | `targets[].runtimeOS` | 未定义 | ❌ **缺失** |

### 修复内容

文件：`entry/build-profile.json5`

```diff
   "targets": [
     {
-      "name": "default"
+      "name": "default",
+      "runtimeOS": "HarmonyOS",
+      "config": {
+        "deviceType": [
+          "phone"
+        ]
+      }
     },
     {
-      "name": "ohosTest",
+      "name": "ohosTest",
+      "runtimeOS": "HarmonyOS",
+      "config": {
+        "deviceType": [
+          "phone"
+        ]
+      }
     }
   ]
```

### 关键教训

1. **第一次尝试失败**：直接将 `"deviceType": "phone"` 放在 target 根级别 → `00303038 Schema validate failed`
2. **正确的 schema**：`deviceType` 必须在 `config` 对象内，且值为**字符串数组** `["phone"]`
3. `runtimeOS` 字段声明了该 target 的运行目标环境（HarmonyOS / OpenHarmony），新版 SDK 要求显式声明

---

## 五、附录：关键配置文件速查表

| 文件路径 | 用途 | 关键字段 |
|---------|------|---------|
| `build-profile.json5`（根） | 工程级构建配置 | `app.products[].targetSdkVersion`、`app.signingConfigs`、`modules[].targets` |
| `entry/build-profile.json5` | 模块级构建配置 | `targets[].config.deviceType`、`targets[].runtimeOS` |
| `entry/src/main/module.json5` | 模块配置 | `deviceTypes`、`abilities`、`requestPermissions` |
| `AppScope/app.json5` | 应用全局配置 | `bundleName`、`versionCode`、`versionName` |
| `entry/oh-package.json5` | 模块依赖配置 | `dependencies`、`devDependencies` |
| `hvigorfile.ts`（根/模块） | 构建脚本 | 自定义 hook 和插件 |
| `main_pages.json` | 页面路由注册 | `src[]` 页面路径列表 |

---

> **版本记录**：v1.0 — 基于 `nanoclaw-on-openharmony` 项目修复实践生成
