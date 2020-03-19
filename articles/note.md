# 知识点笔记

一些知识点笔记，或者题目都会放在这里，比较丰富的内容会单独抽出来。

## 如果我出题

### Object.definne

以下输出什么

```js
let a = {};
Object.defineProperty(a, 'i', {
  value: 0
});

a.i++;

console.log(a.i);
```

```js
let a = 1;

Object.defineProperty(window, 'a', {
  get() {
    return 2;
  }
});

console.log(window.a);
console.log(a);
```

下面会发生什么

```js
var b = 1;

Object.defineProperty(window, 'b', {
  get() {
    return 2;
  }
});
```

## 编程

### 写个程序把 entry 转换成如下对象

```js
var entry = {
  'a.b.c.dd': 'abcdd',
  'a.d.xx': 'adxx',
  'a.e': 'ae'
}

// 要求转换成如下对象
var output = {
  a: {
   b: {
     c: {
       dd: 'abcdd'
     }
   },
   d: {
     xx: 'adxx'
   },
   e: 'ae'
  }
}
```

```js
function process(entry) {
    let result = {};
    Object.keys(entry).forEach((key) => {
        key.split('.').reduce((a, b, index, {length}) => {
            if (index === length - 1) {
                a[b] = entry[key]
            } else if (!(b in a)) {
                a[b] = {}
            }
            return a[b];
        }, result);
    });
    return result;
}
```

### 实现 destructuringArray 方法，达到如下效果

`destructuringArray([1,[2,4],3], "[a,[b],c]")`
`{a: 1, b: 2, c: 3}`

```js
function destructuringArray(value, key) {
    let code = `
        let result = {};
        with(new Proxy(result, {
            has() {
                return true
            }
        })) {
            ${key} = ${JSON.stringify(value)};
        }
        return result;
    `;
    let fn = new Function(code);
    return fn();
}
```

实现是实现了。。。会被打死吧

### 实现一个 normalize 函数，能将输入的特定的字符串转化为特定的结构化数据

字符串仅由小写字母和 [] 组成，且字符串不会包含多余的空格。
示例一: 'abc' --> {value: 'abc'}
示例二：'[abc[bcd[def]]]' --> {value: 'abc', children: {value: 'bcd', children: {value: 'def'}}}

```js
function normalize(str) {
  let current, result;
  while (current = str.match(/\[([a-z]+)\]/)) {
    let [res, value] = current;
    result = result ? { value, children: result } : { value };
    str = str.replace(res, '');
  }
  if (!result) result = {value: str};
  return result;
}
```

### 实现 call、apply、bind

注意 bind 带有柯里化的性质

```js
function call(fn, target, ...args) {
  const name = Symbol(fn.name);
  target[name] = fn;
  return target[name](...args);
}

function apply(fn, target, args) {
  const name = Symbol(fn.name);
  target[name] = fn;
  return target[name](...args);
}

function bind(fn, target, ...preArgs) {
  const name = Symbol(fn.name);
  target[name] = fn;
  return function(...args) {
    return target[name](...[...preArgs, ...args])
  }
}
```

### 二叉树的最大深度

递归，执行一次加个1

```js
let a = {
  right: {
    left: {}
  },
  left: {
    right: {
      left: {}
    }
  }
}
function maxDepth(tree) {
  if (!tree) return 0;
  let { left, right } = tree;
  let leftDepth = maxDepth(left);
  let rightDepth = maxDepth(right);

  return 1 + (rightDepth > leftDepth ? rightDepth : leftDepth);
}

maxDepth(a);
```

### bilibili reverse

用 JavaScript 写一个函数，输入 int 型，返回整数逆序后的字符串。如：输入整型 1234，返回字符串“4321”。要求必须使用递归函数调用，不能用全局变量，输入函数必须只有一个参数传入，必须返回字符串。

