# vue

为了更深入了解 Vuejs 我决定，边读源码边自己实现一些简单的功能。

> note:文中的代码会进行缩减/修改，不然太多环境判断之类的代码不好阅读。

## 数据绑定

为什么从数据绑定开始，一是因为它重要，而是因为它比较独立，不依赖框架，可以单独抽出来讲。

###  Observer

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

依旧很简单，改写了属性的 getter/setter 方法（同时兼容了本身就有 getter/setter 方法的情况）在 getter/setter 时进行操作。简单的说 `Observer` 就是改造目标对象，劫持属性。

接下来需要一个充当主体的角色，将值多路推送给观察者(Observer)，Vuejs 中就是 `Dep` 了：

### Dep
