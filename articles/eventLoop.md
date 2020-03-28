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

***

> 更新一下

我们再深入一下另外两种情况

## requestAnimationFrame

首先 `requestAnimationFrame` 是什么，官方的描述是：

> 该方法告诉浏览器您希望执行动画，并请求浏览器在下一次重绘之前调用指定的函数。

值得注意的是该操作是**一次性的**
也就是说每调用一次 `window.requestAnimationFrame(callBack)` 浏览器只会在下一次重绘之前调用 `callBack` 而不是像注册一个事件一样长久保持。所以正常的使用方法是这样的：

```js
let anim = document.querySelector('#anim');

let time = 0;
function step() {
    anim.style.width = `${anim.clientWidth + 1}px`;

    if (time < 60) {
        time++;
        window.requestAnimationFrame(step)
    }
}

step();
```

这样就会有一个长达 1s 的动画了（正常情况）。

然而为什么是 1s ？
且他究竟是在哪一步执行，是插队到当前的队列中吗？ 我们接着来看看：

```html
<style>
    #anim {
        width: 100px;
        height: 100px;
        background: red;
    }
</style>
<body>
    <div id="anim"></div>
    <button onclick="clickH()" onmouseup="step()">321</button>
</body>
<script>
    let anim = document.querySelector('#anim');
    let time = 0;
    function step() {
        anim.style.width = `${anim.clientWidth + 1}px`;
        if (time < 60) {
            time++;
            window.requestAnimationFrame(step)
        }
    }
    function clickH() {
        console.log(anim.clientWidth);
    }
</script>
```

我们通过 `onmouseup` 来触发一个事件，并开启一个动画，老样子我们通过 chrome 的 performance 来看看具体的情况：

![img-requestAnimationFrame]

我们截取了开始的一小段，可以看到真正执行 `requestAnimationFrame` 的 callBack 的时机是 绘制(Paint)之前，并且就被’解雇‘了 (`Animation Frame Fired`)。OK， **结论**已经出来了，不过我们继续深入下这个渲染：

我们来复习一下浏览器渲染的步骤：js(dom 操作等) => Style(计算样式) => Layout(布局) => Paint(绘制) => Composite(渲染层合并)

实际上这并不是一个连续的流程，我们仔细看看 `mouseup => setp` 下面的两个紫色小块，他们分别是 `Style` 和 `Layout`，也就是说每一次 dom 操作都会同步的执行 `Style` 和 `Layout` 的过程，这并不是插队，所有渲染的过程都在渲染线程完成，我们知道渲染线程和js线程是互斥的，也就是说渲染线程工作时js线程是挂起的，所以并不是在上面说的冲刷 `microtask` 的后面的渲染过程中执行，我们可以加一个微任务简单测试一下

![style&layout]

可以看到很明显的的顺序。这也是为什么设置了 `anim.style.width` 之后能马上拿到 `anim.clientWidth` 的原因。(有意思的是 height 不会触发 Layout 大家可以试一试)

那么什么时候会触发 `Paint` 呢，之前我们讲到每次冲刷 `microtask` 之后都有可能发生渲染，这个可能是怎么判断的呢，我们看到文档中流程模型的 [step11][update-rendering]。浏览器会判断当前是否有渲染机会（Rendering opportunities）以及是否有渲染的必要（Unnecessary rendering）。渲染的必要很好理解，你没有做任何操作自然不需要重绘。渲染机会和浏览器刷新率有关，1s 内 `Paint` 的次数不会超过刷新率，所以我们上边看到真正 `Paint` 的时机是在 `mouseup` 和 `click` 之后的，因为整个执行的过程才 14ms 还不到一帧的时间(60hz)。现在我们给 `mouseup` 事件多加一些操作让其比较耗时：

```html
<style>
    #anim {
        width: 100px;
        height: 100px;
        background: red;
    }
</style>
<body>
    <div id="anim"></div>
    <button onclick="clickH()" onmouseup="mouseupH()">321</button>
</body>
<script>
    let anim = document.querySelector('#anim');
    let time = 0;
    function step() {
        anim.style.width = `${anim.clientWidth + 1}px`;
        if (time < 60) {
            time++;
            window.requestAnimationFrame(step)
        }
    }
    function mouseupH() {
        step();
        for (let i=0;i<1000;i++) {
            console.log(1);
        }
    }
    function clickH() {
        console.log(anim.clientWidth);
    }
</script>
```

