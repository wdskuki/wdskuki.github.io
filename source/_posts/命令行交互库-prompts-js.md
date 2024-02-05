---
title: 命令行交互库-prompts.js
date: 2023-11-10 10:00:00
tags: javascript
---

## 1. 什么是prompts

prompts是一个命令行交互的库，它可以让用户在命令行中输入一些信息，然后程序根据这些信息做出相应的反应。

## 2. 什么是命令行交互

命令行交互是指用户通过命令行输入一些信息，然后程序根据这些信息做出相应的反应。
比如，用户在命令行中输入`npm init`，然后程序就会根据命令的一些参数，比如项目名称，项目类型，模板等等，然后根据这些参数，生成一个package.json文件。

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/40758309181f4b2ab4510b13172be4e3~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=602\&h=208\&s=101234\&e=gif\&f=37\&b=000000)

## 3. 用法

### 简单用法

```javascript
const prompts = require('prompts');

(async () => {
  const response = await prompts({
    type: 'number',
    name: 'value',
    message: 'How old are you?',
    validate: value => value < 18 ? `Nightclub is 18+ only` : true
  });

  console.log(response); // => { value: 24 }
})();
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/824b6963b4824e2a87d00730163836e0~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=602\&h=202\&s=15927\&e=gif\&f=11\&b=010101)

简单解释下上面代码中prompts的参数：

*   type: 输入框的类型，比如number，text，password等等。
*   name: 输入框的名字，用来在后面获取输入框的值。
*   message: 输入框的提示信息。
*   validate: 输入框的校验函数，用来校验输入框的值。

### prompts链

如果prompts构造函数中传递到一个列表，那么就能实现链式的响应式交互。这里需要注意的是，列表中的每个元素的name需要唯一，否则会出现相互覆盖的情况。

```javascript
const prompts = require('prompts');

const questions = [
  {
    type: 'text',
    name: 'username',
    message: 'What is your GitHub username?'
  },
  {
    type: 'number',
    name: 'age',
    message: 'How old are you?'
  },
  {
    type: 'text',
    name: 'about',
    message: 'Tell something about yourself',
    initial: 'Why should I?'
  }
];

(async () => {
  const response = await prompts(questions);

  // => response => { username, age, about }
})();
```

![](https://p3-juejin.byteimg.com/tos-cn-i-k3u1fbpfcp/572fb0a8bae04b5da62d73afb174044d~tplv-k3u1fbpfcp-jj-mark:0:0:0:0:q75.image#?w=602\&h=208\&s=23215\&e=gif\&f=16\&b=010101)

# 4. 总结

*   prompts 是一个命令行交互的库，它可以让用户在命令行中输入一些信息，然后程序根据这些信息做出相应的反应。
*   prompts通常用于一些脚手架项目的交互，比如npm init，npm install，npm run等等。
*   一些知名的开源项目都集成有prompts，比如create-react-app，vue-cli等等。

## 5. 参考资料

*   [prompts](https://github.com/terkelg/prompts)
