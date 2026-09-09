# 前端编码规范与组件设计准则 (Coding & Component Standards)

## 1. TypeScript 强类型约束规范

1. **类型零 `any` 铁律**：
   * 所有新工程必须开启 `strict: true`；
   * 严禁在接口出入参、Store 关键状态、组件 Props 中滥用 `any`；
   * 对于结构未知的动态数据，使用 `unknown` 并配合类型守卫（Type Narrowing）或类型断言函数处理。
2. **接口与实体分离原则**：
   * 请求参数使用 `*DTO`（Data Transfer Object）或 `*Params` 命名；
   * 接口响应体使用 `*VO`（View Object）或 `*Response` 命名；
   * 数据库实体或前端核心领域模型使用 `*Entity` 或直接以领域名词（如 `User`, `Order`, `Product`）命名。
3. **组件实例类型导出**：
   * 对于暴露了 `defineExpose` 方法或需要通过 `ref` 获取实例的组件，必须在其声明文件或组件末尾显式导出其实例类型：
     ```typescript
     export type XtxGuessInstance = InstanceType<typeof XtxGuess>
     ```

---

## 2. Vue 3 (Composition API) 最佳范式

### 2.1 单文件组件 `<script setup lang="ts">` 规范
* **纯 Setup 语法**：新组件强制采用 `<script setup lang="ts">`，严禁回退或混用 Options API。
* **声明式 Props 与 Emits（区分版本分水岭）**：
  * **Vue 3.5+ 推荐写法（Reactive Props Destructuring 稳定特性）**：
    Vue 3.5 正式将解构保持响应式（Reactive Props Destructure）转正为内置特性，解构出的变量可直接在模板与计算属性中使用，并直接在解构中赋默认值：
    ```vue
    <script setup lang="ts">
    interface Props {
      title: string
      count?: number
      active?: boolean
    }
    // Vue 3.5+ 原生支持：直接解构赋默认值，且完美保持响应性追踪
    const { title, count = 0, active = false } = defineProps<Props>()

    // 声明事件
    const emit = defineEmits<{
      (e: 'change', value: number): void
      (e: 'close'): void
    }>()
    </script>
    ```
  * **Vue 3.4 及以前维护项目**：若未开启实验性解构，解构 `defineProps` 会丢失响应性，必须通过 `props.xxx` 访问，或通过 `withDefaults` 与 `toRefs` 访问。
* **模板引用新规范 (Vue 3.5+)**：
  * 使用 `useTemplateRef<T>()` 替代以往同名 `const inputEl = ref<HTMLInputElement | null>(null)` 的魔法绑定，杜绝模板 ref 与脚本变量命名的混淆隐患：
    ```typescript
    import { useTemplateRef, onMounted } from 'vue'
    const inputRef = useTemplateRef<HTMLInputElement>('username-input')
    onMounted(() => {
      inputRef.value?.focus()
    })
    ```
* **响应式 API 选型原则**：
  * **优先选择 `ref()`**：统一使用 `ref()` 定义基本类型与对象类型，语法一致性强，支持直接重新赋值（`data.value = newData`）；
  * **谨慎使用 `reactive()`**：严禁直接对 `reactive` 对象进行解构（若非 Vue 3.5 解构会丢失响应性，需配合 `toRefs()`）；严禁对 `reactive` 变量直接覆盖赋值（如 `state = res.data` 会直接破坏 Proxy 代理引用）。

### 2.2 Composables 组合式函数规范
* **命名约定**：以 `use` 开头（如 `useCountDown.ts`, `useTablePagination.ts`）。
* **生命周期必须配对清理**：在 Composable 内部若开启了定时器（`setInterval`）、订阅了全局事件总线（`eventBus`）或浏览器事件（`addEventListener`），**必须在 `onUnmounted` 钩子中执行逆向清理**：
  ```typescript
  import { ref, onUnmounted } from 'vue'

  export function useInterval(callback: () => void, delay: number) {
    const timer = setInterval(callback, delay)
    onUnmounted(() => {
      clearInterval(timer)
    })
  }
  ```

---

## 3. React 18/19 (Hooks) 最佳范式

### 3.1 函数组件与类型定义
* 优先使用标准函数声明：
  ```tsx
  interface ButtonProps {
    variant?: 'primary' | 'secondary' | 'danger'
    disabled?: boolean
    children: React.ReactNode
    onClick?: (e: React.MouseEvent<HTMLButtonElement>) => void
  }

  export default function Button({
    variant = 'primary',
    disabled = false,
    children,
    onClick,
  }: ButtonProps) {
    return (
      <button
        type="button"
        disabled={disabled}
        className={`btn btn-${variant}`}
        onClick={onClick}
      >
        {children}
      </button>
    )
  }
  ```

### 3.2 副作用管理与 Hooks 纪律
1. **依赖数组完整性**：`useEffect`、`useCallback`、`useMemo` 的依赖数组必须包含 effect 内部使用的所有外部变量与函数，禁止为了绕过执行而故意隐瞒依赖。
2. **清理函数（Cleanup Function）**：异步请求在组件卸载时应通过 `AbortController` 取消，防止组件销毁后仍在尝试调用状态变更；长轮询与定时器在 cleanup 回调中注销：
   ```tsx
   useEffect(() => {
     const controller = new AbortController()
     fetchData({ signal: controller.signal })
     return () => controller.abort()
   }, [query])
   ```
3. **类名动态拼接规范**：在处理条件类名时，强制使用 `classnames` 或 `clsx`，禁止使用易引发空值与格式混乱的多段模板字符串拼接。

---

## 4. 前端组件设计与工程契约

1. **容器组件与展示组件分离（Smart & Dumb Components）**：
   * **展示组件（UI / Dumb）**：仅通过 Props 接收数据，通过 Events / Callbacks 发送通知，内部绝不发起网络请求，绝不直接操作全局 Store。
   * **容器组件（Page / Smart）**：负责编排业务流程、调度 API 请求、绑定全局状态。
2. **插槽与组合优于配置膨胀**：
   * 当一个组件的配置参数超过 10 个且包含大量布尔开关时，应使用 Slot（Vue）或 Children / Render Props（React）将定制控制权交还调用方，避免生成超重组件。
3. **单向数据流原则**：
   * 严禁子组件内部通过 `props.data.xxx = 123` 直接修改父级传入的引用对象属性；必须通过事件通知父级修改，或在本地拷贝后通过副本操作。