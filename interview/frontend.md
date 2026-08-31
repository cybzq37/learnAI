# 前端面试题

## HTML & CSS

### HTML 语义化

用合适的标签表达内容含义（header/nav/main/section/article/footer 等）。好处：① 结构清晰、利于 SEO；② 无障碍（屏幕阅读器）；③ 代码可维护。

### 盒模型与 box-sizing

- **标准盒模型**（content-box）：宽高只包含 content，总宽 = content + padding + border + margin；
- **IE 盒模型**（border-box）：宽高包含 content + padding + border。
- `box-sizing: border-box` 让 padding/border 不撑大元素，布局计算更直观，是项目标配。

### 回流（重排）与重绘

- **回流（Reflow）**：布局几何属性变化（宽高、位置、字体、窗口缩放），浏览器重新计算布局；
- **重绘（Repaint）**：外观变化不影响布局（颜色、背景、阴影）；
- **回流必然引起重绘，重绘不一定回流**；
- 减少回流：① 用 class 批量改样式；② `transform/opacity` 代替 `top/left`（合成层，不走回流重绘）；③ 文档碎片 `documentFragment` 批量插入；④ 避免频繁读取 offsetWidth 等（强制同步布局）。

### 清除浮动

```css
.clearfix::after {
  content: "";
  display: block;
  clear: both;
}
```

父元素也可用 `overflow: hidden`（触发 BFC）或 `display: flex`（更推荐）。

### BFC（块级格式化上下文）

BFC 是一块独立的渲染区域，内部布局不影响外部。**触发条件**：`float`、`position: absolute/fixed`、`overflow: hidden`、`display: inline-block/flex/grid` 等。**作用**：① 清除浮动；② 防止 margin 塌陷；③ 两栏自适应布局。

### 水平垂直居中

```css
/* 方案1：flex（最推荐） */
.parent { display: flex; justify-content: center; align-items: center; }

/* 方案2：绝对定位 + transform */
.parent { position: relative; }
.child  { position: absolute; left: 50%; top: 50%; transform: translate(-50%, -50%); }

/* 方案3：grid */
.parent { display: grid; place-items: center; }
```

### CSS 选择器优先级

`!important` > 内联样式 > ID 选择器 > 类/属性/伪类 > 标签/伪元素 > 通配符。同级别时**后写的生效**。

### flex 布局

- 主轴方向：`flex-direction: row | column`；
- 分配空间：`flex: 1`（flex-grow: 1，占满剩余空间）、`flex-shrink`（收缩）、`flex-basis`（基准尺寸）；
- 常用：`justify-content`（主轴对齐）、`align-items`（交叉轴对齐）、`flex-wrap`（换行）。

### position 的区别

| 值 | 特点 |
|---|---|
| static | 默认，正常文档流 |
| relative | 相对自身原位置偏移，不脱离文档流 |
| absolute | 相对最近的非 static 祖先定位，脱离文档流 |
| fixed | 相对视口定位，脱离文档流 |
| sticky | 吸顶效果：滚动到阈值前是 relative，之后变 fixed |

### 移动端适配方案

- **rem 方案**：根字体随屏幕宽度变化（`html { font-size: 屏幕宽/10 }`），配合 postcss-pxtorem；
- **vw/vh 方案**：`1vw = 视口宽度 1%`，直接按设计稿换算；
- **viewport 适配**：`<meta name="viewport" content="width=device-width, initial-scale=1">`；
- 1px 问题：`transform: scale(0.5)` 或媒体查询。

## JavaScript 基础

### 数据类型与判断

- 基本类型（7 种）：number、string、boolean、null、undefined、symbol、bigint；引用类型：object（含 array、function、date）。
- **判断方式**：
  - `typeof`：判断基本类型，`typeof null === 'object'`（历史 bug），无法区分引用类型；
  - `instanceof`：判断原型链，跨 iframe 会失效；
  - `Object.prototype.toString.call(x)`：最准确，返回 `[object Array]` 等。

