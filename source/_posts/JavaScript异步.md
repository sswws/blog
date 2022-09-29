---
title: 深入 JavaScript 异步编程
abbrlink: 42480
date: 2022-08-09 20:01:56
tags:

---

单线程 JavaScript 异步方案

首先我们需要了解，JavaScript 代码的运行是单线程，采用单线程模式工作的原因也很简单，最早就是在页面中实现 Dom 操作，如果采用多线程，就会造成复杂的线程同步问题，如果一个线程修改了某个元素，另一个线程又删除了这个元素，浏览器渲染就会出现问题；

单线程的含义就是： JS执行环境中负责执行代码的线程只有一个；就类似于只有一个人干活；一次只能做一个任务，有多个任务自然是要排队的；

优点：安全，简单

缺点：遇到任务量大的操作，会阻塞，后面的任务会长时间等待，出现假死的情况；

![截屏2022-08-10 16.38.57](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.38.57.png)

为了解决阻塞的问题，Javascript 将任务的执行模式分成了两种，同步模式（ Synchronous）和 异步模式( Asynchronous)

后面我们将分以下几个内容，来详细讲解 JavaScript 的同步与异步：

1、同步模式与异步模式

2、事件循环与消息队列

3、异步编程的几种方式

4、Promise 异步方案、宏任务/微任务队列

5、Async / Await语法糖

6、Generator 异步方案

7、手写 Promise 源码

## 同步模式

代码依次执行，后面的任务需要等待前面任务执行结束后，才会执行，同步并不是同时执行，而是排队执行；

先来看一段代码：

![截屏2022-08-10 16.44.17](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.44.17.png)

动画形式展现 同步代码 的执行过程：

![截屏2022-08-10 16.45.25](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/91619161%E6%88%AA%E5%B1%8F2022-08-10%2016.45.25.png)

代码会按照既定的语法规则，依次执行，如果中间遇到大量复杂任务，后面的代码则会阻塞等待；

再来看一段异步代码：

![截屏2022-08-10 16.45.47](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.45.47.png)

异步代码的执行，要相对复杂一些：

![截屏2022-08-10 16.46.03](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.46.03.png)

代码首先按照同步模式执行，当遇到异步代码时，会开启异步执行线程，在上面的代码中，setTimeout 会开启环境运行时的执行线程运行相关代码，代码运行结束后，会将结果放入到消息队列，等待 JS 线程结束后，消息队列的任务再依次执行；

流程图如下：

![截屏2022-08-10 16.46.17](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.46.17.png)

## 回调函数

通过上图，我们会看到，在整个代码的执行中，JS 本身的执行依然是单线程的，异步执行的最终结果，依然需要回到 JS 线程上进行处理，在JS中，异步的结果 回到 JS 主线程 的方式采用的是 “ 回调函数 ” 的形式 , 所谓的 回调函数 就是在 JS 主线程上声明一个函数，然后将函数作为参数传入异步调用线程，当异步执行结束后，调用这个函数，将结果以实参的形式传入函数的调用（也有可能不传参，但是函数调用一定会有），前面代码中 setTimeout 就是一个异步方法，传入的第一个参数就是 回调函数，这个函数的执行就是消息队列中的 “回调”；

下面我们自己封装一个 ajax 请求，来进一步说明回调函数与异步的关系

### Ajax 的异步请求封装

![截屏2022-08-10 16.46.34](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.46.34.png)

上面的代码，封装了一个 myAjax 的函数，用于发送异步的 ajax 请求，函数调用时，代码实际是按照同步模式执行的，当执行到 xhr.send() 时，就会开启异步的网络请求，向指定的 url 地址发送网络请求，从建立网络链接到断开网络连接的整个过程是异步线程在执行的；换个说法就是 myAjax 函数执行到 xhr.send() 后，函数的调用执行就已经结束了，如果 myAjax 函数调用的后面有代码，则会继续执行，不会等待 ajax 的请求结果；

