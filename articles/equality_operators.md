# 从一道题了解 `==`

先看一下原题：[下面代码中 a 在什么情况下会打印 1][==]

```js
var a = ?;
if(a == 1 && a == 2 && a == 3){
    console.log(1);
}
```

话不多说，我们直接看看 `==` 的 [ECMA 规范][ECMA ==]，一个平等表达式（无论 `===` 还是 `==`） `EqualityExpression ==/=== RelationalExpression` 都会先执行以下步骤：

1. 将 `lref` 设为左边表达式（`EqualityExpression`）的结果
2. 将 `lref` 设为 `GetValue(lref)` 的结果
3. 将 `rref` 设为右边表达式（`RelationalExpression`）的结果
4. 将 `rref` 设为 `GetValue(rref)` 的结果

这里注意，规范中 `GetValue(V)` 提到：
> The following [[Get]] internal method is used by GetValue when V is a property reference with a primitive base value. It is called using base as its this value and with property P as its argument. 

总之如果有，`GetValue` 会调用 `getter`

紧接着 `==` 会执行抽象比较算法 (`x == y`, Abstract Equality Comparison Algorithm)，`===` 会执行平等比较算法（`x === y`，The Strict Equality Comparison Algorithm）我们不展开了，主要看看[抽象比较算法][ECMA The Abstract Equality Comparison Algorithm]。

内容很多，各种情况有各种的处理方法，这里我就不贴了，自己去看，根据题目我直接定位算法中的两种情况（Type(y) is Number）：

> 5. If Type(x) is String and Type(y) is Number,
return the result of the comparison ToNumber(x) == y.
> 9. If Type(x) is Object and Type(y) is either String or Number,
return the result of the comparison ToPrimitive(x) == y.

简单的说一下，`ToPrimitive(Object)` 会调用对象的 `[[DefaultValue]]`, `[[DefaultValue]]` 会调用 `toString`、`valueOf`， 要注意的是类型不同调用顺序不一样，详情见 [ECME DefaultValue][ECMA DefaultValue]

那么这题就有四个切入点

1. `getValue` 也就是 `[[get]]`
2. `ToNumber`
3. `toString`
4. `valueOf`

> note: 我没找到它隐式调用的 `ToNumber` 方法是啥，可能并未开放，我估计没法用它解决

那么就可以动手解决了，

用 `defineProperty`: 
> note: (如果像题目开头 `var a = ?` 就不能用 `defineProperty` 了会报错)

```js
var i = 0;
Object.defineProperty(window, 'a', {
    get() {
        return ++this.i;
    }
});

if(a == 1 && a == 2 && a == 3){
    console.log(1);
}
```

用 `toString()` ：

```js
var a = {
    i: 0,
    toString() {
        return ++this.i;
    }
}

if(a == 1 && a == 2 && a == 3){
  console.log(1);
}
```

用 `valueOf`:
 
```js
var a = {
    i: 0,
    valueOf() {
        return ++this.i;
    }
}

if(a == 1 && a == 2 && a == 3){
    console.log(1);
}
```

ok, 这题解决了我们来深入了解下 `ToPrimitive`：

首先 js 提供很多内置方法来类型转换：`ToXXX` (详见[类型转换][toxxx])
提供给各种方法显示、隐式地调用，大多情况都是固定的规则 如：

```js
ToBoolean(+0 | -0 | NaN) === false
// 反之为 true
```

也有特殊的情况，转换对象的时候：`ToPrimitive(Object, xxx)`。
> note: `ToNumber(input)` 和 `ToString(input)` 在 `input` 为 `Object` 时会调用 `ToPrimitive`

`ToPrimitive(input, PreferredType)` 方法有两个入参，`input` 为要转换的对象，`PreferredType` 为要转换的类型提示，`PreferredType` 会被各个内置方法默认传入，如：

```js
ToNumber(input) => ToPrimitive(input, Number)
ToString(input) => ToPrimitive(input, String)
```

结合前面将的当传入为对象时 `ToPrimitive(Object, PreferredType)` 会调用 `[[DefaultValue]](hint)`，它接收 `PreferredType` 作为类型提示，现在我们来仔细了解下 `[[DefaultValue]](hint)` ：

* When `hint` is `String`：

1. 如果 `Object.toString()` 可调用且有返回值，return 返回值
2. 如果 `Object.valueOf()` 可调用且有返回值，return 返回值
3. TypeError

* When `hint` is `Number`：

1. 如果 `Object.valueOf()` 可调用且有返回值，return 返回值
2. 如果 `Object.toString()` 可调用且有返回值，return 返回值
3. TypeError

* `hint` 既不是 `String` 也不是 `Number`: 当成 `Number` 处理

> note: 如果是 Date 类型当成 `String` 处理

ok，我们再回到刚才的题目在 `==` 的情况 9 中 `ToPrimitive(x)` 没有传入任何的提示，也就是说会先调 `valueOf`：

```js
let a = {
    toString() {
        return 1;
    },
    valueOf() {
        return 2;
    }
}

let b = {
    toString() {
        return 1;
    }
}

let c = {
    valueOf() {
        return 2;
    }
}

a == 2; // true
b == 1; // true
c == 2; // true
```

再来看看其他情况：

```js
let a = {
    toString() {
        return 1;
    },
    valueOf() {
        return 2;
    }
}

Number(a); // 2
String(a); // "1"
```

最后再列以下规范中会调用 `ToPrimitive` 的情况：

* `new Date (value)`：`ToPrimitive(value)`
* `Date.prototype.toJSON(key)`：`ToPrimitive(this, Number)` 该方法用来给 `JSON.stringify(Date)` 用的
* `+` 操作符：两边都会 `ToPrimitive(value)` 哦
* `==`：是 `Object` 的那个 `ToPrimitive(value)`
* `ToString(Object)、ToNumber(Object)`：这个情况有点多咯

你以为结束了嘛，别眨眼哦，我们还可以改写对象的 `ToPrimitive` 方法：

```js
var a = {
    i: 0,
    toString() {
        return 123;
    },
    valueOf() {
        return 321;
    },
    [Symbol.toPrimitive]() {
        return ++this.i;
    }
}

if(a == 1 && a == 2 && a == 3){
    console.log(1);
}
```

这里再贴一个很有意思的题目:[请实现一个 add 函数，满足以下功能。][add]，看似柯里化，其实是类型转换，不然为啥又能当数字又能当方法调用。

ok,不得不说这题出得真有意思，最后我越来越觉得 规范 真的是个好东西了，遇事不绝看规范，答案都在里面。

[==]:https://juejin.im/post/5d23e750f265da1b855c7bbe#heading-42
[ECMA ==]:http://ecma-international.org/ecma-262/5.1/#sec-11.9.1
[ECMA DefaultValue]: http://ecma-international.org/ecma-262/5.1/#sec-8.12.8
[ECMA The Abstract Equality Comparison Algorithm]:http://ecma-international.org/ecma-262/5.1/#sec-11.9.3
[toxxx]:http://ecma-international.org/ecma-262/5.1/#sec-9
[add]:https://juejin.im/post/5d23e750f265da1b855c7bbe#heading-93