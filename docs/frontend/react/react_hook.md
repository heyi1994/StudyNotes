# React 自定义 Hook 

## 1. 什么是自定义 Hook

自定义 Hook 是一个以 `use` 开头的 JavaScript 函数，它可以调用其他 Hook。

```js
// 这是一个自定义 Hook
function useWindowWidth() {
  const [width, setWidth] = useState(window.innerWidth);

  useEffect(() => {
    const handleResize = () => setWidth(window.innerWidth);
    window.addEventListener("resize", handleResize);
    return () => window.removeEventListener("resize", handleResize);
  }, []);

  return width;
}
```

自定义 Hook 本质上是**逻辑复用**的机制，它让你把组件逻辑提取到可复用的函数中，而不改变组件树结构（不像 HOC 和 render props）。

**关键特征：**

- 函数名必须以 `use` 开头（这是约定，也让 lint 工具能正确检查规则）
- 内部可以调用其他 Hook（内置的或自定义的）
- 每次调用都有完全独立的 state，Hook 之间不共享状态

---

## 2. 为什么需要自定义 Hook

在 Hook 出现之前，React 有两种方式复用有状态逻辑：**高阶组件（HOC）** 和 **render props**。

### 2.1 HOC 的问题

```jsx
// 嵌套地狱，难以追踪数据来源
export default withRouter(withAuth(withTheme(withLocale(MyComponent))));
```

### 2.2 render props 的问题

```jsx
// JSX 嵌套深，可读性差
<Mouse
  render={(mouse) => (
    <Cat mouse={mouse} render={(cat) => <Cheese cat={cat} />} />
  )}
/>
```

### 2.3 自定义 Hook 的优势

```jsx
// 清晰、扁平、语义化
function MyComponent() {
  const router = useRouter();
  const { user } = useAuth();
  const theme = useTheme();
  const locale = useLocale();
  // ...
}
```

- **更简洁**：不增加组件树层级
- **更清晰**：数据来源一目了然
- **更灵活**：可以在 Hook 之间传递数据

---

## 3. Hook 的基本规则

自定义 Hook 必须遵守和内置 Hook 相同的规则，否则会导致 bug。

### 规则一：只在最顶层调用 Hook

```jsx
//  错误：在条件语句中调用
function useCounter(initialValue) {
  if (initialValue > 0) {
    const [count, setCount] = useState(initialValue); // 违反规则！
  }
}

//  正确：始终在顶层调用
function useCounter(initialValue) {
  const [count, setCount] = useState(initialValue);
  // 在逻辑中使用条件
  const increment = () => {
    if (count < 100) setCount((c) => c + 1);
  };
  return { count, increment };
}
```

**原因**：React 依靠 Hook 的调用顺序来正确管理每个 Hook 对应的状态。条件调用会打乱这个顺序。

### 规则二：只在 React 函数组件或自定义 Hook 中调用

```js
//  错误：在普通函数中调用
function fetchData() {
  const [data, setData] = useState(null); // 违反规则！
}

// 正确：在组件或自定义 Hook 中调用
function useFetch(url) {
  const [data, setData] = useState(null);
  // ...
}
```

---

## 4. 第一个自定义 Hook

让我们从一个简单的计数器 Hook 开始，理解提取过程。

### 重构前：逻辑在组件内

```jsx
function Counter() {
  const [count, setCount] = useState(0);

  const increment = () => setCount((c) => c + 1);
  const decrement = () => setCount((c) => c - 1);
  const reset = () => setCount(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
      <button onClick={increment}>+</button>
    </div>
  );
}
```

### 重构后：提取为自定义 Hook