### 闭包

**函数 + 其词法作用域的组合**：内部函数引用外部函数的变量，即使外部函数已执行完，变量仍被保留。用途：① 私有变量（模块化）；② 防抖节流；③ 柯里化。**缺点**：变量常驻内存，滥用会导致内存泄漏。

```js
function createCounter() {
  let count = 0;
  return function () { return ++count; }; // 闭包：count 不会被回收
}
const counter = createCounter();
counter(); // 1
counter(); // 2
```

### 原型与原型链

- 每个函数有 `prototype`（显式原型），每个实例有 `__proto__`（隐式原型，指向构造函数的 prototype）；
- **原型链**：访问对象属性时，先找自身，再沿 `__proto__` 逐级向上找，直到 `Object.prototype`（其 `__proto__` 为 null）；
- `new` 的过程：创建空对象 → 绑定原型 → 执行构造函数 → 返回对象（若构造函数返回对象则返回它）。

### this 指向

**谁调用指向谁**，分四类：

| 调用方式 | this 指向 |
|---|---|
| 普通函数调用 | window（严格模式 undefined） |
| 对象方法调用 | 该对象 |
| call/apply/bind | 指定的第一个参数 |
| 箭头函数 | 定义时外层作用域的 this（无自己的 this） |
| new 调用 | 新创建的实例 |

```js
// 经典题：setTimeout 中的 this
const obj = {
  name: 'tom',
  fn() { console.log(this.name); },
};
setTimeout(obj.fn, 100);     // undefined（this 指向 window）
setTimeout(() => obj.fn(), 100); // tom（箭头函数捕获外层 this）
```

### 事件循环（Event Loop）（必问）

- **同步任务**先执行；异步任务分**宏任务**与**微任务**；
- 宏任务：setTimeout、setInterval、I/O、UI 渲染、事件回调；
- 微任务：Promise.then、queueMicrotask、MutationObserver；
- 执行顺序：**一个宏任务 → 清空所有微任务 → 渲染 → 下一个宏任务**。

```js
console.log('1');                       // 同步
setTimeout(() => console.log('2'), 0);  // 宏任务
Promise.resolve().then(() => console.log('3')); // 微任务
// 输出顺序：1 3 2
```

### 防抖与节流（高频 + 手写）

- **防抖（debounce）**：触发后延迟执行，期间再次触发则重置计时器 → 只执行最后一次（搜索联想、窗口 resize）；
- **节流（throttle）**：固定时间间隔内只执行一次 → 控制执行频率（滚动加载、拖拽、按钮点击）。

```js
// 防抖：最后一次触发后 delay 毫秒执行
function debounce(fn, delay = 300) {
  let timer = null;
  return function (...args) {
    clearTimeout(timer);
    timer = setTimeout(() => fn.apply(this, args), delay);
  };
}

// 节流：delay 毫秒内最多执行一次（时间戳版，立即执行）
function throttle(fn, delay = 300) {
  let last = 0;
  return function (...args) {
    const now = Date.now();
    if (now - last >= delay) {
      last = now;
      fn.apply(this, args);
    }
  };
}
```

### 深浅拷贝（手写）

```js
// 浅拷贝：Object.assign / 展开运算符（只拷贝第一层）
const shallow = { ...obj };

// 深拷贝：JSON 方式（有局限：undefined/函数/循环引用会丢失或报错）
const deep1 = JSON.parse(JSON.stringify(obj));

// 手写深拷贝（递归，处理循环引用）
function deepClone(target, map = new WeakMap()) {
  if (typeof target !== 'object' || target === null) return target;
  if (map.has(target)) return map.get(target);
  const result = Array.isArray(target) ? [] : {};
  map.set(target, result);
  for (const key in target) {
    if (target.hasOwnProperty(key)) {
      result[key] = deepClone(target[key], map);
    }
  }
  return result;
}
```