```js
function reverse(num) {
  num = num + '';
  let result = num[0];
  let right = num.slice(1);
  if (right.length > 0) {
    result = reverse(+right) + result;
  }
  return result;
}
```

### 要求设计 LazyMan 类，实现以下功能

```js
LazyMan('Tony');
// Hi I am Tony

LazyMan('Tony').sleep(10).eat('lunch');
// Hi I am Tony
// 等待了10秒...
// I am eating lunch

LazyMan('Tony').eat('lunch').sleep(10).eat('dinner');
// Hi I am Tony
// I am eating lunch
// 等待了10秒...
// I am eating diner

LazyMan('Tony').eat('lunch').eat('dinner').sleepFirst(5).sleep(10).eat('junk food');
// Hi I am Tony
// 等待了5秒...
// I am eating lunch
// I am eating dinner
// 等待了10秒...
// I am eating junk food
```

一开始想复杂了，一个冲刷器不停冲刷任务队列，一个正常任务队列一个延时队列，看了别人答案的确是链路比较好,有点像 `generator` 的实现

```js
function LazyMan(str) {
    let queue = [];

    function next() {
        currentTask = queue.shift();
        currentTask && currentTask();
    }

    Promise.resolve().then(next);
    str && console.log(`hi Im ${str}`);
    return {
        sleep(num) {
            queue.push(function() {
                setTimeout(next, num*1000);
            });
            return this;
        },
        eat(str) {
            queue.push((function(str) {
                return function() {
                    console.log(`eat ${str}`);
                    next();
                }
            })(str));
            return this;
        },
        sleepFirst(num) {
            queue.unshift(function() {
                setTimeout(next, num*1000);
            });
            return this;
        }
    }
}
```

### 用 setTimeOut 实现 setInterVal

管理了id 实现了清理

```js
function setMyInterval(cb, dur, extar, timer = {}) {
    timer.id = setTimeout(() => {
        cb(extar);
        clearTimeout(timer.id)
        setMyInterval(cb, dur, extar, timer)
    }, dur, extar);
    return timer;
}
function clearMyInterval(timer) {
    clearTimeout(timer.id)
}
```

### 深拷贝，考虑 symbol 和 引用

基本验证了下可以，WeakSet 存对象防止多次引用，多次引用返回引用的对象 不拷贝。

```js
function deepCopy(target, cache = new WeakSet()) {
    let result;
    if (target instanceof Object) {
        if (cache.has(target)) return target;
        cache.add(target);
        if (Array.isArray(target)) {
            result = [];
            for (let i = 0;i < target.length; i++) {
                result[i] = deepCopy(target[i], cache);
            }
        } else {
            result = {};
            Reflect.ownKeys(target).forEach(key => void (result[key] = deepCopy(target[key], cache)))
        }
    } else {
        result = target;
    }

    return result;
}
```

## 浏览器

### http 状态码

* 200: 成功、强缓存未过期、弱缓存，service work
* 204: no content 请求成功但没有返回任何内容，适用于埋点
* 301: 永久重定向
* 302: 临时重定向
* 304: 强缓存过期，url请求，服务器判断未过期可以继续使用
* 400: 错误
* 401: 需要身份验证
* 403: 服务器拒绝
* 404: not found
* 500: 服务器内部错误
* 503: 服务不可用

### webview 通讯机制

* 拦截URL SCHEME

实际上 webview 和浏览器一样会请求 url 而 Native 可以对请求进行拦截，获取 url 即参数数据，对特定 url 进行操作。

通常用 iframe.src 来发送而不是 location.href。

* api 注入

通过 webview 提供的接口直接对 window 注入对象或者方法，让js 调用时直接调用 Native 代码逻辑

api 注入更好，请求需要耗时，url 长度也有一定的限制

* Native 调用 JavaScript

通过接口可以直接执行 js 代码（类似 eval）方法对象需在 window 上实现

### http2

* 二进制传输，取代了 HTTP1.x 的文本传输，更高效
* 多路复用，一个tcp链接，HTTP1.x 中并发请求多个 TCP 连接，大概6-7个限制

