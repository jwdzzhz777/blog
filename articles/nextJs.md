# next.js 使用记录

最近正在写博客(不出意外就是this)，考虑到页面用什么框架，想了许久最后选择了 `next.js`，不为别的就为了 `seo` 虽然内容本身就放在 `Issue` 中可以被搜索引擎找到，但多学点总没错。

### Quick Start
根据[教程][quick_start]来就是了，如果是要用 `typescript` ,还需要依赖 `@types`:
```
npm install --save-dev typescript @types/react @types/node
```
`tsconfig.json` 也会自动帮你生成。
直接在 `pages` 下面新建个文件 `index.tsx`:
```tsx
const Home=() => <h1>Hello world!</h1>;

export default Home;
```
然后运行 `next dev` 就跑起来啦。

### alias

写代码的时候引入其他位置的文件总是要用相对路径很麻烦，所以我一般会用 `webpack` 给一些路径设置别名。<br>
`next.js` 设置别名在配置文件中，默认是没有的，所以我们手动新建一个 `next.config.js` 文件。<br>
比如我要给 `components` 文件夹设置别名：
```js
var path = require('path');

const resolve = (dir) => {
    return path.join(__dirname, dir);
};

module.exports = {
    webpack: (config) => {
        Object.assign(config.resolve.alias, {
            'components': resolve('components')
        });
        return config;
    }
}
```
直接在 `webpack` 配置下修改 `config` 配置就行了。然后试一试：
```tsx
import Some from 'components/some';

const Home = () => {
    return (
        <>
            <h1>Hello world!</h1>
            <Some />
        </>
    );
};

export default Home;
```
> note: 如果项目接入了 `typescript` 可能会报**找不到模块**，
> 在 `tsconfig.json` 的 `compilerOptions` 下设置 `"baseUrl": "."` 即可。

### 环境变量
同样的，我有时候也会设置一些环境变量，也是直接在配置文件中去设置，比如我想设置该项目请求的域名：

```js
module.exports = {
    webpack: ...,
    env: {
        API_HOST: 'api.com'
    }
}
```
代码中就可以直接用 `process.env.API_HOST` 了。<br>
当然这是构建时的配置(`Build-time`)，我们也可以做运行时的配置(`Runtime`):
```js
// next.config.js
module.exports = {
    env: {
        API_HOST: 'api.com'
    }
    serverRuntimeConfig: {
        host: procsss.env.API_HOST
    },
    publicRuntimeConfig: {
        staticFolder: '/static',
    },
}
```
`serverRuntimeConfig` 存放仅有在服务端可以获取的配置, 而 `publicRuntimeConfig` 下的配置在服务端和客户端均可获取。
```js
// pages/index.js
import getConfig from 'next/config'

const { serverRuntimeConfig, publicRuntimeConfig } = getConfig();

console.log(serverRuntimeConfig.host) // 仅在服务端可用
console.log(publicRuntimeConfig.staticFolder) // 均可用
```

### less 支持
官方提供了 [next-less][next-less] 来对 `less` 提供支持。
```
yarn add @zeit/next-less less
```
配置的时候很有意思，它提供了一个方法来返回 `less` 的配置，我们需要把我们原有的配置传入方法
```js
// in next.config.js
const withLess = require('@zeit/next-less');
var path = require('path');
const config = require('./config');

const resolve = (dir) => {
    return path.join(__dirname, dir);
}

module.exports = withLess({
    webpack: (config) => {
        Object.assign(config.resolve.alias, {
            'components': resolve('components')
        });
        return config;
    },
    env: {
        ...config[process.env.NODE_ENV]
    }
});
```
创建一个 `style.less`:
```css
@font-size: 50px;
.example {
    font-size: @font-size;
}
```
我们就可以在页面里引入 `less` 文件了
```
import "./styles.less";
```
或者这么用
```js
mport css from "../styles.less"

export default () => <div className={css.example}>Hello World!</div>
```

### 设置公共部分
假如页面有个公共的 `header` ，我总是希望有一个 `Layout` 里面有页面的公共部分，比如 `<header>`、`<footer>` 然后中间才是我们的页面，在 `Vue` 中我们可以用 `Router` 实现,但在Next.js中所有页面都是独立的 page。官方的教程里也是每个 page 引入一个 `<header>` 这显然不是我想要的。

