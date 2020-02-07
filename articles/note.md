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

## javascript

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

[Array exotic object]:https://www.ecma-international.org/ecma-262/6.0/#sec-array-exotic-objects
[time_slicing]:https://zhuanlan.zhihu.com/p/88996118
[MyPromise]:https://github.com/jwdzzhz777/myPromise