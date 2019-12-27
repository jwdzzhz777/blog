# Eggjs 异步配置

在部署 `eggjs` 大家应该都会遇到的一个问题：有些初始化配置不写在本地，可能是出于安全考虑，会异步获取配置，本文就是如何去异步初始化配置。

> note: 本文讲的是 egg-sequelize

***

这不，某天我部署 `egg` 应用的时候遇到这个问题，初始化 `egg-sequelize` 的配置的时候，
公司的 `mysql` 数据库的信息是通过配置中心获取的，而获取请求是异步的，

一开始，天真的我以为 `config` 返回异步函数就可以，直接写了起来:

```js
// in config.xxx.js
import ConfigCenter from '...';

export default async () => {
    let configCenter = new ConfigCenter(...);

    const sequelize = await configCenter.fetch();

    return { sequelize };
}
```

一跑发现并没有报错，但是初始化用的还是默认 `sequelize` 的配置，
调试了半天，还以为公司提供的东西有问题...
仔细看看根本没等异步请求完成就连接了数据库。

于是看了下github，好嘛人配置压根就是同步的。没办法只能自己搞了。

**找办法**

看一下 `egg-sequelize` 的源码，`app.js` 和 `agent.js` 里面都是直接调用了
```js
require('./lib/loader')(app);
```
`loader` 里面都干了啥:
```js
module.exports = app => {
    const defaultConfig = {
        // ...
    }
    const config = app.config.sequelize;
    // support customize sequelize
    app.Sequelize = config.Sequelize || require('sequelize');

    const databases = [];
    if (!config.datasources) {
        databases.push(loadDatabase(Object.assign({}, defaultConfig, config)));
    } else {
        config.datasources.forEach(datasource => {
            databases.push(loadDatabase(Object.assign({}, defaultConfig, datasource)));
        });
    }

    app.beforeStart(async () => {
        await Promise.all(databases.map(database => authenticate(database)));
    });

    function loadDatabase() {
        // ...
    }
}
```

比较好理解，先解拿到配置然后异步调用 `loadDatabase`。那么 `beforeStart` 是什么？

![life_circle][life_circle_url]

看了一下 `eggjs` 启动的生命周期，
官方文档讲的很清楚了，是个配置生效前的异步钩子，那么不用管 `loadDatabase` 里面干了啥，直接在 `beforeStart`
钩子函数里做文章就好了。

于是：
```js
// in app.js and agent.js
import ConfigCenter from '...';

export default agent => {
    agent.beforeStart(async () => {
        let configCenter = new ConfigCenter(...);

        const sequelize = await configCenter.fetch();

        agent.config.sequelize = sequelize;
    });
};
```
**试一试**

还是加载了默认配置呀？

再仔细看看代码：
```js
const config = app.config.sequelize;

databases.push(loadDatabase(Object.assign({}, defaultConfig, config)));

app.beforeStart(async () => {
    await Promise.all(databases.map(database => authenticate(database)));
});
```
他一来直接就获取了配置，
然后再添加到 `beforeStart` 里去，也就是最后拿的还是一开始获取的 `config` 中的配置，根本不管我后面干了啥...这下看来不能这么用了，
只能手动去注册 `egg-sequelize` 了。

**手动注册**

先把 `plugin.js` 中 `egg-sequelize ` 的注册代码删掉。

然后加入代码
```js
// in app.js and agent.js
import ConfigCenter from '...';

export default agent => {
    agent.beforeStart(async () => {
        // 异步获取配置
        let configCenter = new ConfigCenter(...);
        const sequelize = await configCenter.fetch();
        agent.config.sequelize = sequelize;
        // 模仿 egg-sequelize 的操作
        require('egg-sequelize/lib/loader')(agent);
    });
};
```
	Note: app.js 和 agent.js 都要写 因为有 delegate 和 baseDir 的配置
ok，可以把 config.xxx.js 中和 `sequelize` 有关的配置删掉了
（不需要异步获取的默认配置可以留着）

最后，因为我的项目用了 `typescript` 别忘了加 `d.ts` 文件。
```js
// in typings/index.d.ts
import 'egg-sequelize';
```

运行一下试试，
成功了 完美。

----

**更新**

`beforeStart` 函数替代

官方貌似对生命周期函数进行了优化:[升级你的生命周期事件函数
][update_life_circle]

那么我们也优化下，之前 `app.ts` 和 `agent.ts` 都是函数的形式，现在都改成类的形式。
```ts
import { Application } from 'egg';

class AppBootHook {
    app: Application;
    constructor(app: Application) {
        this.app = app;
    }

    async willReady() {
        // 异步获取配置
        let configCenter = new ConfigCenter(...);
        const sequelize = await configCenter.fetch();
        this.app.config.sequelize = sequelize;
        // 模仿 egg-sequelize 的操作
        require('egg-sequelize/lib/loader')(this.app);
    }

    async didLoad() {
        ...
    }
}

export default AppBootHook;
```
`willReady` 运行在插件加载之前，所以我们获取配置放在这里， `didLoad` 运行在插件加载好之后，我们可以对加载好的插件进行造作
`this.app.xxx = ...`

[life_circle_url]:https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/egg_async_config/WechatIMG4.jpg
[update_life_circle]:https://eggjs.org/zh-cn/advanced/loaderUpdate.html#mobileAside
