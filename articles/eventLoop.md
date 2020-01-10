# Event Loop

## 什么是 event loop
event loop是一个规范，具体看[这里](event-loop-spec)，每个浏览器实现不一样。

*event loop* 有一个或多个*task queues*，一个 *task queue* 是一组 *tasks*
* 每个 *event loop* 都有一个当前正在运行的 *task* ，它可以是 *task* 可以是 null ，在最一开始，它是 null
* 每个*event loop*都有一个 *microtask queue*，是一个*microtasks*的集合，一开始是空的，*microtask* 是通过微任务算法创建的 *task*
* 每个 *event loop* 有一个 boolean 值的 *microtask checkpoint* , 默认为 `false` , 每当到达 *microtask* 的执行点时用来判断。

![eventloop][eventloop]

上面是规范的原文了内容还有有很多，*microtask* 就是我们所说的**微任务**，规范中并未提及 `macrotask queue` ，根据[这个][issue]提到的 *task queue* 就是我们所说的 `macrotask queue` ，一个 *task* 就是一个 `macrotask` ，这个之后讨论。


## tasks & microtask
那么什么是 *task*, 详细的看[这里][task]。

*task* 封装了event、callbacks、资源获取(Using a resource)、对dom操作进行响应(Reacting to DOM manipulation
)、html 解析(Parsing) 这些个算法。

一个 *task* 具有以下结构：Steps(大概是咱们的一行行代码？), Source, document, A script evaluation environment settings object set。其中 *Source* 决定了 *task* 被推向哪个队列(同源)。

## 排列任务

先简单看看一个 *task* 是如何创建并被推到队列中的，([这里][queuing tasks]更详细)：
1. 先看看 *event loop* 是否提供，若否，提供一个 [implied event loop][implied event loop].
2. 设置 *task* 的 Steps、Source、document、Script evaluation environment settings object set

3. 将 *task* 推到 *event loop* 中和 *task* 同源(*Source*)的 *task queue* 中(没有的话创建一个？)
3. 如果这里是 *microtask* 直接推到提供的 *event loop* 的 *microtask queue* 中


> note: *microtask* 也是一个 *task* 其也有 Steps、Source、document、Script evaluation environment settings object set 这四个玩意，但 *microtask queue* 不是 *task queue*

### 如何创建一个 microtask
* process.nextTick (node)
* Promise
* Object.observe
* MutationObserver

通常你可以利用上面这些能帮助你创建一个微任务，而创建微任务需要特定的算法，目前只有部分浏览器提供了相应的api，例如： `queueMicrotask`。

## 流程模型

[完整过程][Processing model]很复杂这里简单梳理下，

只要 *event loop* 存在就会执行以下步骤：

1. 从 *event loop* 取出第一个 *taskQueue* ，如果没有，直接跳到 4 步骤。
2. 取出并移除 *taskQueue* 中第一个 可运行 *task*，放到 *event loop* 的 *currently running task* 中，执行 *task* 的 *setps*
3. 将 *event loop* 的 *currently running task* 设置为 null
4. *microtask checkpoint* 此处**可能**会运行并清空 *microtask queue*
5. 此时**可能**会进行更新视图渲染
6. 重复2-5
7. 其他

![processing][processing]

类似于
```js
while (/** event loop exist */) {
    // run step 1 - 7
}
```

## 以下是个人理解了

流程模型的步骤 2 中执行 *task* 的 *setps* 时可能会有异步任务，产生新的 *task* 或 *microtask*（callbacks），异步任务完成时 *microtask* 会被推到 *microtask queue* ，在下一个 *microtask checkpoint* 可能被执行掉，而 *task* 按照上述 **排列任务** 的流程被推到 *task queue*。而根据同源的策略，这些任务会被推到一个其他 *Source* 的 *task queue*。

![newTask][newTask]

### 通用源
在所有规范中，许多几乎不相关的功能都使用以下任务源。
* DOM
* 用户交互
* netWorking
* history.back() Api
源相同的 *task* 会被推向同源的队列。我们借助此做个试验

### Chrome Performance

通过 chrome 的 Performance 我们来看看浏览器是如何运行的。

因为用户交互有通用的源，我们通过点击事件来观看实际情况

```html
<html>
    <body>
        <button onclick="clickHandler()" onmouseup="mouseupHandler()">test</button>
        <script>            
            function clickHandler(e) {
                consolelog(1);
                Promise.resolve(2).then(consolelog)
                setTimeout(() => {
                    consolelog(3);
                });
            }
            function mouseupHandler () {
                consolelog(4);
                Promise.resolve(5).then(consolelog)
                setTimeout(() => {
                    consolelog(6);
                });
            }
            // 便于观查
            function consolelog(value) {
                console.log(value);
            }
        </script>
    </body>
</html>
```

`onmouseup` 和 `onclick` 他们都来自与一次用户交互，因此他们应该在同一个 *task queue* 中，而两次 `setTimeout` 产生的 *task* 会不会在一个 *task queue* 中呢

![click][click]

得到的结果有一点奇怪，根据 **流程模型** ，*microtask checkpoint* 只会存在于每个 *task* 运行完之后，所以 *图中的 Task* 应该不是我们理解的 *task*，而像是我们说的 *task queue*，而 `Event:click` 和 `Event:mouseup` 更像是我们说的 *task* ，因为他们最后都执行了 `Run Micortasks`。

这样理解就没有问题了，两个 `Event` 都产生自用户交互的 *Source*，所以他们在同一个 *task queue* 中，而每个 `setTimeout` 生成的 *task* 都放在了一个新的、单独的 *task queue* 中，也许他没设置成通用的 *Source*，而是新生成了一个。

而答案也很符合 **流程模型**

```js
4
5
1
2
6
3
```

复杂的例子就不贴了大家自己试试，也就是说基本理解了这个模型，不管浏览器怎么实现，都能推导出正确的流程，当然这只是其中正常流程的一部分，还会有很多特殊的情况，这里就不深入了，最后贴上一篇好的文章:

[tasks-microtasks-queues-and-schedules][tasks-microtasks-queues-and-schedules]


[event-loop-spec]:https://html.spec.whatwg.org/multipage/webappapis.html#event-loop
[issue]:https://stackoverflow.com/questions/25915634/difference-between-microtask-and-macrotask-within-an-event-loop-context
[task]:https://html.spec.whatwg.org/multipage/webappapis.html#concept-task
[Processing model]:https://html.spec.whatwg.org/multipage/webappapis.html#event-loop-processing-model
[queuing tasks]:https://html.spec.whatwg.org/multipage/webappapis.html#queuing-tasks
[implied event loop]:https://html.spec.whatwg.org/multipage/webappapis.html#implied-event-loop
[tasks-microtasks-queues-and-schedules]:https://jakearchibald.com/2015/tasks-microtasks-queues-and-schedules/#comment-2198501019
[eventloop]:https://github.com/jwdzzhz777/blog/blob/master/assets/event_loop/eventloop.jpg?raw=true
[newTask]:https://github.com/jwdzzhz777/blog/blob/master/assets/event_loop/newTask.jpg?raw=true
[processing]:https://github.com/jwdzzhz777/blog/blob/master/assets/event_loop/processing.jpg?raw=true
[click]:https://github.com/jwdzzhz777/blog/blob/master/assets/event_loop/click.jpg?raw=true