### 数组去重 / 扁平化（手写）

```js
// 去重
const unique = [...new Set([1, 2, 2, 3])]; // [1, 2, 3]

// 扁平化
const flat = [1, [2, [3, [4]]]].flat(Infinity); // [1, 2, 3, 4]

// 手写 flat
function myFlat(arr, depth = 1) {
  return depth > 0
    ? arr.reduce((acc, cur) => acc.concat(Array.isArray(cur) ? myFlat(cur, depth - 1) : cur), [])
    : arr.slice();
}
```

### Promise（必问）

- 三种状态：pending / fulfilled / rejected，**状态一经改变不可逆**；
- `then` 返回新 Promise 支持链式调用；`catch` 捕获错误；`finally` 无论结果都执行；
- 静态方法：`Promise.all`（全成功才成功，**一个失败立即失败**）、`Promise.race`（谁先完成用谁）、`Promise.allSettled`（等全部结束，各自返回状态）、`Promise.any`（一个成功即成功）。

```js
// 经典：Promise.all 与 race 的区别
Promise.all([p1, p2, p3]).then(console.log); // 全部成功返回数组；任一失败走 catch
Promise.race([p1, p2, p3]).then(console.log); // 返回最先完成（成功或失败）的那个
```

### async / await 原理

本质是 **Generator + 自动执行器** 的语法糖，把异步代码写成同步风格，异常用 try/catch 捕获。await 后面的表达式会被包装成 Promise，线程让出直到 resolve。

## ES6+ 新特性

### let / const 与 var 的区别

| 对比 | var | let / const |
|---|---|---|
| 作用域 | 函数作用域 | 块级作用域 |
| 变量提升 | 有（undefined） | 有（但存在**暂时性死区**，声明前访问报错） |
| 重复声明 | 允许 | 不允许 |
| 全局声明 | 挂到 window | 不挂到 window |

const 声明常量，引用类型可改属性但**不能重新赋值**。

### 箭头函数与普通函数的区别

1. **没有自己的 this**（继承外层作用域，且 call/apply 无法改变）；
2. 没有 arguments、不能作为构造函数（没有 prototype）；
3. 不能使用 new、没有 super、没有 yield。

### 解构赋值与展开运算符

```js
const { name, age, ...rest } = person; // 对象解构 + 剩余
const [first, ...others] = arr;        // 数组解构
const copy = { ...obj };               // 浅拷贝
```

### 模板字符串

支持多行与插值 `${expr}`，标签模板可自定义处理。

### 可选链与空值合并

```js
const name = user?.info?.name;      // 可选链：中间任一为空返回 undefined，不报错
const count = num ?? 0;             // 空值合并：只有 null/undefined 时用默认值
// ?? 与 || 的区别：|| 对 0、''、false 也会取默认值，?? 不会
```

### Map / Set 与 Object / Array 的区别

- **Map**：key 可以是任意类型（含对象），有 size、迭代顺序是插入顺序，适合频繁增删；
- **Set**：成员唯一，去重利器，`new Set(arr)`；
- 遍历方法：`forEach`、`for...of`、keys/values/entries。

## 浏览器与网络

### 从输入 URL 到页面展示的过程（必问）

1. **DNS 解析**：域名 → IP（浏览器缓存 → 系统缓存 → 递归查询）；
2. **TCP 连接**：三次握手；
3. **发送 HTTP 请求**：请求行 / 请求头 / 请求体；
4. **服务器响应**：返回 HTML / CSS / JS / 资源；
5. **浏览器解析渲染**：
   - 解析 HTML 构建 **DOM 树**，解析 CSS 构建 **CSSOM 树**；
   - 合并成 **Render 树**（display:none 的不进）；
   - **布局（Layout）** → **绘制（Paint）** → **合成（Composite）**；
   - JS 会阻塞 DOM 解析（`<script>` 放底部或加 defer/async）；
