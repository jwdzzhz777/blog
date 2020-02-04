# vue

为了更深入了解 Vuejs 我决定，边读源码边自己实现一些简单的功能。

> note:文中的代码会进行缩减/修改，不然太多环境判断之类的代码不好阅读。建议边看源代码边读。

## 数据绑定

为什么从数据绑定开始，一是因为它重要，而是因为它比较独立，不依赖框架，可以单独抽出来讲。

### Observer

总所周知，Vuejs 是通过 数据劫持(`Object.defineProperty()`) 配合观察者模式来实现的数据绑定，所以首先来看看 Vuejs 实现的观察者：

```js
export class Observer {
    value: any;
    dep: Dep;

    constructor (value: any) {
        this.value = value
        this.dep = new Dep()

        Object.defineProperty(value, '__ob__', {
            value: this,
            enumerable: true,
            writable: true,
            configurable: true
        })

        if (Array.isArray(value)) {
            // 这里重写了数组的方法
            { resetArrayMethods }

            this.observeArray(value)
        } else {
            const keys = Object.keys(value)
            for (let i = 0; i < keys.length; i++) {
                // 变成响应式属性
                defineReactive(obj, keys[i])
            }
        }
    }
    observeArray (items: Array<any>) {
        for (let i = 0, l = items.length; i < l; i++) {
            new Observer(items[i])
        }
    }
}
```

`Observer` 的功能比较简单，就是将一个对象包装成可观察对象（Observable），将自己（this）绑定在对象的 `__ob__` 字段上，劫持每一个字段。如果是数组，则会进行处理，重写数组方法，并将数组内每个对象继续包装成可观察对象。

> note: 这里以及下面的 `new Observer()` 源码中对应的是一个 `function observe` 大多是各种判断的我就直接忽略了，知道就好

这里简单插一下 Vuejs 是如何重写数组的：

```js
const methodsToPatch = [ 'push', 'pop', 'shift', 'unshift', 'splice', 'sort', 'reverse' ]
export const arrayMethods = Object.create(Array.prototype)

methodsToPatch.forEach(method => {
    // 原方法
    const original = arrayProto[method]
    Object.defineProperty(arrayMethods, method, function(...args) {
        // super()
        const result = original.apply(this, args)
        // 中间代码折起来了，也比较好理解，毕竟有些操作(push)会扩充数组，这些扩充的对象也要被包装成 可观察者对象

        // 通知改变
        this.__ob__.dep.notify()
    })
})
```

其实也比较简单，类似继承的方式搞一个新的 `prototype` 重写数组方法，依旧调用原方法的同时增加通知 `ob.dep.notify()`，最后重改原型链 `target.__proto__ = arrayMethods`

接下来我们来看看 `defineReactive` 方法中 Vuejs 如何处理的数据劫持：

```js
function defineReactive (
    obj: Object,
    key: string,
    val: any,
    shallow?: boolean
) {
    const dep = new Dep()
    // 前提：属性可枚举

    // 这里有属性的 getter/setter 已经被定义过的情况，同样的 Vuejs 没有忽略它
    // 满足自己功能的同时，也调用了原方法 getter/setter ? getter/setter.call(obj)
    const getter = property && property.get
    const setter = property && property.set
    if ((!getter || setter) && arguments.length === 2) {
        // 默认值咯
        val = obj[key]
    }

    // 万一也是个对象/数组，深度劫持
    let childOb = !shallow && new Observer(val)
    Object.defineProperty(obj, key, {
        get: function reactiveGetter () {
            const value = getter ? getter.call(obj) : val
            if (Dep.target) {
                dep.depend()
                // 如果是对象
                if (childOb) {
                    childOb.dep.depend()
                    // 如果是数组
                    if (Array.isArray(value)) {
                        dependArray(value)
                    }
                }
            }
            return value
        },
        set: function reactiveSetter (newVal) {
            const value = getter ? getter.call(obj) : val
            // 前后对比
            if (newVal === value || (newVal !== newVal && value !== value)) return

            if (setter) setter.call(obj, newVal)
            else val = newVal
            // 万一设置了个新对象/数组呢
            childOb = !shallow && observe(newVal)
            // 通知
            dep.notify()
        }
    })
}

```

比较好理解，改写了属性的 getter/setter 方法（同时兼容了本身就有 getter/setter 方法的情况）在 getter/setter 时进行操作。简单的说 `Observer` 就是改造目标对象，劫持属性。

这里要注意的是在方法 `defineReactive` 中 `new Dep` 形成了闭包，在 `getter` 时才调用 `dep.depend()`，`setter` 时才通知 `dep.notify()`，这个之后讲

接下来需要一个充当主体的角色，将值多路推送给观察者(Observer)，Vuejs 中就是 `Dep` 了：

### Dep

Dep 就是个内容发布者，它依托在一个可观察对象内（Observable）可以被多个观察者订阅（Watcher），多路推送消息。

```js
class Dep {
  static target: ?Watcher;
  id: number;
  subs: Array<Watcher>;

  constructor () {
    this.id = uid++
    this.subs = []
  }

  addSub (sub: Watcher) {
    this.subs.push(sub)
  }

  removeSub (sub: Watcher) {
    remove(this.subs, sub)
  }

  depend () {
    if (Dep.target) {
      Dep.target.addDep(this)
    }
  }

  notify () {
    // stabilize the subscriber list first
    const subs = this.subs.slice()
    if (process.env.NODE_ENV !== 'production' && !config.async) {
      // subs aren't sorted in scheduler if not running async
      // we need to sort them now to make sure they fire in correct
      // order
      subs.sort((a, b) => a.id - b.id)
    }
    for (let i = 0, l = subs.length; i < l; i++) {
      subs[i].update()
    }
  }
}

// The current target watcher being evaluated.
// This is globally unique because only one watcher
// can be evaluated at a time.
Dep.target = null
const targetStack = []

function pushTarget (target?: Watcher) {
  targetStack.push(target)
  Dep.target = target
}

function popTarget () {
  targetStack.pop()
  Dep.target = targetStack[targetStack.length - 1]
}
```

结构也比较简单，`subs` 是用来存放所有的观察者（Watcher），`addSub` 和 `removeSub` 管理 `subs`，`notify` 用于通知所有的观察者（调用所有 `subs` 的 `update` 方法）。那么 `target` 相关的是什么。

首先我们知道的 `target` 装的就是观察者（Watcher）， `static target` 作为全局就有一点单例的味道，啥时候会用到它呢，我们简单梳理一下初始化的过程：

1. 各种初始化
2. observe data（重写 getter/setter）
3. 调用 `$mount` 方法，这个时候会 `new Watcher()`

这个 `Watcher` 接收一个 `render` 函数并在初始化调用（这个 `Watcher` 的作用是要渲染组件的 ，具体的下面讲），调用 `render` 函数时会获取对应的值，这个时候触发了我们重写的 `getter` ，触发的过程类似这样的：

```js
pushTarget(watcher)
getter()
popTarget()
```

那么结合上面的 `Observer`，每个子属性（`defineReactive`处理过）都有一个 `dep`，这个子属性本身可能也是个对象被处理成可观察对象（Observable）, 它也有个 `dep`，那么子属性的子属性..（套娃）这些 `dep` 都被同一个 `Watcher` 订阅了（上述的那个）。

如果这个时候遇到嵌套的情况，比如 `Router` 或 `Vue.extend` 这时候上面的流程还会再来一次，这时候会生成一个新的 观察者（Watcher2）来观察新的数据：

```js
pushTarget(watcher)
(getter() {
    pushTarget(watcher2)
    getter2()
    popTarget()
})()
popTarget()
```

类似于这样，这个时候 `targetStack` 就派上用场了，它保留了之前的 `Watcher`，因为是单例的，新的 `dep` 会被 `Watcher2` 给订阅，完事移除 `Watcher2` 继续完成之前的 `watcher` 的订阅，有一点洋葱圈模型的味道。

### Watcher

`Watcher` 作为观察者，其主要作用就是接收改变的通知并作出反应（渲染 or 通知给我们）。它的主要构成是这样的：

