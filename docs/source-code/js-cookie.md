## 介绍

在前端的工作中，我们经常会使用 cookie 来存储一些数据。由于 document.cookie 的 API 实在是难以容忍，所以，我们经常会自己封装或者使用第三方的 cookie 读/写操作库。

我再工作中用到的是 js-cookie.js 这个库，其实网上类似的库有好多，可是我们团队为什么选择了它呢？带着好奇心，我去读了一下它的源码。

天呐！！！封装的极其简单又严谨，还提供了两个读/写的钩子，让我们可以做一些个性化的需求。瞬间被这个库给转粉了。接下来，我们就来读一下这个库的源码吧！

> 注：本文使用的 js-cookie 的源码版本为：v3.0.1



---



## 目录

- [API介绍](#API介绍)
  - [set(key, value[, options])](#set(key, value[, options]))
  - [get(key)](#get(key))
  - [remove(key[, options])](#remove(key[, options]))
  - [options详细介绍](#options详细介绍)
    - [Options.domian](#Options.domian)
    - [Options.path](#Options.path)
    - [Options.expires](#Options.expires)
    - [Options[max-age]](#Options[max-age])
    - [Options.secure](#Options.secure)
    - [网络安全相关](#cookie 关于网络相关的属性还有以下几点：)
    - [什么情况下请求会携带cookie](#什么情况下请求会携带cookie)
    - [cookie存储优先级](#cookie存储优先级)
  - [withAttributes(options)](#withAttributes(options))
  - [withConverter(options)](#withConverter(options))
- [源码解读](#源码解读)
  - [整体概括](#整体概括)
  - set(key, value[, options])
  - Converter.read(value, key)
  - get(key)
  - Converter.write(value, key)
  - remove(key[, options])
  - withAttributes(options)
  - withConverter(options)
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

| key 允许的特殊符号 | value 允许的特殊符号                    |
| ------------------ | --------------------------------------- |
| # ￥ & + ^ ` \|    | # ￥ & + / :C :D :E :F @ [ ] ^ ` { \| } ||

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

关于 Cookies.set(key, val[, options]) 的 options，其实就是 document.cookie 的相关设置。所以，这里会对 document.cookie 所有的设置做一个详细的介绍。



##### Options.domian

>  cookie 存储的域名空间。子域可以访问主域的域名，主域不能访问子域的域名

我们先看原生的 Domain：

```js
// 不设置 Domian
document.cookie = 'username=x';

// 设置 Domian
document.cookie = 'username=y; domain=yjh1992.github.io';
```

看一下控制台的截图：

![alt 控制台 Cookies](/blog/source-code/images/js-cookie/cookie_domain_default.png)

当我们不指定 cookie Domian 时，cookie 会把 Domian 设置成当前页面的主域（.yjh1992.github.io）。前面的点是什么意思呢？是说 yjh1992.github.io 的子域名也可以访问这个 cookie。比如：m.yjh1992.github.io、www.yjh1992.github.io 等。当然，我们存储域名的时候也可以自己指定主语。

```js
// 假设现在的页面URL：https://www.yjh.com/account

document.cookie = 'token=xxxx; domain=.yjh.com';
// 此时 yjh.com 下的所有子域名都能使用这个cookie
// 比如：www.yjh.com、m.yjh.com、test.m.yjh.com ...
```



既然我们可以指定 cookie 存储的域名，那我们能不能随便指定一个跟我们页面没关系的域名呢？

```js
// 我们的页面URL：https://yjh1992.github.io/blog/source-code/js-cookie

// 指定百度的主域
document.cookie = 'xxxxxxxx=yyyyyyyy; domain=.baidu.com';
// 指定同级子域名
document.cookie = 'xxxxxxxx=yyyyyyyy; domain=yjh2008.github.io';
```

答案是不可以。我们只能指定当前的域名、当前域名的子域名或者当前的主域名。其他的域 cookie 都会无效。在火狐浏览器还会给出警告信息。



使用 js-cookie 设置 Domain 只需要指定第三个参数即可：

```js
Cookies.set('username', 'yjh', { domain: 'yjh1992.github.io' });
```



##### Options.path

> cookie 存储的路径空间。子路径可以访问主路径的 cookie，主路径不能访问子路径的 cookie。

我们看一下原生的 Path：

```js
// 我们当前的页面地址：https://yjh1992.github.io/user

// 不设置 Path
document.cookie = 'username=x';

// 设置 Path
document.cookie = 'username=y; path=/user';
```

看一下控制台的截图：

![alt 控制台 Cookies](/blog/source-code/images/js-cookie/cookie_path_view.png)



cookie 在不设置 Path 时，默认是当前页面的路径。所以 document.cookie = 

'username=x' 的 Path 是 '/'。这时你可能会疑惑？当前页面的地址不是 https://yjh1992.github.io/user 吗？拿到当前页面的路径不是 ’/user‘ ？

这里我们来说一下，cookie 对当前页面路径是如何解析的呢？看一下下面的例子：

| location.pathname | cookie_path |
| ----------------- | ----------- |
| /user             | /           |
| /user/            | /user       |
| /user/info        | /user       |

也就是说 cookie 解析页面路径是 window.location.pathname 的最后一个 ’/‘ 及 '/' 后面的所有内容都会被抹掉，但最少会有一个 '/' 来标识根路径。



使用 js-cookie 设置 Path 只需要指定第三个参数即可：

```js
Cookies.set('username', 'yjh', { path: '/' });
```



##### Options.expires

> cookie 过期的具体日期。格式：UTC时区事件。例如：Wed, 14 Jun 2017 07:00:00 GMT。
>
> 使用 Date.toUTCString() 来输出这个格式。

我们看一下原生的 expires：

```js
// 不设置 Expired
document.cookie = 'username=yjh';

// 设置 Expired, cookie 将在 2022年8月8日 过期
document.cookie = `age=29; expires=${ new Date('2022/08/08').toUTCString() }`;
```

看一下控制台的截图：

![alt 控制台 Cookies](/blog/source-code/images/js-cookie/cookie_expires_view.png)

我们可以看到指定了 expires 的 cookie，和没指定 expires 的 cookie 区别如下：

| 指定了 expires           | 未指定 expires |
| ------------------------ | -------------- |
| 2022-08-07T16:00:00.000Z | Session        |

我们发现未指定 expires 的 Expires / Max-Age 的值是 Session。那 Session 是什么意思呢？

> Session 含义：cookie 会在对话结束时过期

什么是对话结束时？是关闭浏览器当前tab，还是关闭整个浏览器？测试结果如下：

| 浏览器 | 关闭当前 tab | 关闭浏览器  |
| ------ | ------------ | ----------- |
| Chrome | cookie 存在  | cookie 消失 |
| 火狐   | cookie 存在  | cookie 消失 |
| Safari | cookie 存在  | cookie 消失 |

测试结果很明显了，那就是在关闭浏览器后。Expires / Max-Age 为 Session 的 cookie 就会过期。



因为 Expires 指定的是具体的过期时间。所以，如果我们想设置一个永久有效的 cookie，可以使用下面的日期设置：

```js
document.cookie = `username=yjh; expries=${ new Date('9999/12/32 23:59:59').toUTCString() }`

// 虽然这并不是一个永久有效的日期。但是相信我，我们写的程序活不到那一天的。
```



同样，我们也可以指定一个过去的时间来让 cookie 过期，从而达到删除 cookie 的效果。

```js
document.cookie = `username=yjh; expries=${ new Date('1998/08/08').toUTCString() }`
```



我们再回到 js-cookie。毕竟每次使用 new Date('yyyy/mm/dd').toUTCString() 比较麻烦。所以，js-cookie 对 Expires 参数做了一层封装。我们只需要指定一个数字即可，**注意：这个数字的单位是 天**；

```js
// 1天后过期
Cookies.set('username', 'yjh', { expires: 1 });
// 30天后过期
Cookies.set('username', 'yjh', { expires: 30 });
// 1年后过期
Cookies.set('username', 'yjh', { expires: 365 });
// 半天后过期
Cookies.set('username', 'yjh', { expires: 0.5 });
```

虽然 js-cookie 对 Expires 的封装方便了我们指定 cookie 的天数。但是，在开发过程中，有时候我们一般指定具体的日期更为方便。比如我们有一个10月1日的活动，过期日期是10月7日。但是用户可能参加的日期不一样，有些人是10月1日参加的，有些人是10月3日参加的。我们去算 cookie 存储的天数，就不如指定一个具体的日期更方便。

那我们可不可指定一个具体的日期呢？虽然 js-cookie 在文档中并没有提到可以指定具体的日期，但是通过看源码，我们发现是可以的。指定具体日期的用法如下：

```js
// 2021年10月1日过期
Cookies.set('username', 'yjh', { expires: new Date('2021/10/01') });
```

这就是读源码的好处之一吧！



##### Options[max-age]

>cookie 存储的时长。单位：秒。
>
>相对较新的一个属性，IE8之前的版本不支持。[查看详细支持情况](https://caniuse.com/?search=cookie%20Max-Age)

我们看一下原生的 Max-Age：

```js
// 不设置 Max-Age
document.cookie = 'username=yjh';

// 设置 Max-Age, cookie 将在 10 秒后过期
document.cookie = `age=29; max-age=10`;
```

看一下控制台的截图：

![alt 控制台 Cookies](/blog/source-code/images/js-cookie/cookie_maxage_view.png)

我们看到不设置 Max-Age 时，Expires/Max-Age 部分显示的是 Session，我们上面已经提过了。设置了 Max-Age 后，Expires/Max-Age 部分展示的格式和设置 Expires 时是一样的。

如果，我们同时设置 Expires 和 Max-Age，那么谁会生效呢？测试代码如下：

```js
/*
* 注：做测试当天日期是：2021年10月19日 晚上8点04分
*/

// 先设置 Expires，后设置 Max-Age
document.cookie = `username=yjh; expires=${ new Date('9999/12/32 23:59:59').toUTCString() }; max-age=10`;

// 先设置 Max-Age，后设置 Expires
document.cookie = `age=29; max-age=10; expires=${ new Date('9999/12/32 23:59:59').toUTCString() }`;
```

测试结果：

| 先设置 Expires，后设置 Max-Age | 先设置Max-Age，后设置 Expires |
| ------------------------------ | ----------------------------- |
| Max-Age 生效                   | Max-Age 生效                  |

是的，Expires 和 Max-Age 是有优先级的，而 **Max-Age 的优先级最高**。



其实，js-cookie 并没有提供 Max-Age 这个 API，但是我们如果设置了，也是能够生效的。

```
Cookies.set('username', 'yjh', { 'max-age': '10' });
```

但是有一点**需要特别注意：Max-Age 的 value 应该是一个字符串类型**，要不然会报错。具体原因我们在读源码的时候会详细说明。



##### Options.secure

> 此 cookie 只能通过 https 协议访问。网络安全相关

也就是说设置了 secure 后，只有 https 的请求才能带上这个 cookie。所以，在传输层劫持数据的风险较低。但不代表设置了 secure 属性的 cookie 就是安全。因为这个 cookie 还是可以通过客户端去读写的，能够读写就有被篡改的可能。

使用方法如下：

```js
// 原生。在 https 下设置才能生效。
document.cookie = 'username=yjh; secure'

// js-cookie
Cookies.set('username', 'yjh', { secure: true });
```



##### cookie 关于网络相关的属性还有以下几点：

- HttpOnly

  如果想要避免 cookie 被篡改，服务端响应头的 set-cookie 字段可以添加 HttpOnly 属性。设置了 HttpOnly 属性的 cookie 对客户端来说是透明的，document.cookie 获取 cookie 时会忽略带有 HttpOnly 属性的 cookie。虽然客户端无法获取，但是我们可以在开发者工具中查看这个 cookie。HttpOnly 会显示一个对钩：

  ![alt 控制台 Cookies](/blog/source-code/images/js-cookie/httponly_view.png)

  千万不要觉得不能被篡改就安全了，因为**最危险的不是篡改，而是利用**。下面这个属性会说明这一点。

- SameSite

  这个属性也是属于服务端响应头的 set-cookie 功能。设置了这个属性的 cookie 在跨域请求时是不会被携带的。也就是说 a.yjh.com 页面，请求 b.yjh.com 的接口时，a.yjh.com 页面的 cookie 不会被请求携带的。

  此属性有 3 个值：

  | 值              | 说明                          |
  | --------------- | ----------------------------- |
  | SameSite=None   | 无论是否跨站都会发送 Cookie   |
  | SameSite=Lax    | 允许部分第三方请求携带 Cookie |
  | SameSite=Strict | 仅允许同站请求携带 Cookie     |

  这个属性就是来解决 cookie 被利用的问题。比如：张三在 gouwu.com 网站下来一个单，之后张三又去访问了另一个恶意网站（eyi.com）。eyi.com 这时候发送一个请求 gouwu.com/mai/1000/yuan/gouliang。这个请求就会带上张三在访问 gouwu.com 时存储的 cookie。假如说 gouwu.com 小额不需要支付密码，让我们恭喜张三，在不知情的情况下多了 1000 元狗粮。

  张三可能会说，eyi.com 无法获取到 gouwu.com 域名下存储的 cookie 啊！没错，但是获取不到不代表 cookie 就不存在吧！顺便说一下，这个就是 CSRF 攻击。总之，让我们再次恭喜张三。

- SameParty

  虽然 SameSite 给我们带来了安全，但是，它也给我们带来了麻烦。在现实生活，往往因为公司的不同业务线，会有不同的二级域名。比如：sleep.yjh1992.com、weekup.yjh1992.com。本来大家是可以跨域请求的，加了个 SameSite 就啥也不能够了。于是，Google 就提出来 SameParty（因为他们有太多二级域名了）。

  SameParty 属性可以让同一主体下的域名 cookie 共享。即带来了 SameSite 的安全，也解决了 SameSite 的问题。但是 SameParty 这个属性只是一个开关而已。背后的主体域名关联是 First-Party Sets 策略，想要了解的可以 Google 一下。

  

注意：**document.cookie 是无法设置 HttpOnly、SameSite 和 SameParty 这几个属性的**。



---



### withAttributes(options)

如果我们每次设置 cookie 都要指定一些公共的参数，比如 Domian、Path、Expires等，会显得很繁琐。所以，js-cookie 提供了一个设置默认配置的原型方法。此方法会返回一个新的实例。设置方法如下：

```js
const newCookie = Cookies.withAttributes({
  domain: '.yjh1992.github.io',
  path: '/',
  expires: 10, // 注意：js-cookie 的 expires 参数设置的单位是：天
  secure: true,
  'max-age': '60', // 注意：不在 API 内，但可以使用。value 需要是字符串类型，不然会报错
});

// 设置完默认配置后，使用新返回的实例进行 cookie 操作。
newCookie.set('username', 'yjh');
```



说明：

1. withAttributes 方法需要设置一个 JSON 对象，或者 undefined。其他的类型会报错。
2. withAttributes 方法具有继承特性。返回的新实例会继承祖先实例的所有属性（相同属性会被新设置的 value 覆盖），虽然，并没有祖先实例的概念。但确实会产生这个效果。看一下下面的例子：

```js
const cookie1 = Cookies.withAttributes({ path: '/' });
const cookie2 = cookie1.withAttributes({ domain: '.yjh1992.github.io' });
const cookie3 = cookie2.withAttributes({ expires: 10 });
const cookie4 = cookie3.withAttributes({ secure: true });
// 当前我们也可以设置非 cookie 的属性，同样会被继承，但是不生效
const cookie5 = cookie4.withAttributes({ yjh: 29 });
// 这里不传参也是OK的，会直接继承父实例的所有属性
const cookie6 = cookie5.withAttributes();

// 我们可以使用实例的 attributes 属性查看实例的属性
console.log('cookie1', cookie1.attributes);
console.log('cookie2', cookie2.attributes);
console.log('cookie3', cookie3.attributes);
console.log('cookie4', cookie4.attributes);
console.log('cookie5', cookie5.attributes);
console.log('cookie6', cookie6.attributes);
```

看一下控制台的截图：

![alt 控制台 Cookies](/blog/source-code/images/js-cookie/withAttributes_view.png)

js-cookie 实现这个效果的代码只有一行，极其的简单。我们看源码的时候会看到它如何实现的。



---



### withConverter(options)

这个是我最喜欢的一个 API，读/写拦截器。它允许在我们读/写 cookie 之前做一层拦截处理，并返回一个新的实例。先看一下用法吧！

```js
const cookie = Cookies.withConverter({
  // 在执行 cookie.get(key, value) 时会先走这个方法，最终我们 get 的 value 是 read 函数的返回值
  read(value, key) {
    return value;
  },
  // 在执行 cookie.set(key, value) 时会先走这个方法，最终我们 set 的 value 是 write 函数的返回值
  write(value, key) {
    return value;
  }
});
```

我们知道了它的用法，那他有什么用途呢？因为它把真正读/写的权利完全暴露给了开发者，理论上来说我们可以用它做任何我们想到的事情。

比如：我们之前的例子设置的值都是普通类型，我们如果想要设置要给对象是需要执行 JSON.stringify() 的。因为，js-cookie 并没有为我们提供可以直接设置值为对象的方法。为什么 js-cookie 不给我们提供呢？是因为有 withConverter(options) API 呀！

```js
const cookieJSON= Cookies.withConverter({
  read(value, key) {
    let result = null;
    try {
      result = JSON.parse(value);
    } catch(e) {}
    return result;
  },
  write(value, key) {
    if (typeof value !== 'undefined') {
    	return JSON.stringify(value);
    }
  }
});
```



再比如：我们需要一个只读 cookie，点类似于对象的只读属性。

```js
const cookieJSON= Cookies.withConverter({
  read(value, key) {
    if (key === 'onlyReay') {
    	return 'anything value';
    }
    
    return value;
  },
  write(value, key) {
    if (key === 'onlyReay') {
    	return void 0;
    }
    
    return value;
  }
});
```

其他更多的用法，就等着你自己去发掘吧！



---



## 源码解读

前面说了那么多，终于到了源码解读的部分了。其实我一开始想写的就只有源码解读，但为什么却还要写那么一大堆 API介绍呢？因为只有你 **足够了解 API，读源码才有意义**。



### 整体概括

很多时候读源码时会遇到这样一个问题：**打开一看，一堆代码，不知道如何下手**。

解决这个问题的办法就是：把所有的函数、对象都关闭，先整体看一下源码分为哪几个部分。我们开始啦！

> 备注：为了方便阅读，我删除了代码中的注释，并添加了自己的解读注释。

```js
/*
* 一开始是一堆模块化的判断，这里不需要看，我们直接跳过去
*/
;
(function (global, factory) {
  typeof exports === 'object' && typeof module !== 'undefined' ? module.exports = factory() :
  typeof define === 'function' && define.amd ? define(factory) :
  (global = global || self, (function () {
    var current = global.Cookies;
    var exports = global.Cookies = factory();
    exports.noConflict = function () { global.Cookies = current; return exports; };
  }()));
}(this, (function () { 'use strict';
                      
	// 一个 assign 的方法，我们先不用去管它是做什么的
  function assign (target) { /* ... */ }
 	// 名字以 default 开头，大概是一个默认的配置对象
  var defaultConverter = { /* ... */ };
  // 一个 inti 的方法，看名字就知道这是一个初始化的方法
  function init (converter, defaultAttributes) { /* ... */ }
	// 这里调用了 init 初始化方法，并且传入了默认配置，并且传入了第二个参数 path: '/'。
  var api = init(defaultConverter, { path: '/' });
	// 这里 return 了 api。也就是说我们最终使用的对象，其实是这个 api.
  // 而 api 是通过 init 方法返回的，那么 init 就是我们下一个要看的函数
  return api;

})));
```



init 方法我们也不要着急看代码，也是大概的看一下它分为哪些部分。

```js
function init (converter, defaultAttributes) {
  // 这里是一个 set 方法，看名字和参数我们也能够猜出来，这就是 Cookies.set() 方法了
	function set (key, value, attributes) { /* ... */ }
	// 这里是一个 get 方法，看名字和参数那它应该就是 Cookies.get() 方法了
	function get (key) { /* ... */ }

  // 这里 return 了一个 Object.create 创建的对象。
  // 因为这个 Object.create API 不常用，这里解释一下。
	// - 它会返回一个对象 {}
  // - 第一个参数是返回对象的原型
	// - 第二个参数是设置对象上的属性
	return Object.create(
    {
      // 原型上挂在了上面定义的 set 方法
			set: set,
      
      // 原型上挂在了上面定义的 get 方法
     	get: get,
      
      // 这里是一个 Cookies.remove() 方法
      // 你可能会奇怪，为什么 remove 方法不在上面定义呢？因为它的实现实在是太简单啦！放在下面也不影响阅读
   		remove: function (key, attributes) { /* ... */ },
      
      // 这里是我们前边提到的 withAttributes 方法
      // 作用：设置公共的一些 cookie 参数，并返回一个新的实例对象
      // 它有一个继承效果，可以继承祖先节点的属性。
			// 我还提到过它只用 1 行代码就实现了这个效果。哈，我们一会儿就会看到它
   		withAttributes: function (attributes) { /* ... */ },
      
      // 这里是读/写拦截器，我最喜欢的一个 API，上面我也有提到
      withConverter: function (converter) { /* ... */ }
  	},
  	{
      // 这里是实例当前的 cookie 属性，它的值是 defaultAttributes
      // 这里有两点怕有疑问，解释一下：
      // 1. Object.freeze 是什么？
      // Object.freeze 是冻结对象的方法，被冻结的对象不能再被修改。比如：不能修改/新增/删除已有属性
      // 2. attributes 的值是一个带有 value 的对象，那我们获取时要用 Cookies.attributes.value 吗?
      // Object.create 的第二个参数是设置对象上的属性。属性需要写成描述对象的形式。这里就相当于：
      // { attributes: defaultAttributes }
			// 想要详细了解，可以查看： https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Object/freeze
      attributes: { value: Object.freeze(defaultAttributes) },
      
      // 看名字感觉和 withConverter 有联系对不对，这就是当前实例上的读/写拦截器
   		converter: { value: Object.freeze(converter) }
  	}
	)
}
```



我们看完了 init 方法，知道了 init 最终会返回一个对象，这个对象的控制台打印结果如下:

![alt 控制台 Cookies](/blog/source-code/images/js-cookie/cookies_console_view.png)

