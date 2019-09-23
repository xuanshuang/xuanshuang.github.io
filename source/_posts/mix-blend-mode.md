---
title: CSS中的mix-blend-mode
date: 2019-09-07 23:03:33
tags: 
- css
- mix-blend-mode
---
最近在给大数据部门做可视化，有一个动画因为赶时间，决定采用GIF动图来实现。然后设计师居然给不出透明背景的GIF，需要我们自己采用CSS来处理，即CSS中的mix-blend-mode属性。
<!--more-->

### 什么是mix-blend-mode
>The mix-blend-mode property specifies how the content of an element blends with its background and the content of its direct parent. Elements on overlapping layers will blend with those beneath it. In this case, it's important to mention the isolation property. It stops the elements with mix-blend-property from blending with the backdrop.

* 这个属性用来指定元素的内容如何与其背景和其直接父级的内容混合。
* 这个属性可以用在background属性和img标签上。
* 还有一个类似的属性background-blend-mode。

* 背景不能是transparent，透明色，这样黑色背景合成还是黑色背景。
* background-color初始值：rgba(0, 0, 0, 0), transparent。

### 为什么我失效了
实际操作中，我使用了这条CSS规则，却发现一点效果也没有。随即询问了教我使用这个属性的同学，他说他可以，不清楚我为什么不行，并且发给我一个Demo。在对比了Demo和我的代码后，发现应用了这条规则的元素，它的祖先中有一个position定位为fixed的元素。做了一些对比，如下：
1. 根元素position为fixed，自己为fixed，无效，黑色背景存在
![](/post-images/blend-0.png)
2. 根元素position为absolute，自己为fixed，有效，黑色背景被去除
![](/post-images/blend-1.png)
一开始以为是fixed属性创建了一个新的layer，在这个layer中，我没有设置任何背景颜色，才导致了属性的失效。后来发现，absolute定位也创建了新的layer。
![](/post-images/blend-2.png)

这时候，又发现如果在它的任意一个父节点设置一个不透明的背景颜色，无论是fixed还是absolute定位，都能去除GIF背景中的黑色。搞得我很迷惑。

<!-- ![](/post-images/blend-3.png)
![](/post-images/blend-4.png) -->

想了想，猜测失效的原因是还是与定位的方式有关，不同的定位改变了此元素父元素的内容。因为我这里包裹了很多div，而这些div的背景颜色都是默认的透明色。在fixed定位时，这个元素的父级是以window的内容来进行合成的，而在这里我们并没有什么有颜色的内容。透明色与黑色（我们的GIF背景是黑色的）混合，仍然是黑色的背景了。当我们使用absolute定位，存在一个不为static定位的父元素，则这个被混合元素的父元素是有内容的。

归结起来，mix-blend-mode是与它**直接父级的内容进行混合的**。

### 慎用
费了点力气，搞清楚了为什么属性失效，并在Chrome等主流器中达到了想要等效果。但是，**这个属性在IE、Edge是完全无效的**。
这里记录一个使用canvas来polyfill的方法：

```javascript
var URLreg = /(?:\(['|"]?)(.*?)(?:['|"]?\))/;

document.addEventListener('DOMContentLoaded', function() {
  var supportsBackgroundBlendMode = window.getComputedStyle(document.body).backgroundBlendMode;
  if(typeof supportsBackgroundBlendMode == 'undefined') {  
    // TODO: maybe check for Canvas composite support?
    createBlendedBackgrounds();
  }
}, false);

function createBlendedBackgrounds() {
  var els = document.querySelectorAll('.blend-multiply');
  for(var i = 0; i < els.length; i++) {
    var el = els[i];
    processElement(el);
  }
}

function processElement(el) {
  var style = window.getComputedStyle(el);
  var backgroundImageURL = URLreg.exec(style.backgroundImage)[1];
  var backgroundColor = style.backgroundColor;
  createBlendedBackgroundImageFromURLAndColor(backgroundImageURL, backgroundColor, function(imgData) {
    el.style.backgroundImage = 'url(' + imgData + ')';
  });
}

function createBlendedBackgroundImageFromURLAndColor(url, color, callback) {
  var img = document.createElement('img');
  img.src = url;
  img.onload = function() {
    var canvas = document.createElement('canvas');
    canvas.width = this.naturalWidth;
    canvas.height = this.naturalHeight;
    var context = canvas.getContext('2d');
    // IE9是支持CanvasRenderingContext2D API: globalCompositeOperation的
    context.globalCompositeOperation = 'multiply'
    context.drawImage(this, 0, 0);
    context.fillStyle = color;
    context.fillRect(0, 0, canvas.width, canvas.height);
    var data = canvas.toDataURL('image/jpeg');
    callback(data);
  };
}
```
本想偷点懒，没想到花的时间也不少。。。记录一下以防忘记。

[css-blend-mode](https://kolosek.com/css-blend-mode/)
[globalCompositeOperation](https://developer.mozilla.org/en-US/docs/Web/API/CanvasRenderingContext2D/globalCompositeOperation)
[how-to-hack-unsupported-mix-blend-mode-css-property](https://stackoverflow.com/questions/32613896/how-to-hack-unsupported-mix-blend-mode-css-property)