```js
class Watcher {
  vm: Component;
  cb: Function;
  deps: Array<Dep>;
  newDeps: Array<Dep>;
  depIds: SimpleSet;
  newDepIds: SimpleSet;
  getter: Function;
  value: any;

  constructor (
    vm: Component,
    expOrFn: string | Function,
    cb: Function,
    options?: ?Object,
    isRenderWatcher?: boolean
  ) {
    // options
    this.cb = cb
    this.deps = []
    this.newDeps = []
    this.depIds = new Set()
    this.newDepIds = new Set()
    // parse expression for getter
    if (typeof expOrFn === 'function') {
      this.getter = expOrFn
    } else {
      this.getter = parsePath(expOrFn)
    }
    this.value = this.lazy
      ? undefined
      : this.get()
  }

  /**
   * Evaluate the getter, and re-collect dependencies.
   */
  get () {
    pushTarget(this)
    let value
    const vm = this.vm

    value = this.getter.call(vm, vm)
    traverse(value)

    popTarget()
    this.cleanupDeps()

    return value
  }

  /**
   * Subscriber interface.
   * Will be called when a dependency changes.
   */
  update () {
    /* istanbul ignore else */
    if (this.lazy) {
      this.dirty = true
    } else if (this.sync) {
      this.run()
    } else {
      queueWatcher(this)
    }
  }

  /**
   * Scheduler job interface.
   * Will be called by the scheduler.
   */
  run () {
    const value = this.get()
    const oldValue = this.value
    this.value = value
    this.cb.call(this.vm, value, oldValue)
  }

  /**
   * Add a dependency to this directive.
   */
  addDep (dep: Dep) {
    const id = dep.id
    if (!this.newDepIds.has(id)) {
      this.newDepIds.add(id)
      this.newDeps.push(dep)
      if (!this.depIds.has(id)) {
        dep.addSub(this)
      }
    }
  }

  /**
   * Clean up for dependency collection.
   */
  cleanupDeps () {
    let i = this.deps.length
    while (i--) {
      const dep = this.deps[i]
      if (!this.newDepIds.has(dep.id)) {
        dep.removeSub(this)
      }
    }
    let tmp = this.depIds
    this.depIds = this.newDepIds
    this.newDepIds = tmp
    this.newDepIds.clear()
    tmp = this.deps
    this.deps = this.newDeps
    this.newDeps = tmp
    this.newDeps.length = 0
  }
}
```

我保留了一些我认为重要的部分，我们来看看逐个了解一下：

`get` ：首先构造函数其实干的事很少，做了一些简单的赋值后直接调用了 `get` 方法，`get` 上面有提到（Dep getter 触发），就是设置 `Dep.target` 调用 `expOrFn`，主要目的就是订阅，也就是我们所说的依赖收集。

`expOrFn`：它作为参数传入，其实就是一个 `getter` 触发器，从名字可以看出来它可能是一个表达式，也可能是一个方法。这里就要先提一下哪些情况会新建一个 `Watcher`：

* 初始化(`$mount`)：上面Dep 初始化有提到，这个时候 `expOrFn` 是一个 渲染函数 `() => vm._update(vm._render(), hydrating)` 解析模版渲染页面的同时触发data的 `getter`（具体后面讲），这个时候生成的 `Watcher` 意义重大，控制着 dom 的更新。

* `watch`、 `$watch`：这个时候 `expOrFn` 就是 key ，他会被 `parsePath` 处理成方法调用（`parsePath` 可以处理 `a.b.c` 这种形势的 key）， 还会传入一个 `cb` 方法，在接收更新之后调用。

```js
Vue.extend({
  watch: {
    key: (val) => cbFunction
  }
})
// or
vm.$watch('key', cbFunction)
```

* `computed`：稍稍有点不一样，`expOrFn` 就是我们定义的方法，方法内所有被触发 `getter` 的属性都会被当前的 `Watcher` 订阅，`Watcher` 接收到更新消息后还是会调用 `expOrFn` 然后将返回值设为 `value` (key 会被挂在 vm 上，取的就是这个 `value`)

```js
Vue.extend({
  computed: {
    key: cbFunction
  }
})
```

`traverse`: 深度遍历对象，把所有 `getter` 都触发一遍（我不干啥，我就看看）。

`addDep`：`Dep.depend` 就是调用此方法，用于把当前 `Watcher` 存入 `dep` 实例（来来回回有点绕）

`cleanupDeps`：以及 `deps`、`newDeps`、`depIds`、`newDepIds` 这些个玩意，你只要理解他们是防止重复订阅的就可以了。

`update`、`run`：通知更新（`dep.notify`）后调用的就是 `update` 方法,如果是同步的，调用 `run` 方法，`run` 干的事也很简单就是再调一次 `get` 要么重新渲染，要么 `cb` 给我们（此时会重新处理一次订阅）然后更新 `watch` 的 `value`。如果是异步的 调用 `queueWatcher`。

> note: 关于 `queueWatcher` 的东西在 `scheduler.js` 中，也就是所谓的调度器，简单理解就是放到任务队列延迟执行,执行的任务还是 `update` 方法。

#### scheduler

调度器不是很重要，简单提一下（需要了解 `nextTick` 、`event loop`），可跳过。

```js
const queue = []
let waiting = false
let flushing = false

function flushSchedulerQueue() {
  flushing = true;
  // 冲刷 queue
  flushing = waiting = false;
}

function queueWatcher (watcher: Watcher) {
  if (!flushing) {
    queue.push(watcher)
  } else {
    // if already flushing, splice the watcher based on its id
    // if already past its id, it will be run next immediately.
    let i = queue.length - 1
    while (i > index && queue[i].id > watcher.id) {
      i--
    }
    queue.splice(i + 1, 0, watcher)
  }
  // queue the flush
  if (!waiting) {
    waiting = true
    nextTick(flushSchedulerQueue)
  }
}
```

首先我们知道 `nextTick` 会把 `flushSchedulerQueue` 放进微任务队列(`microtask queue`)。那么 `waiting` 状态就比较好理解，他保证了 `microtask queue` 中只有一个 `flushSchedulerQueue` 任务。而 `flushing` 为 true 的情况发生在 `flushSchedulerQueue` 正在执行时，根据 `watcher.id` 往 `queue` 中插任务如果没有 ‘过号’ 那么新插进来的也会被执行掉，如果 ‘过号’ 了那么保留在 `queue` 等待下一次冲刷。
整个过程模型和微任务的流程模型简直一毛一样。

***

ok, 至此数据绑定相关的东西（也就是 `core/observer` 下的东西）我们都过了一遍了。现在我们把他们串联起来，来看看 `Vue` 的 ‘心路历程’

```js
// 应用初始化
_init() {
  // 初始化数据相关，当然前后还有其他的许多初始化，Lifecycle、Events 等等
  initState() {
    // 处理 data
    initData() {
      // 这里会将 data 可观察化 也就是设置 getter/setter
      observe() {
        new Observer();
      }
      proxy() // 代理到 vm 实例上
    }
    // data 处理好就可以被监听器订阅了
    initComputed() { new Watcher() }
    initWatch() { new Watcher() }
  }
  // 解析模版、渲染
  $mount() { 
    compileToFunctions() {
      // 这里会将模版处理成渲染函数，类似这样：
      vm.$option.render = with(this) {
        return _c('div', ...) // _c 就是我们用的 createElement 方法
      }
    }
    mountComponent() {
      updateComponent = () => vm._update(vm._render(), hydrating);
      new Watcher(vm, updateComponent); // 调用 updateComponent， 渲染页面， 触发 getter ，依赖收集
    }
  }
}
```

大概就这样，里面很多细枝末节就不提了，这些方法名都是 `Vue` 里面存在的，一搜就能找到，这里提一下 `proxy`：
此 `proxy` 不是我们想的 `es6 proxy` 他的作用是将 `$data` 的属性代理到 `vm` 实例上，方便我们 `this.xxx` 调用。

## Virtual DOM

ok 我们了解完了 `Vuejs` 数据绑定的过程，接下来我们看看编译渲染 DOM 的过程。之前提到，在实例化方法 `_init` 的最后会调用 `$mount` 来解析模版、渲染，现在我们来仔细看看。

### Compiler

首先是编译，`$mount` 中调用 `compileToFunctions` 将 `<template>` 文本转化成一个 `render` 函数，我们来看看 `compileToFunctions` 都做了什么：

```js
function baseCompile (
  template: string,
  options: CompilerOptions
): CompiledResult {
  const ast = parse(template.trim(), options)
  // ast 优化
  if (options.optimize !== false) {
    optimize(ast, options)
  }
  const code = generate(ast, options)
  return {
    ast,
    render: code.render,
    staticRenderFns: code.staticRenderFns
  }
}

function createCompiler (baseOptions: CompilerOptions) {
  function compile (
    template: string,
    options?: CompilerOptions
  ): CompiledResult {
    const finalOptions = Object.create(baseOptions)
    // 省略了一些 option 的处理

    const compiled = baseCompile(template.trim(), finalOptions)

    return compiled
  }

  return {
    compile,
    compileToFunctions: createCompileToFunctionFn(compile)
  }
}
```

提取了其中代码，把生成方法和 option 的处理部分删掉了，完整的可以到 `src/compile` 中去看，`compileToFunctions` 方法由 `createCompiler` 方法生成，不过主要的内容在 `baseCompile` 方法中，我们主要来看看这个方法。

