# TypeScript + React 快速入门指南

> 面向有 JavaScript 基础的 Java 开发者，帮你快速看懂 Ragent 前端代码。

## 一、TypeScript 速成：给 JavaScript 加类型

TypeScript 就是带类型的 JavaScript。你写的 JS 代码 99% 都是合法的 TS 代码，只是多了类型标注。

### 1.1 基础类型

```typescript
// JS 写法（你已经会的）
let name = "张三";
let age = 25;
let isAdmin = true;

// TS 写法（加上类型标注）
let name: string = "张三";
let age: number = 25;
let isAdmin: boolean = true;

// 数组
let tags: string[] = ["RAG", "Agent", "MCP"];
let scores: number[] = [95, 88, 72];

// 对象（用 interface 定义结构，类似 Java 的接口）
interface User {
  userId: string;
  username: string;
  role: string;
  avatar?: string;  // ? 表示可选字段，类似 Java 的 @Nullable
}

// 使用
const user: User = {
  userId: "1001",
  username: "admin",
  role: "admin"
  // avatar 可以不写，因为是可选的
};
```

### 1.2 函数类型

```typescript
// JS 写法
function add(a, b) {
  return a + b;
}

// TS 写法：标注参数类型和返回值类型
function add(a: number, b: number): number {
  return a + b;
}

// 箭头函数（ES6 语法，TS 中更常用）
const add = (a: number, b: number): number => a + b;

// 异步函数
async function fetchUser(id: string): Promise<User> {
  const response = await fetch(`/api/users/${id}`);
  return response.json();
}

// 回调函数类型
function onMessage(callback: (msg: string) => void): void {
  callback("hello");
}
```

### 1.3 联合类型和字面量类型

```typescript
// 联合类型：变量可以是多种类型之一
let id: string | number = "abc";
id = 123; // 也合法

// 字面量类型：限定具体的值（类似 Java 的枚举）
type Role = "user" | "assistant";  // 只能是这两个值
type MessageStatus = "streaming" | "done" | "cancelled" | "error";

// Ragent 项目中的真实例子（types/index.ts）
export type FeedbackValue = "like" | "dislike" | null;

// 使用
let feedback: FeedbackValue = "like";  // OK
// feedback = "love";  // 编译报错！不在允许的值里
```

### 1.4 泛型（和 Java 泛型几乎一样）

```typescript
// Java: List<String> names = new ArrayList<>();
// TS:
let names: Array<string> = [];

// 泛型函数
function first<T>(arr: T[]): T | undefined {
  return arr[0];
}

const num = first<number>([1, 2, 3]);  // num 的类型是 number | undefined
const str = first(["a", "b"]);         // TS 自动推断为 string | undefined

// 泛型接口（Ragent 中 API 响应的通用结构）
interface Result<T> {
  code: string;
  message: string;
  data: T;
}

// 使用
type UserResult = Result<User>;
// 等价于 { code: string; message: string; data: User }
```

### 1.5 类型导出和导入

```typescript
// types/index.ts - 定义类型
export interface Session {
  id: string;
  title: string;
  lastTime?: string;
}

export interface Message {
  id: string;
  role: Role;
  content: string;
  thinking?: string;
  status?: MessageStatus;
}

// 其他文件中导入使用
import type { Session, Message } from "@/types";
// import type 只导入类型，不会产生运行时代码
```

### 1.6 类型断言（类似 Java 的强制类型转换）

```typescript
// Java: String s = (String) obj;
// TS:
const input = document.getElementById("root") as HTMLElement;

// 另一种写法
const input = document.getElementById("root")!;  // ! 表示"我确定这不是 null"
```

---

## 二、React 速成：用组件搭界面

React 的核心思想：UI = f(state)。状态变了，界面自动更新。

### 2.1 组件 = 函数

React 组件就是一个返回 JSX（类似 HTML）的函数。