6. 断开连接：四次挥手（HTTP/1.1 keep-alive 可复用连接）。

### 三次握手与四次挥手

- **三次握手**：① 客户端 SYN=1（我能发吗）② 服务端 SYN+ACK（收到，我也能发）③ 客户端 ACK（收到）→ 建立连接。目的：确认双方收发能力正常。
- **四次挥手**：① 客户端 FIN（我要关了）② 服务端 ACK（收到）③ 服务端 FIN（我也要关了）④ 客户端 ACK → 关闭。因为 TCP 是**全双工**，两个方向要分别关闭。TIME_WAIT 等待 2MSL 保证最后一个 ACK 送达。

### HTTP 与 HTTPS

- HTTPS = HTTP + TLS/SSL 加密，默认端口 443；
- 加密流程：客户端发起 → 服务端返回**证书**（含公钥）→ 客户端验证证书并生成**对称密钥**，用公钥加密发送 → 服务端用私钥解密拿到对称密钥 → 之后用对称加密通信；
- 作用：防窃听（加密）、防篡改（摘要）、防冒充（证书）。

### HTTP 1.0 / 1.1 / 2.0 / 3.0

| 版本 | 核心改进 |
|---|---|
| 1.0 | 每个请求独立连接 |
| 1.1 | **keep-alive 长连接**、管线化、Host 头 |
| 2.0 | **多路复用**（一个连接并发多个请求）、头部压缩（HPACK）、服务器推送、二进制分帧 |
| 3.0 | 基于 **QUIC（UDP）**，解决队头阻塞，连接迁移更快 |

### 常见 HTTP 状态码

- **200** 成功；**201** 已创建；
- **301** 永久重定向；**302** 临时重定向；**304** 协商缓存命中（未修改）；
- **400** 参数错误；**401** 未认证；**403** 无权限；**404** 不存在；
- **500** 服务器内部错误；**502** 网关错误；**503** 服务不可用；**504** 网关超时。

### 浏览器缓存（必问）

- **强缓存**：不发请求，直接读本地缓存。字段：`Cache-Control: max-age=3600`（优先级高）、`Expires`（HTTP/1.0，绝对时间）。
- **协商缓存**：发请求问服务器，服务器判断未修改则返回 **304**。字段：`Last-Modified` / `If-Modified-Since`（秒级，不精确）、`ETag` / `If-None-Match`（内容指纹，优先）。
- 缓存位置：memory cache（内存）→ disk cache（磁盘）→ 网络请求。
- **应用**：静态资源文件名加 hash（webpack 打包），HTML 不缓存或协商缓存。

### 跨域与解决方案

跨域：**协议 / 域名 / 端口**任一不同。浏览器同源策略限制请求读取响应。

解决方案：
1. **CORS**：后端设置 `Access-Control-Allow-Origin` 等头（主流方案）；简单请求直接放行，复杂请求（自定义头/PUT 等）先发 **OPTIONS 预检**；
2. **JSONP**：利用 `<script>` 标签不受同源限制，只支持 GET，需后端配合；
3. **代理**：开发环境 devServer proxy / nginx 反向代理，前端无感知；
4. **postMessage**：iframe 跨域通信。

### Cookie / localStorage / sessionStorage / IndexedDB

| 存储 | 大小 | 有效期 | 作用域 | 特点 |
|---|---|---|---|---|
| Cookie | 4KB | 可设过期 | 同源（可设 path/domain） | 每次请求自动携带，有安全风险 |
| localStorage | 5MB | 永久 | 同源 | 纯前端存储 |
| sessionStorage | 5MB | 关闭标签页 | 同标签页 | 页面间共享 |
| IndexedDB | 大 | 永久 | 同源 | 存储结构化数据，异步 |

### WebSocket 与轮询