但是，myAjax 函数调用结束后，ajax 的网络请求却依然在进行着，如果想要获取到 ajax 网络请求的结果，我们就需要在结果返回后，调用一个 JS 线程的函数，将结果以实参的形式传入：

![截屏2022-08-10 16.46.55](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.46.55.png)

### 回调地狱的诞生

回调函数让我们轻松处理异步的结果，但是，如果代码是异步执行的，而逻辑是同步的； 就会出现 “回调地狱”，举个栗子：

代码B需要等待代码A执行结束才能执行，而代码C又需要等待代码B，代码D又需要等待代码C，而代码 A、B、C都是异步执行的；

![截屏2022-08-10 16.47.13](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.47.13.png)

没错，代码执行是异步的，但是异步的结果，是需要有强前后顺序的，著名的"**回调地狱**"就是这么诞生的;

相对来说，代码逻辑是固定的，但是，这个编码体验，要差很多，尤其在后期维护的时候，层级嵌套太深，让人头皮发麻；

如何让我们的代码不在地狱中受苦呢？

有请 Promise 出山，拯救程序员的头发；

## Promise 异步处理

### Promise 概念及应用

![截屏2022-08-10 16.47.30](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.47.30.png)

Promise 译为 承诺、许诺、希望，意思就是异步任务交给我来做，一定(承诺、许诺)给你个结果；在执行的过程中，Promise 的状态会修改为 pending ，一旦有了结果，就会再次更改状态，异步执行成功的状态是 Fulfilled , 这就是承诺给你的结果，状态修改后，会调用成功的回调函数 onFulfilled 来将异步结果返回；异步执行成功的状态是 Rejected， 这就是承诺给你的结果，然后调用 onRejected 说明失败的原因(异常接管)；

将前面对 ajax 函数的封装，改为 Promise  的方式；

### 重构 Ajax 的异步请求封装

![截屏2022-08-10 16.47.47](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.47.47.png)

使用 Promise

![截屏2022-08-10 16.48.04](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.48.04.png)

还是前面提到的逻辑，如果返回的结果中，又有 ajax 请求需要发送，可一定记得使用链式调用，不要在then中直接发起下一次请求，否则，又是地狱见了：

如果返回的结果中，又有 ajax 请求需要发送：

![截屏2022-08-10 16.48.26](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.48.26.png)

链式的意思就是在上一次 then 中，返回下一次调用的 Promise 对象，我们的代码，就不会进地狱了；

![截屏2022-08-10 16.48.42](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.48.42.png)

虽然我们脱离了回调地狱，但是 .then 的链式调用依然不太友好，频繁的 .then 并不符合自然的运行逻辑，Promise 的写法只是回调函数的改进，使用then方法以后，异步任务的两段执行看得更清楚了，除此以外，并无新意。Promise 的最大问题是代码冗余，原来的任务被 Promise 包装了一下，不管什么操作，一眼看去都是一堆 then，原来的语义变得很不清楚。于是，在 Promise 的基础上，Async 函数来了；

终极异步解决方案，千呼万唤的在 ES2017中发布了；

### Async/Await 语法糖

Async 函数使用起来，也是很简单，将调用异步的逻辑全部写进一个函数中，函数前面使用 async  关键字，在函数中异步调用逻辑的前面使用 await ，异步调用会在 await 的地方等待结果，然后进入下一行代码的执行，这就保证了，代码的后续逻辑，可以等待异步的 ajax 调用结果了，而代码看起来的执行逻辑，和同步代码几乎一样；

![截屏2022-08-10 16.48.59](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.48.59.png)

注意：await 关键词 只能在 async 函数内部使用

Promise 的写法只是回调函数的改进，使用`then`方法以后，异步任务的两段执行看得更清楚了，除此以外，并无新意。

Promise 的最大问题是代码冗余，原来的任务被 Promise 包装了一下，不管什么操作，一眼看去都是一堆`then`，原来的语义变得很不清楚。

使用`async` / `await` 关键字就可以在异步代码中使用普通的 `try` / `catch`代码块接管错误了

![截屏2022-08-10 16.49.19](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.49.19.png)

