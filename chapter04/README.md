# 图解 Promise 实现原理（四）—— Promise 静态方法实现

## 摘要

很多同学在学习 Promise 时，知其然却不知其所以然，对其中的用法理解不了。**本系列文章由浅入深逐步实现 Promise，并结合流程图、实例以及动画进行演示，达到深刻理解 Promise 用法的目的。**

1. [图解 Promise 实现原理（一）—— 基础实现](../chapter01/README.md)
2. [图解 Promise 实现原理（二）—— Promise 链式调用](../chapter02/README.md)
3. [图解 Promise 实现原理（三）—— Promise 原型方法实现](../chapter04/README.md)
4. [图解 Promise 实现原理（四）—— Promise 静态方法实现](./README.md)

## 前言

上一节中，实现了 Promise 的原型方法。包括增加异常状态，catch以及 finally。截至目前，Promise 的实现如下：

```js
class Promise {
  callbacks = [];
  state = 'pending';//增加状态
  value = null;//保存结果
  constructor(fn) {
    fn(this._resolve.bind(this), this._reject.bind(this));
  }
  then(onFulfilled, onRejected) {
    return new Promise((resolve, reject) => {
      this._handle({
        onFulfilled: onFulfilled || null,
        onRejected: onRejected || null,
        resolve: resolve,
        reject: reject
      });
    });
  }
  catch(onError) {
    return this.then(null, onError);
  }
  finally(onDone) {
    if (typeof onDone !== 'function') return this.then();

    let Promise = this.constructor;
    return this.then(
      value => Promise.resolve(onDone()).then(() => value),
      reason => Promise.resolve(onDone()).then(() => { throw reason })
    );
  }
  _handle(callback) {
    if (this.state === 'pending') {
      this.callbacks.push(callback);
      return;
    }

    let cb = this.state === 'fulfilled' ? callback.onFulfilled : callback.onRejected;

    if (!cb) {//如果then中没有传递任何东西
      cb = this.state === 'fulfilled' ? callback.resolve : callback.reject;
      cb(this.value);
      return;
    }

    let ret;

    try {
      ret = cb(this.value);
      cb = this.state === 'fulfilled' ? callback.resolve : callback.reject;
    } catch (error) {
      ret = error;
      cb = callback.reject
    } finally {
      cb(ret);
    }

  }
  _resolve(value) {

    if (value && (typeof value === 'object' || typeof value === 'function')) {
      var then = value.then;
      if (typeof then === 'function') {
        then.call(value, this._resolve.bind(this), this._reject.bind(this));
        return;
      }
    }

    this.state = 'fulfilled';//改变状态
    this.value = value;//保存结果
    this.callbacks.forEach(callback => this._handle(callback));
  }
  _reject(error) {
    this.state = 'rejected';
    this.value = error;
    this.callbacks.forEach(callback => this._handle(callback));
  }
}
```

接下来再介绍一下 Promise 中静态方法的实现，譬如 Promise.resolve、Promise.reject。其它静态方法的实现也是类似的。

## 静态方法

除了前文中提到的 Promise 实例的原型方法外，Promise 还提供了 Promise.resolve 和 Promise.reject 方法。用于将非 Promise 实例包装为 Promise 实例。例如：

```js
Promise.resolve('foo')
// 等价于
new Promise(resolve => resolve('foo'))
```
Promise.resolve 的参数不同对应的处理也不同，如果 Promise.resolve 的参数是一个 Promise 的实例，那么 Promise.resolve 将不做任何改动，直接返回这个 Promise 实例，如果是一个基本数据类型，譬如上例中的字符串，Promise.resolve 就会新建一个 Promise 实例返回。这样当我们不清楚拿到的对象到底是不是 Promise 实例时，为了保证统一的行为，Promise.resolve 就变得很有用了。看一个例子：

```js
const Id2NameMap = {};
const getNameById = function (id) {

  if (Id2NameMap[id]) return Id2NameMap[id];

  return new Promise(resolve => {
    mockGetNameById(id, function (name) {
      Id2NameMap[id] = name;
      resolve(name);
    })
  });
}
getNameById(id).then(name => {
  console.log(name);
});
```

