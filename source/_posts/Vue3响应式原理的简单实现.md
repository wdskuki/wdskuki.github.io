---
title: Vue3响应式原理的简单实现
date: 2022-04-23 10:00:00
tags: vue
---

# 背景
首先用一个例子来解释什么是响应式。
1. 设置变量`a`为`1`；
2. 设置变量`b`为`a`和`2`之和；
3. 当修改`a`的值，`b`的值也会随之发生改变。
# 初步实现
 ```javascript
 let a = 1
 let b = a + 2
 a = 3 // b 为 3
 a = 4 // b 为 3
 ```
 上述代码显然无法满足例子中的第三点，即`b`随`a`的变化而变化。原因很简单，JS代码是从上往下执行的。`a`的赋值操作无法通知上一行的`b = a + 2`。
 如果一定要希望`b`及时能够响应`a`的变化，可以在每次`a`的赋值操作之后紧跟`b = a + 2`
 # 简单代码的改进
 ```javascript
 let a = 1
 let b = undefined
 b = a + 2
 // ...
 a = 3
 b = a + 2
 // ...
 a = 4
 b = a + 2
 ```
方便起见，这里将`b = a + 2`包在一个函数里。
 ```javascript
 let a = 1
 let b = undefined
 function foo() {
    b = a + 2
 }
 foo()
 // ...
 a = 3
 foo()
 // ...
 a = 4
 foo()
 ```

但这里有一个问题，就是`a`的赋值操作必须手工紧跟这个foo函数的执行。
那么，是否有一个自动的方式来处理这个依赖关系呢？

# Proxy对象
ES6里有一个`Proxy`，可以对对象属性的`get/set`操作进行拦截处理。
如果要将`Proxy`的这个功能运用到这里，需要将`a`包装成一个对象。
```javascript
let a = 1
let b = undefined
let obj_a = {a} // 将a包装成对象
let proxy_a = new Proxy(obj_a, {
    get(target, property) {
       return target[property]
    },
    set(target, property, value) {
        target[property] = value
        foo() // 拦截set操作，执行foo函数
        return true
    }
})
function foo() {
    b = proxy_a.a + 2
}

foo()
// ...
proxy_a.a = 3 // b 为 5
// ...
proxy_a.a = 4 // b 为 6
```
* 首先，将`a`封装成目标对象`obj_a`，用`Proxy`对`obj_a`进行代理，返回的对象`proxy_a`为目标对象`obj_a`的代理对象；
* 其次，因为对`obj_a`的`set`操作进行拦截调用`foo`函数，因此`proxy_a`的属性赋值操作，会实时的运行`foo`函数，这样就实时更新`b`值。
* 只有对`proxy_a`的属性赋值才会自动调用`foo`函数，如果是对`obj_a`的属性赋值，或者直接对`a`赋值，则不会自动调用`foo`函数。


为了简化后续分析，我们将上面的问题进行进一步抽象和简化：

* 因为这里重点关注的是响应式机制的实现，而`Proxy`的输入又只能适用于`对象`，因此文章后面部分我们只考虑`对象类型`的参数（因为基本类型也可以通过简单封装，改造成对象）。
* 另外，`b`存在的意义在于观察程序是否能够响应`proxy_a.a`的变化，因此用一个`console.log(proxy_a.a)`也能购代替`b`的作用，而且还少一个变量。
可以将代码改成如下：

```javascript
let obj_a = {a}
let proxy_a = new Proxy(obj_a, {
    get(target, property) {
       return target[property]
    },
    set(target, property, value) {
        target[property] = value
        foo()
        return true
    }
})
functon foo() {
    console.log('foo ', proxy_a.a)
}

```
只要在每次`proxy_a.a`赋值，看下`console.log`输出的值是否也有响应变化，就能判断响应式机制是否正确实现了。

# 多个类似的`foo`函数的情况
前面的例子中正好只有一个`foo`函数，且这个foo函数正好用到了`proxy_a.a`，因此我们在`set`操作的时候可以精准的知道需要调用的哪个函数和函数名。
但如果是如下情况呢？
```javascript
let obj_a = {a:1}
let proxy_a = new Proxy(obj_a, {
    get(target, property) {
       return target[property]
    },
    set(target, property, value) {
        target[property] = value
        foo() // 这里调用的是foo，而不是foo1还foo2，原因是什么？
        return true
    }
})
function foo() {
    console.log('foo ', proxy_a.a)
}
function foo1() {
    console.log('foo1')
}
function foo2() {
    console.log('foo2', proxy_a.a)
}
.....
```
这里有多个类似的`foo`函数，有些和`proxy_a`有关，有些又没有关系。如果我们想事前就实现一个通用的`proxy_a`，在定义`proxy_a`的`set`操作的时候，如何知道该调用哪个函数呢?

一个有效的实现：
- 首先运行每个`foo`函数，运行过程中检查是否有用到`proxy_a`的`get`方法；
- 如果有的话，就将当前`foo`函数和当前的代理对象`proxy_a`的关系记录下来；
- 后续如果有用到`proxy_a`的`set`方法时，再从这个对应关系获得相关联的`foo`函数，并依次执行。

```javascript
let obj_a ={a: 1}
let set = new Set() // 保存和proxy_a有关联的函数变量
let activeFunc = undefined // 于记录当前运行的是哪个函数变量

let foo = function() {
    activeFunc = foo
    console.log('foo =', proxy_a.a)
    activeFunc = undefined
}
let foo1 = function() {
    activeFunc = foo1
    console.log('foo1')
    activeFunc = undefined
}
let foo2 = function() {
    activeFunc = foo2
    console.log('foo2 =', proxy_a.a)
    activeFunc = undefined
}

let proxy_a = new Proxy(obj_a, {
    get(target, property) {
        if(activeFunc) {
          set.add(activeFunc)
        }
        return target[property]
    },
    set(target, property, value) {
        for(let item of set) {
            item()
        }
        target[property] = value
        return true
    }
})

foo()
foo1()
foo2()

//....
proxy_a.a = 3
// ...
proxy_a.a = 4

```
* `set`保存和`proxy_a`有关联的函数变量；
* `activeFunc`用于记录当前运行的是哪个函数；
* 如果当前运行`foo`函数中，存在`proxy_a.a`，那么就会进入到`get`拦截器，将当前的函数变量添加到`Set`集合；
* 运行完全部的`foo`函数，`Set`集合保存了所有和`proxy_a.a`有关系的函数变量；
* 后续在给`proxy_a.a`赋值的时候，进入到`set`拦截器，将`Set`集合中保存的函数逐个执行。

# 总结
以上就是Vue3响应式实现的思路原型。我们总结下：
* Vue3的响应式处理的是对象，我们称为目标对象；如果不是对象，可以通过处理，封装成一个对象；
* 利用`Proxy`，从目标对象生成一个代理对象，代理对象的属性读写操作可以进行拦截预处理；
* 将响应式关联逻辑封装到函数中，且响应式逻辑中的目标对象须替换为代理对象；
* 程序运行的同时也同时运行上述封装的函数，并收集和代理对象`get`操作有关的函数变量，保存到一个`Set`集合中;
* 执行完毕之后，如有对代理对象的属性赋值操作，则会触发`Set`集合中的函数变量依次执行，从而完成响应式流程。
