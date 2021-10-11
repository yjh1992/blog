## 介绍

在我们进行移动端开发的时候，click 事件会存在 300ms 延时的现象。具体的原因在于：移动端的屏幕偏小，为了便于用户阅读。所以，在用户进行双击时，浏览器会出现放大视图的默认行为。

当用户点击屏幕两次时，浏览器如何判定用户的行为是双击，还是点击了两次呢？就在于两次点击行为前后的间隔是不是在 300ms 之内。如果在 300ms 之内，浏览器就判定为 “双击行为”。否则，就认为点击了 2 次。

所以，当 用户点击行为 发生时，浏览器不会立即触发 click 事件 。而是等待 300ms 。如果在 300ms 之内再次发生了 用户点击行为 ，那么浏览器就会触发 dblclick 事件。如果在 300ms 之内未再次发生 用户点击行为，则浏览器触发 click 事件 。

---



## 解决方案

fastclick.js 就是用来解决这 300ms 延迟的一个 JS 库。它的原理非常简单：使用移动端的 touch 事件来模拟 click 事件。

比如：在 docuemnt.body 上绑定 touchStart 和 touchEnd 事件，touchStart 事件执行时记录一下 event.target 。touchEnd 事件执行时判断一下当前的 event.target 是否等于 touchStart 事件执行时记录的 event.target 。如果相等，就创建一个 click 自定义事件。最后，使用 event.target 触发自定义的 click 事件。

当然，以上知识核心的思路。真正实现起来并没有这么简单，还要考虑很多场景和各个手机下的兼容性问题。接下来，就让我们开始源码的阅读吧！