```jsx
// hooks/useCounter.js
function useCounter(initialValue = 0) {
  const [count, setCount] = useState(initialValue);

  const increment = useCallback(() => setCount((c) => c + 1), []);
  const decrement = useCallback(() => setCount((c) => c - 1), []);
  const reset = useCallback(() => setCount(initialValue), [initialValue]);

  return { count, increment, decrement, reset };
}

// 使用 Hook 的组件
function Counter() {
  const { count, increment, decrement, reset } = useCounter(0);

  return (
    <div>
      <p>Count: {count}</p>
      <button onClick={decrement}>-</button>
      <button onClick={reset}>Reset</button>
      <button onClick={increment}>+</button>
    </div>
  );
}

// 同一个 Hook 可以在多处复用，每处状态独立
function AnotherCounter() {
  const { count, increment } = useCounter(100);
  return <button onClick={increment}>{count}</button>;
}
```

---

## 5. 常见模式与实战案例

### 5.1 数据请求 `useFetch`

这是最常用的自定义 Hook 之一，封装异步数据获取逻辑。

```jsx
// hooks/useFetch.js
function useFetch(url, options = {}) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);

  useEffect(() => {
    // 用于取消请求，防止组件卸载后 setState 报错
    const controller = new AbortController();

    const fetchData = async () => {
      setLoading(true);
      setError(null);

      try {
        const response = await fetch(url, {
          ...options,
          signal: controller.signal,
        });

        if (!response.ok) {
          throw new Error(`HTTP error! status: ${response.status}`);
        }

        const result = await response.json();
        setData(result);
      } catch (err) {
        if (err.name !== "AbortError") {
          setError(err.message);
        }
      } finally {
        setLoading(false);
      }
    };

    fetchData();

    return () => controller.abort();
  }, [url]); // url 变化时重新请求

  return { data, loading, error };
}

// 使用示例
function UserProfile({ userId }) {
  const { data: user, loading, error } = useFetch(`/api/users/${userId}`);

  if (loading) return <Spinner />;
  if (error) return <ErrorMessage message={error} />;
  return <div>{user.name}</div>;
}
```

**扩展版：支持手动刷新**

```jsx
function useFetch(url) {
  const [data, setData] = useState(null);
  const [loading, setLoading] = useState(true);
  const [error, setError] = useState(null);
  // 通过改变 refreshKey 触发重新请求
  const [refreshKey, setRefreshKey] = useState(0);

  useEffect(() => {
    const controller = new AbortController();

    const fetchData = async () => {
      setLoading(true);
      try {
        const res = await fetch(url, { signal: controller.signal });
        const result = await res.json();
        setData(result);
        setError(null);
      } catch (err) {
        if (err.name !== "AbortError") setError(err.message);
      } finally {
        setLoading(false);
      }
    };

    fetchData();
    return () => controller.abort();
  }, [url, refreshKey]);

  const refresh = useCallback(() => setRefreshKey((k) => k + 1), []);

  return { data, loading, error, refresh };
}
```

---

### 5.2 本地存储 `useLocalStorage`

```jsx
// hooks/useLocalStorage.js
function useLocalStorage(key, initialValue) {
  // 惰性初始化：只在首次渲染时读取 localStorage
  const [storedValue, setStoredValue] = useState(() => {
    try {
      const item = window.localStorage.getItem(key);
      return item ? JSON.parse(item) : initialValue;
    } catch (error) {
      console.error("Error reading localStorage key:", key, error);
      return initialValue;
    }
  });

  const setValue = useCallback(
    (value) => {
      try {
        // 支持函数式更新（和 setState 一样的用法）
        const valueToStore =
          value instanceof Function ? value(storedValue) : value;

        setStoredValue(valueToStore);
        window.localStorage.setItem(key, JSON.stringify(valueToStore));
      } catch (error) {
        console.error("Error setting localStorage key:", key, error);
      }
    },
    [key, storedValue],
  );

  const removeValue = useCallback(() => {
    try {
      setStoredValue(initialValue);
      window.localStorage.removeItem(key);
    } catch (error) {
      console.error("Error removing localStorage key:", key, error);
    }
  }, [key, initialValue]);

  return [storedValue, setValue, removeValue];
}

// 使用示例
function ThemeToggle() {
  const [theme, setTheme] = useLocalStorage("theme", "light");

  return (
    <button onClick={() => setTheme((t) => (t === "light" ? "dark" : "light"))}>
      当前主题：{theme}
    </button>
  );
}
```

