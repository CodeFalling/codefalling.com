title: 利用 generator 解决回调地狱
author: codefalling
tags:
  - javascript
  - promise
  - async
  - generator
categories:
  - 编程
date: 2016-08-04 00:00:00
---
在 JS 中，常常出现多个异步操作需要逐步完成的情况，例如相继获取多个 API 的结果，最简单的方法是通过嵌套的回调实现：

<!-- more -->

```js
function sleep(cb){
  setTimeout(cb, 1000)
}

function getAuthor(cb){
  sleep(() => {
    cb('codefalling')
  })
}

function getBlog(cb, author){
  sleep(() => {
    if (author === 'codefalling') {
      cb('https://codefalling.com')
    }
  })
}

getAuthor(author => {
  console.log(`Author: ${author}`)
  getBlog(blog => {
    console.log(`Blog: ${blog}`)
  }, author)
})
```

这种方法看起来不够自然，而且当异步的操作越来越多时，回调的嵌套会越来越深，代码的执行顺序会非常不直观，不利于维护。


## generator

generator(生成器) 也是 ES6 的新特性，它并不是为了解决回调地狱而生，但是却可以用于解决回调地狱。写过 Python 的同学应该熟悉这个特性，generator 的本质是一个迭代器。

```js
function* gen(){
  for (let i = 0; i < 3; i++) {
    yield i;
  }
}

let f = gen()

console.log(f.next().value)
console.log(f.next().value)
console.log(f.next().value)
```

这种特性允许用 `function` 声明一个生成器，生成器执行后返回一个迭代器(`f`)，和普通的函数不同，生成器内可以使用 `yield` 关键字而不是 `return`。当我们第一次执行 `f.next()` 时，函数一直执行到 `yield` 关键字，暂停整个函数的执行，并且返回 `yield` 语句的值。当第二次调用 `f.next()` 时则从上次暂停的地方继续执行，直到下次碰到 `yield` 或者执行完毕。

值得一提的是，`yield i` 作为一个表达式，也是有返回值的，`f.next()` 的参数就是其执行时的返回值。

到目前看，yield 看起来似乎都和回调没什么关系，其设计的目的也是为了更加优雅的实现迭代器而非解决回调地狱。但是由于其奇特的控制流，可以实现一些 amazing 的特性。


## yield + callback

```js
function* tasker(cb){
  const author = yield getAuthor(cb)
  console.log(`Author: ${author}`)
  const blog = yield getBlog(cb, author)
  console.log(`Blog: ${blog}`)
}

const runner = tasker(resume)
function resume(res){
    runner.next(res)
}
runner.next()
```

输出的效果和上面相同，乍一看让人很惊讶，`tasker` 里本身是异步执行的代码写起来顺序却和同步执行的代码一样。这正是依赖 `generator` 奇异的控制流实现的。当我们第一次执行到 `runner.next()` 时，执行到 `yield getAuthor(resume)` 暂停了（对 `author` 的赋值还没有执行），`getAuthor(resume)` 被执行，`resume` 作为回调函数得到结果并且被调用，执行了 `runner.next(res)`，导致 `generator` 继续执行下去，`author` 被赋值并且输出，然后暂停在下一个 `yield`。

这段代码看起来是顺序执行的，但控制流其实跳跃了多次，画一张图来表示一下：

![generator](\images\generator.png)
看到，整个控制流是按照我们预期的思路进行，每次异步操作返回时再去进行接下来的操作。而事实上我们的逻辑代码（`tasker`) 看起来就像同步的一样，我们只需要关心这一块的逻辑顺序而不需要关心异步调用什么时候返回，`resume` 则会保证整个流程向前推动。

我们可以将推动流程的代码写的更加通用：

```js
function magic(gen){
  const runner = gen(resume);
  function resume(){
    runner.next(...arguments)
  }
  runner.next();
}

magic(function*(cb){
  const author = yield getAuthor(cb)
  console.log(`Author: ${author}`)
  const blog = yield getBlog(cb, author)
  console.log(`Blog: ${blog}`)
})
```