上面的场景我们会经常碰到，为了减少请求，经常会缓存数据，我们获取到 id 对应的名字后，存到 Id2NameMap 对象里，下次再通过 id 去请求 id 对应的 name 时先看 Id2NameMap里有没有，如果有就直接返回对应的 name，如果没有就发起异步请求，获取到后放到 Id2NameMap 中去。

其实上面的代码是有问题的，如果命中 Id2NameMap 里的值，getNameById 返回的结果就是 name，而不是 Promise 实例。此时 getNameById(id).then 会报错。在我们不清楚返回的是否是 Promise 实例的情况下，就可以使用 Promise.resolve 进行包装：

```js
Promise.resolve(getNameById(id)).then(name => {
  console.log(name);
});
```
这样一来，不管 getNameById(id) 返回的是什么，逻辑都没有问题。看下面的Demo:

[demo-Promise.resolve 的源码](https://repl.it/@morrain2016/demo-Promiseresolve)

在实现 Promise.resolve 之前，我们先看下它的参数分为哪些情况：

**1.参数是一个 Promise 实例**

如果参数是 Promise 实例，那么 Promise.resolve 将不做任何修改、原封不动地返回这个实例。

**2.参数是一个 thenable 对象**

thenable 对象指的是具有 then 方法的对象，比如下面这个对象。

```js
let thenable = {
  then: function(onFulfilled) {
    onFulfilled(42);
  }
};
```
Promise.resolve 方法会将这个对象转为 Promise 对象，然后就立即执行 thenable 对象的 then 方法。

```js
let thenable = {
  then: function(onFulfilled) {
    onFulfilled(42);
  }
};

let p1 = Promise.resolve(thenable);
p1.then(function(value) {
  console.log(value);  // 42
});
```

上面代码中，thenable对象的then方法执行后，对象p1的状态就变为resolved，从而立即执行最后那个then方法指定的回调函数，输出 42。

**3.参数不是具有 then 方法的对象，或根本就不是对象**

如果参数是一个原始值，或者是一个不具有then方法的对象，则 Promise.resolve 方法返回一个新的 Promise 对象，状态为 resolved。

**4.不带任何参数**

Promise.resolve 方法允许调用时不带参数，直接返回一个 resolved 状态的 Promise 对象。

```js
  static resolve(value) {
    if (value && value instanceof Promise) {
      return value;
    } else if (value && typeof value === 'object' && typeof value.then === 'function') {
      let then = value.then;
      return new Promise(resolve => {
        then(resolve);
      });

    } else if (value) {
      return new Promise(resolve => resolve(value));
    } else {
      return new Promise(resolve => resolve());
    }
  }
```

[Promise 的实现源码](https://repl.it/@morrain2016/Promise)

**Promise.reject 与 Promise.resolve 类似，区别在于 Promise.reject 始终返回一个状态的 rejected 的 Promise 实例，而 Promise.resolve 的参数如果是一个 Promise 实例的话，返回的是参数对应的 Promise 实例，所以状态不一定。**

## 总结

刚开始看 Promise 源码的时候总不能很好的理解 then 和 resolve 函数的运行机理，但是如果你静下心来，反过来根据执行 Promise 时的逻辑来推演，就不难理解了。这里一定要注意的点是：Promise 里面的 then 函数仅仅是注册了后续需要执行的代码，真正的执行是在 resolve 方法里面执行的，理清了这层，再来分析源码会省力的多。

现在回顾下 Promise 的实现过程，其主要使用了设计模式中的观察者模式：

1. 通过 Promise.prototype.then 和 Promise.prototype.catch 方法将观察者方法注册到被观察者 Promise 对象中，同时返回一个新的 Promise 对象，以便可以链式调用。
2. 被观察者管理内部 pending、fulfilled 和 rejected 的状态转变，同时通过构造函数中传递的 resolve 和 reject 方法以主动触发状态转变和通知观察者。


## 参考资料

[【翻译】Promises/A+规范](https://www.ituring.com.cn/article/66566)

[Promise 实现详解](https://zhuanlan.zhihu.com/p/25178630)

[30分钟，让你彻底明白Promise原理](https://mengera88.github.io/2017/05/18/Promise%E5%8E%9F%E7%90%86%E8%A7%A3%E6%9E%90/)