- 轮询/长轮询：基于 HTTP，有请求头开销，实时性差；
- **WebSocket**：全双工长连接，一次握手（HTTP Upgrade）后双向推送，适合聊天、游戏、实时行情。心跳保活：定时 ping/pong。

### 前端安全（XSS / CSRF）

- **XSS（跨站脚本攻击）**：注入恶意脚本。**防御**：转义输出、CSP（Content-Security-Policy）、HttpOnly Cookie、对用户输入过滤。
- **CSRF（跨站请求伪造）**：利用已登录 Cookie 诱导用户发请求。**防御**：同源校验（Referer）、Token 校验、双重 Cookie、SameSite。

## Vue

### Vue 2 与 Vue 3 的区别（必问）

| 对比 | Vue 2 | Vue 3 |
|---|---|---|
| 响应式原理 | Object.defineProperty（对象/数组需特殊处理） | **Proxy**（全量拦截，支持新增属性、数组索引） |
| 组合 API | Options API | **Composition API**（setup / ref / reactive） |
| 性能 | — | 更快更小，Tree-shaking 友好，diff 优化（静态节点提升） |
| 生命周期 | beforeDestroy / destroyed | beforeUnmount / unmounted |
| 类型支持 | 需配合 TS 麻烦 | 全面 TypeScript 支持 |

### 响应式原理（Vue 3）

- `reactive` 用 **Proxy** 拦截 get/set，收集依赖（effect）；`ref` 用对象包装基本类型；
- **依赖收集与触发**：get 时把当前 effect 收集到 dep；set 时遍历触发 dep 中的 effect 重新执行；
- 相比 Vue 2：Proxy 可以监听**新增属性、删除属性、数组索引/长度变化**，且无需深度遍历（惰性）。

### 生命周期

创建：beforeCreate → created（可访问 data，未挂载）→ beforeMount → mounted（DOM 已挂载，请求接口常见于此）→ beforeUpdate → updated → beforeUnmount → unmounted（清理定时器/监听器）。

### v-if 与 v-show

- `v-if`：**条件渲染**，不满足不创建 DOM（切换开销大，初始渲染省）；
- `v-show`：**display: none 切换**，始终渲染（切换开销小，初始渲染浪费）。
- 场景：频繁切换用 v-show；几乎不变用 v-if。

### computed 与 watch 的区别

- **computed**：依赖数据变化自动计算，**有缓存**（依赖不变不重新计算），适合模板中派生值；
- **watch**：监听数据变化执行副作用（异步、DOM 操作），支持 deep / immediate，无缓存。

```js
const fullName = computed(() => firstName.value + lastName.value);
watch(() => props.id, (newVal, oldVal) => { fetchData(newVal); }, { immediate: true });
```

### 组件通信方式

1. **props / emit**：父传子、子传父；
2. **v-model**：表单双向绑定（语法糖：value + input）；
3. **provide / inject**：跨层级注入（父提供，任意后代注入）；
4. **ref**：父组件获取子组件实例直接调用方法；
5. **eventBus**：任意组件通信（Vue 3 用 mitt）；
6. **Pinia / Vuex**：全局状态管理；
7. **$attrs / $listeners**：透传属性与事件。

### 插槽 slot

- 默认插槽、具名插槽（`<template #header>`）、作用域插槽（子传数据给父插槽内容，如表格列自定义）。

### keep-alive

缓存组件实例，切换时避免重新渲染（保留状态）。`include` / `exclude` 指定缓存哪些组件，配合 `activated` / `deactivated` 生命周期。

### 虚拟 DOM 与 diff 算法

- 虚拟 DOM：用 JS 对象描述真实 DOM 结构，操作成本低；
- 好处：跨平台（渲染器可替换）、批量更新（diff 后统一 patch）；
- **diff 算法**：同层比较，key 复用节点；先比较 type 与 key，不同则替换，相同则递归比较子节点，用 key 做**最长递增子序列**优化移动。