### preload & prefetch

```html
<link rel="preload" href="style.css" as="style">
<link rel="prefetch" href="js/chunk-xxx.js">
```

两者都是预加载资源并缓存，加载后并不执行，等真正用到时在执行，不过优先级不一样。

`prefetch` 为预先加载之后**可能**会用到的资源，浏览器会在空闲时间加载这些资源。

`preload` 为预先加载之后**一定**会用到的资源，标记了 `preload` 的资源会在页面生命周期的早期加载，再进行浏览器的主渲染，并不会阻塞进程。

## 函数式编程

### 柯里化

```js
function curry(fn, arr = []) {
  let { length } = fn;
  return arr.length === fn.length ?
    fn(...arr) :
    function(...args) {
      return curry(fn, [...arr, ...args])
    }
}
```

### compose

组合，从后到前 `compose(f, g, h) = f(g(h()))`

```js
function compose(...args) {
  let res, current;
  return function(arg) {
    let fns = args.slice();
    res = arg;
    while (current = fns.pop()) {
      res = current(res);
    }
    return res;
  }
}
```

### pipe

管道，从前往后 `pipe(f,g,h) = h(g(f()))`

```js
function pipe(...args) {
  let res, current;
  return function(arg) {
    let fns = args.slice();
    res = arg;
    while (current = fns.shift()) {
      res = current(res);
    }
    return res;
  }
}
```

## javascript

### Generator & async

这里有一篇不错的[文章][how_generators_work]

简单的说 `async` 是 `Generator` 的语法糖 `await` 修饰的语句会被 `Promise` 包裹，异步任务完成后调用 `Generator` 的 `next` 方法。

而 `Generator` 函数会返回一个迭代器器 (iterator)，调用迭代器的 `next` 方法执行函数直到遇到 `yield`， 根据 [ECMA][ECMA_generator] 所说，此时 内部的 `[[GeneratorState]]` 状态将会被置为暂停（task 的 `step` 停止执行？）并返回迭代器。下一次调用 `next` 时 `[[GeneratorState]]` 为恢复状态，代码继续运行。

简单的说～嗯！

### 深度优先 & 广度优先

用 dom 元素来模拟

```html
<div id="a">
    <div id="b">
        <div id="d"></div>
        <div id="e"></div>
    </div>
    <div id="c">
        <div id="f"></div>
        <div id="g"></div>
    </div>
</div>
```

深度优先 递归就可以了

```js
function deepTraversal(node) {
  if (!node) return;
  console.log(node); // 处理 node
  if (node.children && node.children.length > 0) {
    Array.from(current.children).forEach(deepTraversal);
  }
}

deepTraversal(document.querySelector('#a')) // a b d e c f g
```

广度优先, 用队列存储要处理的任务

```js
function widthTraversal(node) {
  let queue = [node];

  let current;
  while (queue.length > 0) {
    current = queue.shift();
    // 处理
    console.log(current);

    if (current.children && current.children.length > 0) {
      queue = queue.concat(Array.from(current.children))
    }
  }
}

widthTraversal(document.querySelector('#a')) // a b c d e f g
```

### 防抖 & 节流

都是控制触发的频率，比如n秒内触发一次。防抖和节流不一样的是防抖在过程中再次触发会重新计数。

```js
// 防抖
function debounce(fn) {
  let last = Date.now();
  return function() {
    let now = Date.now();
    if ((now - last) > 1000) {
      fn();
    }
    last = now;
  }
}
// 节流
function throttle(fn) {
  let last = Date.now();
  return function() {
    let now = Date.now();
    if ((now - last) > 1000) {
      last = now;
      fn();
    }
  }
}
```

代码上也只是细微的区别

### WeakMap & WeakSet

`WeakMap` `WeakSet` 相比 `Map` 和 `Set` 总结来说有两个特点