```tsx
// 最简单的组件
function Hello() {
  return <h1>你好，Ragent！</h1>;
}

// 带参数的组件（props，类似 Java 方法的参数）
interface GreetingProps {
  name: string;
  role?: string;  // 可选参数
}

function Greeting({ name, role = "user" }: GreetingProps) {
  return (
    <div>
      <h1>你好，{name}！</h1>
      <p>角色：{role}</p>
    </div>
  );
}

// 使用组件（像 HTML 标签一样）
function App() {
  return (
    <div>
      <Greeting name="张三" role="admin" />
      <Greeting name="李四" />
    </div>
  );
}
```

### 2.2 JSX 语法速查

```tsx
function Demo() {
  const isAdmin = true;
  const items = ["RAG", "Agent", "MCP"];

  return (
    <div>
      {/* 条件渲染（类似 if-else） */}
      {isAdmin ? <span>管理员</span> : <span>普通用户</span>}

      {/* 条件渲染（只有 if，没有 else） */}
      {isAdmin && <button>管理后台</button>}

      {/* 三元表达式也可以用 null */}
      {isAdmin ? <AdminPanel /> : null}

      {/* 列表渲染（类似 for 循环） */}
      <ul>
        {items.map((item, index) => (
          <li key={index}>{item}</li>
        ))}
      </ul>

      {/* className 代替 class（因为 class 是 JS 关键字） */}
      <div className="container">内容</div>

      {/* style 用对象写法 */}
      <div style={{ color: "red", fontSize: "16px" }}>红色文字</div>

      {/* 事件处理 */}
      <button onClick={() => alert("点击了！")}>点我</button>
    </div>
  );
}
```

### 2.3 useState：状态管理（最核心的 Hook）

```tsx
import { useState } from "react";

function Counter() {
  // useState 返回 [当前值, 设置函数]
  // 类似 Java 的 private int count = 0; + setCount()
  const [count, setCount] = useState(0);
  const [name, setName] = useState("张三");

  return (
    <div>
      <p>计数：{count}</p>
      <button onClick={() => setCount(count + 1)}>+1</button>
      <button onClick={() => setCount(prev => prev - 1)}>-1</button>

      <input
        value={name}
        onChange={(e) => setName(e.target.value)}
      />
      <p>你好，{name}</p>
    </div>
  );
}
```

### 2.4 useEffect：副作用（类似生命周期）

```tsx
import { useState, useEffect } from "react";

function UserProfile({ userId }: { userId: string }) {
  const [user, setUser] = useState<User | null>(null);
  const [loading, setLoading] = useState(true);

  // useEffect = 组件渲染后执行的副作用
  // 第二个参数是依赖数组：只有依赖变化时才重新执行
  useEffect(() => {
    // 类似 Java 的 @PostConstruct 或 componentDidMount
    setLoading(true);
    fetch(`/api/users/${userId}`)
      .then(res => res.json())
      .then(data => {
        setUser(data);
        setLoading(false);
      });
  }, [userId]);  // userId 变化时重新请求

  // 空依赖数组 = 只在组件首次渲染时执行一次
  useEffect(() => {
    console.log("组件加载了");
    return () => {
      console.log("组件卸载了");  // 清理函数，类似 @PreDestroy
    };
  }, []);

  if (loading) return <p>加载中...</p>;
  if (!user) return <p>用户不存在</p>;

  return <div>{user.username}</div>;
}
```

### 2.5 useRef：引用 DOM 元素

```tsx
import { useRef } from "react";

function ChatInput() {
  // useRef 类似 Java 的成员变量，不会触发重新渲染
  const textareaRef = useRef<HTMLTextAreaElement>(null);
  const isComposingRef = useRef(false);

  const focusInput = () => {
    textareaRef.current?.focus();  // ?. 是可选链，null 安全
  };

  return (
    <div>
      <textarea ref={textareaRef} placeholder="输入问题..." />
      <button onClick={focusInput}>聚焦输入框</button>
    </div>
  );
}
```

### 2.6 useCallback 和 useMemo：性能优化