[fastclick.js github 地址](https://github.com/ftlabs/fastclick)

---



## 源码解读

源码解读我们会分为一下几个部分：

1.  [API介绍](#API介绍) -- 了解如何使用，才能知道内部代码是用完成了哪些事情
2.  [整体概括](#整体概括) -- 整体看一下 fastclick.js 的源码分为几个部分
3.  [详细解读](#详细解读) -- 从 fastclick.js 的入口开始分析整个链路做了哪些事情

---



### API介绍

```js
/**
* @param {HTMLElement} layer
* @param {Object} options
* 	- options.touchBoundary
* 	- options.tapDelay
* 	- options.tapTimeout
*/
FastClick.attach(layer[, options])
```

参数说明：

| params  | 说明                     |
| ------- | ------------------------ |
| layer   | fastclick 的事件代理 DOM |
| options | fastclick 配置对象       |

options 说明

| option        | 默认值 | 单位 | 说明                                                                                                 |
| ------------- | ------ | ---- | ---------------------------------------------------------------------------------------------------- |
| touchBoundary | 10     | px   | touchMove 事件的移动边界，移动距离超过这个值时，fastclick 将不在派发 click 事件。认为 click 事件取消 |
| tapDelay      | 200    | ms   | fastclick 事件触发触发间隔，在此间隔内，fastclick 只会触发一次                                       |
| tapTimeout    | 700    | ms   | 从 touchEnd 与 touchStart 的触发间隔超过这个值，将会取消 fastclick 事件                              |

初始化：

```js
if ("addEventListener" in document) {
	document.addEventListener("DOMContentLoaded", function () {
    FastClick.attach(document.body);
	}, false);
}
```

---



## 整体概括

我们已经了解了 Fastclick 相关的 API ，接下来我们来看一 Fastclick 源码 有哪几个部分。

> tip：为了方便阅读，我删除了源码相关的注释。并把相关的代码调整了位置

```js
(function () {
	"use strict";

	/*
	* 首先是定义了一些跟设备相关的内部变量，因为在这些设备中需要做一些特殊处理
	* 我们先不用关心在这些设备下具体要做哪些处理，我们目前就是先整体过一遍 Fastclick 的整体结构，对整体有个大概的印象即可。
	* 当我们开始详细的解读时，会对这些特殊设备做详细的说明
	*/

	// 检测设备是否为： Window Phone
	var deviceIsWindowsPhone = navigator.userAgent.indexOf("Windows Phone") >= 0;
	// 检测设备操作系统是否为：Android
	var deviceIsAndroid = navigator.userAgent.indexOf("Android") > 0 && !deviceIsWindowsPhone;
	// 检测设备操作系统是否为：iOS
	var deviceIsIOS = /iP(ad|hone|od)/.test(navigator.userAgent) && !deviceIsWindowsPhone;
	// 检测设备操作系统版本是否为：iOS 4+
	var deviceIsIOS4 = deviceIsIOS && /OS 4_\d(_\d)?/.test(navigator.userAgent);
	// 检测设备操作系统是否为：iOS 6+ ~ iOS 7+
	var deviceIsIOSWithBadTarget = deviceIsIOS && /OS [6-7]_\d/.test(navigator.userAgent);
	// 检测设备是否为：黑莓手机
	var deviceIsBlackBerry10 = navigator.userAgent.indexOf("BB10") > 0;


	/*
	* 然后我们看到了一个 Fastclick 的构造函数，这个函数支持两个参数 layer 和 options。
	* 官方文档给出的初始化 API：FastClick.attach(layer[, options])
	* 我们看到 调用 FastClick.attach 也是用了 layer 和 options 两个参数
	* 那 Fastclick.attach 和 FastClick 构造函数之间又有什么关系呢？我们可以记一下这个点，在详细解读源码的时候，我们去看看他们之间的关系
	*/

  	function FastClick(layer, options) { /*...*/ }


	/*
	* 接下来定义了一些 FastClick 的原型方法，我们先大概过一下都有哪些方法
	*/

	// 是否需要使用原始的 click。因为一些特殊的元素，只能通过原生的 click 事件来触发
	FastClick.prototype.needsClick = function (target) { /*...*/ };
	// 点击的是否需要获取焦点。比如我们点击的是一个 input 输入框。如果元素是需要获取焦点的，那 FastClick 又会做哪些操作呢？
	FastClick.prototype.needsFocus = function (target) { /*...*/ };
	// 触发 自定义 click 事件。也就是没有 300 毫秒延迟的 Click 事件
	FastClick.prototype.sendClick = function (targetElement, event) { /*...*/ };
	// 确定一个自定义事件的类型。FastClick 自定义事件不是 click 吗？难道还会自定义其他的事件？是的，在某种情况下定义的是 mousedown 事件，而非 click 事件。哈哈哈，好奇吗？
	FastClick.prototype.determineEventType = function (targetElement) { /*...*/ };
	// 让点击的元素获取焦点。因为 Fastclick 解决了 click 事件延迟，也解决了 获取焦点事件 的延迟。因为同样都是通过 点击行为 触发
	FastClick.prototype.focus = function (targetElement) { /*...*/ };
	// 用来解决父节点是一个带滚动条可以滑动的元素所带来的问题
	FastClick.prototype.updateScrollParent = function (targetElement) { /*...*/ };
	// 找到当前点击的节点 DOM 元素。因为有可能点击的是 文本 元素
	FastClick.prototype.getTargetElementFromEventTarget = function (eventTarget) { /*...*/ };

	/* ------ 核心代码 Start ------ */
	// touchstart 事件，用来模拟 click。
	FastClick.prototype.onTouchStart = function (event) { /*...*/ };
	// touchmove 事件，用来模拟 click。核心代码
	FastClick.prototype.onTouchMove = function (event) { /*...*/ };
	// touchend 事件，用来模拟 click。核心代码
	FastClick.prototype.onTouchEnd = function (event) { /*...*/ };
	/* ------ 核心代码 End ------ */

	// 检测 touchmove 事件有没有超过 option.touchBoundary 的值。如果超过会认为用户行为不是 click 行为
	FastClick.prototype.touchHasMoved = function (event) { /*...*/ };
	// 找到点击元素的目标元素。哈哈哈，不太理解对不对。也就是说点击元素是 Label  标签时，找到 Label 指向的目标节点。
	FastClick.prototype.findControl = function (labelElement) { /*...*/ };
	// touch 事件取消方法。用来重置一些标记
	FastClick.prototype.onTouchCancel = function () { /*...*/ };
	// 阻止一些 Event 事件。比如下面的 onClick 方法
	FastClick.prototype.onMouse = function (event) { /*...*/ };
	// 主要用于阻止 click 事件。为什么要阻止？因为 FastClick 已经触发了自定义的 Click 事件。如果不阻止，300ms 后就会再次触发原生的 Click 事件。
	FastClick.prototype.onClick = function (event) { /*...*/ };
	// 销毁方法。做了一些 Event 事件的移除
	FastClick.prototype.destroy = function () { /*...*/ };


	/*
	* 下面是 FastClick 构造函数上的静态方法
	*/

	// 当前是否需要 Fastclick。在一些浏览器或者某些情况下是不存在 300 毫秒延迟的，所以使用默认浏览器即可。具体哪些情况不存在 300 毫秒延迟，先留个小悬念
	FastClick.notNeeded = function (layer) { /*...*/ };
	// FastClick 构造函数的工厂方法。参数和 FastClick 构造函数一样，到这里，我们大概也能猜到它内部做了什么事情了
	FastClick.attach = function (layer, options) { /*...*/ };


	/*
	* 最后就是 js 模块化的判断了。
	*/

	if (typeof define === "function" && typeof define.amd === "object" && define.amd) {
		define(function () {
			return FastClick;
		});
	} else if (typeof module !== "undefined" && module.exports) {
		module.exports = FastClick.attach;
		module.exports.FastClick = FastClick;
	} else {
		window.FastClick = FastClick;
	}
})();
```

以上就是 FastClick 的整体情况了。现在就让我们开始看看详细的细节吧！

---



## 详细解读

我们首先从 Fastclick 初始化开始，看看都做了哪些事情。FastClick 官方示例：

```js
if ("addEventListener" in document) {
	document.addEventListener("DOMContentLoaded", function () {
    FastClick.attach(document.body);
	}, false);
}
```

Fastclick 初始化，调用了 attach 静态方法。我们在 [整体概括](#整体概括) 中提到 attach 是 FastClick 构造函数的一个工厂方法。参数如下：

| params  | 说明                     |
| ------- | ------------------------ |
| layer   | fastclick 的事件代理 DOM |
| options | fastclick 配置对象       |

在 FastClick 官方示例中，第一个参数 layer 传递的是 document.body。第二个参数 options 并没有传。那就让我们先从 attach 方法入手。



### FastClick.attach

```js
FastClick.attach = function(layer, options) {
  return new FastClick(layer, options);
};
```

我们看到，attach 方法的代码就只有一行，return 了一个 FastClick 构造函数的实例。也就是说，我们把 FastClick 初始化的方式改成下面这个样子，也是能够正常执行的：

```js
if ("addEventListener" in document) {
	document.addEventListener("DOMContentLoaded", function () {
    // 这里改成了直接 new FastClick 构造函数
    new FastClick(document.body);
	}, false);
}
```

在看到这里，也许你会有个疑问。既然直接 new FastClick(layer) 就能够进行初始化，FastClick 为什么还要脱裤子放屁搞个 **工厂方法** 呢？

首先我们要说明一下 **工厂方法** 这种设计模式的意义是什么。以下是个人的理解：

> 工厂方法：提供一个统一对外的实例化出口，用户只需要知道传递什么参数，不需要关心内部的具体实现。

因为 FastClick 的实例化足够简单，不足以体现 **工厂方法** 的好处。那就举个实例化稍微复杂一点的例子。比如：JQuery 的 $()。

$() 也是一个 **工厂方法** ，我们看一下它的几种常见用法：

```js
// 获取 DOM 节点
const ulDOM = $('body .content ul');

// 创建 DOM 节点
$('<li><p>我是一个 list-item</p></li>').appendTo(ulDom);

// DOM Ready 的回调
$(function() {
	//... DOMContentLoaded 事件完成
});
```

同样一个 $()，根据参数的不同，创建了不同的实例对象。用户只需要知道如何使用即可，不需要关心内部的实现细节。这就是 **工厂方法** 的好处。

再看 FastClick.attach 工厂方法，FastClick 实例化足够简单，也不会根据参数不同有太大的变化。所以，就算你觉得没必要使用 **工厂方法**，也不能有人去反驳你。但是做为我个人来说，我是非常认可这里使用一个 **工厂方法** 来作为实例化的统一出口的。就算没必要，那装个B它不香吗？

回归到源码上，既然 attach 直接调用的 FastClick 构造函数，那我们就看一下 FastClick 构造函数做了哪些事情吧！



### new FastClick

```js
function FastClick(layer, options) {
		var oldOnClick;

		options = options || {};

		/**
		 * Whether a click is currently being tracked.
		 *
		 * @type boolean
		 */
		this.trackingClick = false;


		/**
		 * Timestamp for when click tracking started.
		 *
		 * @type number
		 */
		this.trackingClickStart = 0;


		/**
		 * The element being tracked for a click.
		 *
		 * @type EventTarget
		 */
		this.targetElement = null;


		/**
		 * X-coordinate of touch start event.
		 *
		 * @type number
		 */
		this.touchStartX = 0;


		/**
		 * Y-coordinate of touch start event.
		 *
		 * @type number
		 */
		this.touchStartY = 0;


		/**
		 * ID of the last touch, retrieved from Touch.identifier.
		 *
		 * @type number
		 */
		this.lastTouchIdentifier = 0;


		/**
		 * Touchmove boundary, beyond which a click will be cancelled.
		 *
		 * @type number
		 */
		this.touchBoundary = options.touchBoundary || 10;


		/**
		 * The FastClick layer.
		 *
		 * @type Element
		 */
		this.layer = layer;

		/**
		 * The minimum time between tap(touchstart and touchend) events
		 *
		 * @type number
		 */
		this.tapDelay = options.tapDelay || 200;

		/**
		 * The maximum time for a tap
		 *
		 * @type number
		 */
		this.tapTimeout = options.tapTimeout || 700;

		if (FastClick.notNeeded(layer)) {
			return;
		}

		// Some old versions of Android don't have Function.prototype.bind
		function bind(method, context) {
			return function() { return method.apply(context, arguments); };
		}


		var methods = ['onMouse', 'onClick', 'onTouchStart', 'onTouchMove', 'onTouchEnd', 'onTouchCancel'];
		var context = this;
		for (var i = 0, l = methods.length; i < l; i++) {
			context[methods[i]] = bind(context[methods[i]], context);
		}

		// Set up event handlers as required
		if (deviceIsAndroid) {
			layer.addEventListener('mouseover', this.onMouse, true);
			layer.addEventListener('mousedown', this.onMouse, true);
			layer.addEventListener('mouseup', this.onMouse, true);
		}

		layer.addEventListener('click', this.onClick, true);
		layer.addEventListener('touchstart', this.onTouchStart, false);
		layer.addEventListener('touchmove', this.onTouchMove, false);
		layer.addEventListener('touchend', this.onTouchEnd, false);
		layer.addEventListener('touchcancel', this.onTouchCancel, false);

		// Hack is required for browsers that don't support Event#stopImmediatePropagation (e.g. Android 2)
		// which is how FastClick normally stops click events bubbling to callbacks registered on the FastClick
		// layer when they are cancelled.
		if (!Event.prototype.stopImmediatePropagation) {
			layer.removeEventListener = function(type, callback, capture) {
				var rmv = Node.prototype.removeEventListener;
				if (type === 'click') {
					rmv.call(layer, type, callback.hijacked || callback, capture);
				} else {
					rmv.call(layer, type, callback, capture);
				}
			};

			layer.addEventListener = function(type, callback, capture) {
				var adv = Node.prototype.addEventListener;
				if (type === 'click') {
					adv.call(layer, type, callback.hijacked || (callback.hijacked = function(event) {
						if (!event.propagationStopped) {
							callback(event);
						}
					}), capture);
				} else {
					adv.call(layer, type, callback, capture);
				}
			};
		}

		// If a handler is already declared in the element's onclick attribute, it will be fired before
		// FastClick's onClick handler. Fix this by pulling out the user-defined handler function and
		// adding it as listener.
		if (typeof layer.onclick === 'function') {

			// Android browser on at least 3.2 requires a new reference to the function in layer.onclick
			// - the old one won't work if passed to addEventListener directly.
			oldOnClick = layer.onclick;
			layer.addEventListener('click', function(event) {
				oldOnClick(event);
			}, false);
			layer.onclick = null;
		}
}
```