* 只接接受 `Object` 作为 `key` / `value`
* 弱引用

意味着 `WeakMap`、`WeakSet` 对象没有没有存储当前对象的列表所以他们都是不可枚举的，也只有简单的 `set/add`、`has`、`delete`（`WeakMap`还有个`get`）。

当然同样是因为弱引用，引用对象是有可能被垃圾回收机制销毁掉的：

```html
<html lang="en" dir="ltr">
    <head>
        <meta charset="utf-8">
        <title></title>
    </head>
    <body>
        <button onclick="clickHandler(this)">123</button>
        <script type="text/javascript">
            let a = {};
            let wm = new WeakMap();
            let ws = new WeakSet();
            wm.set(a, 1);
            ws.add(a);
            function clickHandler() {
                console.log(wm);
                console.log(ws);
            }
            a = null;
        </script>
    </body>
</html>
```

在浏览器中打开，我们点击按钮打印出了正常的对象

```js
WeakMap {{…} => 1}
WeakSet {{…}}
```

现在我们打开 chrome 的开发者工具 => Performance 点击垃圾桶（Collect garbage）来手动触发垃圾回收。现在我们在点击按钮

```js
WeakMap {}
WeakSet {}
```

引用对象已经被回收了。

### String.protorype.match

参数是一个正则表达式，返回一个数组。

如果正则带有 `g` 标记（全局），仅仅返回所有表达式匹配的结果

```js
const varName = '[a-zA-Z0-9]+';
const position = `https?://[0-9:.]+/${varName}.js:[0-9]+:[0-9]+`;
const preChrome = `at ${varName} `;
const preFirefox = `${varName}@`;
const errorPath = new RegExp(`(${preChrome}|${preFirefox})(${position})`, 'g');

let a = `
    bar@http://192.168.31.8:8000/c.js:2:9
    foo@http://192.168.31.8:8000/b.js:4:15
    calc@http://192.168.31.8:8000/a.js:4:3
    <anonymous>:1:11
    http://192.168.31.8:8000/a.js:22:3
`.match(errorPath)
// 结果：
a = [
  'bar@http://192.168.31.8:8000/c.js:2:9',
  'foo@http://192.168.31.8:8000/b.js:4:15',
  'calc@http://192.168.31.8:8000/a.js:4:3'
]
```

如果未使用g标志，则仅返回第一个完整匹配及其相关的捕获组（子项）。 在这种情况下，返回的项目将具有如下所述的其他属性。

* `groups`: 一个捕获组数组 或 `undefined`（如果没有定义命名捕获组）。
* `index`: 匹配的结果的开始位置
* `input`: 搜索的字符串.

```js
// 同上
const errorPath = new RegExp(`(${preChrome}|${preFirefox})(${position})`);
//同上
// 结果：
a = [
  'bar@http://192.168.31.8:8000/c.js:2:9',
  'bar@',
  'http://192.168.31.8:8000/c.js:2:9',
  index: 5,
  input: '\n' +
    '    bar@http://192.168.31.8:8000/c.js:2:9\n' +
    '    foo@http://192.168.31.8:8000/b.js:4:15\n' +
    '    calc@http://192.168.31.8:8000/a.js:4:3\n' +
    '    <anonymous>:1:11\n' +
    '    http://192.168.31.8:8000/a.js:22:3\n',
  groups: undefined
]
```

### 经典的继承算法

```js
let _extends = (function() {
  function F() {};
  return function(target, parent) {
    F.prototype = parent.prototype;
    target.prototype = new F();
    target.prototype.constructor = target;
  }
})()

function Child(age, name) {
  Parent.call(this, name);
  this.age = age;
}

_extends(Child, Parent);

Child.prototype.getAge = function() {return this.age}

function Parent(name) {
  this.name = name;
}