---

### 5.3 防抖 `useDebounce`

防抖：在事件停止触发一段时间后才执行，适合搜索框输入。

```jsx
// hooks/useDebounce.js
function useDebounce(value, delay = 300) {
  const [debouncedValue, setDebouncedValue] = useState(value);

  useEffect(() => {
    const timer = setTimeout(() => {
      setDebouncedValue(value);
    }, delay);

    // 每次 value 或 delay 变化时清除上一个定时器
    return () => clearTimeout(timer);
  }, [value, delay]);

  return debouncedValue;
}

// 使用示例：搜索框
function SearchInput() {
  const [inputValue, setInputValue] = useState("");
  const debouncedSearch = useDebounce(inputValue, 500);

  // 只有停止输入 500ms 后才会触发搜索
  useEffect(() => {
    if (debouncedSearch) {
      console.log("发起搜索请求：", debouncedSearch);
      // fetch(`/api/search?q=${debouncedSearch}`)
    }
  }, [debouncedSearch]);

  return (
    <input
      value={inputValue}
      onChange={(e) => setInputValue(e.target.value)}
      placeholder="搜索..."
    />
  );
}
```

---

### 5.4 节流 `useThrottle`

节流：在指定时间内只执行一次，适合滚动、拖拽等高频事件。

```jsx
// hooks/useThrottle.js
function useThrottle(value, interval = 200) {
  const [throttledValue, setThrottledValue] = useState(value);
  const lastUpdated = useRef(null);

  useEffect(() => {
    const now = Date.now();
    if (lastUpdated.current === null || now - lastUpdated.current >= interval) {
      lastUpdated.current = now;
      setThrottledValue(value);
    } else {
      const timer = setTimeout(
        () => {
          lastUpdated.current = Date.now();
          setThrottledValue(value);
        },
        interval - (now - lastUpdated.current),
      );

      return () => clearTimeout(timer);
    }
  }, [value, interval]);

  return throttledValue;
}

// 使用示例：监听滚动位置
function ScrollTracker() {
  const [scrollY, setScrollY] = useState(0);
  const throttledScrollY = useThrottle(scrollY, 100);

  useEffect(() => {
    const handleScroll = () => setScrollY(window.scrollY);
    window.addEventListener("scroll", handleScroll);
    return () => window.removeEventListener("scroll", handleScroll);
  }, []);

  return <p>滚动位置（节流后）：{throttledScrollY}px</p>;
}
```

---

### 5.5 事件监听 `useEventListener`

```jsx
// hooks/useEventListener.js
function useEventListener(eventName, handler, element = window, options) {
  // 使用 ref 保存 handler，避免每次重新绑定事件
  const savedHandler = useRef(handler);

  useEffect(() => {
    savedHandler.current = handler;
  }, [handler]);

  useEffect(() => {
    const target = element?.current ?? element;
    if (!target?.addEventListener) return;

    const eventListener = (event) => savedHandler.current(event);
    target.addEventListener(eventName, eventListener, options);

    return () => target.removeEventListener(eventName, eventListener, options);
  }, [eventName, element, options]);
}

// 使用示例
function KeyboardShortcut() {
  useEventListener("keydown", (event) => {
    if (event.ctrlKey && event.key === "s") {
      event.preventDefault();
      console.log("保存！");
    }
  });

  return <p>按 Ctrl+S 保存</p>;
}
```

---

### 5.6 媒体查询 `useMediaQuery`