因为使用简单，很多人也不会探究其使用的原理，无非就是两个 单词，加到前面，用就好了，虽然会用，日常开发看起来也没什么问题，但是一遇到 Bug 调试，就凉凉，面试的时候也总是知其然不知其所以然，咱们先来一个面试题试试，你看你能运行出正确的结果吗？

### async 面试题

请写出以下代码的运行结果：

![截屏2022-08-10 16.49.35](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.49.35.png)

![截屏2022-08-10 16.49.52](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.49.52.png)

想要把结果搞清楚，我们需要引入另一个内容：Generator 生成器函数；

## Generator 函数

https://developer.mozilla.org/zh-CN/docs/Web/JavaScript/Reference/Global_Objects/Generator

Generator 函数是 ES6 提供的一种异步编程解决方案，语法行为与传统函数完全不同；

Generator 函数有多种理解角度。语法上，首先可以把它理解成，Generator 函数是一个状态机，封装了多个内部状态。

执行 Generator 函数会返回一个遍历器对象，也就是说，Generator 函数除了状态机，还是一个遍历器对象生成函数。返回的遍历器对象，可以依次遍历 Generator 函数内部的每一个状态。

形式上，Generator 函数是一个普通函数，但是有两个特征。一是，function 关键字与函数名之间有一个星号；二是，函数体内部使用 yield 表达式，定义不同的内部状态（yield 在英语里的意思就是“产出”）。

### Generator 基础用法

![截屏2022-08-10 16.50.11](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.50.11.png)

![截屏2022-08-10 16.50.31](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.50.31.png)

![截屏2022-08-10 16.50.52](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.50.52.png)

Generator 生成器函数，返回 遍历器对象，先看一段代码：

![截屏2022-08-10 16.51.08](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/91619161%E6%88%AA%E5%B1%8F2022-08-10%2016.51.08.png)

你会发现，在函数声明的地方，函数名前面多了 * 星号，函数体中的代码有个 yield ，用于函数执行的暂停；简单点说就是，这个函数不是个普通函数，调用后不会立即执行全部代码，而是在执行到 yield 的地方暂停函数的执行，并给调用者返回一个遍历器对象，yield 后面的数据，就是遍历器对象的 value 属性值，如果要继续执行后面的代码，需要使用 遍历器对象中的 next() 方法，代码会从上一次暂停的地方继续往下执行；

是不是so easy 啊；

同时，在调用next 的时候，还可以传递参数，函数中上一次停止的 yeild 就会接受到当前传入的参数；

![截屏2022-08-10 16.51.29](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.51.29.png)

Generator 的最大特点就是让函数的运行，可以暂停，不要小看他，有了这个暂停，我们能做的事情就太多，在调用异步代码时，就可以先 yield 停一下，停下来我们就可以等待异步的结果了；那么如何把 Generator 写到异步中呢？

**Generator  异步方案**

将调用ajax的代码写到 生成器函数的 yield 后面，每次的异步执行，都要在 yield 中暂停，调用的返回结果是一个 Promise 对象，我们可以从 迭代器对象的 value 属性获取到Promise 对象，然后使用 .then 进行链式调用处理异步结果，结果处理的代码叫做 执行器，就是具体负责运行逻辑的代码；![截屏2022-08-10 16.51.43](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.51.43.png)



而如果有多个请求，那么我们的代码就会陷入嵌套中

![截屏2022-08-10 16.52.00](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.52.00.png)

而执行器的逻辑中，是相同嵌套的，因此可以写成递归的方式对执行器进行改造：

![截屏2022-08-10 16.52.16](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.52.16.png)

然后，再将执行的逻辑，进行封装复用，形成独立的函数模块；

![截屏2022-08-10 16.52.32](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.52.32.png)

封装完成后，我们再使用时，只需要关注 Generator 中的 yield 部分就行了

此时你会发现，使用 Generator 封装后，异步的调用就变的非常简单了，但是，这个封装还是有点麻烦，有大神帮我们做了这个封装，相当强大：https://github.com/tj/co ，感兴趣看一研究一下，而随着 JS 语言的发展，更多的人希望类似 co 模块的封装，能够写进语言标准中，我们直接使用这个语法规则就行了；