Parent.prototype.getName = function() {
  return this.name;
}
```

### 平等表达式 `==`

```js
var a = ?;
if(a == 1 && a == 2 && a == 3){
  console.log(1);
}
```

见 blog 从一道题了解 `==`

### 实现 flatten 函数

简单的说就是数组降维

递归实现：

```js
// 去重且升序
function flatten(arr, cache) {
  cache = cache || {};
  let result = [];
  arr.forEach(item => {
    if (Array.isArray(item)) {
      result = result.concat(flatten(item, cache));
    } else {
      if (!cache[item]) {
        result.push(item);
        cache[item] = 1;
      }
    }
  });
  return result.sort((a, b) => a - b);
}
```

内置函数实现：

```js
arr.toString().split(',');
```

### 自己实现 Object.create

```js
function create(proto, propertiesObject) {
    // 不支持 null、undefined 等乱七八糟的值
    function F() {}
    F.prototype = proto;

    let f = new F();

    Object.defineProperties(f, propertiesObject);

    return f;
}
```

### 如何判断数组

他们之间有什么区别和优劣

```js
Object.prototype.toString.call()
instanceof
Array.isArray
```

`Object.prototype.toString.call()` 取得是对象的 `Symbol.toStringTag` 接口的值(内置对象即便没有也能被 toString 方法识别 比如 Array)，一般是类似于 `[object xxx]` 的字符串，可修改。<br>
`instanceof` 判断前者原型链上是否出现后者构造函数的 `prototype`，讲道理也可以改 <br>
`Array.isArray` 我们来看看它的规范：

1. 如果 Type(argument) 不是 `Object` 返回 false
2. 如果 argument 是一个 [Array exotic object][Array exotic object]，返回 true
3. 如果 argument 是一个 Proxy exotic object 则判断 `Array.isArray([[ProxyHandler]])`
4. return false

只要继承了 Array, 就会继承 Array 的 `Symbol.toStringTag`(其实没有，但能被识别) 和 constructor ，所以如果不改变内置值，从某种意义上将这两个方法差不多。
Array.isArray 第二步的具体实现不清楚，每个浏览器不一样，如果不存在 `Array.isArray()` 官方建议用 `Object.prototype.toString.call(arg)` 代替 所以就结果来说和前两者差不多，唯一区别是第三步，如果是个代理对象可以判断那个被代理的对象是否是 `Array` ，注意这里的代理对象不是指 `Proxy` 生成的对象(实际上 `instanceof` 和 `isArray` 都能判断 `Proxy`)，而是指外来对象，比如值 iframes 中的 `Array` 生成的对象，所以 `Array.isArray` 支持更广。

> note: 他们都支持 Proxy 对象的判断

### 属性遍历的5种方法

1. for...in <br> `for...in` 循环遍历对象自身的属性和原型链上的可枚举属性（不含 `Symbol` 属性)。
2. Object.keys <br> `Object.keys` 返回一个数组，包括自身的可枚举属性的键名(不包括 `Symbol`)。
3. Object.getOwnPropertyNames <br> `Object.getOwnPropertyNames` 返回一个数组，包括自身的所有属性(包括不可枚举，不包括 `Symbol`)的键名。
4. Object.getOwnPropertySymbols <br> `Object.getOwnPropertySymbols` 返回一个数组，包括自身的所有 `Symbol` 属性(仅) 的键名。
5. Reflect.ownKeys <br> `Reflect.ownKeys` 返回一个数组，包括自身的所有属性的键名。

```js
const c = Symbol('c');
const d = Symbol('d');

let test = Object.create(Object.create(Object.prototype, {
    pa: {
        enumerable: true,
        value: 'can be enumerated'
    },
    pb: {
        enumerable: false,
        value: 'can not be enumerated'
    }
}), {
    a: {
        enumerable: true,
        value: 'can be enumerated'
    },
    b: {
        enumerable: false,
        value: 'can not be enumerated'
    },
    [c]: {
        enumerable: true,
        value: 'symbol & can be enumerated'
    },
    [d]: {
        enumerable: false,
        value: 'symbol & can not be enumerated'
    }
});

