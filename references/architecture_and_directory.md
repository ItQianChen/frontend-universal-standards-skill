# 前端工程架构与目录分层规范 (Architecture & Directory Standards)

## 1. 统一工程目录骨架

无论是 Vue 3、React 还是跨端 Uni-app，团队所有前端工程必须遵循**领域驱动 + 职责分离**的分层规范。

### 1.1 Vue 3 + Vite 标准工程目录
```text
src/
├── api/                    # 统一接口层 (按领域拆分子目录，如 user/, order/)
│   ├── [domain]/           # 具体领域
│   │   ├── index.ts        # 请求函数 (export const reqLogin = ...)
│   │   └── type.ts         # RequestDTO 与 ResponseVO 强类型声明
│   ├── http.ts             # Axios 底层实例配置与拦截器链
│   └── request.ts          # 统一泛型请求门面
├── assets/                 # 静态资源 (图片、SVG图标、字体)
│   └── icons/              # 本地 SVG 图标库 (供 vite-plugin-svg-icons 采集)
├── components/             # 全局可复用公共组件 (无路由绑定、高内聚)
│   ├── SvgIcon/            # SVG 雪碧图渲染器
│   └── index.ts            # 全局组件自动注册插件 (app.use(components))
├── composables/            # 业务组合式逻辑 (useCountDown, useTablePagination 等)
├── directives/             # 全局自定义指令 (v-img-lazy, v-permission)
├── layout/                 # 核心后台骨架 (Header, Sidebar, Tabbar, Main, Breadcrumb)
├── router/                 # 路由配置
│   ├── routes.ts           # 路由静态/动态分级配置 (constantRoutes, asyncRoutes)
│   ├── permission.ts       # 全局路由前置/后置守卫与鉴权流
│   └── index.ts            # createRouter 实例
├── stores/                 # Pinia 状态管理
│   ├── modules/            # 领域子模块 (user.ts, setting.ts, cart.ts)
│   └── index.ts            # Pinia 根实例与持久化插件挂载
├── styles/                 # 样式体系
│   ├── variable.scss       # 全局 SCSS 变量 (主题色、间距、菜单尺寸)
│   ├── reset.scss          # 样式重置
│   └── index.scss          # 样式汇总入口
├── types/                  # 全局类型定义 (*.d.ts)
├── utils/                  # 纯函数通用工具集 (token.ts, time.ts, validate.ts)
├── views/                  # 业务页面视图 (与路由严格 1:1 映射)
├── App.vue                 # 顶层 Router-View 与全局 Provider 挂载
├── main.ts                 # 应用引导入口
└── setting.ts              # 应用全局基础配置 (Title, Logo)
```

### 1.2 React 18/19 + Vite 标准工程目录
```text
src/
├── api/                    # 领域接口层 (包含 http.ts 基础实例与 types.ts)
├── assets/                 # 静态素材
├── components/             # 通用组件
│   ├── AuthGuard.tsx       # 声明式路由访问守卫
│   ├── Layout.tsx          # 骨架布局
│   └── ...                 # 表格、弹窗等展示组件
├── hooks/                  # 自定义 React Hooks
├── pages/                  # 页面级视图 (通过 lazy 按需加载)
├── router/                 # 路由中心 (基于 createBrowserRouter)
│   └── index.tsx
├── stores/                 # 状态管理 (Zustand 或 Redux Toolkit)
├── utils/                  # 工具库 (Storage, EventBus, Crypto)
├── App.tsx                 # 根组件
├── index.css               # 全局样式 / Tailwind 引入
└── main.tsx                # createRoot 引导入口
```

---

## 2. 路由分级与 RBAC 动态鉴权规范

### 2.1 路由分类解耦三层架构
1. **常量路由（Constant Routes）**：
   * 包含无需鉴权即可访问的基础页面：`/login`（登录）、`/404`（未找到）、`/403`（无权限）。
   * 必须在项目初始化时直接注册到路由器中。
2. **异步权限路由（Async Routes）**：
   * 按功能模块分组的管理路由（如用户管理、商品管理、订单中心）。
   * 包含 `meta: { title: string, icon: string, roles: string[] }` 鉴权元数据。
   * 严禁直接硬编码全量注册，必须由用户登录后返回的角色权限数组（`routes` / `roles`）进行动态过滤。
3. **通配任意路由（Any / Catch-all Route）**：
   * `{ path: '/:pathMatch(.*)*', redirect: '/404', name: 'Any' }`。
   * **铁律**：通配路由必须在动态路由过滤完成并调用 `router.addRoute()` 注入后，作为**最后一条路由**追加；严禁在常量路由中直接注册，否则会导致页面刷新时动态路由尚未注册而直接误判跳转 404。

### 2.2 全局导航守卫状态机 (Vue Router 4)
```typescript
import router from '@/router'
import nprogress from 'nprogress'
import 'nprogress/nprogress.css'
import pinia from '@/stores'
import { useUserStore } from '@/stores/modules/user'

const userStore = useUserStore(pinia)
const whiteList = ['/login', '/register', '/404']

router.beforeEach(async (to, from, next) => {
  nprogress.start()
  const token = userStore.token

  if (token) {
    if (to.path === '/login') {
      next({ path: '/' })
    } else {
      // 检查是否已有用户信息和动态权限路由
      if (userStore.username) {
        next()
      } else {
        try {
          // 异步获取用户权限清单并动态计算加载路由
          await userStore.getUserInfo()
          // 动态路由添加后，必须通过 replace 重新触发当前跳转
          next({ ...to, replace: true })
        } catch (error) {
          // Token 过期或失效，清空状态并跳登录
          await userStore.userLogout()
          next({ path: '/login', query: { redirect: to.fullPath } })
        }
      }
    }
  } else {
    if (whiteList.includes(to.path)) {
      next()
    } else {
      next({ path: '/login', query: { redirect: to.fullPath } })
    }
  }
})

router.afterEach((to) => {
  if (to.meta?.title) {
    document.title = `${to.meta.title} - 管理系统`
  }
  nprogress.done()
})
```

---

## 3. 状态管理架构模式

### 3.1 选型准则
1. **Vue 3 工程**：强制使用 **Pinia**，严禁新建 Vuex 工程。
2. **React 工程**：
   * 中小型或高内聚 Web 应用首选 **Zustand**（无 Provider 包裹，体积极小，心智负担低）；
   * 复杂企业级系统选用 **Redux Toolkit (RTK)**（严格规范 Action 与 DevTools 追踪）。
3. **移动端/小程序**：强制对 Store 注入跨端 Storage 适配器。

### 3.2 State 最小化与衍生计算原则
* 状态树必须保持最小正交性，**严禁在 State 中存储可通过已有状态推导的冗余字段**。
* 计算范例：
  * **错误做法**：在 State 中同时维护 `cartList`、`totalCount`、`totalPrice`，每次修改 `cartList` 时手动累加计算后赋值；容易因漏改导致金额不同步。
  * **标准做法**：在 State 中仅维护核心原始数组 `cartList`，总件数和总金额全部通过 Pinia 的 `getters`（或 Vue 3 `computed`）动态派生：
    ```typescript
    export const useCartStore = defineStore('cart', () => {
      const cartList = ref<CartItem[]>([])
      const totalCount = computed(() => cartList.value.reduce((sum, item) => sum + item.count, 0))
      const totalPrice = computed(() => cartList.value.reduce((sum, item) => sum + item.count * item.price, 0))
      return { cartList, totalCount, totalPrice }
    })
    ```