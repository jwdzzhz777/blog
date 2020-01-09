# 函数声明（function declaration） 与 函数表达式 （function expression）

为什么想了解这个，起因是这道题：[下面代码打印什么内容，为什么？][heading-37]

```js
var b = 10;
(function b(){
    b = 20;
    console.log(b);
})();
```

答案是 `f b() {...}`。为什么，我们从头开始看起。

定义一个函数有三种方法
* function declaration 函数声明
* function expression 函数表达式
* `Function` 构造函数

我们主要看看前两种

## function declaration

[函数声明][function declaration]比较简单，其语法是这样的：
```js
function name([param[, param,[..., param]]]) {
   [statements]
}
```
> note:`name` 是必须的

### 函数声明提升

函数声明会被提升到封闭函数或全局范围的顶部

```js
hoisted(); // logs "foo"

function hoisted() {
  console.log('foo');
}
```

函数表达式 本身不会被提升。

```js
notHoisted // undefined
notHoisted(); // TypeError: notHoisted is not a function

var notHoisted = function() {
   console.log('bar');
};
```

**带条件的函数提升**

函数声明如果嵌套在 `if` 语句中各个浏览器结论不一样：

```js
var hoisted = "foo" in this;
console.log(`'foo' name ${hoisted ? "is" : "is not"} hoisted. typeof foo is ${typeof foo}`);

if (...) {
    function foo(){ return 1; }
}

// In Chrome:
// 'foo' name is hoisted. typeof foo is undefined
//
// In Firefox:
// 'foo' name is hoisted. typeof foo is undefined
//
// In Edge:
// 'foo' name is not hoisted. typeof foo is undefined
//
// In Safari:
// 'foo' name is hoisted. typeof foo is function
```

所以建议这里用 函数表达式。

## Function expression

[函数表达式][Function expression] 的表达式是这样的：
```js
var myFunction = function [name]([param1[, param2[, ..., paramN]]]) {
   statements
};
```

这里 `name` 不是必须的，当你指定另一个 `name` 的时候他是一种特殊情况叫具名函数表达式(Named function expression, 以下称 NFE)。在 [ECMA 规范][ECMA] 中有一段描述：
> The Identifier in a FunctionExpression can be referenced from inside the FunctionExpression's FunctionBody to allow the function to call itself recursively. However, unlike in a FunctionDeclaration, the Identifier in a FunctionExpression cannot be referenced from and does not affect the scope enclosing the FunctionExpression.

也就是说在函数体内部可以访问到名为 `name（Identifier）` 的变量，从而允许你调用自身来实现递归，而具 NFE 不同，Identifier 不能在 函数表达式 所在的作用域（scope）中被引用，也不能影响 函数表达式 所在的作用域（scope）。

所以：

```js
function a() {
    console.log(a);
}

a; // f a()
a(); // f a()
a.name; // 'a'

var b = function () {
    console.log(b);
}

b; // f b()
b(); // f b()
b.name; // 'b'

var c = function d() {
    console.log('c', c);
    console.log('d', d);
}

c; // f d()
d; // Uncaught ReferenceError: d is not defined
c(); // 'c' f d() / 'd' f d()
c.name; // 'd'
```

上面代码可以看到， 这里 d 不能被全局引用也无法影响到全局。再来看看题目那种事先声明的情况：

```js
var a = function b() {
    console.log(b);
    console.log(window.b);
    b = 20;
    console.log(b);
}
var b = 10;

a();
// step.1 f b()
// step.2 1
// step.3 f b()
```

可以看到这种情况方法体内 b 是 `Identifier` 而不是全局的 b 但是为什么 `b = 20;` 这一句并没有执行呢？依旧是看 [ECMA 规范][ECMA] 中，创建一个 NFE 的步骤，它比其他情况要复杂的多：
> 1. Let funcEnv be the result of calling NewDeclarativeEnvironment passing the running execution context’s Lexical Environment as the argument
2. Let envRec be funcEnv’s environment record.
3. Call the CreateImmutableBinding concrete method of envRec passing the String value of Identifier as the argument.
4. Let closure be the result of creating a new Function object as specified in 13.2 with parameters specified by FormalParameterListopt and body specified by FunctionBody. Pass in funcEnv as the Scope. Pass in true as the Strict flag if the FunctionExpression is contained in strict code or if its FunctionBody is strict code.
5. Call the InitializeImmutableBinding concrete method of envRec passing the String value of Identifier and closure as the arguments.
6. Return closure.

仔细看 step3 和 step5 的 `CreateImmutableBinding` 和 `InitializeImmutableBinding` 将 `Identifier` 作为参数传递。虽然咱不知道他是干啥的但是通过名字咱们能猜测出来，创建、初始化**不可修改**的绑定。所以 NFE 的 方法体中 `Identifier` 变量是不可修改的。这就解释了为什么上述代码中 `b = 20;` 这句话没有起作用了。

```js
var a = function b() {
    b = 1;
    console.log(b);
}

a(); // f b()

function c() {
    c = 1;
    console.log(c);
}

c(); // 1
c // 1
```

那我们再回归题目为什么一个自执行函数也会是这样的情况呢？

### Immediately Invoked Function Expression

[自执行函数][IIFE] 的全名是 Immediately Invoked Function Expression 从名字中就可以看出来了它是一个 立即执行的函数表达式，所以他是一个 函数表达式 `Function expression`。

那么再来看一遍题目：

```js
var b = 10;
(function b(){
    b = 20;
    console.log(b);
})();
```

我们给题目中这种情况一个更明确的定义，那就是 立即执行的具名函数表达式，既然是 NFE 那就好解释了。

再来看一种情况，如果返回一个 function 会怎么样：

```js
function a() {
    return function b() {
        b = 1;
        console.log(b);
    }
}

let aa = a();

aa(); // f b()
b // Uncaught ReferenceError: b is not defined

let c = (function() {
    return function d() {
        d = 1;
        console.log(d);
    }
});

c(); // f d()
d; // Uncaught ReferenceError: b is not defined
```

也就是说在返回一个 `function a() {}` 的情况下，返回的方法还是一个 NFE 。

还有什么情况也是如此呢？MDN 中给了一个例子：

```js
var a = {
    b: function c() {
        c = 1;
        console.log(c);
    }
};

a.b(); // f c()
```

那么我**估计?** `function` 前面有表达式就是 函数表达式，否则是函数定义。

最后，参考：https://segmentfault.com/q/1010000002810093

[heading-37]:https://juejin.im/post/5d23e750f265da1b855c7bbe#heading-37
[function declaration]:https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Statements/function
[Function expression]:https://developer.mozilla.org/en-US/docs/Web/JavaScript/Reference/Operators/function
[IIFE]:https://developer.mozilla.org/en-US/docs/Glossary/IIFE
[ECMA]:http://ecma-international.org/ecma-262/5.1/#sec-13