```jsx
// hooks/useMediaQuery.js
function useMediaQuery(query) {
  const [matches, setMatches] = useState(
    () => window.matchMedia(query).matches,
  );

  useEffect(() => {
    const mediaQuery = window.matchMedia(query);
    const handler = (event) => setMatches(event.matches);

    // 现代浏览器使用 addEventListener
    mediaQuery.addEventListener("change", handler);
    return () => mediaQuery.removeEventListener("change", handler);
  }, [query]);

  return matches;
}

// 使用示例
function ResponsiveLayout() {
  const isMobile = useMediaQuery("(max-width: 768px)");
  const isDarkMode = useMediaQuery("(prefers-color-scheme: dark)");

  return (
    <div>
      <p>当前是：{isMobile ? "移动端" : "PC端"}</p>
      <p>系统主题：{isDarkMode ? "深色" : "浅色"}</p>
    </div>
  );
}
```

---

### 5.7 上一次的值 `usePrevious`

```jsx
// hooks/usePrevious.js
function usePrevious(value) {
  const ref = useRef(undefined);

  // useEffect 在渲染后执行，所以 ref 保存的是上一次的值
  useEffect(() => {
    ref.current = value;
  }, [value]);

  return ref.current;
}

// 使用示例：对比前后变化
function PriceDisplay({ price }) {
  const previousPrice = usePrevious(price);

  const trend =
    previousPrice === undefined
      ? null
      : price > previousPrice
        ? "↑"
        : price < previousPrice
          ? "↓"
          : "—";

  return (
    <p>
      价格：{price} {trend}
      {previousPrice !== undefined && <small>（上次：{previousPrice}）</small>}
    </p>
  );
}
```

---

### 5.8 点击外部区域 `useOutsideClick`

常用于下拉菜单、弹窗等需要点击外部关闭的场景。

```jsx
// hooks/useOutsideClick.js
function useOutsideClick(callback) {
  const ref = useRef(null);

  useEffect(() => {
    const handleClick = (event) => {
      if (ref.current && !ref.current.contains(event.target)) {
        callback();
      }
    };

    document.addEventListener("mousedown", handleClick);
    document.addEventListener("touchstart", handleClick);

    return () => {
      document.removeEventListener("mousedown", handleClick);
      document.removeEventListener("touchstart", handleClick);
    };
  }, [callback]);

  return ref;
}

// 使用示例
function Dropdown() {
  const [isOpen, setIsOpen] = useState(false);
  const dropdownRef = useOutsideClick(() => setIsOpen(false));

  return (
    <div ref={dropdownRef}>
      <button onClick={() => setIsOpen((o) => !o)}>打开菜单</button>
      {isOpen && (
        <ul>
          <li>选项一</li>
          <li>选项二</li>
        </ul>
      )}
    </div>
  );
}
```

---

### 5.9 表单处理 `useForm`