里面就两步，`parse`，`generate`，我们依次来看。

#### parse

`parse`方法的代码都在 `compile/parse` 中，其主要功能就是将 `template` 编译成 AST。我们先了解以下语法树的基本结构：

```js
function createASTElement (
  tag: string,
  attrs: Array<ASTAttr>,
  parent: ASTElement | void
): ASTElement {
  return {
    type: 1,
    tag,
    attrsList: attrs,
    attrsMap: makeAttrsMap(attrs),
    rawAttrsMap: {},
    parent,
    children: []
  }
}
```

因为是将 `html` 代码转换成 AST 所以结构简单多了，主要是标签名 `tag` (`<tag>`) 和 属性 `attrsMap`（key: value）以及层级关系 `children`,`parent`。

接下来了解下 `html-parse` ，他是转换 AST 的核心代码，其本身来自于一个开源代码的扩展：[simplehtmlparser][simplehtmlparser]。代码就不贴了，自行了解下。

> note: 注意这不是 `Vuejs` 的源码，而是在这之上做的改造，我们借助它来简单了解

其实就是正则判断，判断 `template` 文本的开头是什么，对应有这么几种情况：

* `<!--`开头，也就是注释，这种情况再找到结尾 `-->` 直接删了。
* `<!DOCTYPE` 开头，文档类型，和上面一样，删了。
* `<name` 开头，开始标签(`start tag`)，执行 `parseStartTag`
* `</name` 开头，结束标签(`end tag`)，执行 `parseEndTag`

那么我们回归 `Vuejs` 本身，其实主要逻辑差不多，先简单看看流程模型：

```js
function parse(tamplate) {
  let currentParent, stack;
  parseHTML(tamplate, start, end) {
    if (startTag) {
      let match = parseStartTag(template)
      start(match)
    }
    if (endTag) {
      parseEndTag(template);
      end()
    }

    function parseStartTag() {}
    function parseEndTag() {}
  }

  function start() {}
  function end() {}
}
```

代码精简了很多，主要的解析流程就是如此，首先是 `parseHTML` 里面是如何解析的，我们结合上面了解的一起看：

```js
const attribute = /^\s*([^\s"'<>\/=]+)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const dynamicArgAttribute = /^\s*((?:v-[\w-]+:|@|:|#)\[[^=]+\][^\s"'<>\/=]*)(?:\s*(=)\s*(?:"([^"]*)"+|'([^']*)'+|([^\s"'=<>`]+)))?/
const ncname = `[a-zA-Z_][\\-\\.0-9_a-zA-Z${unicodeRegExp.source}]*`
const qnameCapture = `((?:${ncname}\\:)?${ncname})`
const startTagOpen = new RegExp(`^<${qnameCapture}`)
const startTagClose = /^\s*(\/?)>/

function parseStartTag(html) {
  const start = html.match(startTagOpen)
  if (start) {
    const match = {
      tagName: start[1],
      attrs: [],
      start: index
    }
    advance(start[0].length)
    let end, attr
    while (!(end = html.match(startTagClose)) && (attr = html.match(dynamicArgAttribute) || html.match(attribute))) {
      attr.start = index
      advance(attr[0].length)
      attr.end = index
      match.attrs.push(attr)
    }
    if (end) {
      match.unarySlash = end[1]
      advance(end[0].length)
      match.end = index
      return match
    }
  }
}
```

首先是 `parseStartTag` 方法，其通过正则查找类似 `<tagName` 的文本，记录其位置并通过子表达式得到 `tagName`，`advance` 方法用于将匹配到的值剔除，`while` 循环中不断通过正则查找并记录 `attribute` 然后删除，当匹配到 `startTagClose` 时结束(`>` 或 `/>`)，
记录子表达式（`/` 或 ` `），最后得到一个 `match` 对象，大概长这样：

```js
match = {
  tagName: 'div',
  attrs: [
    ['v-model="xxx"', 'v-model', '=', 'xxx']
  ],
  start: Number,
  end: Number,
  unarySlash: '/' || ''
}
```

得到 `match` 对象后调用 `startHandler` 进一步处理，现在的 `attrs` 还是字符串（不过已经把属性名和值都取到了，这个正则还是给力的），先处理成 `Map` ，然后才是创建 AST，然后经过一些处理（`v-for`、`v-if` 之类的属性），最后放入 `stack` 中。

```js
function start(match) {
  let tag = match.tagName;
  let attrs = ...; // 处理成 map
  let element: ASTElement = createASTElement(tag, attrs, currentParent)
  processPre(element)
  processRawAttrs(element)
  processFor(element)
  processIf(element)
  processOnce(element)

  if (!unary) {
    currentParent = element
    stack.push(element);
  } else {
    currentParent.children.push(element)
    element.parent = currentParent
  }
}
```

这里说明以下，`stack` 和 `currentParent` 在父级作用域，也就是 `parse` 方法中，用来处理父子关系。而 `unary` 是用来判断 `element` 是不是非特殊标签 (`<br/>`这种)，并且是不是 `/>` 结尾。如果是就传入 `stack` 继续处理，直到遇到 `endTag`：

```js
function parseEndTag (html) {
  let endTagMatch = html.match(endTag)
  advance(endTagMatch[0].length)
  ... // 主要还是判断那些个什么<br />啦<p /> 啦
  end(endTagMatch[1])
}

function end(tagName) {
  const element = stack[stack.length - 1]
  // pop stack
  stack.length -= 1
  currentParent = stack[stack.length - 1]
  currentParent.children.push(element)
  element.parent = currentParent
}
```

`end` 的主要功能就是取出 `stack` 栈顶，和其前面一项确认父子关系，把 `currentParent` 设置为取出后的栈顶。

简单的说：

* 解析到 `startTag` 就创建 `ASTElement`。
   如果是 `/>` 就和 stact 最后一项（`currentParent`）确认父子关系
   不是就将 `element` 推入 `stack`
* 解析到 `endTag` 就取出栈尾（`element`），`currentParent` 往前移(新的栈尾)，
  `element` 和 `currentParent` 确认父子关系

如此不断的循环。

这些就是 `parse` 的主要功能了，当然还有其他的，比如 `text-parse` 主要负责文本的处理(主要处理双括号的语法`{{expression}}`，转换成 `_s(expression)`)，而`filter-parse` 负责管道的处理 (`{{value | filter}}` => `_s(_f("filter")(value))`)，注意这里都是处理成字符串哦，至于 `_s`、`_f` 是什么，我们接下来讲。最后看一下例子，之后我们也会按照这个例子走下去：

##### examples

```html
<div id="editor">
  <input v-model="input"></input>
  {{input | nothing}}
</div>

<script>
new Vue({
  el: '#editor',
  data: {
    input: 'input'
  },
  filter: {
    nothing(a) {
      return a;
    }
  }
})
</script>
```

`parse` 方法得到：

```js
ast = {
  type: 1,
  tag: 'div',
  attrsList: [{name: 'id', value: 'editor'}]
  attrsMap: {id: 'editor'},
  children: [
    {
      type: 1,
      tag: 'input',
      attrsList: [{name: 'v-model', value: 'input'}]
      attrsMap: {'v-model': 'input'},
      parent: ast,
      events: {
        input: {value: '"if($event.target.composing)return;input=$event.target.value"'}
      }
      children: []
    },
    {
      type: 2,
      expression: `"\n      "+_s(_f("nothing")(input))+"\n    "`,
      tokens: [{'@binding': "_f("nothing")(input)"}]
    }
  ]
}
```

> note: 提一嘴，`type` 1 是元素， 3 是注释，其他当 文本处理

可以看到，除了解析 `标签` , `属性` , `父子关系` 之外，遇到 `vue` 特性（`@click` 啥的）会有特殊的处理：`{{expression}}` 会被解析成 `_s(expression)`，`filter` 是 `_f(expression)`，`@name="expression"` 则会在 `ASTElement.event` 上多加一个字段 `name: expression`，`v-if` 则会在 `ASTElement` 上多加标记(`ifProcessed: expression`) 等等，就不一一列了。

#### codegen

`parse` 方法将模版解析成 `ast`，那么接下来 `generate` 方法将会把 `ast` 转化成一个 `render` 函数，我们来简单看看：

```js
function generate(ast) {
  const code = ast ? genElement(ast) : '_c("div")'
  return {
    render: `with(this){return ${code}}`
  }
}