for (let key in test) {console.log(key)} // 'a'、'pa'
Object.keys(test); // ['a']
Object.getOwnPropertyNames(test); // ['a', 'b']
Object.getOwnPropertySymbols(test); // [Symbol(c), Symbol(d)]
Reflect.ownKeys(test); // ['a', 'b', Symbol(c), Symbol(d)]
```

### Event Loop

见 blog [event loop][event_loop]

### Promise

见 github [MyPromise][MyPromise]

## Vue

### 在 Vue 中，子组件为何不可以修改父组件传递的 Prop

如果修改了，Vue 是如何监控到属性的修改并给出警告的。

原因是单向数据流，只能从父往子流动，因为父只有一个，容易追溯来源，而子可以有很多个，如果都能修改则很难追溯来源。

实际上 vue 是这样工作的：

更新视图过程中遇到组件时会处理 props, 过程类似于这样：

```js
this.$props = {};

for (let key in propsOption) {
  if (...) {
    this.$props[key] = vnode.props[key]
    Object.defineProperty(this.$props, key, {
      get() {...},
      set() {...}
    })
  }
}

```

vnode.props 是从父组件中取的值，有两种情况，非引用类型和引用类型, 类似这样

```js
let data = {
  a: '123'
  b: {
    c: '321'
  }
}

let props = {};

props.a = data.a;
props.b = data.b;
```

之后无论哪种情况你去修改 `props[key]` 都不会影响 data，这和 vue 无关，js 引用的性质

```js
props.a = '321'
props.b = {
  d: '123'
}
// data 不可能变
```

而如果改了引用对象，父组件的数据也是会跟着变得

```js
props.b.c = '123'
console.log(data.b.c) // 123 
```

这个毋庸置疑，实际上 vue 也是这样的，他并没有做什么限制所以，vue 可以子组件可以修改父组件的数据吗？答案是可以。

当然 vue 也不希望你这么做。在 `$props` 的 `set` 中 vue 如果判断出你手动修改了 `$props` 就会发出提醒，vue 是如何判断的呢？

首先 `$props` 这个对象他只有在 `render => update` 的过程中才可能被创建/修改，所以在这一过程 vue 有一个全局状态 `isUpdatingChildComponent` 来标记，在 `$props` 的 `set` 中如果发现 `isUpdatingChildComponent === false` 就意味着 `$props` 被人为修改了，vue 就会发出提示，但是并不阻止你这么做。

有趣的是因为劫持的是 `$props` 的 `set` 只有上述 `props[key]` 的情况才会发出提示，而你真正修改父组件的情况（也就是上述修改引用的情况）实际上根本无法判断，也没发发出提示。

### v-if & v-show

* `v-if` diff 时候会删除、增加节点
* `v-show` 会设置 `display` 属性

### nextTick

其实就是微任务 `microtask`。之所以是微任务也是经过各种情况累计出来的。

```js
import { noop } from 'shared/util'
import { handleError } from './error'
import { isIE, isIOS, isNative } from './env'

export let isUsingMicroTask = false

const callbacks = []
let pending = false

function flushCallbacks () {
  pending = false
  const copies = callbacks.slice(0)
  callbacks.length = 0
  for (let i = 0; i < copies.length; i++) {
    copies[i]()
  }
}

let timerFunc

if (typeof Promise !== 'undefined' && isNative(Promise)) {
  const p = Promise.resolve()
  timerFunc = () => {
    p.then(flushCallbacks)
    if (isIOS) setTimeout(noop)
  }
  isUsingMicroTask = true
} else if (!isIE && typeof MutationObserver !== 'undefined' && (
  isNative(MutationObserver) ||
  // PhantomJS and iOS 7.x
  MutationObserver.toString() === '[object MutationObserverConstructor]'
)) {
  let counter = 1
  const observer = new MutationObserver(flushCallbacks)
  const textNode = document.createTextNode(String(counter))
  observer.observe(textNode, {
    characterData: true
  })
  timerFunc = () => {
    counter = (counter + 1) % 2
    textNode.data = String(counter)
  }
  isUsingMicroTask = true
} else if (typeof setImmediate !== 'undefined' && isNative(setImmediate)) {
  timerFunc = () => {
    setImmediate(flushCallbacks)
  }
} else {
  timerFunc = () => {
    setTimeout(flushCallbacks, 0)
  }
}