```jsx
// hooks/useForm.js
function useForm(initialValues, validationRules = {}) {
  const [values, setValues] = useState(initialValues);
  const [errors, setErrors] = useState({});
  const [touched, setTouched] = useState({});
  const [isSubmitting, setIsSubmitting] = useState(false);

  const validate = useCallback(
    (fieldValues = values) => {
      const newErrors = {};
      Object.keys(validationRules).forEach((field) => {
        const rule = validationRules[field];
        const error = rule(fieldValues[field], fieldValues);
        if (error) newErrors[field] = error;
      });
      return newErrors;
    },
    [values, validationRules],
  );

  const handleChange = useCallback((e) => {
    const { name, value, type, checked } = e.target;
    setValues((prev) => ({
      ...prev,
      [name]: type === "checkbox" ? checked : value,
    }));
  }, []);

  const handleBlur = useCallback(
    (e) => {
      const { name } = e.target;
      setTouched((prev) => ({ ...prev, [name]: true }));
      const newErrors = validate();
      setErrors(newErrors);
    },
    [validate],
  );

  const handleSubmit = useCallback(
    (onSubmit) => async (e) => {
      e.preventDefault();
      // 提交时标记所有字段为已触碰
      const allTouched = Object.keys(values).reduce(
        (acc, key) => ({ ...acc, [key]: true }),
        {},
      );
      setTouched(allTouched);

      const validationErrors = validate();
      setErrors(validationErrors);

      if (Object.keys(validationErrors).length === 0) {
        setIsSubmitting(true);
        try {
          await onSubmit(values);
        } finally {
          setIsSubmitting(false);
        }
      }
    },
    [values, validate],
  );

  const reset = useCallback(() => {
    setValues(initialValues);
    setErrors({});
    setTouched({});
  }, [initialValues]);

  return {
    values,
    errors,
    touched,
    isSubmitting,
    handleChange,
    handleBlur,
    handleSubmit,
    reset,
  };
}

// 使用示例
function LoginForm() {
  const {
    values,
    errors,
    touched,
    isSubmitting,
    handleChange,
    handleBlur,
    handleSubmit,
  } = useForm(
    { email: "", password: "" },
    {
      email: (v) =>
        !v ? "邮箱不能为空" : !/\S+@\S+\.\S+/.test(v) ? "邮箱格式错误" : "",
      password: (v) =>
        !v ? "密码不能为空" : v.length < 6 ? "密码至少6位" : "",
    },
  );

  return (
    <form
      onSubmit={handleSubmit(async (data) => {
        await login(data);
      })}
    >
      <div>
        <input
          name="email"
          value={values.email}
          onChange={handleChange}
          onBlur={handleBlur}
          placeholder="邮箱"
        />
        {touched.email && errors.email && <span>{errors.email}</span>}
      </div>
      <div>
        <input
          name="password"
          type="password"
          value={values.password}
          onChange={handleChange}
          onBlur={handleBlur}
          placeholder="密码"
        />
        {touched.password && errors.password && <span>{errors.password}</span>}
      </div>
      <button type="submit" disabled={isSubmitting}>
        {isSubmitting ? "登录中..." : "登录"}
      </button>
    </form>
  );
}
```

---

### 5.10 复制到剪贴板 `useClipboard`

```jsx
// hooks/useClipboard.js
function useClipboard(resetInterval = 2000) {
  const [copied, setCopied] = useState(false);

  const copy = useCallback(async (text) => {
    if (!navigator?.clipboard) {
      console.warn("Clipboard API 不可用");
      return false;
    }
    try {
      await navigator.clipboard.writeText(text);
      setCopied(true);
      return true;
    } catch (error) {
      console.error("复制失败：", error);
      setCopied(false);
      return false;
    }
  }, []);

  useEffect(() => {
    if (!copied) return;
    const timer = setTimeout(() => setCopied(false), resetInterval);
    return () => clearTimeout(timer);
  }, [copied, resetInterval]);

  return { copied, copy };
}

// 使用示例
function CopyButton({ text }) {
  const { copied, copy } = useClipboard();

  return (
    <button onClick={() => copy(text)}>{copied ? "已复制！" : "复制"}</button>
  );
}
```

---

### 5.11 网络状态 `useOnlineStatus`

```jsx
// hooks/useOnlineStatus.js
function useOnlineStatus() {
  const [isOnline, setIsOnline] = useState(navigator.onLine);

  useEffect(() => {
    const handleOnline = () => setIsOnline(true);
    const handleOffline = () => setIsOnline(false);

    window.addEventListener("online", handleOnline);
    window.addEventListener("offline", handleOffline);

    return () => {
      window.removeEventListener("online", handleOnline);
      window.removeEventListener("offline", handleOffline);
    };
  }, []);

  return isOnline;
}

// 使用示例
function NetworkBanner() {
  const isOnline = useOnlineStatus();

  if (isOnline) return null;

  return (
    <div style={{ background: "red", color: "white", padding: 8 }}>
      网络已断开，请检查您的网络连接
    </div>
  );
}
```

---

### 5.12 元素尺寸观察 `useResizeObserver`