function genElement(ast) {
  if (el.staticRoot) genStatic(el)
  else if (el.once) genOnce(el)
  else if (el.for) genFor(el)
  else if (el.if) genIf(el)
  else if (el.tag === 'template') genChildren(el)
  else if (el.tag === 'slot') genSlot(el)
  else {
    if (el.component) {
      code = genComponent(el.component, el, state)
    } else {
      let data = genData(el, state)
      let children = genChildren(el, state, true)
      let code = `_c('${el.tag}'${
        data ? `,${data}` : '' // data
      }${
        children ? `,${children}` : '' // children
      })`
    }
  }
  return code;
}

function genChildren(el) {
  return `[${el.children.map(c => genElement(c)).join(',')}]${
    normalizationType ? `,${normalizationType}` : ''
  }`
}

function genComponent (componentName, comment: ASTText): string {
  const children = el.inlineTemplate ? null : genChildren(el)
  return `_c(${componentName},${genData(el, state)}${
    children ? `,${children}` : ''
  })`
}

function genData (el: ASTElement, state: CodegenState): string {
  let data = '{'
  // directives first.
  // directives may mutate the el's other properties before they are generated.
  const dirs = genDirectives(el, state)
  if (dirs) data += dirs + ','
  if (dirs) data += dirs + ','
  if (el.key) data += `key:${el.key},`
  if (el.ref) data += `ref:${el.ref},`
  if (el.refInFor) data += `refInFor:true,`
  if (el.pre) data += `pre:true,`
  if (el.component) data += `tag:"${el.tag}",`
  if (el.attrs) data += `attrs:${genProps(el.attrs)},`
  if (el.props) data += `domProps:${genProps(el.props)},`
  if (el.events) data += `${genHandlers(el.events, false)},`
  if (el.nativeEvents) data += `${genHandlers(el.nativeEvents, true)},`
  if (el.slotTarget && !el.slotScope) data += `slot:${el.slotTarget},`

  data = data.replace(/,$/, '') + '}'

  return data
}
```

截取了其中一小部分，主要就是通过 `genElement` 生成类似 `_c(tagName, Data, [children])` 这样的字符串，之后会被当作表达式调用，`_c` 方法其实就是 `render` 函数中提供的 `createElement` 方法。

接触过 `render` 函数的话应该很好理解，也就是说不管怎么样最终都会回到 `render` 函数，如果你直接用 `render` 函数还能提高性能（省去了 `parse` 的过程）。那么我们看看[官方文档][createElement]。

`createElement` 有三个参数，第一个参数是 `html` 的标签名字，直接通过 `element.tag` 得到(`parse` 解析取得)。第二个参数是一个数据对象，其值通过 `genData` 方法处理（普通属性、dom属性、style、class、事件等等），第三个参数是子虚拟节点的集合，其也由 `createElement` 构建而成，在 `genElement` 中循环处理 `element.children`。

最终 `generate` 会将 `ast` 包装成类似 `with(this){return _c(...)}` 的字符串，还是上面的<a href='#examples'>例子</a>，最终会输出这样：

```js
const render = `with(this){return _c('div',{attrs:{"id":"editor"}},[_c('input',{directives:[{name:"model",rawName:"v-model",value:(input),expression:"input"}],domProps:{"value":(input)},on:{"input":function($event){if($event.target.composing)return;input=$event.target.value}}}),_v("\n      "+_s(_f("nothing")(input))+"\n    ")])}`
```

最后我们通过 `Function` 将其转换成一个方法，并绑定在 `vm.options` 上：

```js
vm.options.render = new Function(code);
```

还是之前的<a href='#examples'>例子</a>，其结果大概是这样：

```js
vm.options.render = function() {
  with (this) {
    return _c(
      'div',{attrs:{"id":"editor"}},
      [
        _c(
          'input',
          {
            directives:[
              {
                name:"model",
                rawName:"v-model",
                value:(input),
                expression:"input"
              }
            ],
            domProps:{"value":(input)},
            on:{"input":function($event){if($event.target.composing)return;input=$event.target.value}}
          }
        ),
        _v("\n      "+_s(_f("nothing")(input))+"\n    ")
      ]
    )
  }
}
```

注意了，之所以这么做，以及使用 `with` 语法，主要原因是 `Compiler` 并未对表达式进行解析处理，而是完整保留了表达式，通过 `with` 语句保证真正执行 `render` 函数时能够正确执行表达式，并从 `vm` 实例中取到正确的值。

ok,至此 `Compiler` 的任务完成。

### VNode

之前提到在生成 `render` 函数之后 `new Watcher` 时会传入一个渲染函数：

```js
const updateComponent = () => {
  vm._update(vm._render(), hydrating)
}
```

抛去各种判断，`_render` 方法其实就是调用了 `render.call(vm)` 得到 `vnode`，而 `render` 中调用各种 `_x()`
 方法处理各种情况，这些代码都在 `core/instance/render-helpers` 中，有兴趣可以看下，其中最主要的 `_c`  也就是 `createElement` ，代码位于 `core/vdom/create-element` 中。其主要功能也很简单，说白了就是通过传入的 `tagName`、 `Data` 和 `[children]` 生成 `vnode`:

 ```js
function createElement(
  context: Component, // vm 实例
  tag: any,
  data: any,
  children: any
) {
  // 内置的 html 标签 或 svg 标签
  if (isHTMLTag(tag) || isSVG(tag)) {
    vnode = new VNode(tag, data, children, undefined, undefined, context)
  } else if (isDef(Ctor = resolveAsset(context.$options, 'components', tag))) { // 处理组件
    vnode = createComponent(Ctor, data, context, children, tag)
  } else {
    // 其他不知道的莫名其妙的标签名
    vnode = new VNode(tag, data, children, undefined, undefined, context)
  }
}
```

这里说一下，如果遇到不认识的 `tag` 会判断 `$options.components` 中有没有相应 `name` 的对象，返回相应的对象 `Ctor`，如果有，会执行 `createComponent` ：

```js
function createComponent(
  Ctor: Class<Component> | Function | Object | void,
  data: ?VNodeData,
  context: Component,
  children: ?Array<VNode>,
  tag?: string
) {
  if (isObject(Ctor)) {
    // 这个就是当前环境的 Vue class 
    // Vue.options._base = Vue
    const baseCtor = context.$options._base

    Ctor = baseCtor.extend(Ctor)

    const name = Ctor.options.name || tag
    const vnode = new VNode(
      `vue-component-${Ctor.cid}${name ? `-${name}` : ''}`,
      data, undefined, undefined, undefined, context,
      { Ctor, data, data.on, tag, children }
    )
  }
}
```

其实就是执行 `Vue.extend(Ctor)` 生成一个带有 `componentOptions` 的 `vnode`。 说到底最终都是生成 `vnode` 那么我们来了解以下它的结构： 

```js
class VNode {
  tag: string | void;
  data: VNodeData | void;
  children: ?Array<VNode>;
  text: string | void;
  elm: Node | void;
  ns: string | void;
  context: Component | void; // rendered in this component's scope
  key: string | number | void;
  componentOptions: VNodeComponentOptions | void;
  componentInstance: Component | void; // component instance
  parent: VNode | void; // component placeholder node

  // strictly internal
  raw: boolean; // contains raw HTML? (server only)
  isStatic: boolean; // hoisted static node
  isRootInsert: boolean; // necessary for enter transition check
  isComment: boolean; // empty comment placeholder?
  isCloned: boolean; // is a cloned node?
  isOnce: boolean; // is a v-once node?
  asyncFactory: Function | void; // async component factory function
  asyncMeta: Object | void;
  isAsyncPlaceholder: boolean;
  ssrContext: Object | void;
  fnContext: Component | void; // real context vm for functional nodes
  fnOptions: ?ComponentOptions; // for SSR caching
  devtoolsMeta: ?Object; // used to store functional render context for devtools
  fnScopeId: ?string; // functional scope id support

  constructor (
    tag?: string,
    data?: VNodeData,
    children?: ?Array<VNode>,
    text?: string,
    elm?: Node,
    context?: Component,
    componentOptions?: VNodeComponentOptions,
    asyncFactory?: Function
  ) {
    this.tag = tag
    this.data = data
    this.children = children
    this.text = text
    this.elm = elm
    this.ns = undefined
    this.context = context
    this.fnContext = undefined
    this.fnOptions = undefined
    this.fnScopeId = undefined
    this.key = data && data.key
    this.componentOptions = componentOptions
    this.componentInstance = undefined
    this.parent = undefined
    this.raw = false
    this.isStatic = false
    this.isRootInsert = true
    this.isComment = false
    this.isCloned = false
    this.isOnce = false
    this.asyncFactory = asyncFactory
    this.asyncMeta = undefined
    this.isAsyncPlaceholder = false
  }