图就不放了大家可以试一试，结果是和之前一样，还是没有中间渲染...（mouseup 和 click 之间） 这又是为什么呢，不符合规范啊。

我们来看看 step.11-5 的描述，在这一步会删除出于某种原因最好跳过更新渲染的所有 Document 对象。下面有这么一段提示

> note: This step enables the user agent to prevent the steps below from running for other reasons, for example, to ensure certain tasks are executed immediately after each other, with only microtask checkpoints interleaved (and without, e.g., animation frame callbacks interleaved). Concretely, a user agent might wish to coalesce timer callbacks together, with no intermediate rendering updates.

微任务交错可以理解，冲刷微任务时肯定不希望一直触发渲染，可是并没有举例其他情况，不过我们只要知道，每一次 task => microtask queue 之后都会触发 `Paint` 只不过大多情况被’跳过‘了，所以真正 `Paint` 的时机就到了最后一次任务处理，也就是之后没有可执行的 task 时，而 `requestAnimationFrame` 的 callBack 始终是在 `Paint` 之前的。

## requestIdleCallback

接着我们来看看 `requestIdleCallback` 官方给的描述是：

> 该方法将在浏览器的空闲时段内调用回调函数队列。这使开发者能够在主事件循环上执行后台和低优先级工作，而不会影响延迟关键事件，如动画和输入响应。函数一般会按先进先调用的顺序执行，然而，如果回调函数指定了执行超时时间timeout，则有可能为了在超时前执行函数而打乱执行顺序

简单的说其处理的回调将在空闲时间执行。这是他的用法：

```js
window.requestIdleCallback(callBack, { timeout: 1000 });
```

实际上他和 `setTimeout` 差不多，`setTimeout` 是延时一段时间后把 callBack 推到任务队列，而 `requestIdleCallback` 是延时一段时间后把 callBack 推到 与空闲任务相关的源（Source）的队列中。这里是详细的[定义][requestIdleCallback]

现在我们把他加到例子中去：

```html
<body>
    <div id="anim"></div>
    <button onclick="clickH()" onmouseup="mouseupH()">321</button>
</body>
<script>
    let anim = document.querySelector('#anim');
    let time = 0;
    function step() {
        anim.style.width = `${anim.clientWidth + 1}px`;
        if (time < 60) {
            time++;
            window.requestAnimationFrame(step)
        }
    }
    function relax() {
        if (time < 60) {
            time2++;
            window.requestIdleCallback(relax)
        }
    }
    function mouseupH() {
        step();
        relax();
    }
    function clickH() {
        console.log(anim.clientWidth);
    }
</script>
```

看看结果：

![img-requestIdleCallback]

可以看到回调的执行时间在每次渲染之后，我们来看看 [流程模型][Processing model]的 step.12，满足下列条件时会执行 [REQUESTIDLECALLBACK]:

1. 当前是一个 window event loop
2. 事件循环中没有 task 且没有文档出于活动状态
3. 微任务队列也为空
4. hasARenderingOpportunity 标志位为false （每次微任务之后的渲染过程才会为 true）

渲染过程在 step.11，空闲任务的执行在 step.12 这也就是为什么我们看到 `requestIdleCallback` 的 callBack 总是在 渲染之后执行。

最后，通过这两个东西，我也算是把之前模棱两可的渲染这一步算是补完了，不过 Event loop 的内容其实很多，后续可能还会更新。

[event-loop-spec]:https://html.spec.whatwg.org/multipage/webappapis.html#event-loops
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
[update-rendering]:https://html.spec.whatwg.org/multipage/webappapis.html#update-the-rendering
[img-requestAnimationFrame]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/event_loop/requestAnimaFrame.jpg
[style&layout]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/event_loop/style%26layout.jpg
[requestIdleCallback]:https://w3c.github.io/requestidlecallback/#the-requestidlecallback-method
[img-requestIdleCallback]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/event_loop/idleCallback.jpg
