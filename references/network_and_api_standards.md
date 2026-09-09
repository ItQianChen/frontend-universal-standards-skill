# 前端网络请求与 API 契约工业级规范 (Network & API Standards)

## 1. 核心架构与安全纪律 (Non-negotiable Rules)

1. **【铁律一：纯函数 API 原则】**
   * API 模块函数必须是**纯无副作用的数据 I/O 函数**；
   * **绝对禁止**在 API 接口函数内部直接唤起 UI 提示（如 `ElMessage`、`ElNotification`、`Toast`）；
   * **绝对禁止**在 API 接口函数内部直接执行本地持久化操作（如 `sessionStorage.setItem('token', ...)`）；
   * 接口层只负责参数校验、发送网络请求、返回数据或抛出错误，UI 反馈与状态变更必须由调用方（Store 或 Component）处理。
2. **【铁律二：严禁使用 GET 传递鉴权与敏感凭据】**
   * 登录、修改密码、支付确认等任何涉及安全凭据的接口，必须使用 **POST 请求** 且数据封装在 Request Body 中；
   * 严禁将密码、Token 或敏感个人身份信息（PII）作为 GET 请求的 Query 参数暴露在 URL 中（防止代理日志、浏览器历史、CDN 日志泄露）。
3. **【铁律三：严禁直接篡改入参对象】**
   * 严禁在调用或拦截过程中直接修改调用方传递的对象属性（如 `params.password = md5(params.password)`）；
   * 必须使用浅拷贝或深拷贝副本后提交变异数据。
4. **【铁律四：禁止硬编码环境地址】**
   * 基础路径必须由环境变量驱动（如 `import.meta.env.VITE_APP_BASE_API` 或 `process.env.VUE_APP_BASE_API`），禁止在源码中写入 `localhost:8080`、内网 IP 等固定地址。

---

## 2. 工业级 Axios 封装标准模板 (含无感刷新与错误字典)

