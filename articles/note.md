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

## 算法

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

## HTML

### preload & prefetch

```html
<link rel="preload" href="style.css" as="style">
<link rel="prefetch" href="js/chunk-xxx.js">
```

两者都是预加载资源并缓存，加载后并不执行，等真正用到时在执行，不过优先级不一样。

`prefetch` 为预先加载之后**可能**会用到的资源，浏览器会在空闲时间加载这些资源。

`preload` 为预先加载之后**一定**会用到的资源，标记了 `preload` 的资源会在页面生命周期的早期加载，再进行浏览器的主渲染，并不会阻塞进程。

## javascript

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
Array.isArray 第二步的具体实现不清楚，每个浏览器不一样，如果不存在 `Array.isArray()` 官方建议用 `Object.prototype.toString.call(arg)` 代替 所以就结果来说和前两者差不多，唯一区别是第三步，如果是个代理对象可以判断那个被代理的对象是否是 `Array` 所以 `Array.isArray` 支持更广。

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

见 blog event loop

### Promise

见 github [MyPromise][MyPromise]

## Vue

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

个人理解，具体去看 eventloop

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

`?.` 操作符是 TypeScript 3.7 引入的 [Optional Chaining][optional_chaining] 其作用是当编写代码时遇到 `null` 或 `undefiend` 时立即停止运行表达式:

```ts
let x = foo?.bar.baz();
```

此时 `foo` 还未定义，所以后面的代码不会执行，也就不会报错，`x` 会被赋值为 `undefiend`。

### leetcode 题目

题目链接][leetcode_ts]

```ts
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

[Array exotic object]:https://www.ecma-international.org/ecma-262/6.0/#sec-array-exotic-objects
[time_slicing]:https://zhuanlan.zhihu.com/p/88996118
[MyPromise]:https://github.com/jwdzzhz777/myPromise
[non_null_operator]:https://www.typescriptlang.org/docs/handbook/release-notes/typescript-2-0.html#non-null-assertion-operator
[optional_chaining]:http://www.typescriptlang.org/docs/handbook/release-notes/typescript-3-7.html#optional-chaining
[leetcode_ts]:https://github.com/LeetCode-OpenSource/hire/blob/master/typescript_zh.md