### nextTick 原理

数据更新后 DOM 是**异步**更新的（同一事件循环内多次修改只更新一次），`nextTick` 在 DOM 更新完成后执行回调，底层用 Promise.then 模拟微任务。

### Vuex / Pinia 与状态管理

- Vuex：state / mutations（同步）/ actions（异步）/ getters / modules；
- **Pinia**（Vue 3 推荐）：去掉了 mutations，state + actions + getters，更简洁，天然支持 TS，支持 setup 风格。

### 路由 Vue Router

- 两种模式：**hash**（#/xxx，兼容性好，不走服务器）与 **history**（HTML5 History API，URL 美观，需服务器配置 fallback）；
- 导航守卫：全局（beforeEach 登录鉴权）、路由独享、组件内（beforeRouteEnter）；
- 懒加载：`() => import('xxx.vue')` 按路由分包。

## React

### React 核心概念

- 组件化（函数组件 + Hooks）、**虚拟 DOM**、单向数据流（props 向下）；
- JSX 本质是 `React.createElement` 的语法糖。

### 函数组件与类组件

| 对比 | 类组件 | 函数组件 |
|---|---|---|
| 定义 | class + render | 纯函数 |
| 状态 | this.state / setState | **useState** |
| 生命周期 | componentDidMount 等 | **useEffect** |
| this | 需绑定 | 无 this |
| 推荐 | Vue 3 后不推荐 | **官方推荐** |

### Hooks 详解（必问）

- **useState**：声明状态，`const [count, setCount] = useState(0)`；
- **useEffect**：副作用（请求、订阅），依赖数组控制执行；返回清理函数（取消订阅）；`[]` 相当于 componentDidMount；
- **useMemo**：缓存计算结果（依赖不变不重算）；
- **useCallback**：缓存函数引用（防止子组件无效重渲染）；
- **useRef**：可变引用（不触发渲染），常用于获取 DOM / 保存定时器；
- **useContext**：跨层级共享数据；
- **自定义 Hook**：以 use 开头，封装复用逻辑。

```js
function useDebounce(value, delay = 300) {
  const [debounced, setDebounced] = useState(value);
  useEffect(() => {
    const timer = setTimeout(() => setDebounced(value), delay);
    return () => clearTimeout(timer); // 值变化时先清理上一次
  }, [value, delay]);
  return debounced;
}
```

### 受控组件与非受控组件

- **受控**：value 由 React state 控制，每次输入触发 onChange 更新 state（推荐）；
- **非受控**：由 DOM 自身维护，用 `useRef` 读取。

### React 渲染机制与性能优化

- 组件更新由 **state/props 变化**触发，默认**父组件更新会连带子组件**（除非 memo）；
- 优化手段：① `React.memo`（props 不变跳过渲染）+ useCallback/useMemo；② key 稳定且唯一；③ 大列表虚拟滚动（react-window）；④ 懒加载 `React.lazy + Suspense`；⑤ 避免内联对象/函数创建新引用。

### React 事件机制

React 事件是**合成事件（SyntheticEvent）**，统一委托到根容器，解决浏览器兼容性，自动批量更新；`e.stopPropagation()` 阻止冒泡。

### key 的作用

diff 时通过 key 识别节点是否复用，key 应**唯一且稳定**（不要用数组 index，会导致列表错乱），同层比较减少不必要的 DOM 重建。

### HOC 与 Render Props 与 Hooks

- **HOC（高阶组件）**：接收组件返回新组件，实现逻辑复用（老写法，如 withRouter）；
- **Render Props**：通过函数 prop 共享代码；
- **Hooks**：现代推荐方案，组合优于继承。

## 工程化与构建

### Webpack 核心概念