```jsx
// hooks/useResizeObserver.js
function useResizeObserver(ref) {
  const [size, setSize] = useState({ width: 0, height: 0 });

  useEffect(() => {
    if (!ref.current) return;

    const observer = new ResizeObserver(([entry]) => {
      const { width, height } = entry.contentRect;
      setSize({ width, height });
    });

    observer.observe(ref.current);
    return () => observer.disconnect();
  }, [ref]);

  return size;
}

// 使用示例
function ResponsiveChart() {
  const containerRef = useRef(null);
  const { width, height } = useResizeObserver(containerRef);

  return (
    <div ref={containerRef} style={{ width: "100%", height: "400px" }}>
      <p>
        容器尺寸：{width} x {height}
      </p>
      {/* 根据尺寸渲染不同的图表 */}
    </div>
  );
}
```

---

## 6. 进阶模式

### 6.1 Hook 组合

自定义 Hook 最强大的地方在于可以相互组合，构建更复杂的逻辑。

```jsx
// 组合 useDebounce 和 useFetch 实现防抖搜索
function useSearch(initialQuery = "") {
  const [query, setQuery] = useState(initialQuery);
  const debouncedQuery = useDebounce(query, 400);

  const { data, loading, error } = useFetch(
    debouncedQuery
      ? `/api/search?q=${encodeURIComponent(debouncedQuery)}`
      : null,
  );

  return {
    query,
    setQuery,
    results: data,
    loading,
    error,
    isSearching: query !== debouncedQuery, // 正在等待防抖
  };
}

// 使用
function SearchPage() {
  const { query, setQuery, results, loading, isSearching } = useSearch();

  return (
    <div>
      <input value={query} onChange={(e) => setQuery(e.target.value)} />
      {isSearching && <small>输入中...</small>}
      {loading && <Spinner />}
      {results?.map((item) => (
        <ResultItem key={item.id} item={item} />
      ))}
    </div>
  );
}
```

---

### 6.2 带有 Reducer 的 Hook

当状态逻辑复杂时，结合 `useReducer` 让状态转换更可预测。

```jsx
// hooks/useShoppingCart.js
const cartReducer = (state, action) => {
  switch (action.type) {
    case "ADD_ITEM": {
      const existing = state.items.find((i) => i.id === action.payload.id);
      if (existing) {
        return {
          ...state,
          items: state.items.map((i) =>
            i.id === action.payload.id ? { ...i, quantity: i.quantity + 1 } : i,
          ),
        };
      }
      return {
        ...state,
        items: [...state.items, { ...action.payload, quantity: 1 }],
      };
    }
    case "REMOVE_ITEM":
      return {
        ...state,
        items: state.items.filter((i) => i.id !== action.payload),
      };
    case "UPDATE_QUANTITY":
      return {
        ...state,
        items: state.items
          .map((i) =>
            i.id === action.payload.id
              ? { ...i, quantity: Math.max(0, action.payload.quantity) }
              : i,
          )
          .filter((i) => i.quantity > 0),
      };
    case "CLEAR":
      return { ...state, items: [] };
    default:
      return state;
  }
};

function useShoppingCart() {
  const [state, dispatch] = useReducer(cartReducer, { items: [] });

  const addItem = useCallback(
    (item) => dispatch({ type: "ADD_ITEM", payload: item }),
    [],
  );

  const removeItem = useCallback(
    (id) => dispatch({ type: "REMOVE_ITEM", payload: id }),
    [],
  );

  const updateQuantity = useCallback(
    (id, quantity) =>
      dispatch({ type: "UPDATE_QUANTITY", payload: { id, quantity } }),
    [],
  );

  const clearCart = useCallback(() => dispatch({ type: "CLEAR" }), []);

  const totalItems = state.items.reduce((sum, i) => sum + i.quantity, 0);
  const totalPrice = state.items.reduce(
    (sum, i) => sum + i.price * i.quantity,
    0,
  );

  return {
    items: state.items,
    totalItems,
    totalPrice,
    addItem,
    removeItem,
    updateQuantity,
    clearCart,
  };
}
```

---

### 6.3 工厂模式