```tsx
import { useState, useCallback, useMemo } from "react";

function SearchList({ items }: { items: string[] }) {
  const [query, setQuery] = useState("");

  // useMemo：缓存计算结果，只有依赖变化时才重新计算
  // 类似 Java 的缓存/懒加载
  const filtered = useMemo(() => {
    return items.filter(item => item.includes(query));
  }, [items, query]);

  // useCallback：缓存函数引用，避免子组件不必要的重新渲染
  const handleSearch = useCallback((value: string) => {
    setQuery(value);
  }, []);

  return (
    <div>
      <input onChange={(e) => handleSearch(e.target.value)} />
      <ul>
        {filtered.map(item => <li key={item}>{item}</li>)}
      </ul>
    </div>
  );
}
```

---

## 三、Zustand：全局状态管理（Ragent 用的方案）

Zustand 是一个极简的状态管理库，比 Redux 简单得多。你可以把它理解为一个全局的 "数据仓库"，任何组件都能读写。

### 3.1 创建 Store

```typescript
// stores/counterStore.ts
import { create } from "zustand";

// 1. 定义状态的类型（interface）
interface CounterState {
  count: number;           // 状态
  increment: () => void;   // 修改状态的方法
  decrement: () => void;
  reset: () => void;
}

// 2. 创建 store（类似 Java 的单例 Service）
export const useCounterStore = create<CounterState>((set) => ({
  count: 0,
  increment: () => set((state) => ({ count: state.count + 1 })),
  decrement: () => set((state) => ({ count: state.count - 1 })),
  reset: () => set({ count: 0 }),
}));
```

### 3.2 在组件中使用 Store

```tsx
// 任何组件都可以直接使用，不需要 Provider 包裹
import { useCounterStore } from "@/stores/counterStore";

function CounterDisplay() {
  // 从 store 中取出需要的状态和方法
  const { count, increment, decrement } = useCounterStore();

  return (
    <div>
      <p>计数：{count}</p>
      <button onClick={increment}>+1</button>
      <button onClick={decrement}>-1</button>
    </div>
  );
}

// 另一个组件也能访问同一个 store
function CounterBadge() {
  const count = useCounterStore((state) => state.count);
  return <span>当前值：{count}</span>;
}
```

### 3.3 Ragent 中的真实 Store 示例

```typescript
// Ragent 的 authStore.ts 简化版
import { create } from "zustand";

interface AuthState {
  user: User | null;
  token: string | null;
  isAuthenticated: boolean;
  login: (username: string, password: string) => Promise<void>;
  logout: () => Promise<void>;
}

export const useAuthStore = create<AuthState>((set, get) => ({
  user: null,
  token: null,
  isAuthenticated: false,

  login: async (username, password) => {
    // 调用后端 API
    const data = await loginRequest(username, password);
    // 更新状态（所有使用这个 store 的组件会自动重新渲染）
    set({
      user: data,
      token: data.token,
      isAuthenticated: true
    });
  },

  logout: async () => {
    await logoutRequest();
    set({ user: null, token: null, isAuthenticated: false });
  },
}));
```

---

## 四、Ragent 前端项目结构对照表

