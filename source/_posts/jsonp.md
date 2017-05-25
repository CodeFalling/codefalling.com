title: JSONP 实现
author: codefalling
tags:
  - javascript
  - jsonp
  - frontend
categories:
  - 前端
date: 2016-09-05 00:00:00
---

JSONP 是解决跨域问题的一种方案，不同于 JSON，其并不是一种数据交换格式，而只是一种绕过跨域的技巧。

<!-- more -->

## JSONP

JSONP 的原理非常简单，为了克服跨域问题，利用没有跨域限制的 script 标签加载预设的 callback 将内容传递给 js。一般来说我们约定通过一个参数来告诉服务器 JSONP 返回时应该调用的回调函数名，然后拼接出对应的 js。已微博 API 为例，这个参数名是 `_cb`。

我们可以写一个简单的版本：

```js
function JSONP({
  url,
  params,
  callbackKey,
  callback
}) {
  // 在参数里制定 callback 的名字
  params = params || {}
  params[callbackKey] = 'jsonpCallback'
    // 预留 callback
  window.jsonpCallback = callback
    // 拼接参数字符串
  const paramKeys = Object.keys(params)
  const paramString = paramKeys
    .map(key => `${key}=${params[key]}`)
    .join('&')
    // 插入 DOM 元素
  const script = document.createElement('script')
  script.setAttribute('src', `${url}?${paramString}`)
  document.body.appendChild(script)
}

JSONP({
  url: 'http://s.weibo.com/ajax/jsonp/suggestion',
  params: {
    key: 'test',
  },
  callbackKey: '_cb',
  callback(result) {
    console.log(result.data)
  }
})
```

会在命令行看到 `["TEST", "特殊泰帮承"]`，注意这里新浪微博的 API 只支持 HTTP，所以我们只能在 HTTP 页面上测试。

![upload successful](/images\jsonp.png)
## 同时进行多个请求

上面的流程有一个问题，就是在只有一个 JSONP 调用时它工作的很正常，但是当出现两个或者以上的请求，回调函数就会被覆盖，这样会出现混乱。为了解决这个问题，我们需要对所有的回调函数进行编码，并且在调用时告诉后端对应的独一无二的编号。

除此之外，污染全局空间显然是个不明智的选择，这个问题解决起来倒是非常简单，扔到 `JSONP.xxx` 下即可。

```js
function JSONP({
  url,
  params,
  callbackKey,
  callback
}) {
  // 唯一 id，不存在则初始化
  JSONP.callbackId = JSONP.callbackId || 1
  params = params || {}
    // 传递的 callback 名，和下面预留的一致
  params[callbackKey] = `JSONP.callbacks[${JSONP.callbackId}]`
    // 不要污染 window
  JSONP.callbacks = JSONP.callbacks || []
    // 按照 id 放置 callback
  JSONP.callbacks[JSONP.callbackId] = callback
  const paramKeys = Object.keys(params)
  const paramString = paramKeys
    .map(key => `${key}=${params[key]}`)
    .join('&')
  const script = document.createElement('script')
  script.setAttribute('src', `${url}?${paramString}`)
  document.body.appendChild(script)
    // id 占用，自增
  JSONP.callbackId++
}

JSONP({
  url: 'http://s.weibo.com/ajax/jsonp/suggestion',
  params: {
    key: 'test',
  },
  callbackKey: '_cb',
  callback(result) {
    console.log(result.data)
  }
})
JSONP({
  url: 'http://s.weibo.com/ajax/jsonp/suggestion',
  params: {
    key: 'excited',
  },
  callbackKey: '_cb',
  callback(result) {
    console.log(result.data)
  }
})
```

可以看到现在请求的都是 `http://s.weibo.com/ajax/jsonp/suggestion?key=test&_cb=JSONP.callbacks[1]` 这样的，然后得到的 js 也是 `JSONP.callbacks[1]()`，这样就不会有冲突的问题，也不污染全局域。


## URI编码

上面的代码仍然存在一个小问题，

```js
JSONP({
  url: 'http://s.weibo.com/ajax/jsonp/suggestion',
  params: {
    a: '545&b=3'
    b: '5',
  },
  callbackKey: '_cb',
  callback(result) {
    console.log(result.data)
  }
})
```

会导致 `a` 的内容直接被拼写进字符串，导致覆盖了 `b` 的值，而用户真的只是想让 `a` 的值为 `trdgd&b=2` 而已。解决方案也简单，进行 URI 编码即可，`encodeURIComponent('trdgd&b=2')` 的结果为 `trdgd%26b%3D2`。即将上面参数处理的部分改为

```js
const paramString = paramKeys
  .map(key => `${key}=${encodeURIComponent(params[key])}`)
  .join('&')
```

这里值得一提的是，由于最终的 URL 不能包含 ASCII 码以外的字符，所以其实当使用中文或者特殊字符时其实会被自动编码。而 +，空格，/，?，%，#，&，= 等字符在 URL 中则会出现歧义，只有手动编码后才能让服务器端正确解析。


