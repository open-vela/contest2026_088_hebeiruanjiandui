---
name: vela-quickapp-dev
description: Vela 快应用（Quick App）开发调试技能——环境搭建、构建命令、manifest.json 配置、@system 传感器/震动 API 用法与常见坑。当用户在小米 Vela / AIoT IDE 上开发、构建、调试手表快应用，或遇到传感器订阅无数据、list-item 渲染异常、API undefined 等问题时使用。
---

# Vela 快应用开发调试

面向小米 Vela（aiot-toolkit 工具链）手表快应用的开发经验，来源于 2026 openvela AI 大赛作品的实际开发过程。

## 1. 环境与构建

- **Node.js ≥ 16**（实测 22.x 可用）
- 工具链通过项目 `devDependencies` 引入：`aiot-toolkit` + `@aiot-toolkit/jsc`，`npm install` 即装
- 推荐 **AIoT IDE**（VS Code 内核 + `vela.aiot-core` 扩展），用户数据在 `AppData/Roaming/AIoT IDE`

```bash
npm run start     # 开发模式：--watch 热更新，自动打开 Vela 模拟器
npm run build     # 构建
npm run release   # 发布构建
```

注意：`aiot build` 清理临时目录（`.temp_项目名`）在部分 Windows 环境需提权运行，报错时先检查临时目录残留。

## 2. manifest.json 要点

- **用到的每个 system API 必须在 `features` 数组声明**，漏声明时运行时 `import` 得到 undefined：

```json
"features": [
  { "name": "system.router" },
  { "name": "system.sensor" },
  { "name": "system.vibrator" }
]
```

- `router.pages` 是**对象映射**（不是数组），`router.entry` 指定入口页
- 国际化文案放 `src/i18n/zh-CN.json` 等，按语言文件自动匹配

## 3. 传感器 API（@system.sensor）

```js
import sensor from '@system.sensor'

sensor.subscribeAccelerometer({
  interval: 'normal',
  callback: function (ret) {  // ret.x / ret.y / ret.z
    // 处理加速度数据
  },
  fail: function (msg, code) {
    // 订阅失败（如模拟器无传感器）
  }
})
// 用完必须反订阅，否则后台持续耗电
sensor.unsubscribeAccelerometer()
```

**模拟器降级策略**：模拟器可能没有真实加速度计。两种情况都要处理：

1. `fail` 回调被触发 → 切换模拟数据源
2. 订阅"成功"但长时间无回调 → 用定时器检测 N 秒内是否有数据，没有则降级

建议在界面显示 `[模拟传感器]` 标签，让演示状态可辨识。

## 4. 震动 API（@system.vibrator）

```js
import vibrator from '@system.vibrator'

try { vibrator.vibrate({ mode: 'short' }) } catch (e) {}
// mode: 'short' | 'long'
```

务必 try/catch 包裹——模拟器上可能不支持。连续强提醒可以 `long` + setTimeout 间隔再震一次。

## 5. 常见坑（实战踩过）

1. **`list-item` 必须带 `type` 属性**，否则列表渲染异常
2. 页面间传值用 `this.$app.$def.globalData`（app.ux 中定义）
3. 路由：`router.push({ uri: 'pages/detail', params: {...})` / `router.back()`
4. 回调里的 `this` 不指向页面实例，先 `var self = this` 再在闭包里用 `self`
5. 数据驱动的条件渲染用 `if="{{cond}}"`，注意布尔值变更要整体替换才会触发更新
