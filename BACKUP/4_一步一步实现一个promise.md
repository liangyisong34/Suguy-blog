# [一步一步实现一个promise](https://github.com/liangyisong34/Suguy-blog/issues/4)

**因为自己之前一直对promise没有搞太明白，尤其是then的链式调用那里更是云里雾里，因此决定手写一遍彻底搞懂promise**

- _首先来看下一个promise的简单功能以及使用方式_
```
let promise = new Promise((resolve, reject) => {
        resolve('success')
        reject('error')
    })

promise.then(value => {
    console.log(value)
}, reason => {
    console.log(reason)
})
```

> promise首先是一个构造函数，传入一个函数，这个函数的入参是内部设置好的resolve和reject方法。promise的原型对象上有一个then方法，它的参数是成功和失败的回调函数，会接收来自resolve和reject的参数。

- _根据上面的结论，来写一个promise的大概结构_
```

function myPromise(fn) {
        function resolve(){}
        function reject(){}
        fn(resolve,reject)
    }

 myPromise.prototype.then=function(onFulfilled,onRejected){}
```

> 下面来填充这个骨架，实现基本功能

- _实现基本功能_

```
function myPromise(fn) {
        let self = this
        self.value=null
        self.error=null
        self.onFulfilled=null
        self.onRejected=null
        function resolve(value){
            self.value=value
            self.onFulfilled(self.value)
        }
        function reject(error){
            self.error=error
            self.onRejected(self.error)
        }
        fn(resolve,reject)
    }

    myPromise.prototype.then=function(onFulfilled,onRejected){
        this.onFulfilled=onFulfilled
        this.onRejected=onRejected
    }
```

> 下面来做个测试，可以发现异步任务是没问题的，但是同步问题会报错，因为在初始化时调用resolve()，这个时候onFulfilled还没有赋值
```
//异步任务 通过
let promise = new myPromise((resolve,reject)=>{
      setTimeout(function(){
          resolve('success')
      },0)
  })
  promise.then(reason=>{
      console.log(reason)
  })

  //同步任务 报错
  let promise = new myPromise((resolve,reject)=>{
      resolve('success')
  })
  promise.then(reason=>{
      console.log(reason)
  })
```

- _支持同步任务_
```
function myPromise(fn) {
        let self = this
        self.value = null
        self.error = null
        self.onFulfilled = null
        self.onRejected = null

        function resolve(value) {
            setTimeout(function () {
                self.value = value
                self.onFulfilled(self.value)
            }, 0)
        }

        function reject(error) {
            setTimeout(function () {
                self.error = error
                self.onRejected(self.error)
            }, 0)
        }
        fn(resolve, reject)
    }

    myPromise.prototype.then = function (onFulfilled, onRejected) {
        this.onFulfilled = onFulfilled
        this.onRejected = onRejected
    }
```

> 思路很简单，增加一个setTimeout来延迟执行

- _接下来给promise添加状态_
```
const PENDING = "pending"
const FULFILLED = "fulfilled"
const REJECTED = "rejected"

function myPromise(fn) {
  let self = this
  self.status = PENDING
  self.value = null
  self.error = null
  self.onFulfilled = null
  self.onRejected = null

  function resolve(value) {
      if (self.status === PENDING) {
          setTimeout(function () {
              self.status = FULFILLED
              self.value = value
              self.onFulfilled(self.value)
          }, 0)
      }
  }

  function reject(error) {
      if (self.status === PENDING) {
          setTimeout(function () {
              self.status = REJECTED
              self.error = error
              self.onRejected(self.error)
          }, 0)
      }
  }
  fn(resolve, reject)
}

myPromise.prototype.then = function (onFulfilled, onRejected) {
  if (this.status === PENDING) {
      this.onFulfilled = onFulfilled
      this.onRejected = onRejected
  } else if (this.status === FULFILLED) {
      onFulfilled(this.value)
  } else if (this.status === REJECTED) {
      onRejected(this.error)
  }
}
```

> 这个也比较简单，添加三种状态，resolve,reject会分别改变成对应的状态，then的回调函数里根据当前不同的状态分别执行不同的操作