  // DEPRECATED: alias for componentInstance for backwards compat.
  /* istanbul ignore next */
  get child (): Component | void {
    return this.componentInstance
  }
}
```

`vnode` 构造函数主要接收的就是 `createElemetn` 的参数，其本质也是生成一个结构对象，不过和 `ast` 不同的是，`ast` 保留了表达式，而 `vnode` 是将表达式执行完的结果，他是静态的，也就是说与之对应的是一个确定的渲染结果。依旧是那个<a href='#examples'>例子</a>：

```js
const vnode = {
  tag: 'div',
  data: {
    attrs: {id: 'editor'}
  },
  children: [
    {
      tag: 'input',
      data: {
        directives: [{name: 'model', rawName: 'v-model', value: 'input', expression:'input'}]
      },
      ...others
    },
    {
      tag: undefined,
      data: undefined,
      children: undefined,
      text: '↵      input↵    ',
      ...others
    }
  ],
  ...others
}
```

至此，我们已经通过 `render` 函数和 `vm` 实例的数据生成了 `vnode`，要注意的是，`render` 函数是在初始化生成的，是不会变的，但是我们的数据层是会变的，每次调用 `_render` 都有可能生成不同的 `vnode`，接着就可以通过 `vnode` 去生成真正的 `dom` 节点了。

### patch

生成 `vnode` 之后就是将其传入 `_update`，其内部本质就是调用 `patch` 方法，代码在 `core/vdom/patch`，我们来大致看一下：

```js
function patch(oldVnode, vnode, hydrating, insertedVnodeQueue) {
  if (isUndef(oldVnode)) createElm(vnode, insertedVnodeQueue);
  else {
    const isRealElement = isDef(oldVnode.nodeType)
      if (!isRealElement && sameVnode(oldVnode, vnode)) {
        // patch existing root node
        patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
      } else {
        if (isRealElement) {
          // mounting to a real element
          // check if this is server-rendered content and if we can perform
          // a successful hydration.
          if (oldVnode.nodeType === 1 && oldVnode.hasAttribute(SSR_ATTR)) {
            oldVnode.removeAttribute(SSR_ATTR)
            hydrating = true
          }
          if (isTrue(hydrating) && hydrate(oldVnode, vnode, insertedVnodeQueue)) return oldVnode
          // either not server-rendered, or hydration failed.
          // create an empty node and replace it
          oldVnode = emptyNodeAt(oldVnode)
        }
        // replacing existing element
        const oldElm = oldVnode.elm
        const parentElm = nodeOps.parentNode(oldElm)
        // create new node
        createElm(vnode, insertedVnodeQueue, parentElm)
      }
  }
}
```

我们一步步来，`patch` 方法接收两个参数 `vnode` 就是我们新生成的虚拟DOM, 而 `oldVnode` 是上一个状态的 `DOM element` 对象，它也可以是初始化时的 `el` 指向的元素：

```js
<div id="app"></div>

new Vue({
  el: '#app',
  ...
})
```

比如这个时候 `oldVnode` 就是 `<div id="app"></div>`。所以哪怕是初始化的时候 `oldVnode` 也是存在的。

接着看方法体，有下面几种情况：

1. 首先是判断 `oldVnode` 是否存在，比如操作对象是 `Component` 时他就是空的，这个时候要创建一个新的根元素 `createElm(vnode)`。

2. 如果 `oldVnode` 不为空，先拿到 `oldVnode.nodeType`。注意，`oldVnode` 他可能是真实的 `DOM element` 对象，也有可能是个虚拟DOM `vnode`,这里的 [nodeType][nodeType] 是 `DOM element` 的属性，当它值为 1 时代表它是个元素 (`element`)，我们通过 `nodeType` 来判断 `oldVnode` 是虚拟DOM 还是 真实DOM，也就是 `isRealElement`。

3. 当 `oldVnode` 是虚拟DOM，且新旧虚拟DOM相同时（`!isRealElement && sameVnode(oldVnode, vnode)`）调用 `patchVnode(oldVnode, vnode)`，一般来讲第一次渲染时 `oldVnode` 是真实DOM，更新节点时 `oldVnode` 是虚拟DOM。

4. 当 `oldVnode` 是真实DOM时，说明是第一次调用，则需要判断一个特殊情况： `hydrating` 是否为 `true`，`hydrating` 是一个是否保持的标记，他的作用是是否沿用 `oldVnode`。比如说服务端渲染的情况，我们第一次 `patch` 时 `oldVnode` 已经是渲染好了的真实DOM了，此时我们要做的就是存一下真实DOM(`oldVnode`)即可。实际调用 `hydrate` 方法。

5. 剩下的就是 `oldVnode` 是真实DOM、`hydrating === false` 和 `oldVnode` 是虚拟DOM、`!sameVnode(oldVnode, vnode)`
两种情况，这两种情况直接用新的替换老的 (`createElm(vnode)`)。

>note: 这里的 `sameVnode` 不是 diff !! 它只是确认根节点是否被替换了。

ok，各种情况我们总结完了，我们来一一细看。

#### `createElm`

当处于上述情况 1 和 5 时，会调用 `createElm` 方法，该方法的作用是将虚拟DOM 转换为真实DOM 并替换当前的真实DOM(`oldVnode`)

```js
function createElm(
  vnode,
  insertedVnodeQueue,
  parentElm,
  refElm,
  nested,
  ownerArray,
  index
) {
  // 递归调用 处理 children 时 vnode 指向正确的 child (ownerArray[index])
  if (isDef(vnode.elm) && isDef(ownerArray)) {
    // This vnode was used in a previous render!
    // now it's used as a new node, overwriting its elm would cause
    // potential patch errors down the road when it's used as an insertion
    // reference node. Instead, we clone the node on-demand before creating
    // associated DOM element for it.
    vnode = ownerArray[index] = cloneVNode(vnode)
  }
  // 处理 Component
  if (createComponent(vnode, insertedVnodeQueue, parentElm, refElm)) {
    return
  }
  const data = vnode.data
  const children = vnode.children
  const tag = vnode.tag

  if (tag) {
    vnode.elm = document.createElement(tag)
    // style 的 scope 相关
    setScope(vnode)

    createChildren(vnode, children, insertedVnodeQueue)
    if (data) {
      invokeCreateHooks(vnode, insertedVnodeQueue)
    }
    insert(parentElm, vnode.elm, refElm)
  }
}

function createChildren (vnode, children, insertedVnodeQueue) {
  if (Array.isArray(children)) {
    for (let i = 0; i < children.length; ++i) {
      createElm(children[i], insertedVnodeQueue, vnode.elm, null, true, children, i)
    }
  } else if (isPrimitive(vnode.text)) {
    // 不是数组直接 document.appendChild 到父元素上。
    nodeOps.appendChild(vnode.elm, nodeOps.createTextNode(String(vnode.text)))
  }
}

function invokeCreateHooks() {
  for (let i = 0; i < cbs.create.length; ++i) {
    cbs.create[i](emptyNode, vnode)
  }
  i = vnode.data.hook // Reuse variable
  if (isDef(i)) {
    if (isDef(i.create)) i.create(emptyNode, vnode)
    if (isDef(i.insert)) insertedVnodeQueue.push(vnode)
  }
}
```

`createComponent`： 处理当前是组件的情况，这个我们稍后再讲。

`document.createElement(tag)` 生成一个元素，然后赋值到 `vnode.elm`，此时元素只是一个空壳，什么都没有。

`setScope` 方法是用来处理样式的 `scope` 属性。

`createChildren` 用来递归调用 `createElm` 处理 `children`,`createElm` 的参数 `parentElm` 指的是父元素，`ownerArray[index]` 为当前子 `vnode`。

`invokeCreateHooks` 是用来方法处理 `data` 的，其中 `cbs` 存放这一组 `hooks`: `['create', 'activate', 'update', 'remove', 'destroy']` 每个 `hooks` 中存放这一组方法，这些方法用与在不同周期处理 `data` 数据，其代码在 `core/vdom/modules` 和 `platforms/xxx/runtime/modules` 中，这里就不展开了，主要是处理 `class`、`attrs`、`events`、`props` 等属性，并合并到 `vnode.elm` 上。而 `vnode.data.hook` 和 `component` 相关，稍后讲。

`insert` 方法就是将全部处理好的元素子元素 `vnode.elm` 放到父元素的内部 `document.appendChild(parent, vnode.elm)` 。注意这里都是真实的 DOM 操作，如果处理的是根节点，操作的 `parentElm` 是页面上的真实DOM。比如情况 5:

```js
// replacing existing element
const oldElm = oldVnode.elm
const parentElm = nodeOps.parentNode(oldElm)
// create new node
createElm(vnode, insertedVnodeQueue, parentElm)
```

这种情况就直接将当前元素整个替换。

#### createComponent

之前有提到，在 `_render` 时会处理 `Component`，当时只是处理组件对象，将其继承 `Vue` 实例的属性，生成 `vnode`，其实此时还会在组件对象的 `data` 上绑定一些 `hooks` :

```js
// inline hooks to be invoked on component VNodes during patch
const componentVNodeHooks = {
  init (vnode: VNodeWithData, hydrating: boolean): ?boolean {
    if (
      vnode.componentInstance &&
      !vnode.componentInstance._isDestroyed &&
      vnode.data.keepAlive
    ) {
      // kept-alive components, treat as a patch
      const mountedNode: any = vnode // work around flow
      componentVNodeHooks.prepatch(mountedNode, mountedNode)
    } else {
      const child = vnode.componentInstance = createComponentInstanceForVnode(
        vnode,
        activeInstance
      )
      child.$mount(hydrating ? vnode.elm : undefined, hydrating)
    }
  },
  prepatch() {...},
  insert() {...},
  destroy() {...}
}

