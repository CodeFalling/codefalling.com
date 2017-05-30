title: 执行 innerHTML 里的 script
author: codefalling
tags:
  - javascript
  - script
categories:
  - 编程
date: 2017-03-17 00:00:00
---
## 背景
有时候我们会有把一整段 HTML 动态塞进页面的需求，例如渲染了一个模板，从服务器端获取了一段广告代码等。一般情况下我们使用 `container.innerHTML` 即可。但是当 HTML 中出现 `script` 标签时，直接使用 `innerHTML` 并不会执行它。

### 一个例子
```html
<div id="test">Hello HTML</div>
<script>
	document.getElementById('test').innerHTML = 'Hello JS';
</script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.4.2/react.min.js"></script>
<script src="https://cdnjs.cloudflare.com/ajax/libs/react/15.4.2/react-dom.min.js"></script>
<script>
	ReactDOM.render(React.createElement('div', null, 'Hello React'), document.getElementById('test'));
</script>
```

一个常见的例子里包含普通的 HTML 内容，`<script>` 里的 inline script，通过 `src` 引用的外部 script。如果我们尝试直接用 `innerHTML` 赋值只会得到一个 `Hello HTML`。而后面的 `<script>` 标签无一例外没有执行。

## appendChild

我们知道通过 `appendChild` 把 `<script>` 标签直接塞进页面是可以执行和加载里面的 js 的（`JSONP` 就是通过这种方法实现的，参见之前的文章：[JSONP 的实现 - 知乎专栏](https://zhuanlan.zhihu.com/p/22338759)。

所以其实我们需要做的就只是把所有的 `<script>` 找出来，然后通过 `appendChild` 塞到页面里即可。

```js
function runScript(script){
  // 直接 document.head.appendChild(script) 是不会生效的，需要重新创建一个
  const newScript = document.createElement('script');
  // 获取 inline script
  newScript.innerHTML = script.innerHTML;
  // 存在 src 属性的话
  const src = script.getAttribute('src');
  if (src) newScript.setAttribute('src', src);

  document.head.appendChild(newScript);
  document.head.removeChild(newScript);
}

function setHTMLWithScript(container, rawHTML){
  container.innerHTML = rawHTML;
  const scripts = container.querySelectorAll('script');
  for (let script of scripts) {
    runScript(script);
  }
}
```

## 执行顺序
当我们尝试用上面的 `setHTMLWithScript(document.body, html)` 时有一个问题，就是 `script` 的加载和执行并非同步的，我们会得到一个 `Hello, JS`。

而下面的 `<script>` 依赖前面的 `<script>` 执行加载完成是一个非常常见的需求，因为在正常的静态网页里就是这样的，虽然所有的远程脚本都是异步加载的，但后面的 `<script>` 会等待前面的加载执行后才开始执行。

为了让异常处理和异步流程的控制更方便，我们让 `runScript` 返回一个 Promise，然后只需要一个简单的 `reduce` 就可以把异步逻辑串联起来：

```
function runScript(script){
  return new Promise((reslove, rejected) => {
    // 直接 document.head.appendChild(script) 是不会生效的，需要重新创建一个
    const newScript = document.createElement('script');
	// 获取 inline script
    newScript.innerHTML = script.innerHTML;
    // 存在 src 属性的话
    const src = script.getAttribute('src');
    if (src) newScript.setAttribute('src', src);
    
    // script 加载完成和错误处理
    newScript.onload = () => reslove();
    newScript.onerror = err => rejected();
    document.head.appendChild(newScript);
    document.head.removeChild(newScript);
    if (!src) {
	    // 如果是 inline script 执行是同步的
	    reslove();
    }
  })
}

function setHTMLWithScript(container, rawHTML){
  container.innerHTML = rawHTML;
  const scripts = container.querySelectorAll('script');

  return Array.prototype.slice.apply(scripts).reduce((chain, script) => {
    return chain.then(() => runScript(script));
  }, Promise.resolve());
}
```

得到预期的 `Hello React`。

*其实这里有一点和直接渲染不一致的地方，就是脚本的加载也是同步的，后面的脚本会等待之前的脚本执行完才会加载，不过从 js 层面似乎没有办法解决这个问题。*


## JQuery.html

熟悉 JQuery 的同学可能知道 `$.html` 其实会直接执行里面的 `<script>` 标签，不过是同步的，在 `$.html` 的代码中，可以看到 jQuery 判断满足一定条件下直接使用 `innerHTML`，随便执行一个 `$('body').html(test<script></script>)` 然后打个断点，

![jquery-source](/images\pasted-1.png)

![jquery-source](/images\pasted-2.png)


可以看到这里做了一个简单的正则判断，如果碰到 `<script><style><link>` 标签就用 jQuery 自己实现的 `append`，继续追踪下去，

![upload successful](/images\pasted-0.png)
显然 jQuery 在这里完全没有考虑 `<script>` 前后的依赖。对于 inline script 的标签也是直接通过 `eval` 实现的而不是新建一个插入到文档里。

JQuery 也有几个 issue 讨论是否要按照顺序执行，但最后决定保持现状：[Scripts in inner html are not exectuted sequentially in order · Issue #2538 · jquery/jquery](https://github.com/jquery/jquery/issues/2538)。



## 其他
## createContextualFragment

除了写进去再用 `querySelectorAll` 把 script 全都拿出来复制一遍外，IE11 以上的浏览器也可以通过 `createContextualFragment` 直接把 html 转换成 DOM 节点然后 append 到页面上：

```js
var tagString = "<div>I am a div node</div><script>console.log('test')</script>";
var range = document.createRange();
// make the parent of the first div in the document becomes the context node
range.selectNode(document.body);
var documentFragment = range.createContextualFragment(tagString);
undefined
document.body.appendChild(documentFragment)
```

也可以用这种方法来实现上面的功能。
### 兼容性

上面的代码都只是顺手的探索，没有考虑兼容性方面的问题，例如 IE 不支持 script 的 onload 事件等，可能需要 `onreadystatechange` 来实现。


### DOMContentLoaded
`DOMContentLoaded` 早已经完成，如果有需要，我们可能要在脚本加载完成后，重新触发一下

```js
setHTMLWithScript(document.body, rawHTML)
.then(() => {
  var DOMContentLoadedEvent = document.createEvent('Event');
  DOMContentLoadedEvent.initEvent('DOMContentLoaded', true, true);
  document.dispatchEvent(DOMContentLoadedEvent);
})
```

### document.write
在静态页面中，`<script>` 标签里如果出现 `document.write`，会直接在 `<script>` 插入的位置写入，这种方法常被用于广告投放脚本来定位自己的位置。

而当我们在动态插入时文档已经关闭，会直接 `write` 到整个页面上，如果有必要可以暂时替换 `document.write` 来实现。

### getCurrentScript
`getCurrentScript` 是另一个定位 `<script>` 标签所在位置的方法，之所以不太常用是因为 IE 不兼容它，如果我们要考虑兼容这个方法新产生的 `<script>` 标签就不应该往 `<head>` 里 append，而是插入到原来所在的位置。

## 总结
以上方法都只是模拟静态 `<script>` 解析的过程，一般来说我们不要求行为完全一致（毕竟跨域异步加载同步执行这点 JS 就无法模拟），但是可以按照我们的需求去实现它的行为。

这种方法也只适用于一部分场景，如果有更复杂的 JS 动态加载需求应该考虑使用 `requirejs` 等 AMD Loader。
## 拓展阅读

- [javascript - load and execute order of scripts - Stack Overflow](http://stackoverflow.com/questions/8996852/load-and-execute-order-of-scripts)
- [Scripts in inner html are not exectuted sequentially in order · Issue #2538 · jquery/jquery](https://github.com/jquery/jquery/issues/2538)
- [javascript - Load and execution sequence of a web page? - Stack Overflow](http://stackoverflow.com/questions/1795438/load-and-execution-sequence-of-a-web-page)

