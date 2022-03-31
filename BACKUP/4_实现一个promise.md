# [实现一个promise](https://github.com/liangyisong34/Suguy-blog/issues/4)

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