```
frontend/src/
├── main.tsx              # 入口文件（类似 Java 的 main 方法）
├── App.tsx               # 根组件（路由配置）
├── router.tsx            # 路由定义（URL -> 页面的映射）
│
├── types/                # 类型定义（类似 Java 的 DTO/VO）
│   └── index.ts          # Session, Message, User 等接口
│
├── services/             # API 调用层（类似 Java 的 Service 调后端）
│   ├── api.ts            # Axios 实例 + 拦截器（统一请求/响应处理）
│   ├── authService.ts    # 登录/登出 API
│   ├── chatService.ts    # 对话相关 API
│   ├── sessionService.ts # 会话管理 API
│   ├── knowledgeService.ts
│   └── ...
│
├── stores/               # 全局状态（类似 Java 的单例 Service）
│   ├── authStore.ts      # 认证状态（用户信息、token）
│   ├── chatStore.ts      # 对话状态（会话列表、消息、流式状态）
│   └── themeStore.ts     # 主题状态
│
├── hooks/                # 自定义 Hook（可复用的逻辑）
│   ├── useAuth.ts        # 认证相关逻辑
│   ├── useChat.ts        # 对话相关逻辑
│   └── useStreamResponse.ts  # SSE 流式响应处理
│
├── components/           # 可复用组件
│   ├── chat/             # 对话相关组件
│   │   ├── ChatInput.tsx       # 输入框
│   │   ├── MessageItem.tsx     # 消息气泡
│   │   ├── MessageList.tsx     # 消息列表
│   │   ├── MarkdownRenderer.tsx # Markdown 渲染
│   │   ├── ThinkingIndicator.tsx # 思考动画
│   │   └── FeedbackButtons.tsx  # 点赞/点踩
│   ├── session/          # 会话管理组件
│   ├── layout/           # 布局组件（Header, Sidebar）
│   ├── common/           # 通用组件（Loading, Toast, Avatar）
│   └── ui/               # 基础 UI 组件（Button, Input, Dialog）
│
├── pages/                # 页面组件（每个 URL 对应一个页面）
│   ├── ChatPage.tsx      # 主对话页
│   ├── LoginPage.tsx     # 登录页
│   └── admin/            # 管理后台页面
│       ├── AdminLayout.tsx
│       ├── dashboard/
│       ├── knowledge/
│       ├── intent-tree/
│       ├── ingestion/
│       └── traces/
│
├── utils/                # 工具函数
│   ├── helpers.ts
│   ├── storage.ts        # localStorage 封装
│   └── error.ts
│
├── lib/                  # 第三方库封装
│   └── utils.ts          # cn() 工具函数（合并 CSS 类名）
│
└── styles/               # 全局样式
    └── globals.css
```

**Java 开发者对照理解：**

| 前端概念 | Java 对应概念 |
|----------|--------------|
| Component（组件） | View / Fragment |
| Props（属性） | 方法参数 / 构造函数参数 |
| State（状态） | 成员变量 |
| Hook（钩子） | 注解 / AOP |
| Store（Zustand） | 单例 Service + 观察者模式 |
| Service（API 调用） | Feign Client / RestTemplate |
| Router（路由） | Spring MVC 的 @RequestMapping |
| types/ | DTO / VO 类 |

---

## 五、Ragent 真实代码逐行解读

### 5.1 API 封装层（services/api.ts）

这是所有 API 请求的基础，类似 Java 中配置 RestTemplate 或 Feign。

```typescript
import axios from "axios";
import { storage } from "@/utils/storage";

// 创建 axios 实例（类似 Java 的 RestTemplate 配置）
export const api = axios.create({
  baseURL: import.meta.env.VITE_API_BASE_URL || "",  // 从环境变量读取后端地址
  timeout: 60000  // 超时 60 秒
});

// 请求拦截器（类似 Java 的 HandlerInterceptor.preHandle）
// 每个请求自动带上 token
api.interceptors.request.use((config) => {
  const token = storage.getToken();
  if (token) {
    config.headers.Authorization = token;
  }
  return config;
});

// 响应拦截器（类似 Java 的 HandlerInterceptor.afterCompletion）
// 统一处理响应格式和错误
api.interceptors.response.use(
  (response) => {
    const payload = response.data;
    // 后端返回 { code: "0", message: "ok", data: {...} }
    if (payload.code !== "0") {
      return Promise.reject(new Error(payload.message));
    }
    return payload.data;  // 直接返回 data，调用方不用再 .data.data
  },
  (error) => {
    if (error?.response?.status === 401) {
      // token 过期，跳转登录页
      storage.clearAuth();
      window.location.href = "/login";
    }
    return Promise.reject(error);
  }
);
```

### 5.2 Service 层调用示例

```typescript
// services/chatService.ts
import { api } from "./api";

// 停止生成（POST 请求）
export function stopTask(taskId: string) {
  return api.post("/api/rag/chat/stop", { taskId });
}

// 提交反馈（POST 请求）
export function submitFeedback(messageId: string, vote: number) {
  return api.post(`/api/rag/messages/${messageId}/feedback`, { vote });
}

// services/sessionService.ts
// 获取会话列表（GET 请求）
export function listSessions() {
  return api.get("/api/rag/conversations");
}

// 获取消息列表（GET 请求带参数）
export function listMessages(conversationId: string) {
  return api.get(`/api/rag/conversations/${conversationId}/messages`);
}

// 删除会话（DELETE 请求）
export function deleteSession(conversationId: string) {
  return api.delete(`/api/rag/conversations/${conversationId}`);
}
```