```typescript
// src/api/http.ts
import axios, {
  type AxiosInstance,
  type AxiosRequestConfig,
  type InternalAxiosRequestConfig,
  type AxiosResponse,
} from 'axios'
import { ElMessage } from 'element-plus'
import { getToken, getRefreshToken, setToken, removeToken } from '@/utils/token'

// 1. 动态加载环境基础路径并提供安全降级
const baseURL = import.meta.env.VITE_APP_BASE_API || '/api'

export const http: AxiosInstance = axios.create({
  baseURL,
  timeout: 10000,
  headers: {
    'Content-Type': 'application/json;charset=utf-8',
  },
})

// 2. 业务与 HTTP 错误码字典映射
const ERROR_STATUS_MAP: Record<number | string, string> = {
  400: '请求参数错误或格式不合法',
  401: '认证失败或登录状态已过期，请重新登录',
  403: '拒绝访问：您没有操作该资源的权限',
  404: '请求的服务器资源不存在',
  408: '网络请求超时，请稍后重试',
  500: '服务器内部错误，请联系系统管理员',
  502: '网关错误或服务正在重启',
  503: '服务不可用，服务器当前无法处理请求',
  504: '网关超时',
  default: '系统未知异常，请检查网络连接',
}

// 3. 无感刷新锁与挂起队列机制 (Silent Token Refresh)
let isRefreshing = false
let requestsQueue: Array<(token: string) => void> = []

// 4. 请求拦截器
http.interceptors.request.use(
  (config: InternalAxiosRequestConfig) => {
    // 自动注入 Token
    const token = getToken()
    if (token && config.headers) {
      config.headers.Authorization = `Bearer ${token}`
    }
    return config
  },
  (error) => {
    return Promise.reject(error)
  },
)

// 5. 响应拦截器
http.interceptors.response.use(
  (response: AxiosResponse) => {
    // 核心载荷解包
    const res = response.data

    // 针对后端的业务 code 规范处理
    if (res && typeof res.code === 'number' && res.code !== 200) {
      const errorMsg = res.message || res.msg || ERROR_STATUS_MAP[res.code] || ERROR_STATUS_MAP.default
      ElMessage.error(errorMsg)
      return Promise.reject(new Error(errorMsg))
    }

    return res
  },
  async (error) => {
    const originalRequest = error.config

    // 处理 401 鉴权失效
    if (error.response?.status === 401 && !originalRequest._retry) {
      const refreshToken = getRefreshToken()

      // 若具备 Refresh Token 则触发无感静默换票
      if (refreshToken) {
        if (!isRefreshing) {
          isRefreshing = true
          originalRequest._retry = true

          try {
            // 调用换票接口 (需使用纯原生 axios 避免死循环拦截)
            const { data } = await axios.post(`${baseURL}/auth/refresh-token`, { refreshToken })
            const newToken = data.data.token
            setToken(newToken)

            // 执行队列中挂起的重放请求
            requestsQueue.forEach((callback) => callback(newToken))
            requestsQueue = []

            originalRequest.headers.Authorization = `Bearer ${newToken}`
            return http(originalRequest)
          } catch (refreshErr) {
            // Refresh Token 亦失效，彻底登出
            requestsQueue = []
            removeToken()
            if (typeof window !== 'undefined') {
              window.dispatchEvent(new CustomEvent('auth:unauthorized'))
            }
            return Promise.reject(refreshErr)
          } finally {
            isRefreshing = false
          }
        } else {
          // 当前已有换票请求在进行，将后续 401 请求加入等待队列
          return new Promise((resolve) => {
            requestsQueue.push((newToken: string) => {
              originalRequest.headers.Authorization = `Bearer ${newToken}`
              resolve(http(originalRequest))
            })
          })
        }
      } else {
        // 无 Refresh Token 直接广播注销
        removeToken()
        if (typeof window !== 'undefined') {
          window.dispatchEvent(new CustomEvent('auth:unauthorized'))
        }
      }
    }

    // 常规 HTTP 错误提示
    let message = ERROR_STATUS_MAP.default
    if (error.response) {
      const status = error.response.status
      message = error.response.data?.message || ERROR_STATUS_MAP[status] || ERROR_STATUS_MAP.default
    } else if (error.message?.includes('timeout')) {
      message = ERROR_STATUS_MAP[408]
    } else if (typeof window !== 'undefined' && !window.navigator.onLine) {
      message = '网络已断开，请检查您的网络连接'
    }

    ElMessage.error(message)
    return Promise.reject(error)
  },
)

export default http
```

---

## 3. 泛型请求门面与领域模块化规范

### 3.1 泛型 Request 门面函数
```typescript
// src/api/request.ts
import http from './http'
import type { AxiosRequestConfig } from 'axios'

export interface ApiResponse<T = any> {
  code: number
  message: string
  data: T
}

export async function request<TResponse = unknown, TData = unknown>(
  config: AxiosRequestConfig<TData>,
): Promise<TResponse> {
  const response = await http.request<TResponse>({
    ...config,
  })
  return response as unknown as TResponse
}
```

### 3.2 业务领域模块声明标准 (`src/api/user/`)
```typescript
// src/api/user/type.ts
export interface LoginRequestDTO {
  username: string
  password: string
}

export interface UserInfoVO {
  userId: string
  username: string
  nickname: string
  avatar: string
  roles: string[]
}

export interface LoginResponseVO {
  token: string
  refreshToken: string
  user: UserInfoVO
}

// src/api/user/index.ts
import { request } from '@/api/request'
import type { LoginRequestDTO, LoginResponseVO, UserInfoVO } from './type'

enum UserApi {
  Login = '/auth/login',
  UserInfo = '/auth/user-info',
  Logout = '/auth/logout',
}

export const reqLogin = (data: LoginRequestDTO) => {
  return request<LoginResponseVO, LoginRequestDTO>({
    url: UserApi.Login,
    method: 'POST',
    data,
  })
}

export const reqUserInfo = () => {
  return request<UserInfoVO>({
    url: UserApi.UserInfo,
    method: 'GET',
  })
}
```