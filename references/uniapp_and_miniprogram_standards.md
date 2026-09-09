# Uni-app 与小程序跨端开发工程规范 (Uni-app & Mini-program Standards)

## 1. 2MB 主包体积红线与分包架构 (Subpackaging)

### 1.1 分包设计原则
微信等主流小程序平台对主包体积有严苛的 **2MB 上限限制**。中大型项目必须执行**分包隔离架构**：
1. **主包仅容纳首屏与全局底座**：
   * TabBar 核心 3~5 个顶层页面（如首页、分类、购物车、个人中心）；
   * 登录/注册授权入口页；
   * 全局公共组件与公共静态资源（主包引用的静态图片建议经过 TinyPNG 极限压缩或上云存储 CDN）。
2. **所有纵深业务强行打入分包**：
   * 订单域全部页面（提单、支付结果、物流跟踪、退款）归入 `pagesOrder` 分包；
   * 会员域复杂页面（收货地址管理、实名认证、安全设置）归入 `pagesMember` 分包；
   * 营销域（秒杀、拼团、抽奖大转盘）归入独立营销分包。

### 1.2 `pages.json` 分包与预加载配置规范
```json
{
  "pages": [
    { "path": "pages/index/index", "style": { "navigationBarTitleText": "首页" } },
    { "path": "pages/my/my", "style": { "navigationBarTitleText": "我的" } }
  ],
  "subPackages": [
    {
      "root": "pagesOrder",
      "pages": [
        { "path": "create/create", "style": { "navigationBarTitleText": "确认订单" } },
        { "path": "detail/detail", "style": { "navigationBarTitleText": "订单详情" } }
      ]
    },
    {
      "root": "pagesMember",
      "pages": [
        { "path": "address/address", "style": { "navigationBarTitleText": "收货地址" } }
      ]
    }
  ],
  "preloadRule": {
    "pages/index/index": {
      "network": "all",
      "packages": ["pagesOrder"]
    }
  }
}
```

---

## 2. 跨端统一请求拦截器 (`uni.addInterceptor`)

```typescript
// src/utils/http.ts
import { useMemberStore } from '@/stores'

const baseURL = import.meta.env.VITE_APP_BASE_API || 'https://api.example.com'

// 1. 双轨请求拦截配置 (拦截 request 与 uploadFile)
const httpInterceptor: UniApp.RequestInterceptor = {
  invoke(options: UniApp.RequestOptions) {
    // 1.1 动态补全相对地址
    if (!options.url.startsWith('http')) {
      options.url = baseURL + options.url
    }
    // 1.2 超时保护 (默认 10s)
    options.timeout = 10000

    // 1.3 注入客户端跨端平台标识
    options.header = {
      ...options.header,
      'source-client': 'miniapp',
    }

    // 1.4 注入鉴权 Token
    const memberStore = useMemberStore()
    const token = memberStore.token
    if (token) {
      options.header.Authorization = `Bearer ${token}`
    }
  },
}

uni.addInterceptor('request', httpInterceptor)
uni.addInterceptor('uploadFile', httpInterceptor)

// 2. 统一 Promise 封装
interface ApiResponse<T> {
  code: number
  msg: string
  data: T
}

export const http = <T>(options: UniApp.RequestOptions): Promise<ApiResponse<T>> => {
  return new Promise((resolve, reject) => {
    uni.request({
      ...options,
      success(res) {
        // HTTP 2xx
        if (res.statusCode >= 200 && res.statusCode < 300) {
          resolve(res.data as ApiResponse<T>)
        } else if (res.statusCode === 401) {
          // 401 凭证过期 -> 清空用户信息并重定向到登录
          const memberStore = useMemberStore()
          memberStore.clearProfile()
          uni.navigateTo({ url: '/pages/login/login' })
          reject(res)
        } else {
          uni.showToast({
            icon: 'none',
            title: (res.data as any)?.msg || '请求发生错误',
          })
          reject(res)
        }
      },
      fail(err) {
        uni.showToast({
          icon: 'none',
          title: '网络连接异常，请检查网络设置',
        })
        reject(err)
      },
    })
  })
}
```

---

## 3. 跨端持久化存储适配器模式

在小程序中，浏览器原生的 `localStorage` 不存在，必须将 `pinia-plugin-persistedstate` 桥接至 `uni.getStorageSync` 与 `uni.setStorageSync`：

```typescript
// src/stores/modules/member.ts
import { defineStore } from 'pinia'
import { ref } from 'vue'

export const useMemberStore = defineStore(
  'member',
  () => {
    const token = ref<string>('')
    const setToken = (val: string) => { token.value = val }
    const clearProfile = () => { token.value = '' }
    return { token, setToken, clearProfile }
  },
  {
    persist: {
      storage: {
        getItem: (key: string) => uni.getStorageSync(key),
        setItem: (key: string, value: any) => uni.setStorageSync(key, value),
      },
    },
  },
)
```

---

## 4. Easycom 与组件规范

在 `pages.json` 中配置 `easycom` 规范，实现组件无需在每个页面中显式 `import` 即可直接使用：
```json
{
  "easycom": {
    "autoscan": true,
    "custom": {
      "^uni-(.*)": "@dcloudio/uni-ui/lib/uni-$1/uni-$1.vue",
      "^App(.*)": "@/components/App$1.vue"
    }
  }
}
```

---

## 5. 条件编译与多端生命周期陷阱

1. **条件编译严谨性**：
   * 平台特有代码必须使用 `// #ifdef MP-WEIXIN` 或 `// #ifdef H5` 包裹；
   * 严禁无节制泛滥条件编译，跨端差异过大时应封装为独立的文件级平台模块（如 `payment.h5.ts` 与 `payment.mp.ts`）。
2. **生命周期宿主归属**：
   * 页面特有生命周期（`onLoad`, `onShow`, `onPullDownRefresh`, `onReachBottom`）**只在页面级 SFC 中有效**；
   * 普通自定义子组件内部无法监听这些钩子，子组件必须使用标准 Vue 3 钩子（`onMounted`, `onUpdated`, `onUnmounted`）。