export function nextTick (cb?: Function, ctx?: Object) {
  let _resolve
  callbacks.push(() => {
    if (cb) {
      try {
        cb.call(ctx)
      } catch (e) {
        handleError(e, ctx, 'nextTick')
      }
    } else if (_resolve) {
      _resolve(ctx)
    }
  })
  if (!pending) {
    pending = true
    timerFunc()
  }
  if (!cb && typeof Promise !== 'undefined') {
    return new Promise(resolve => {
      _resolve = resolve
    })
  }
}
```
简单的说 `callbacks` 存放任务的队列， `flushCallbacks` 执行任务队列

`timerFunc` 用于把 `flushCallbacks` 放入微任务队列。
调用 `nextTick` 把回调放入 `callbacks` 然后执行 `timerFunc`。

`timerFunc` 会先做一层判断，根据环境支持依次选择以下方法

1. Promise
2. MutationObserver
3. setImmediate
4. setTimeout

为什么优先选择微任务?其实不论微任务还是宏任务都有一些问题，但微任务的优点很明显，每个任务（比如触发事件）最后 vue 都会修改 dom ,我们并不需要等待页面渲染，只要 dom 更改了对我们来说就可以拿到正确的 dom ，整个过程是这样的：

task => task 的最后修改 dom => run `microtask` => ui 渲染 => task(宏任务)

也就是说如果 `nextTick` 是微任务，不管我们嵌套多少个 nexttick ，回调都能拿到正确的 dom 并且在渲染之前修改 dom 。也就是我们最终只会渲染一次。

个人理解，可以去看 [blog][event_loop]

### 3.0

#### time slicing

vue3 曾经的新功能，时间切片
[看这篇][time_slicing]

## typescript

### `?.` 操作符 和 `!.` 操作符

`!.` 操作符是 TypeScript 2.0 引入的，它的作用是告诉类型检察器我确定 `a` 肯定不是 `null` 或者 `undefined`

```ts
let a;
// ... a do something

// 告诉检察器这里 a 一定存在
a!.b();
```

也就是 `!.` 并不会转换多余的代码，也不会改变结果，如果此时 `a.b` 不存在（就像上述代码）就会报错。详情见[官方文档][non_null_operator]。

`?.` 操作符是 TypeScript 3.7 / ES10 引入的 [Optional Chaining][optional_chaining] 其作用是当编写代码时遇到 `null` 或 `undefiend` 时立即停止运行表达式:

```ts
let x = foo?.bar.baz();
```

此时 `foo` 还未定义，所以后面的代码不会执行，也就不会报错，`x` 会被赋值为 `undefiend`。

### leetcode 题目

[题目链接][leetcode_ts]

```ts
interface Action<T> {
  payload?: T;
  type: string;
}

class EffectModule {
  count = 1;
  message = "hello!";
  delay(input: Promise<number>) {
    return input.then(i => ({
      payload: `hello ${i}!`,
      type: 'delay'
    }));
  }

  setMessage(action: Action<Date>) {
    return {
      payload: action.payload!.getMilliseconds(),
      type: "set-message"
    };
  }
}

type FilterFunction<T, U> = {
  [P in keyof T]: T[P] extends U ? P : never
}[keyof T];

type Mix<T> = Promise<T> | Action<T>;