我们就可以在 `magic` 内用同步的思路写异步的逻辑。


## thunk

上面的代码仍然不够自动化，我们还是需要手动传递 `cb` 这个和逻辑并没有什么关系的回调函数。而 `yield` 后面的表达式是求值后再传递到 `next` 外的，我们并没有什么办法进行干预。唯一的办法就是通过 thunk 化来延迟求值。

所谓 thunk 化可以举个非常简单的例子，例如

```js
function getBlog(cb, author){
  sleep(() => {
    if (author === 'codefalling') {
      cb('https://codefalling.com')
    }
  })
}
```

可以通过

```js
function getBlogThunk(author) {
    return function(cb){
        return getBlog(cb, author);
    }
}
```

来 thunk 化，这样我们就可以通过 `getBlogThunk(author)(cb)` 来实现原来的调用，而当我们想延迟求值时，则可以只返回 `getBlogThunk(author)`。

所以我们的流程能够进一步自动化：

```js
function getBlogThunk(author) {
    return function(cb){
        return getBlog(cb, author);
    }
}

function getAuthorThunk(author) {
    return function(cb){
        return getAuthor(cb);
    }
}

function magic(gen){
  const runner = gen(resume);
  function resume(){
    const thunkcall = runner.next(...arguments)
    if(!thunkcall.done){
        thunkcall.value(resume)
    }
  }
  resume()
}

magic(function*(cb){
  const author = yield getAuthorThunk()
  console.log(`Author: ${author}`)
  const blog = yield getBlogThunk(author)
  console.log(`Blog: ${blog}`)
})
```

异步调用被移交到了 `resume` 中进行，这样就不需要逻辑代码去关系回调的问题。

![thunk](\images\generator_thunk.png)
## Promise

上面有提到，generator 并不是 ES6 针对回调地狱给出的解决方案，现阶段已经在大量使用的方案是 Promise。如果用 Prmoise 改写上面的代码：

```js
function sleep(cb){
  setTimeout(cb, 1000)
}

function getAuthor(cb){
  return new Promise(reslove => {
    sleep(() => {
        reslove('codefalling')
    })
  })
}

function getBlog(author){
  return new Promise(reslove => {
    sleep(() => {
        if (author === 'codefalling') {
        reslove('https://codefalling.com')
        }
    })
  })
}

getAuthor()
.then(author => {
  console.log(author)
  return getBlog(author)
}).then(blog => {
  console.log(blog)
})
```

可以看到这样也避免了层数越来越深的问题，但是仍然躲不开要写回调，我们可以把 generator 和 Promise 结合起来，这样就避免了 thunk 的繁琐，且与现在的标准相统一。

```js
function sleep(cb){
  setTimeout(cb, 1000)
}

function getAuthor(cb){
  return new Promise(reslove => {
    sleep(() => {
        reslove('codefalling')
    })
  })
}

function getBlog(author){
  return new Promise(reslove => {
    sleep(() => {
        if (author === 'codefalling') {
        reslove('https://codefalling.com')
        }
    })
  })
}

function magic(gen){
  const runner = gen(resume);
  function resume(){
    const pro = runner.next(...arguments)
    if(!pro.done){
        pro.value.then(resume)
    }
  }
  resume()
}

magic(function*(){
  const author = yield getAuthor()
  console.log(author)
  const blog = yield getBlog(author)
  console.log(blog)
})
```


## 结尾

其实本文讲述的就是大名鼎鼎的 TJ 大神开发的 [co](https://github.com/tj/co) 的基本原理，颇有点滥用 generator 这个特性的意思，但是现在 ES7 中的 `async/await` 其实就是相当于 genetaor + Promise 的组合，堪称 JS 异步的终极解决方案。

另外，推荐一个 Chrome 插件：[Scratch JS](https://chrome.google.com/webstore/detail/scratch-js/alploljligeomonipppgaahpkenfnfkn)，可以直接在 Chrome 的 Dev Tools 里获得一个多行的 ES6 编辑器并且随时运行，本文的代码是直接在这个插件里写好并且测试的。