其实你也可以对比一下，使用 co 模块后的 Generator 和 async 这两段代码：

![截屏2022-08-10 16.52.47](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.52.47.png)

你应该也发现了，async 函数就是 Generator 语法糖，不需要自己再去实现 co 执行器函数或者安装 co 模块，写法上将 * 星号 去掉换成放在函数前面的 async  ，把函数体的 yield 去掉，换成 await； 完美……

![截屏2022-08-10 16.53.01](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.53.01.png)

我们再来看一下 Generator ，相信下面的代码，你能很轻松的阅读；



![截屏2022-08-10 16.53.14](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.53.14.png)

![截屏2022-08-10 16.53.31](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.53.31.png)

### async 的面试题

带着 Generator  的思路，我们再回头看看那个 async 的面试题；

请写出以下代码的运行结果：

![截屏2022-08-10 16.53.47](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.53.47.png)

运行结果：

![截屏2022-08-10 16.54.09](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2016.54.09.png)

是不是恍然大明白呢……

**手写 Promise 源码**

**Promise 源码封装**

首先确保对Promise应用很熟悉，根据应用方式，实现核心逻辑代码

Promise 就是一个类在执行这个类的时候需要传递一个执行器进去,执行器会立即执行

index.js：

![截屏2022-08-10 17.32.33](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.32.33.png)

Promise 中有三种状态分别为 成功 fulfilled 失败 rejected 等待 pending

- pending -> fulfilled
- pending -> rejected

一旦确定就不可修改；

根据使用方式，我们可以完成 MyPromise.js 的基础代码

![截屏2022-08-10 17.32.53](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.32.53.png)

但是，我们知道状态一旦确定就不可修改，但是在前面调用时，连续执行了resolve, reject，因此我们需要添加阻止状态修改的代码

![截屏2022-08-10 17.33.10](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.33.10.png)

完成后，我们就可以使用 .then 语法了，then 接受两个函数参数，第一个表示成功，第二个表示失败；也就是说，then方法的执行，首先要判断成功还是失败的状态，然后才能针对不同状态调用函数；

index.js

![截屏2022-08-10 17.33.24](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.33.24.png)

在 myPromise.js 中添加 then 方法

![截屏2022-08-10 17.33.40](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.33.40.png)

then中传入的回调函数方法，也是有参数的，分别是成功的返回值，及失败的日志信息；那么成功的数据和失败的日志在哪里呢？

业务代码都是放在 new promise 的回调函数中的，所以，成功和失败的信息，都是在 resolve 和 reject  中产生的，产生数据之后，使用属性的方式记录下来，然后在 then 方法中，获取使用

修改 myPromise.js

![截屏2022-08-10 17.33.58](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.33.58.png)

![截屏2022-08-10 17.34.17](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.34.17.png)

最后，我们导出，并引入后应用；

myPromise.js

![截屏2022-08-10 17.34.34](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.34.34.png)

![截屏2022-08-10 17.34.48](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.34.48.png)

![截屏2022-08-10 17.35.03](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.35.03.png)

### Promise 异步实现

在 New  promise 中，使用异步代码：

![截屏2022-08-10 17.35.18](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.35.18.png)

then 方法是立即执行的，而在then方法中，判断了成功和失败的状态，并没有判断等待的状态，也就是没有判断异步执行的情况；

myPromise.js

![截屏2022-08-10 17.35.54](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.35.54.png)

目前已经在等待中，存储了不同状态的回调，那么什么时候调用不同状态的回调函数呢？

当然是在修改状态中了，

![截屏2022-08-10 17.36.12](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.36.12.png)

封装一个读取文件的 Promise ,

index.js

![截屏2022-08-10 17.36.31](https://xingqiu-tuchuang-1256524210.cos.ap-shanghai.myqcloud.com/9161%E6%88%AA%E5%B1%8F2022-08-10%2017.36.31.png)