- _支持链式调用，借用jQuery的思路，每次then都把当前实例返回_
```
const PENDING = "pending"
const FULFILLED = "fulfilled"
const REJECTED = "rejected"

function myPromise(fn) {
  let self = this
  self.status = PENDING
  self.value = null
  self.error = null
  self.onFulfilled = null
  self.onRejected = null

  function resolve(value) {
      if (self.status === PENDING) {
          setTimeout(function () {
              self.status = FULFILLED
              self.value = value
              self.onFulfilled(self.value)
          }, 0)
      }
  }

  function reject(error) {
      if (self.status === PENDING) {
          setTimeout(function () {
              self.status = REJECTED
              self.error = error
              self.onRejected(self.error)
          }, 0)
       }
  }
  fn(resolve, reject)
}

myPromise.prototype.then = function (onFulfilled, onRejected) {
  if (this.status === PENDING) {
      this.onFulfilled = onFulfilled
      this.onRejected = onRejected
  } else if (this.status === FULFILLED) {
      onFulfilled(this.value)
  } else if (this.status === REJECTED) {
      onRejected(this.error)
  }
  return this
}  
```

> 通过return this来实现链式调用。但其实是有问题的，比如  `1.后面then的回调函数会覆盖前面的，最终返回的是最后一个的结果`  ` 2.每次传递的都是最开始定义的那个promise实例，与实际不符合，期望应该是每次都返回一个新的promise实例`

- _解决问题一，采用一个队列来保存then的回调函数_
```
const PENDING = "pending"
const FULFILLED = "fulfilled"
const REJECTED = "rejected"

function myPromise(fn) {
    let self = this
    self.status = PENDING
    self.value = null
    self.error = null
    self.onFulfilledCallbacks = []
    self.onRejectedCallbacks = []

    function resolve(value) {
        if (self.status === PENDING) {
            setTimeout(function () {
                self.status = FULFILLED
                self.value = value
                self.onFulfilledCallbacks.forEach(callback => {
                    callback(self.value)
                })
            }, 0)
        }
    }

    function reject(error) {
        if (self.status === PENDING) {
            setTimeout(function () {
                self.status = REJECTED
                self.error = error
                self.onRejectedCallbacks.forEach(callback => {
                    callback(self.error)
                })
            }, 0)
        }
    }
    fn(resolve, reject)
}

myPromise.prototype.then = function (onFulfilled, onRejected) {
    if (this.status === PENDING) {
        this.onFulfilledCallbacks.push(onFulfilled)
        this.onRejectedCallbacks.push(onRejected)
    } else if (this.status === FULFILLED) {
        onFulfilled(this.value)
    } else if (this.status === REJECTED) {
        onRejected(this.error)
    }
    return this
}
```

- _解决问题二，不使用return this ，而是新创建一个promise并返回_

> 这里会比较抽象，我自己也是理解了好久，举一个场景吧，现在我要获取一个用户信息，需要几个异步操作：getName，getAge，getInfo，每一次异步操作都会取上一个异步操作的返回值作为入参，按照平时的写法应该是这样的：
```
getName().
        then(getAge).
        then(getInfo).
        then(function(info){
            console.log(info)
        })
```

> 如果套用我们现在写的promise，是没有办法执行完一个异步任务之后返回一个新的promise再执行下一个，也获取不到上一个异步任务的返回值，因为它是把所有异步任务注册到最开始定义的promise上的回调函数队列里，然后在依次执行。举个例子：
```
let promise = new myPromise((resolve, reject) => {
    setTimeout(function () {
        resolve('success')
    }, 1000)

})
let promise1 = promise.then(
    reason => {
        a(reason)
    }
)

let promise2 = promise1.then(
    reason => {
        b(reason)
    }
)

function a(reason) {
    setTimeout(function () {
        console.log(reason + '1')
    }, 5000)
}

function b(reason) {
    setTimeout(function () {
        console.log(reason + '2')
    }, 1000)
}
```