当需要创建具有不同配置的同类 Hook 时，可以使用工厂函数。

```jsx
// 创建带有命名空间的 localStorage Hook 工厂
function createNamespacedStorage(namespace) {
  return function useNamespacedStorage(key, initialValue) {
    return useLocalStorage(`${namespace}:${key}`, initialValue);
  };
}

// 创建针对特定域的 storage Hook
const useUserPreferences = createNamespacedStorage("user_prefs");
const useAppSettings = createNamespacedStorage("app_settings");

// 使用
function UserSettings() {
  const [fontSize, setFontSize] = useUserPreferences("fontSize", 14);
  const [language, setLanguage] = useUserPreferences("language", "zh");
  // 存储键为 'user_prefs:fontSize'，不会与 'app_settings:fontSize' 冲突
}
```

---

## 7. 测试自定义 Hook

使用 `@testing-library/react` 的 `renderHook` 来测试自定义 Hook。

```jsx
// hooks/useCounter.test.js
import { renderHook, act } from "@testing-library/react";
import { useCounter } from "./useCounter";

describe("useCounter", () => {
  test("初始值默认为 0", () => {
    const { result } = renderHook(() => useCounter());
    expect(result.current.count).toBe(0);
  });

  test("支持自定义初始值", () => {
    const { result } = renderHook(() => useCounter(10));
    expect(result.current.count).toBe(10);
  });

  test("increment 增加计数", () => {
    const { result } = renderHook(() => useCounter(0));
    act(() => {
      result.current.increment();
    });
    expect(result.current.count).toBe(1);
  });

  test("reset 重置到初始值", () => {
    const { result } = renderHook(() => useCounter(5));
    act(() => {
      result.current.increment();
      result.current.increment();
      result.current.reset();
    });
    expect(result.current.count).toBe(5);
  });
});

// 测试异步 Hook
describe("useFetch", () => {
  beforeEach(() => {
    global.fetch = jest.fn();
  });

  test("成功获取数据", async () => {
    global.fetch.mockResolvedValueOnce({
      ok: true,
      json: async () => ({ id: 1, name: "Alice" }),
    });

    const { result } = renderHook(() => useFetch("/api/user/1"));

    expect(result.current.loading).toBe(true);

    await act(async () => {
      await new Promise((resolve) => setTimeout(resolve, 0));
    });

    expect(result.current.loading).toBe(false);
    expect(result.current.data).toEqual({ id: 1, name: "Alice" });
    expect(result.current.error).toBeNull();
  });
});
```

---

## 8. 性能优化

### 8.1 正确使用 `useCallback` 和 `useMemo`

```jsx
function useFilteredList(items, filterFn) {
  // 用 useMemo 缓存计算结果，只在 items 或 filterFn 变化时重新计算
  const filteredItems = useMemo(
    () => items.filter(filterFn),
    [items, filterFn],
  );

  // 用 useCallback 稳定函数引用，避免子组件不必要的重渲染
  const getItemById = useCallback(
    (id) => filteredItems.find((item) => item.id === id),
    [filteredItems],
  );

  return { filteredItems, getItemById };
}
```

### 8.2 避免在 Hook 中创建不稳定的引用

```jsx
//  问题：每次渲染都创建新的 options 对象，导致 useEffect 无限触发
function useData(userId) {
  const { data } = useFetch(`/api/users/${userId}`, {
    headers: { "Content-Type": "application/json" }, // 每次渲染都是新对象
  });
}

//  修复：用 useMemo 稳定 options 对象
function useData(userId) {
  const options = useMemo(
    () => ({
      headers: { "Content-Type": "application/json" },
    }),
    [],
  );

  const { data } = useFetch(`/api/users/${userId}`, options);
}
```

### 8.3 使用 `useRef` 存储不需要触发渲染的值

