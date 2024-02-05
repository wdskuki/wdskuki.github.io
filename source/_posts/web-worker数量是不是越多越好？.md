---
title: web worker数量是不是越多越好？
date: 2024-01-25 10:00:00
tags:
---

# 背景

一个CPU密集型的任务，如果可以用web worker并行处理，创建的Web Worker线程是否越多越好呢？
如果不是的话，创建多少个最合适呢？

# 实验

我们以用JS实现从1累加到100亿来模拟这个问题。

```js
// index.js
const num = 10000000000
let sum = 0
for(let i = 1; i < num; i++) {
    sum += i
}
```

上面的运算如何放在JS主线程里运行话，会造成阻塞。因此为了加速运算，需要开启web worker来并行处理。

1.  新建多个web worker对象；
2.  每个web worker只做一段数据的累加；
3.  每当任意一个web worker运算完成之后，将值累加到一个全局变量上；
4.  所有的web worker运算完成，则全局变量的值即为结果值。

```js
// index.js
const workerArray = [
  new Worker("worker.js"), 
  new Worker("worker.js"), 
  ...
]

let sum = 0 // 累加结果
let count = 0 // worker运行完成的个数
const startTimestamp = performance.now();

// 监听返回值
workerArray.forEach(item => {
    item.onmessage = function(event) {
        sum += event.data
        count++
        if(count === workArray.length) {
        // 当所有的worker运行完成
            const endTimestamp = performance.now()
            console.log(`Worker total sum: ${sum}, time cost: ${endTimestamp - startTimestamp}`)
        }
    }
})
 
// 调用worker.js进行运算
let start = 1 // 每个worker计算的起始值
const MAX_DURATION = Math.ceil(num / workerArray.length)
workerArray.forEach(item => {
    // 每个worker计算的结束值
    const end = Math.min(start + MAX_DURATION, num)
    item.postMessage({ start, end })
    start = end + 1 // 更新起始值
}); 

/////////////////////////////////
// worker.js
self.onmessage = function(event) {
  const start = event.data.start
  const end = event.data.end
  let sum = 0
  for(let i = start; i <= end; i++) {
    sum += i
  }
  self.postMessage(sum);
}

```

下图是我们运行多次的结果。x轴表示我们开启的web worker数量，y轴表示运行完获取总累加值需要的时间（ms）。

随着x值的增加，y值减少。当x到了一定值以后，y的减少就不明显了。这个值大概是6。

![](https://dsweiblog.oss-cn-shanghai.aliyuncs.com/technote/SCR-20240125-kvf.png)

原因其实很简单，因为线程的并行需要依赖于CPU的核数。下图是我当前电脑的基本配置，可以看到一共有6个核。

![](https://dsweiblog.oss-cn-shanghai.aliyuncs.com/technote/SCR-20240125-p89.png)

查阅资料可以知道，可以通过API`navigator.hardwareConcurrency`获取当前机器的CPU的逻辑核数。因此代码我们可以简单修改下：

```js
const workerArray = Array.from({length: navigator.hardwareConcurrency}, () => new Worker('./worker.js'))

...
// 其他代码不变
```

# 结论

大部分需要通过web worker并行运算的场景下，JS主线程开启web worker的数量最好不要超过`navigator.hardwareConcurrency`的数量，否则过多的web worker不仅不能带来性能的提升，反而会增加额外通信和内存开销。

# 其他

代码地址：<https://github.com/wdskuki/js-tech-demo/tree/master/webworker>