const hooksToMerge = Object.keys(componentVNodeHooks)

function installComponentHooks(data) {
  const hooks = data.hook || (data.hook = {})
  for (let i = 0; i < hooksToMerge.length; i++) {
    const key = hooksToMerge[i]
    const existing = hooks[key]
    const toMerge = componentVNodeHooks[key]
    if (existing !== toMerge && !(existing && existing._merged)) {
      hooks[key] = existing ? mergeHook(toMerge, existing) : toMerge
    }
  }
}
```

这些 `hooks` 封装了组件的添加和销毁方法，我们就只看 `init`，这里的 `Vnode` 是组件vnode，其对象本身在 `vnode.componentOptions.Ctor` 中，是继承得到的构造方法，这里 `createComponentInstanceForVnode` 实际就是 `new Ctor()` 得到实例，这里会调用 `_init` 方法不过因为组件没有 `$el` 字段，所以不会调用 `$moumt` 方法，所以 `init hook` 手动调用了 `$moumt`。也就是 `init hook` 封装了组件的初始化操作，并将实例放到了 `vnode.componentInstance` 中。

ok，了解了这个 我们再来看 `patch` 中的 `createComponent`:

```js
function createComponent (vnode, insertedVnodeQueue, parentElm, refElm) {
  let i = vnode.data
  if (isDef(i)) {
    const isReactivated = isDef(vnode.componentInstance) && i.keepAlive
    if (isDef(i = i.hook) && isDef(i = i.init)) {
      i(vnode, false /* hydrating */)
    }
    // after calling the init hook, if the vnode is a child component
    // it should've created a child instance and mounted it. the child
    // component also has set the placeholder vnode's elm.
    // in that case we can just return the element and be done.
    if (isDef(vnode.componentInstance)) {
      initComponent(vnode, insertedVnodeQueue)
      insert(parentElm, vnode.elm, refElm)
      if (isTrue(isReactivated)) {
        reactivateComponent(vnode, insertedVnodeQueue, parentElm, refElm)
      }
      return true
    }
  }
}

function initComponent (vnode, insertedVnodeQueue) {
  if (isDef(vnode.data.pendingInsert)) {
    insertedVnodeQueue.push.apply(insertedVnodeQueue, vnode.data.pendingInsert)
    vnode.data.pendingInsert = null
  }
  vnode.elm = vnode.componentInstance.$el
  if (isPatchable(vnode)) {
    invokeCreateHooks(vnode, insertedVnodeQueue)
    setScope(vnode)
  } else {
    // empty component root.
    // skip all element-related modules except for ref (#3455)
    registerRef(vnode)
    // make sure to invoke the insert hook
    insertedVnodeQueue.push(vnode)
  }
}
```

`createComponent` 方法就是直接调用 `component` 的 `create hook`，生成实例存在 `vnode.componentInstance` 中，如果成功，就调用 `initComponent`，并插入父元素中 `insert(parentElm, vnode.elm, refElm)`。

值得注意的是 `create hook` 只是负责组件内部的初始化，而组件使用的地方依旧需要处理当作一个元素处理，即 `initComponent` 中除了将生成的组件实例的真实dom绑定在 `vnode.elm`，依旧调用了 `invokeCreateHooks` 来处理打他，比如需要传入的 `dom props`。

还有，想象以下，有组件时 `patch` 方法会执行两次，页面执行一次，组件内部执行一次（init hook `$mount`），而组件内部执行 `patch` 是没有 `oldVnode` 的，也就是说组件本身生成的 `element` 仅仅是存在组件的 `vnode.$el` 中，最后通过 `vnode.elm = vnode.componentInstance.$el` 的方法拿到，然后 `insert` 到父元素中去。

#### `hydrate`

调用 `$mount` 时会传入一个标记 `hydrating`，并一直拓传到 `patch` 方法，当 `oldVnode` 是真实dom时，它决定是否沿用真实dom。一个很好的例子就是 `SSR` 服务端渲染，初次调用 `$mount` 时页面已经渲染好了，`oldVnode` 就是渲染好的元素节点，此时 `Vue` 就会调用 `hydrate` 方法

```js
// in patch function
if (isTrue(hydrating)) {
  if (hydrate(oldVnode, vnode, insertedVnodeQueue)) {
    return oldVnode
  }
}
```

`hydrate` 会处理 `vnode` 并返回一个 `boolean`，如果为true，`patch` 就直接返回 `oldVnode`，我们来看看里面做了哪些处理：

```js
function hydrate (elm, vnode, insertedVnodeQueue) {
    let i
    const { tag, data, children } = vnode
    vnode.elm = elm

    if (isDef(data)) {
      if (isDef(i = data.hook) && isDef(i = i.init)) i(vnode, true /* hydrating */)
      if (isDef(i = vnode.componentInstance)) {
        // child component. it should have hydrated its own tree.
        initComponent(vnode, insertedVnodeQueue)
        return true
      }
    }
    if (isDef(tag)) {
      if (isDef(children)) {
        // empty element, allow client to pick up and populate children
        if (!elm.hasChildNodes()) {
          createChildren(vnode, children, insertedVnodeQueue)
        } else {
          // v-html and domProps: innerHTML
          if (isDef(i = data) && isDef(i = i.domProps) && isDef(i = i.innerHTML)) {
            if (i !== elm.innerHTML) return false
          } else {
            // iterate and compare children lists
            let childrenMatch = true
            let childNode = elm.firstChild
            for (let i = 0; i < children.length; i++) {
              if (!childNode || !hydrate(childNode, children[i], insertedVnodeQueue, inVPre)) {
                childrenMatch = false
                break
              }
              childNode = childNode.nextSibling
            }
            // if childNode is not null, it means the actual childNodes list is
            // longer than the virtual children list.
            if (!childrenMatch || childNode) return false
          }
        }
      }
      if (isDef(data)) {
        let fullInvoke = false
        for (const key in data) {
          if (!isRenderedModule(key)) {
            fullInvoke = true
            invokeCreateHooks(vnode, insertedVnodeQueue)
            break
          }
        }
        if (!fullInvoke && data['class']) {
          // ensure collecting deps for deep class bindings for future updates
          traverse(data['class'])
        }
      }
    } else if (elm.data !== vnode.text) {
      elm.data = vnode.text
    }
    return true
  }
```

首先是组件，我们之前讲过，真正的组件初始化在 `patch` 方法中，现在虽然我们的页面已经渲染完了，但 `vnode` 中并没有组件的实例`componentInstance` 所以我们需要生成它，调用 `data.hook.init` 方法生成，然后调用 `initComponent` 绑定 elm，注意这里调用时传入了 `hydrating = true` (`i(vnode, true)`) 那么 `init hook` 内部调用 `$mount` 时也会带入 `hydrating`, 保证组件只初始化，不操作dom。

然后是在 `render` 函数有 `children` 的情况下如果页面元素没有子元素（`!elm.hasChildNodes()`）,这种情况调用 `createChildren` ，循环添加子元素到当前元素中 (`createElm`)。还有个特殊情况就是 `innerHTML`，如果当前元素的 `innerHTML` 和 `vnode.data.domProps.innerHTML` 就会直接返回 false，执行之前的替换流程。

再然后是 `isRenderedModule(key)`，我们知道 `invokeCreateHooks` 会处理 `data` 将各种标签属性添加到元素中，还会处理 `props` 和 `event`，但是当页面已经渲染完成时，有些属性不需要再处理了，所以这个 `isRenderedModule` 维护了一个数组 `attrs,class,staticClass,staticStyle,key` 如果 `data` 中没有除这些之外的属性，那么就跳过 `invokeCreateHooks` 毕竟这是多余的操作，但是遇到其他的还是要正常处理，比如 `event`，服务端只会传html 过来，并不会自己处理事件。

最终处理完之后返回 `true`，将 `oldVnode` 绑定在 `vnode.elm` 上，`patch` 直接返回 `vnode`。这样这个 `vnode` 就会绑定在 `vm.$el` 上。保证下次更新时 `oldVnode` 拿到的是这个我们处理好的 `vnode`。

#### patchVnode

之前讲的都是替换的情况，最后就是更新的情况，当 `oldVnode` 不是真实dom 且节点没有替换时，调用 `patchVnode` 。

```js
if (!isRealElement && sameVnode(oldVnode, vnode)) {
  // patch existing root node
  patchVnode(oldVnode, vnode, insertedVnodeQueue, null, null, removeOnly)
}
```

我们先来看看 `sameVnode` 方法：

```js
function sameVnode (a, b) {
  return (
    a.key === b.key && (
      (
        a.tag === b.tag &&
        a.isComment === b.isComment &&
        isDef(a.data) === isDef(b.data) &&
        sameInputType(a, b)
      ) || (
        isTrue(a.isAsyncPlaceholder) &&
        a.asyncFactory === b.asyncFactory &&
        isUndef(b.asyncFactory.error)
      )
    )
  )
}

