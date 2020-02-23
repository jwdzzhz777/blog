# github Api v4 (GraphQL) 记录

最近想接入 `github Api` 做一些事情，去文档一看发现无论 github 还是 gitlab 都更新了，都拥抱了 `GraphQL` 。之前接入 gitlab api 的时候都还是用 `REST API` 的，这下一下子不习惯了，折腾了好久，就在这里记录下。

> note: 以下请求均以 curl 为例

## GraphQL

### why GraphQL?

为什么 github 要选择 GraphQL ? 这里有官方的[公告博客文章][blog_post]。简单说就是三点：

* 可拓展性。尽管 `REST API` 覆盖了很全的信息，但有时候我们需要两到三个请求才能拿到我们需要的资源，而且通常一个 api 给了我们太多的信息，而我们只需要其中一部分。
* 想从终端获取更多元数据信息。比如确定每个终端所需的 OAuth 范围、更智能的发放数据、确保参数类型的安全性。
* 从代码生成文档。而不是手动更新维护

### 前端如何调用

调用 GraphQL Api 需要前端传 GraphQL 语句，而 GraphQL 语句是单纯的文本，所以请求的时候要如此操作：

```js
curl(HOST, {
    data: JSON.stringify({
        query: `
            query($some_variables: String!) {
                ...
            }
        `,
        variables: {
            'some_variables': 'eiheihei',
        }
    })
});
```

## Github Api

不得不说 github 的数据真的很多结构也很复杂，看文档也是一头雾水，这里就记录一些我开发中遇到的一些实例。

这里是[官方文档][developer_github_v4]

### 用户信息

首先调用大部分 api 需要用户登录信息，首先获取 [personal access token][personal_access_token]。

根据教程点点点就完事了，获得 `token` 记得保存，因为出于安全考虑再次进入页面你就看不到了。

然后就可以开始调接口了，GraphQL API v4 只有一个端点

```
https://api.github.com/graphql
```

> note:请求必须是 `POST`

```js
curl('https://api.github.com/graphql', {
    headers: {
        Authorization: `Bearer ${token}`
    },
    method: 'POST',
    dataType: 'json',
    contentType: 'json',
    data: JSON.stringify(query)
});
```

这样请求就会带上登录信息了，一般已经登录的用户可以通过 [`viewer`][viewer_url] 来获取信息，比如我要拿到我的 用户名、昵称、email、头像 等等 :

```js
{
    query: `
        query {
            viewer {
                name
                email
                bio
                avatarUrl
            }
        }
    `
}
```

![返回结果][result_user]

### 获取 repository 下的文件列表

[`repository`][repository_url] 仓库， [`object`][object_url] 为当前仓库对象，[`tree`][tree_url] 目录树。

```js
{
    query: `
        query($name_of_repository: String!) {
            viewer {
                repository (name: $name_of_repository) {
                    name
                    object (expression: "master:") {
                        ... on Tree {
                            entries {
                                name
                                type
                            }
                        }
                    }
                }
            }
        }
    `,
    variables: {
        'name_of_repository': name
    }
}
```

> note: `$name_of_repository` 是 repository 的名字，`expression` 是一个表达式 `'xx：xx'` 前面是分支，后面是路径。
> 这里 'master:' 代表root

<img src="https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574241878332.jpg" width="256" >

### 获取单个文件数据

和上面一样类似拿到单个 [`object`][object_url]，再通过 [`Blob`][blob_url]。

```js
{
    query: `
        query($name_of_repository: String!) {
            viewer {
                repository (name: $name_of_repository) {
                    name
                    object (expression: "master:.gitignore") {
                        ... on Blob {
                            text
                        }
                    }
                }
            }
        }
    `,
    variables: {
        'name_of_repository': name
    }
}
```

> note: `object expression 'master:.gitignore'` 指向了 .gitignore 文件

<img src="https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574242351509.jpg" height="64" >


### 获取提交信息

通过 [`ref`][ref_url] 和 [`Commit`][commit_url] 获取提交列表、某个文件的提交列表

```js
query: `
    query($name_of_repository: String!) {
        viewer {
            repository (name: $name_of_repository) {
                ref (qualifiedName: "master") {
                    target {
                        ... on Commit {
                            history(first: 10, path: "README.md") {
                                edges {
                                    node {
                                        messageHeadline
                                        committedDate
                                    }
                                }
                            }
                        }
                    }
                }
            }
        }
    }
`,
variables: {
    'name_of_repository': 'web-test-zhihu'
}
```

> note: `ref` :repository 的 引用，`qualifiedName: 'master'` 代表 master 分子的引用
> `history path` 路径，非必填，不填写则返回 该分支所有的 commit list ,否则返回和路径匹配的对象的 commit list

结果：<img src="https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574243101776.jpg" width="256">

### 获取所有 带有某个 label 的 Issue

关键字： [`label`][label_url] 、 [`Issue`][issue_url]。
两种方法。

第一种，直接查找某 label 有关的 issue

```js
{
    query: `
        query($name_of_repository: String!) {
            viewer {
                repository (name: $name_of_repository) {
                    issues (labels: ["article"], first: 10) {
                        nodes {
                            title
                        }
                    }
                }
            }
        }
    `,
    variables: {
        'name_of_repository': 'xxx',
    }
}
```

> note: labels 参数接收一个数组

结果：<img src="https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574337041329.jpg" height="148">

第二种，先找 label 再找到该 label 有关的所有 issue

```js
{
    query: `
        query($name_of_repository: String!) {
            viewer {
                repository (name: $name_of_repository) {
                    labels (first: 10, query: "article") {
                        nodes {
                            issues (first: 10) {
                                nodes {
                                    title
                                }
                            }
                        }
                    }
                }
            }
        }
    `,
    variables: {
        'name_of_repository': 'xxx'
    }
}
```

结果：<img src="https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574337333984.jpg" width="286">

两种方法其实差不多,从这里可以看出 graphql 真的很灵活！

****

## 未完待续

[blog_post]:https://github.blog/2016-09-14-the-github-graphql-api/
[developer_github_v4]:https://developer.github.com/v4/
[personal_access_token]:https://help.github.com/en/github/authenticating-to-github/creating-a-personal-access-token-for-the-command-line

[viewer_url]:https://developer.github.com/v4/object/user/
[repository_url]:https://developer.github.com/v4/object/repository/
[object_url]:https://developer.github.com/v4/interface/gitobject/
[tree_url]:https://developer.github.com/v4/object/tree/
[blob_url]:https://developer.github.com/v4/object/blob/
[ref_url]: https://developer.github.com/v4/object/ref/
[commit_url]: https://developer.github.com/v4/object/commit/
[issue_url]: https://developer.github.com/v4/object/issue/
[label_url]: https://developer.github.com/v4/object/labelconnection/

[result_user]: https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574239379939.jpg
[result_file_list]: https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574241878332.jpg
[result_file_blob]: https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574242351509.jpg
[result_commit]: https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574243101776.
[reault_issue]: https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574337041329.jpg
[result_label]: https://raw.githubusercontent.com/jwdzzhz777/blog/master/assets/github_api_v4/1574337333984.jpg
