## 介绍

在前端的工作中，我们经常会使用 cookie 来存储一些数据。由于 document.cookie 的 API 实在是难以容忍，所以，我们经常会自己封装或者使用第三方的 cookie 读/写操作库。

我再工作中用到的是 js-cookie.js 这个库，其实网上类似的库有好多，可是我们团队为什么选择了它呢？带着好奇心，我去读了一下它的源码。

天呐！！！封装的极其简单又严谨，还提供了两个读/写的钩子，让我们可以做一些个性化的需求。瞬间被这个库给转粉了。接下来，我们就来读一下这个库的源码吧！



---



## 目录

- API介绍
  - set
  - get
  - remove
  - options详细介绍
  - withAttributes
  - withConverter
- 源码解读
  - 整体概括
  - get
  - Converter.read
  - set
  - Converter.write
  - remove
  - withAttributes
  - withConverter
  - 防止冲突
- 个人思考



---



## API介绍

> 引入方式这里就不介绍了，既然都要阅读它的源码了，就默认理解为你已经使用过它了。

API 的介绍部分是很重要的，因为这里面我会掺入很多跟 API 相关但又不是 API 的内容。当看完这些内容后，我们后期读起源码来会非常容易。所以，请一定不要跳过它。



### set(key, value[, options])

最简单的例子

```js
Cookies.set('key', 'value');
```



如果我们想设置一些参数的话，可以传第 3 个参数。参数所有的介绍可以查看 [options详细介绍](#options详细介绍)

```js
// 设置 Domain
Cookies.set('name', 'value', { domain: 'yjh1992.github.io' });

// 设置 Path
Cookies.set('name', 'value', { path: '/blog/source-code' });
```



这里有一个点需要注意：**key 和 value 的内容仅允许部分特殊符号，其余特殊符号都会被 encodeURIComponent 转码**。

> key 允许的特殊符号：#、￥、&、+、^、`、|。
>
> value 允许的特殊符号：#、￥、&、+、/、:C、:D、:E、:F、@、[、]、^、`、{、|、}。



为什么会有这些限制呢？因为 [RFC 6265](https://datatracker.ietf.org/doc/html/rfc6265) 规范中对 cookie 做了一些要求，其中的 [第4章](https://datatracker.ietf.org/doc/html/rfc6265#section-4) 是针对服务器 Set-Cookie 的一些约束，[第5章](https://datatracker.ietf.org/doc/html/rfc6265#section-5) 是对浏览器的要求。

当然，你不需要去看，大概意思就是说：服务端应当更严谨的对待 cookie 设置的内容，而浏览器不需要有特殊符号的限制，可以更宽松一点。也就是说上面的符号限制，在浏览器中 **几乎都是支持的**。

之所以 js-cookie 库会做这些限制，就是因为也许你的程序在大部分浏览器都运行正常，所以一旦遇到严格遵守规范的浏览器，那对你的应用程序来说无疑是灾难性的。js-cookie 为了避免这种情况发生，所以做了相关的限制。



---



### get(key)

说到 get 方法就简单了许多。它有且只有 1 个参数。下面是一个 get 的例子：

```js
Cookies.get('key');

// 当我们获取一个不存在的 cookie 时，get 方法返回 undefined
console.log(Cookies.get('nothing'));
// 结果：undefined
```



我不知道读者是否会有一个疑惑，为什么 get 只有一个参数？难道不支持 options 吗？是不支持的。这不是 js-cookie 这个库的原因，而是原生 cookie 的原因。让我们看看下面这个例子：

```js
/*
* 假设当前页面地址是： https://yjh1992.github.io/blog
*/
document.cookie = 'username=yjh; Domain=yjh1992.github.io';
document.cookie = 'age=29; Domain=.github.io';

console.log(document.cookie);
// 结果：username=yjh; age=29
```

也就是说，我们通过 document.cookie 获取到的内容，是不包含除了 key 和 value 之外的信息的。那我们这是其他的信息有什么用呢？会在 [options详细介绍](#options详细介绍) 中说明。



那假如我设置了两个同名但不同路径的 cookie，那浏览器会如何表现呢？

```js
/*
* 假设当前页面的地址是 https://yjh1992.github.io/login
* 我们设置了以下 cookie。
* 浏览器是显示最后一条，还是会显示两条呢？
*/

Cookies.set('username', 'x', { path: '/' });
Cookies.set('username', 'y', { path: '/login' });
```

此时浏览器（Chrome、Safari和火狐）的控制台表现如下：

![alt 控制台 Cookies](/blog/source-code/images/js-cookie/username_cookie_control.png)



获取一下 cookie 的值，看一下浏览器中的表现：

```js
console.log(document.cookie);
// 测试的3个浏览器结果一致：username=y; username=x
```

我们看到，username=y 是排在前面的。这也非常符合预期，我们最先获取到的数据，就应该是最后一次设置的内容。也就是说如果我们获取 cookie 的方式合理的话，那么，我们一定能获取到我们最新设置的内容。



---



### remove(key[, options])

remove 是支持传递 options 的，也就是说我们可以指定 options 来参数一些特定的内容。

最简单的 remove 例子：

```js
Cookies.set('username', 'yjh');
console.log(Cookies.get('username'));
// 结果：yjh

Cookies.remove('username');
console.log(Cookies.get('username'));
// 结果: undefined
```



看一下参数 options 的例子

```js
Cookies.set('username', 'yjh', { path: '/' });

Cookies.remove('username', { path: '/' });
```



既然 remove 方法可以传参 options，那你一定还记得我们在 [Cookies.get 小节](#get(key)) 提到的 **同名但不同路径** 的例子。当时浏览器（Chrome、Safari和火狐）的存储内容如下：

![alt 控制台 Cookies](/blog/source-code/images/js-cookie/username_cookie_control.png)

那如果我们使用 Cookie.remove('username', { path: '/' }) 会同时删除两个 username，还是只能删除一个 username 呢？

```js
/*
* 当前页面地址：http://192.168.236.11:1234/jsCookie.html
*/
Cookies.set('username', 'x', { path: '/' });
Cookies.set('username', 'y', { path: '/jsCookie.html' });

Cookies.remove('username', { path: '/' });
```

测试浏览器表示一直，结果如下：

![alt 控制台 Cookies](/blog/source-code/images/js-cookie/remove_cookie_console.png)

也就是说，现在我们去 Cookie.get('username') 依然会有值。**如果我们存储的字段不是 username，而是用户登录态 token。那就会出现用户无法退出登录的情况**。所以，一定要小心这种情况。

---



### options详细介绍