```jsx
function useTimer() {
  const [elapsed, setElapsed] = useState(0);
  // 用 ref 而不是 state，因为改变 intervalId 不需要重渲染
  const intervalRef = useRef(null);

  const start = useCallback(() => {
    intervalRef.current = setInterval(() => {
      setElapsed((e) => e + 1);
    }, 1000);
  }, []);

  const stop = useCallback(() => {
    clearInterval(intervalRef.current);
  }, []);

  useEffect(() => () => clearInterval(intervalRef.current), []);

  return { elapsed, start, stop };
}
```

---

## 9. 常见陷阱与最佳实践

### 陷阱一：遗漏 useEffect 依赖项

```jsx
//  错误：遗漏 userId 依赖，userId 变化时不会重新获取
function useUserData(userId) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, []); // 缺少 userId
}

//  正确：包含所有依赖
function useUserData(userId) {
  const [user, setUser] = useState(null);
  useEffect(() => {
    fetchUser(userId).then(setUser);
  }, [userId]);
}
```

### 陷阱二：在 useEffect 中直接使用 async 函数

```jsx
// 错误：useEffect 的回调不能是 async 函数
useEffect(async () => {
  const data = await fetchData(); // 这会导致警告
}, []);

// 正确：在内部定义并立即调用 async 函数
useEffect(() => {
  const load = async () => {
    const data = await fetchData();
    setData(data);
  };
  load();
}, []);
```

### 陷阱三：闭包陈旧值问题

```jsx
// 问题：handler 闭包捕获了初始的 count 值（0），永远不会更新
function useStaleCounter() {
  const [count, setCount] = useState(0);

  useEffect(() => {
    const timer = setInterval(() => {
      console.log(count); // 始终是 0（陈旧闭包）
    }, 1000);
    return () => clearInterval(timer);
  }, []); // count 被遗漏在依赖中

  return [count, setCount];
}

// 修复一：加入依赖（但每次 count 变化都会重建 interval）
useEffect(() => {
  const timer = setInterval(() => {
    console.log(count);
  }, 1000);
  return () => clearInterval(timer);
}, [count]);

// 修复二：使用 ref 获取最新值（推荐用于不需要依赖的情况）
function useLatestCounter() {
  const [count, setCount] = useState(0);
  const countRef = useRef(count);

  useEffect(() => {
    countRef.current = count;
  }, [count]);

  useEffect(() => {
    const timer = setInterval(() => {
      console.log(countRef.current); // 始终是最新值
    }, 1000);
    return () => clearInterval(timer);
  }, []); // 安全地省略依赖

  return [count, setCount];
}
```

### 最佳实践总结

| 实践              | 说明                                                                  |
| ----------------- | --------------------------------------------------------------------- |
| 以 `use` 开头命名 | 确保 lint 工具能检查 Hook 规则                                        |
| 单一职责          | 每个 Hook 只做一件事，保持小而专注                                    |
| 返回有意义的接口  | 返回对象（具名属性）比返回数组更清晰，除非顺序很重要（如 `useState`） |
| 清理副作用        | `useEffect` 中始终返回清理函数                                        |
| 稳定函数引用      | 对外暴露的函数用 `useCallback` 包裹                                   |
| 避免过早抽象      | 只有逻辑真正被复用时才提取为 Hook                                     |
| 配合 TypeScript   | 为参数和返回值添加类型，让 Hook 更易用                                |

---

## 10. 总结

自定义 Hook 是 React 中最重要的代码复用机制。通过本文，我们了解了：

1. **本质**：自定义 Hook 是封装了有状态逻辑的普通函数，`use` 开头只是约定
2. **优势**：相比 HOC 和 render props，它不改变组件树结构，代码更清晰
3. **规则**：必须在函数顶层调用，只能在 React 函数中使用
4. **模式**：从简单的状态封装，到数据请求、表单处理、Hook 组合等复杂模式
5. **性能**：合理使用 `useMemo`、`useCallback`、`useRef` 避免不必要的渲染
6. **陷阱**：注意依赖数组完整性、异步函数写法、闭包陈旧值问题