// 修改 Connect 的类型，让 connected 的类型变成预期的类型
type Connect = (module: EffectModule) => {
    [P in FilterFunction<EffectModule, Function>]: (
      input: Parameters<EffectModule[P]>[0] extends Mix<infer T> ? T : never
    ) => ReturnType<EffectModule[P]> extends Promise<infer T>
      ? T
      : ReturnType<EffectModule[P]>;
};

const connect: Connect = m => ({
  delay: (input: number) => ({
    type: 'delay',
    payload: `hello 2`
  }),
  setMessage: (input: Date) => ({
    type: "set-message",
    payload: input.getMilliseconds()
  })
});

type Connected = {
  delay(input: number): Action<string>;
  setMessage(action: Date): Action<number>;
};

export const connected: Connected = connect(new EffectModule());
```

知识点:

* 怎么从 `{a: string, b: Function}` 的类型中排除非 `function` 的字段，我知道 `{ [P in keyof T]: T[P] extends U ? P : never }` 可以做到 `{xxx: never, xxx: Function}`，但是 `never` 很奇怪， 没想到后面跟上 `[keyof T]` 就可以变成 `'b, xxx, xxx'` 这些全是类型为 `function` 的字段名。
* 获取方法的参数类型 `Parameters<Function>`
* 获取方法的返回类型 `ReturnType<Function>`
* 从 `AA<BB>` 类型中拿到 `BB` 类型，主要是 `infer` 语句： `AA<BB> extends AA<infer T> ? T : other` 其中 `T` 就是拿到的 `BB` 类型。

## 工程化

### 异常监控

#### 错误捕获

1. `try catch`: 用一个大的 `try catch` 来捕获所有错误

2. `onerror`： 事件 监听全局的 `onerror` 事件，重写 `window.onerror` 或者 `window.addEventListener('error')` 都行 (注意冒泡和捕获)
  
3. `object.onerror`：资源加载错误监控，比如 `image`

```js
let img = new Image()
img.onerror = funtion() {...}
img.src = 'haha'
```

4. `performance`：借助 `performance` 可以找到那些资源没加载。`performance.getEntriesByType('resource')` 会获得资源加载的条目 (initiatorType: img|script|css 等等)，这些都是成功的，如果知道总共有哪些资源需要加载，对比下就找到没加载成功的。如 `document.getElementsByTagName('link')` 和 `initiatorType = link` 的条目进行对比。

5. `onunhandledrejection`：监听所有未处理的 Promise reject handler。

```js
window.onunhandledrejection = function(e) {
  console.log('window', e.reason);
}

Promise.reject(123).then();
// window 123
Promise.reject(123).then(null, reason => void console.log('promise', reason));
// promise 123
```

#### 上传

* 正常 AJAX 上传
* image 上传

image 方法比较好，因为跨域友好无阻塞，性能好，无需关心返回。后端返回个 204 连返回都不需要。

### 性能监控

主要就是用 `Performance`

`Performance.getEntries()` 可以导出所有性能相关的条目，其中有内存相关`memory`、来源相关`navigation`、事件相关`timing`。

`Performance,mark()` 可以打标记，手动添加条目

`Performance.getEntriesByName()`、`Performance.getEntriesByType()` 可以筛选条目等等

[Array exotic object]:https://www.ecma-international.org/ecma-262/6.0/#sec-array-exotic-objects
[time_slicing]:https://zhuanlan.zhihu.com/p/88996118
[MyPromise]:https://github.com/jwdzzhz777/myPromise
[non_null_operator]:https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-0.html#non-null-assertion-operator
[optional_chaining]:http://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#optional-chaining
[leetcode_ts]:https://github.com/LeetCode-OpenSource/hire/blob/master/typescript_zh.md
[event_loop]:https://github.com/jwdzzhz777/blog/blob/master/articles/eventLoop.md
[how_generators_work]:https://www.freecodecamp.org/news/yield-yield-how-generators-work-in-javascript-3086742684fc/
[ECMA_generator]:https://tc39.es/ecma262/#sec-generatoryield