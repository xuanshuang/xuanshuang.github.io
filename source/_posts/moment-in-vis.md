---
title: moment in vis with antd
date: 2019-03-06 18:47:14
---
有人问我，我明明在antd里全局设置了moment的locale，怎么vis里就是不生效啊！决定水一篇。
<!--more-->

### 问题
在项目中，遇到明明import了zh-cn，在vis使用zh-cn的locale却总不生效，依旧是英文。
代码是这么写的：
``` js
import moment from 'moment';
import 'moment/locale/zh-cn'; // 引入中文locale
import vis from 'vis';

const groups = new vis.DataSet();
groups.add({ id: 0, content: '执法过程' });
groups.add({ id: 1, content: '侦查过程' });
timeline = new vis.Timeline(instance, data, groups, {
  align: 'left',
  groupOrder: 'id',
  width: '100%',
  zoomMin: 604800000,
  locale: 'zh-cn', // 指定vis使用中文locale
});
```
此时，antd datepicker组件使用了中文的locale；但是，vis库却仍然倔强的显示英文。
![](/post-images/vis-in-moment-1.png)

### 为啥子嘞
通过debugger vis源码发现，vis每次使用的moment都不是第一行import的那个moment；*时间过久，忘记了为什么vis使用的moment不是前面import进去的那个了*。
在vis源码中，看到了下面代码：
``` js
module.exports = (typeof window !== 'undefined') && window['moment'] || require('moment');
```
并且vis的dist文件还把moment打包进去了。只好在`import vis`之前加入两行代码来实现需求:

``` js
import moment from 'moment';
window.moment = moment;
```
￼
![](/post-images/vis-in-moment-2.png)

### 由此补充的知识点

在我们的项目中，每次`import moment from 'moment';`的moment都执行一次**初始化**吗？

> ES6模块的设计思想，是尽量的静态化，使得编译时就能确定模块的依赖关系（这种加载称为“编译时加载”），以及输入和输出的变量。CommonJS（用于服务器）和AMD（用于浏览器）模块，都只能在运行时确定这些东西。可以参考[es6-module](http://es6.ruanyifeng.com/#docs/module)。

> AMD 定义了`define`函数，我们可以使用`typeof`探测该函数是否已定义。若要更严格一点，可以继续判断`define.amd`是否有定义。另外，SeaJS也使用了`define`函数，但和AMD的define又不太一样。
对于 CommonJS，可以检查`exports`或是`module.exports`是否有定义。使用`require`引入。

这是momont初始化部分的源码：
``` js
(function (global, factory) {
    typeof exports === 'object' && typeof module !== 'undefined' ? module.exports = factory() :
    typeof define === 'function' && define.amd ? define(factory) :
    global.moment = factory()
}(this, (function() { /** moment的初始化代码 **/ })) // 此处编译后的代码，没有把moment挂在window下面
```