### 5.3 ChatInput 组件解读（简化版）

```tsx
// components/chat/ChatInput.tsx 的核心逻辑
import * as React from "react";
import { useChatStore } from "@/stores/chatStore";

export function ChatInput() {
  // ① 本地状态：输入框的值
  const [value, setValue] = React.useState("");

  // ② 从全局 store 取出需要的状态和方法
  const { sendMessage, isStreaming, cancelGeneration } = useChatStore();

  // ③ ref：引用 DOM 元素（输入框）
  const textareaRef = React.useRef<HTMLTextAreaElement>(null);

  // ④ 提交处理
  const handleSubmit = async () => {
    if (isStreaming) {
      cancelGeneration();  // 正在生成时点击 = 停止
      return;
    }
    if (!value.trim()) return;  // 空内容不提交
    const msg = value;
    setValue("");  // 清空输入框
    await sendMessage(msg);  // 调用 store 的方法发送消息
  };

  // ⑤ 渲染 UI
  return (
    <div>
      <textarea
        ref={textareaRef}
        value={value}
        onChange={(e) => setValue(e.target.value)}
        onKeyDown={(e) => {
          // Enter 发送，Shift+Enter 换行
          if (e.key === "Enter" && !e.shiftKey) {
            e.preventDefault();
            handleSubmit();
          }
        }}
        placeholder="输入你的问题..."
      />
      <button onClick={handleSubmit}>
        {isStreaming ? "停止" : "发送"}
      </button>
    </div>
  );
}
```

### 5.4 SSE 流式响应处理

这是 Ragent 前端最复杂的部分，理解它就理解了整个对话流程。

```typescript
// hooks/useStreamResponse.ts 的核心逻辑（简化版）
export function createStreamResponse(url: string, body: object) {
  const controller = new AbortController();

  // 用 fetch 发起 SSE 请求（不是 WebSocket，是 Server-Sent Events）
  fetch(url, {
    method: "POST",
    headers: { "Content-Type": "application/json" },
    body: JSON.stringify(body),
    signal: controller.signal,  // 用于取消请求
  }).then(async (response) => {
    const reader = response.body!.getReader();
    const decoder = new TextDecoder();

    // 持续读取流数据
    while (true) {
      const { done, value } = await reader.read();
      if (done) break;

      const text = decoder.decode(value);
      // 解析 SSE 格式：每条消息以 "data: " 开头
      // event: stream-meta     -> 流开始，拿到 conversationId
      // event: message-delta   -> 内容增量，追加到消息中
      // event: completion      -> 流结束
      parseSSEEvents(text);
    }
  });

  // 返回取消函数
  return () => controller.abort();
}
```

---

## 六、动手练习：写一个迷你对话组件

把上面学到的知识串起来，写一个完整的小组件：