> 返回结果是sucess2,sucess1，甚至还会因为异步执行完成时间打乱调用顺序！！！结合前面获取用户信息的例子来看，如果使用我们的promise，它会有以下问题：获取用户信息时拿不到上一步获取的年龄，获取年龄获取信息执行顺序不能确定......因此返回this是行不通的，需要返回一个桥梁promsie，做到承上启下的作用，代码具体实现如下：
```
const PENDING = "pending"
const FULFILLED = "fulfilled"
const REJECTED = "rejected"

function myPromise(fn) {
  let self = this
  self.status = PENDING
  self.value = null
  self.error = null
  self.onFulfilledCallbacks = []
  self.onRejectedCallbacks = []

  function resolve(value) {
      if (self.status === PENDING) {
          setTimeout(function () {
              self.status = FULFILLED
              self.value = value
              self.onFulfilledCallbacks.forEach(callback => {
                  callback(self.value)
              })
          }, 0)
      }
  }

  function reject(error) {
      if (self.status === PENDING) {
          setTimeout(function () {
              self.status = REJECTED
              self.error = error
              self.onRejectedCallbacks.forEach(callback => {
                  callback(self.error)
              })
          }, 0)
      }
  }
  fn(resolve, reject)
}

myPromise.prototype.then = function (onFulfilled, onRejected) {
  const self = this
  let bridgePromise
  //防止使用者不传成功或失败回调函数，所以成功失败回调都给了默认回调函数
  onFulfilled = typeof onFulfilled === "function" ? onFulfilled : value => value
  onRejected = typeof onRejected === "function" ? onRejected : error => {
      throw error
  }
  if (self.status === FULFILLED) {
      return bridgePromise = new MyPromise((resolve, reject) => {
          setTimeout(() => {
              try {
                  let x = onFulfilled(self.value)
                  resolvePromise(bridgePromise, x, resolve, reject)
              } catch (e) {
                  reject(e)
              }
          })
      })
  }
  if (self.status === REJECTED) {
      return bridgePromise = new MyPromise((resolve, reject) => {
          setTimeout(() => {
              try {
                  let x = onRejected(self.error)
                  resolvePromise(bridgePromise, x, resolve, reject)
              } catch (e) {
                  reject(e)
              }
          })
      })
  }
  if (self.status === PENDING) {
      return bridgePromise = new MyPromise((resolve, reject) => {
          self.onFulfilledCallbacks.push((value) => {
              try {
                  let x = onFulfilled(value)
                  resolvePromise(bridgePromise, x, resolve, reject)
              } catch (e) {
                  reject(e)
              }
          });
          self.onRejectedCallbacks.push((error) => {
              try {
                  let x = onRejected(error)
                  resolvePromise(bridgePromise, x, resolve, reject)
              } catch (e) {
                  reject(e)
              }
          })
      })
  }
}

//catch方法其实是个语法糖，就是只传onRejected不传onFulfilled的then方法
myPromise.prototype.catch = function (onRejected) {
  return this.then(null, onRejected)
}
//用来解析回调函数的返回值x，x可能是普通值也可能是个promise对象
function resolvePromise(bridgePromise, x, resolve, reject) {
  //如果x是一个promise
  if (x instanceof myPromise) {
      //如果这个promise是pending状态，就在它的then方法里继续执行resolvePromise解析它的结果，直到返回值不是一个pending状态的promise为止
      if (x.status === PENDING) {
          x.then(y => {
              resolvePromise(bridgePromise, y, resolve, reject)
          }, error => {
              reject(error)
          })
      } else {
          x.then(resolve, reject)
      }
      //如果x是一个普通值，就让bridgePromise的状态fulfilled，并把这个值传递下去
  } else {
      resolve(x)
  }
}
```

> 简单解释一下，就以then中的FULFILLED状态为例：
> 
> - 必须返回一个新的promise
> - 先获取到成功回调函数的值x，调用resolvePromise解析，根据是否是promise进行不同的处理
> - 如果x是promise，不断用resolvePromise递归解析直到它的状态不是PENDING，然后调用 x.then(resolve, reject)，注意：这里其实是x.then(value => resolve(value), reason => reject(reason))的简写，它的作用是对bridgePromise进行resolve或者reject，结合前面获取用户信息场景来看，就意味着getName后，getAge(name)返回的是一个promise，它会在拿到age后进行resolve(age)。getName.then也要返回一个bridgePromise，这个bridgePromise要在什么时候resolve呢？很容易理解他需要在前面那个promise resolve(age)后，因此就直接在它的then里调用bridgePromise的resolve，并且把获取到的age传递下去。
> - 如果x不是promise，就直接调用bridgePromise的resolve把这个值传下去

**现在就可以实现了promise的链式调用了，要达到promise A+规范其实还有挺多需要完善的，比如解决循环引用的问题（其实就是加一个判断bridgePromise和x是否相等），还有一些promise的其他方法，后续再补充吧~**