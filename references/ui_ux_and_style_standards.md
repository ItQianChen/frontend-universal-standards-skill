# 前端 UI/UX、样式方案与多端适配规范 (UI/UX & Style Standards)

## 1. 样式方案分型与选型矩阵

1. **企业级中后台 (Element Plus / Ant Design)**：
   * 采用 **SCSS 模块化**，统一在 `src/styles/variable.scss` 沉淀系统级色彩、字阶与菜单排版常量；
   * 在 `vite.config.ts` 中通过 `additionalData` 注入全局变量，支持深层覆盖组件库主题色。
2. **现代化 Web / SaaS 应用 (React / Next / Vue 3 轻量应用)**：
   * 首选 **Tailwind CSS v4**；
   * 利用原子化类名消除庞大的传统 CSS 文件负担，结合 Design Tokens 配置主色（Primary）与表面色（Surface）。
3. **移动端跨端 (Uni-app / 小程序)**：
   * 采用 SCSS 配合平台原生单位 `rpx`，或集成 `@cool-vue/vite-plugin` 注入 Tailwind CSS。

---

## 2. 样式作用域与深度穿透规范

1. **严格的作用域隔离**：
   * Vue SFC 组件必须声明 `<style scoped>`（或 `<style scoped lang="scss">`）；
   * React 组件推荐使用 CSS Modules（`[name].module.scss`）或 Tailwind 类名，严禁裸写全局类名造成样式污染。
2. **现代深度选择器规范**：
   * 在需要覆盖第三方 UI 库（如 Element Plus、Vant、Antd）内部子节点样式时：
   * **强制使用**：Vue 3 规范的 `:deep(.el-table__row)`；
   * **严禁使用**：已在现代编译管线中废弃并引发编译警告的 `/deep/`、`::v-deep` 裸语法或 `>>>`。

---

## 3. 多端屏幕与响应式适配方案

### 3.1 移动端 H5 动态 rem 方案
* **原理**：`amfe-flexible` 监听屏幕变化并动态修改 `<html>` 的 `font-size`；编译期由 `postcss-pxtorem` 自动将业务代码中的 `px` 转化为 `rem`。
* **`postcss.config.js` 配置规范**：
  ```javascript
  module.exports = {
    plugins: {
      'postcss-pxtorem': {
        rootValue: 37.5, // Vant 基于 375px 设计稿；若为 750px 设计稿则配置为 75
        propList: ['*'], // 匹配所有 CSS 属性
        selectorBlackList: ['.ignore', 'van-circle__layer'], // 忽略特定类名
      },
    },
  }
  ```

### 3.2 小程序 `rpx` 响应式标准
* 以 750rpx 作为全屏基准宽度；
* 自定义导航栏（`"navigationStyle": "custom"`）必须通过 `uni.getSystemInfoSync()` 动态计算 `statusBarHeight` 状态栏高度以及胶囊按钮位置，防止页面内容与顶部信号栏或胶囊发生视觉重叠。

---

## 4. 极致用户体验与性能提升模式

### 4.1 高性能视口交叉懒加载 (`IntersectionObserver`)
对于电商长列表、新闻瀑布流中的海量图片，严禁直接绑定 `src`，必须封装全局指令 `v-img-lazy` 进行视口交叉加载：
```typescript
import { useIntersectionObserver } from '@vueuse/core'
import type { App, DirectiveBinding } from 'vue'

export const lazyPlugin = {
  install(app: App) {
    app.directive('img-lazy', {
      mounted(el: HTMLImageElement, binding: DirectiveBinding<string>) {
        const { stop } = useIntersectionObserver(el, ([{ isIntersecting }]) => {
          if (isIntersecting) {
            el.src = binding.value
            stop() // 触发后立即停止监听，释放浏览器观察器资源
          }
        })
      },
    })
  },
}
```

### 4.2 路由平复滚动 (`scrollBehavior`)
在 Web 路由切换时，必须配置滚动行为复位，避免用户在长列表下翻后进入详情页依然停留在下半屏幕的糟糕体验：
```typescript
scrollBehavior() {
  return { top: 0 }
}
```

### 4.3 骨架屏与三态闭环 (Empty / Loading / Error)
* 数据列表必须具备完整的状态闭环：
  1. **加载中状态（Loading）**：优先使用与真实 UI 排版契合的骨架屏（Skeleton），避免单调的居中 Spinner；
  2. **空状态（Empty）**：当列表长度为 0 时，展示富有业务针对性的插画与引导动作按钮；
  3. **错误状态（Error）**：网络异常或服务熔断时提供轻量重试（Retry）按钮。