function sameInputType (a, b) {
  if (a.tag !== 'input') return true
  let i
  const typeA = isDef(i = a.data) && isDef(i = i.attrs) && i.type
  const typeB = isDef(i = b.data) && isDef(i = i.attrs) && i.type
  return typeA === typeB || isTextInputType(typeA) && isTextInputType(typeB)
}
```

该方法主要判断两个 `vnode` 的 `key`、`tag`、`isComment` 这些字段值是否一样（可以都为 undefined）,`data` 字段是否都 存在/不存在，如果 `tag` 为 `input`，其 `type` 也要相等。
只要上述情况满足就会被认定为 `sameNode`,它不管你 `data` 是否相等(哪怕 `data` 中事件、样式等等都不一样)，也不管你 `children` 如何。`sameNode` 不是指同一个元素，而是相似，也就是可复用。之后就会对两个相似的 `vnode` 调用 `patchVnode` 方法。

```js
function patchVnode (
    oldVnode,
    vnode
  ) {
    // 如果指向同一个对象，直接 return 就好了
    if (oldVnode === vnode) return

    const elm = vnode.elm = oldVnode.elm

    // reuse element for static trees.
    // note we only do this if the vnode is cloned -
    // if the new node is not cloned it means the render functions have been
    // reset by the hot-reload-api and we need to do a proper re-render.
    if (isTrue(vnode.isStatic) &&
      isTrue(oldVnode.isStatic) &&
      vnode.key === oldVnode.key &&
      (isTrue(vnode.isCloned) || isTrue(vnode.isOnce))
    ) {
      vnode.componentInstance = oldVnode.componentInstance
      return
    }

    let i
    const data = vnode.data
    // 处理 components
    if (isDef(data) && isDef(i = data.hook) && isDef(i = i.prepatch)) {
      i(oldVnode, vnode)
    }

    const oldCh = oldVnode.children
    const ch = vnode.children
    // 处理data
    if (isDef(data) && isPatchable(vnode)) {
      for (i = 0; i < cbs.update.length; ++i) cbs.update[i](oldVnode, vnode)
      if (isDef(i = data.hook) && isDef(i = i.update)) i(oldVnode, vnode)
    }
    // 处理 children
    if (isUndef(vnode.text)) {
      if (isDef(oldCh) && isDef(ch)) {
        if (oldCh !== ch) updateChildren(elm, oldCh, ch, insertedVnodeQueue, removeOnly)
      } else if (isDef(ch)) {
        if (isDef(oldVnode.text)) nodeOps.setTextContent(elm, '')
        addVnodes(elm, null, ch, 0, ch.length - 1, insertedVnodeQueue)
      } else if (isDef(oldCh)) {
        removeVnodes(oldCh, 0, oldCh.length - 1)
      } else if (isDef(oldVnode.text)) {
        nodeOps.setTextContent(elm, '')
      }
    } else if (oldVnode.text !== vnode.text) {
      nodeOps.setTextContent(elm, vnode.text)
    }
  }