#### Custom `<App>`
Next.js 用一个 `<App>` 组件去渲染每一个 page ,所以我可以自定义 App 组件来实现一个公共的 `<Layout>`。<br>
创建 `/pages/_app.tsx`:
```ts
import React from 'react'
import App from 'next/app'

class MyApp extends App {
    render() {
        const { Component, pageProps } = this.props
        return <Component {...pageProps} />
    }
}

export default MyApp
```
原始的 App 组件就长这样，现在我可以加一些自己的东西。先搞个 `Layout` 组件：
```ts
// in components/layout.tsx
import React from 'react';
import Head from 'next/head'
import Header from './header';

export default class Layout extends React.Component {
    render() {
        const { children, viewer } = this.props;
        const { snakeBar } = this.state;
        return (
            <>
                <Head>
                    <meta name="viewport" content="initial-scale=1.0, width=device-width" key="viewport" />
                </Head>
                <Header/>
                {children}
            </>
        )
    }
}
```
比如我设置一个统一的 `viewport` 或者 `title`,再给页面搞个统一的 `Header` 。
再改造下我们的 `_app.tsx` 就可以了：
```ts
import React from 'react'
import App from 'next/app'
import Layout from 'components/layout';

class MyApp extends App {
    render() {
        const { Component, pageProps } = this.props
        return (
            <Layout>
                <Component {...pageProps} />
            </Layout>  
        );
    }
}

export default MyApp
```
这样我们还可以在 `Layout` 里面设置全局样式
```ts
// in layout.tsx
import 'styles/global.less';
```
虽然可以直接在任意文件设置全局样式,但我并不喜欢这种方式：
```ts
export default () => {
    return (
        <>
            <p>hi</p>
            <style global jsx>{`somestyle`}</style>
        </>
    )
};
```

### 给每个页面提供数据
现在我有了公共的 `header` 现在我想给每个页面提供数据。我们甚至可以在 MyApp 中定义   `getInitialProps` 方法。
```ts
import App from 'next/app';
import Layout from 'components/layout';

export default class MyApp extends App {
    static async getInitialProps({ Component, ctx }) {
        const pageProps = await Component.getInitialProps(ctx);
        let data = await curl(...);
        return { pageProps: {...pageProps, data} };
    }
    render() {
        const { Component, pageProps } = this.props as any;
        return (
            // 也可以传给 layout
            <Layout data={pageProps.data}>
                <Component {...pageProps} />
            </Layout>
        );
    }
}
```
> note:在 `_app.tsx` 中使用 `getInitialProps` 有个弊端，这样每个页面都会被服务端渲染，next.js 有个功能是 [Automatic Static Optimization][static-optimization]
即会自动把没有阻塞数据要求的页面优化为静态页面，它是根据 `getInitialProps` 方法判断的。

这样每个页面的 props 里都能拿到 data 了。同样的我也可以把请求错误的 error 传到 layout 中做一些处理，这样即便我在服务端调的接口报错了，我也能在页面上提示出来。
```ts
import App from 'next/app';
import Layout from 'components/layout';

export default class MyApp extends App {
    static async getInitialProps({ Component, ctx }) {
        const pageProps = await Component.getInitialProps(ctx);
        try {
            let data = await curl(...);
            return { pageProps: {...pageProps, data} };
        } catch (error) {
            return { pageProps, error: error.message };
        }
    }
    render() {
        const { Component, pageProps, error } = this.props as any;
        return (
            <Layout error={error} data={pageProps.data}>
                <Component {...pageProps} />
            </Layout>
        );
    }
}
```
```ts
// in components/layout.tsx
import React from 'react';
import Head from 'next/head'
import Header from './header';

export default class Layout extends React.Component {
    render() {
        const { children, error } = this.props;
        const { snakeBar } = this.state;

        if (error) {
            // do something in this
        }

        return (
            <>
                <Head>
                    <meta name="viewport" content="initial-scale=1.0, width=device-width" key="viewport" />
                </Head>
                <Header/>
                {children}
            </>
        )
    }
}
```

### 未完待续
[quick_start]:https://nextjs.org/learn/basics/getting-started/setup
[next-less]:https://github.com/zeit/next-plugins/tree/master/packages/next-less
[static-optimization]:https://nextjs.org/docs#automatic-static-optimization