- **entry**：入口；**output**：输出；**loader**：处理非 JS 资源（css-loader、babel-loader、file-loader），**从右到左执行**；**plugin**：扩展构建能力（HtmlWebpackPlugin、MiniCssExtractPlugin）；
- **devServer**：热更新（HMR）；
- 打包流程：读取配置 → 从入口递归解析依赖构建模块图 → loader 转换 → plugin 干预 → 输出 bundle。

### Webpack 与 Vite 的区别

| 对比 | Webpack | Vite |
|---|---|---|
| 启动 | 全量打包，慢 | **esbuild 预构建 + 原生 ESM**，秒级启动 |
| 热更新 | 需重新打包模块 | 按需加载，快 |
| 生产构建 | 成熟稳定 | Rollup |
| 场景 | 大型老项目 | 新项目推荐 |

### 前端模块化

- CommonJS（Node，require/module.exports，同步）；
- ES Module（import/export，静态分析，支持 tree-shaking）；
- AMD（require.js）、UMD（兼容两者）。

### 打包体积优化

1. 路由/组件**按需加载**（动态 import）；
2. **tree-shaking** 去掉未使用代码（ES Module 才能摇）；
3. 公共依赖抽离（splitChunks）；按需引入 UI 库（babel-plugin-import）；
4. 压缩（terser）、gzip/brotli；
5. 图片压缩、CDN 引入大依赖。

### git 常用操作

`git add` / `git commit` / `git push` / `git pull` / `git merge` / `git rebase`（变基，整理提交历史）/ `git stash`（暂存）/ `git cherry-pick`（挑提交）/ `git reset --hard`（回退）/ `git log --oneline`。

## 性能优化

### 前端性能优化清单（必问）

**加载阶段**：
1. 资源压缩（JS/CSS/图片）、gzip、CDN 加速；
2. 图片懒加载（loading="lazy"）、骨架屏、WebP 格式；
3. 代码分包、按需加载、Tree-shaking；
4. 减少重定向、预加载（preload / prefetch）；
5. HTTP/2 多路复用、合理设置缓存。

**渲染阶段**：
1. 减少 DOM 层级、避免频繁回流重绘；
2. 长列表虚拟滚动；
3. 避免内存泄漏（定时器、事件监听器及时清理）；
4. Web Worker 处理计算密集型任务；
5. 骨架屏 + SSR/预渲染提升首屏。

### 首屏加载优化

- 路由懒加载、组件异步化；
- 首屏只加载必要资源，非首屏延迟加载；
- 使用 CSS 优化渲染路径（关键 CSS 内联）；
- 骨架屏、SSR、图片占位。

### 虚拟滚动原理

只渲染可视区域的列表项，上下留空白占位，滚动时通过 transform 移动并复用节点，支撑十万级列表。库：react-window、vue-virtual-scroller。

### 前端监控

- 错误监控：window.onerror、unhandledrejection、Vue errorHandler、React ErrorBoundary；
- 性能监控：Performance API（FP、FCP、LCP、TTFB）、web-vitals；
- 上报方式：img 打点（1x1 gif，无跨域问题）、sendBeacon（页面卸载也能发送）。

### 常见手写题汇总

- 防抖 / 节流、深拷贝、数组去重 / 扁平化；
- `new` 实现、call / apply / bind 实现；
- `Promise.all` / `Promise.race` 实现；
- 发布订阅（EventEmitter）、柯里化、函数组合 compose；
- 手写 JSONP、实现 instanceof；
- 大数相加、千分位格式化、字符串反转。

```js
// 手写 bind（高频）
Function.prototype.myBind = function (ctx, ...args) {
  const fn = this;
  return function (...rest) {
    return fn.apply(ctx, args.concat(rest));
  };
};

// 手写 Promise.all
function myAll(promises) {
  return new Promise((resolve, reject) => {
    const results = [];
    let count = 0;
    promises.forEach((p, i) => {
      Promise.resolve(p).then((val) => {
        results[i] = val;
        if (++count === promises.length) resolve(results);
      }, reject);
    });
  });
}
```