```

一步一步来，首先拿到 `oldVnode.elm` 也就是真实的dom元素，而且赋值给 `vnode.elm`,紧接着是静态节点优化，这里提一下，在 `ast` 转 `render` 函数之间，有一步静态节点优化 `optimize(ast)`，前面省略了，如果当前节点没有任何动态的元素，就会添加一个静态标记 `isStatic`，在此时，遇到这个标记就直接 return，复用之间的元素（如果是组件就会复用组件实例 `vnode.componentInstance = oldVnode.componentInstance`）。

接着,拿到 `data` (`const data = vnode.data`), 如果当前节点是组件节点，调用组件的 `prepatch hook`，该 `hook` 的作用是处理组件实例，更新属性，并需要的话调用组件更新方法 `$forceUpdate` 也就是 `_update(_render())`。接着处理 `data` 只要 `data` 存在（`isPatchable` 主要就是判断 vnode/组件实例有没有 `tag` 字段）即调用 `update hooks` 处理，大部分和 `create hooks` 处理一样，处理属性、样式、事件

最后是处理 `children`, 拿到新老 `vnode` 的 `children`，如果 `vnode` 有 `text` 字段，且与 `oldVnode.text` 不同则直接将 当前 `dom element` 的 `textContext` 设为 `vnode.text` （`nodeOps.setTextContent(elm, vnode.text)`）。 `text` 的优先级要大于 `children`。抛去 `text` 的特殊情况则有三种情况，`oldCh` 和 `ch` 同时存在，则调用 `updateChildren` 进一步处理，如果 `ch` 存在 `oldCh` 不存在，则调用 `addVnodes` 处理 `children` 并直接插入当前dom 元素(就是循环对子 `vnode` 调用 `createElm`)，如果 `oldCh` 存在而 `ch` 不存在则调用 `removeVnodes` 将其子节点从元素中删除。

简单的说，只要新老 `vnode` 相似，抛去特殊情况，就会拿到旧元素复用，在旧元素上处理 `data`，也就是说 `sameNode` 判定很简单，而复用也只是简单的复用了dom 元素的壳，其属性还是要处理过，而且只有相似的节点才回去深入比较其 `children`。接着我们看下如何处理 `children`:

#### updateChildren

与 `patchVnode` 不同的是，`updateChildren` 对比的是两个新/老 `vnode` 数组，老数组中任意元素都有可能被新元素复用：

```js
function updateChildren (parentElm, oldCh, newCh, insertedVnodeQueue) {
    let oldStartIdx = 0
    let newStartIdx = 0
    let oldEndIdx = oldCh.length - 1
    let oldStartVnode = oldCh[0]
    let oldEndVnode = oldCh[oldEndIdx]
    let newEndIdx = newCh.length - 1
    let newStartVnode = newCh[0]
    let newEndVnode = newCh[newEndIdx]
    let oldKeyToIdx, idxInOld, vnodeToMove, refElm

    while (oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx) {
      if (isUndef(oldStartVnode)) {
        oldStartVnode = oldCh[++oldStartIdx] // Vnode has been moved left
      } else if (isUndef(oldEndVnode)) {
        oldEndVnode = oldCh[--oldEndIdx]
      } else if (sameVnode(oldStartVnode, newStartVnode)) {
        patchVnode(oldStartVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        oldStartVnode = oldCh[++oldStartIdx]
        newStartVnode = newCh[++newStartIdx]
      } else if (sameVnode(oldEndVnode, newEndVnode)) {
        patchVnode(oldEndVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        oldEndVnode = oldCh[--oldEndIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldStartVnode, newEndVnode)) { // Vnode moved right
        patchVnode(oldStartVnode, newEndVnode, insertedVnodeQueue, newCh, newEndIdx)
        canMove && nodeOps.insertBefore(parentElm, oldStartVnode.elm, nodeOps.nextSibling(oldEndVnode.elm))
        oldStartVnode = oldCh[++oldStartIdx]
        newEndVnode = newCh[--newEndIdx]
      } else if (sameVnode(oldEndVnode, newStartVnode)) { // Vnode moved left
        patchVnode(oldEndVnode, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
        canMove && nodeOps.insertBefore(parentElm, oldEndVnode.elm, oldStartVnode.elm)
        oldEndVnode = oldCh[--oldEndIdx]
        newStartVnode = newCh[++newStartIdx]
      } else {
        if (isUndef(oldKeyToIdx)) oldKeyToIdx = createKeyToOldIdx(oldCh, oldStartIdx, oldEndIdx)
        idxInOld = isDef(newStartVnode.key)
          ? oldKeyToIdx[newStartVnode.key]
          : findIdxInOld(newStartVnode, oldCh, oldStartIdx, oldEndIdx)
        if (isUndef(idxInOld)) { // New element
          createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
        } else {
          vnodeToMove = oldCh[idxInOld]
          if (sameVnode(vnodeToMove, newStartVnode)) {
            patchVnode(vnodeToMove, newStartVnode, insertedVnodeQueue, newCh, newStartIdx)
            oldCh[idxInOld] = undefined
            canMove && nodeOps.insertBefore(parentElm, vnodeToMove.elm, oldStartVnode.elm)
          } else {
            // same key but different element. treat as new element
            createElm(newStartVnode, insertedVnodeQueue, parentElm, oldStartVnode.elm, false, newCh, newStartIdx)
          }
        }
        newStartVnode = newCh[++newStartIdx]
      }
    }
    if (oldStartIdx > oldEndIdx) {
      refElm = isUndef(newCh[newEndIdx + 1]) ? null : newCh[newEndIdx + 1].elm
      addVnodes(parentElm, refElm, newCh, newStartIdx, newEndIdx, insertedVnodeQueue)
    } else if (newStartIdx > newEndIdx) {
      removeVnodes(oldCh, oldStartIdx, oldEndIdx)
    }
  }
```

可以看到，`updateChildren` 利用了前后指针法来遍历两个数组 `oldCh` 和 `newCh`，在最一开始分别记录了

* `oldStartIdx`、`newStartIdx` 新/老数组起始指针位置
* `oldEndIdx`、`newEndIdx` 新/老数组结束指针位置
* `oldStartVnode`、`newStartVnode` 新老数组起始位置的 `vnode`
* `oldEndVnode`、`newEndVnode` 新老数组结束位置的 `vnode`

就像这样：分别对应四个 `vnode` 和指针

<table>
  <tr>
    <td></td><td>newStartVnode</td>
    <td></td><td></td><td></td><td>newEndVnode</td>
  </tr>
  <tr>
    <td>ch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td>oldch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td>oldStartVnode</td>
    <td></td><td></td><td></td><td>oldEndVnode</td>
  </tr>
</table>

ok，有了这些值就可以开始我们的循环操作了，只要 `oldStartIdx <= oldEndIdx && newStartIdx <= newEndIdx` 新老数组的前后指针之间还有元素存在就一直执行如下判断，有那么几种情况：

1. `oldStartVnode` 不存在，那么老数组的前指针后移，`oldStartVnode` 也指向移动后的 `vnode`：`oldStartVnode = oldCh[++oldStartIdx]`
2. `oldEndVnode` 不存在，那么老数组的后指针前移，`oldStartVnode` 也指向移动后的 `vnode`：`oldStartVnode = oldCh[++oldStartIdx]`

> note: 后续匹配 `key` 的逻辑中会将 `oldch` 中匹配到的位置置空 `undefined`，防止多次匹配，这里就是处理这种空的情况，直接跳到下一个/前一个

3. `sameVnode(oldStartVnode, newStartVnode)` 也就是新老数组的起始 `vnode` 相似，如此直接调用 `patchVnode`,复用 `oldVnode` 中存的 dom element，在该元素上直接处理 `newVnode`。完事后将新老前指针后移，同时新老 `vnode` 也指向移动后的：

<table>
  <tr>
    <td>patch:</td><td>newStartVnode</td>
    <td></td><td></td><td></td><td></td>
  </tr>
  <tr>
    <td>ch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td>patch sameNode</td>
  </tr>
  <tr>
    <td>oldch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td>oldStartVnode</td>
    <td></td><td></td><td></td><td></td>
  </tr>
</table>

<table>
  <tr>
    <td>移动指针</td><td></td>
    <td>newStartVnode</td><td></td><td></td><td></td>
  </tr>
  <tr>
    <td>ch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td>oldch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td></td>
    <td>oldStartVnode</td><td></td><td></td><td></td>
  </tr>
</table>

4. `sameVnode(oldEndVnode, newEndVnode)` 类似的，这个情况是新老数组的后指针 `vnode` 相似，调用 `patchVnode`,复用 `oldVnode` 中存的 dom element 并处理。完事后将新老后指针前移，同时新老 `vnode` 也指向移动后的：

<table>
  <tr>
    <td>patch:</td><td></td>
    <td></td><td></td><td></td><td>newEndVnode</td>
  </tr>
  <tr>
    <td>ch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td></td><td></td><td></td><td></td><td>patch sameNode</td>
  </tr>
  <tr>
    <td>oldch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td></td>
    <td></td><td></td><td></td><td>oldEndVnode</td>
  </tr>
</table>

<table>
  <tr>
    <td>移动指针</td><td></td><td></td><td></td><td>newEndVnode</td><td></td>
  </tr>
  <tr>
    <td>ch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td>oldch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td></td><td></td><td></td><td>oldEndVnode</td><td></td>
  </tr>
</table>

5. `sameVnode(oldStartVnode, newEndVnode)` 也就是‘老前’ 和 ‘新后’ `vnode` 相似，这种情况不仅要进行 `patchVnode` 处理，还需要将真实dom元素移动到正确位置： `parentElm.insertBefore(oldStartVnode.elm, oldEndVnode.elm.nextSibling` 也就是将 `oldStartVnode.elm` 放在 `oldEndVnode.elm` 的后面 (注意不是最后！已经处理好的元素不动，放在还没处理的元素列表的最后)。完事后将 `oldStartIdx` 右移，`newEndIdx` 左移。

<table>
  <tr>
    <td>patch:</td><td></td>
    <td></td><td></td><td></td><td>newEndVnode</td>
  </tr>
  <tr>
    <td>ch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td></td><td></td><td></td><td></td><td>patch sameNode</td>
  </tr>
  <tr>
    <td>oldch：</td>
    <td></td><td></td><td></td><td></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td></td>
    <td></td><td></td><td></td><td>oldStartVnode</td>
  </tr>
</table>

<table>
  <tr>
    <td>移动指针</td><td></td><td></td><td></td><td>newEndVnode</td><td></td>
  </tr>
  <tr>
    <td>ch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td>oldch：</td><td></td><td></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td></td><td></td><td></td><td>oldStartVnode</td><td></td>
  </tr>
</table>

此时真实元素会这样排列：

<table>
  <tr>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
  </tr>
</table>

6. `sameVnode(oldEndVnode, newStartVnode)` 类似的，这种情况是 ‘新前’ 和 ‘老后’ 相似，同样也是 `patchVnode`，然后将当前元素 `oldEndVnode.elm` 放在未处理元素 `oldStartVnode.elm` 的最前面  `parentElm.insertBefore(oldEndVnode.elm, oldStartVnode.elm)`。完事将 `oldEndIdx` 左移，`newStartIdx` 右移。

> note: 注意 5，6 中所谓的未处理元素列表指的是 `oldStartIdx`和`oldEndIdx` 之间的位置的 `vnode` 所对应的元素。dom 元素移动只在他们之间，处理好的dom元素不会改变

<table>
  <tr>
    <td>patch:</td><td></td>
    <td></td><td></td><td></td><td>newStartVnode</td>
  </tr>
  <tr>
    <td>ch：</td><td></td><td></td><td></td><td></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td></td><td></td><td></td><td></td><td>patch sameNode</td>
  </tr>
  <tr>
    <td>oldch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td></td>
    <td></td><td></td><td></td><td>oldEndVnode</td>
  </tr>
</table>

<table>
  <tr>
    <td>移动指针</td><td></td><td></td><td></td><td>newStartVnode</td><td></td>
  </tr>
  <tr>
    <td>ch：</td><td></td><td></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td>oldch：</td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
  </tr>
  <tr>
    <td></td><td></td><td></td><td></td><td>oldEndVnode</td><td></td>
  </tr>
</table>

此时真实元素会这样排列：

<table>
  <tr>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">5</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">1</div></td>
    <td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">2</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">3</div></td><td><div style="border: 1px solid black; height: 30px;width: 30px;text-align: center;line-height: 30px;margin: 2px;">4</div></td>
  </tr>
</table>

7. 最后一种情况就是以上都没有匹配到，无论 '前前'、'后后'、'前后',这个时候就要利用  `key` 值进行搜索了。

[simplehtmlparser]:http://erik.eae.net/simplehtmlparser/simplehtmlparser.js
[createElement]:https://cn.vuejs.org/v2/guide/render-function.html#createElement-%E5%8F%82%E6%95%B0
[vnode]:https://github.com/vuejs/vue/blob/dev/src/core/vdom/vnode.js
[nodeType]:https://www.w3school.com.cn/jsref/prop_node_nodetype.asp