```tsx
// MiniChat.tsx - 一个最简单的对话组件
import { useState } from "react";

// ① 定义消息类型
interface ChatMessage {
  id: number;
  role: "user" | "assistant";
  content: string;
}

// ② 组件
export function MiniChat() {
  const [messages, setMessages] = useState<ChatMessage[]>([]);
  const [input, setInput] = useState("");
  const [loading, setLoading] = useState(false);

  // ③ 发送消息
  const handleSend = async () => {
    if (!input.trim() || loading) return;

    // 添加用户消息
    const userMsg: ChatMessage = {
      id: Date.now(),
      role: "user",
      content: input,
    };
    setMessages(prev => [...prev, userMsg]);
    setInput("");
    setLoading(true);

    // 模拟 AI 回复（实际项目中这里调后端 API）
    setTimeout(() => {
      const aiMsg: ChatMessage = {
        id: Date.now() + 1,
        role: "assistant",
        content: `你问的是：${userMsg.content}。这是 AI 的回答。`,
      };
      setMessages(prev => [...prev, aiMsg]);
      setLoading(false);
    }, 1000);
  };

  // ④ 渲染
  return (
    <div style={{ maxWidth: 600, margin: "0 auto", padding: 20 }}>
      <h2>迷你对话</h2>

      {/* 消息列表 */}
      <div style={{ minHeight: 300, border: "1px solid #eee", padding: 16 }}>
        {messages.map(msg => (
          <div
            key={msg.id}
            style={{
              textAlign: msg.role === "user" ? "right" : "left",
              margin: "8px 0",
            }}
          >
            <span
              style={{
                display: "inline-block",
                padding: "8px 12px",
                borderRadius: 8,
                background: msg.role === "user" ? "#3B82F6" : "#F3F4F6",
                color: msg.role === "user" ? "white" : "black",
              }}
            >
              {msg.content}
            </span>
          </div>
        ))}
        {loading && <p style={{ color: "#999" }}>AI 思考中...</p>}
      </div>

      {/* 输入区域 */}
      <div style={{ display: "flex", gap: 8, marginTop: 12 }}>
        <input
          value={input}
          onChange={(e) => setInput(e.target.value)}
          onKeyDown={(e) => e.key === "Enter" && handleSend()}
          placeholder="输入问题..."
          style={{ flex: 1, padding: 8, borderRadius: 4, border: "1px solid #ddd" }}
        />
        <button
          onClick={handleSend}
          disabled={loading}
          style={{
            padding: "8px 16px",
            borderRadius: 4,
            background: "#3B82F6",
            color: "white",
            border: "none",
            cursor: loading ? "not-allowed" : "pointer",
          }}
        >
          发送
        </button>
      </div>
    </div>
  );
}
```

---

## 七、从 JavaScript 到 TypeScript 的常见困惑

### 7.1 "any" 是什么？

```typescript
// any = 关闭类型检查，等于回到 JS
let data: any = "hello";
data = 123;      // 不报错
data.foo.bar();  // 不报错（但运行时会崩）

// 尽量避免 any，用 unknown 代替
let data: unknown = "hello";
// data.foo();  // 编译报错！必须先检查类型
if (typeof data === "string") {
  data.toUpperCase();  // OK，TS 知道这里是 string
}
```

### 7.2 "as const" 是什么？

```typescript
// 普通写法：TS 推断为 string[]
const roles = ["admin", "user"];

// as const：TS 推断为 readonly ["admin", "user"]（字面量元组）
const roles = ["admin", "user"] as const;
// roles[0] 的类型是 "admin"，不是 string
```

### 7.3 解构赋值（ES6 语法，React 中大量使用）

```typescript
// 对象解构
const user = { name: "张三", age: 25, role: "admin" };
const { name, age } = user;  // name = "张三", age = 25

// 带默认值
const { role = "user" } = user;

// 数组解构（useState 就是这个原理）
const [first, second] = [1, 2];  // first = 1, second = 2
const [count, setCount] = useState(0);  // 解构 useState 返回的数组

// 函数参数解构（React 组件的 props）
function Greeting({ name, role }: { name: string; role: string }) {
  return <p>{name} - {role}</p>;
}
```

### 7.4 展开运算符（...）

```typescript
// 数组展开
const arr1 = [1, 2, 3];
const arr2 = [...arr1, 4, 5];  // [1, 2, 3, 4, 5]

// 对象展开（React 中更新状态常用）
const user = { name: "张三", age: 25 };
const updated = { ...user, age: 26 };  // { name: "张三", age: 26 }

// React 中的典型用法
setMessages(prev => [...prev, newMessage]);  // 追加消息
set({ ...state, isLoading: true });          // 更新部分状态
```

---

## 八、学习路径建议

1. 先把本文档过一遍，理解 TS 类型和 React 组件的基本概念（1 小时）
2. 打开 Ragent 的 `frontend/src/types/index.ts`，看看项目定义了哪些类型
3. 打开 `frontend/src/services/api.ts`，理解 API 封装层
4. 打开 `frontend/src/stores/chatStore.ts`，理解全局状态管理
5. 打开 `frontend/src/components/chat/ChatInput.tsx`，对照本文档的解读看真实代码
6. 尝试修改一个小功能（比如改输入框的 placeholder 文字），跑起来看效果

不需要精通前端，能看懂代码、能做小修改、面试时能说清楚前后端怎么交互就够了。
