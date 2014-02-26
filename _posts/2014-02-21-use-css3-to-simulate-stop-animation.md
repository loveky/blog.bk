---
layout: post
title: "使用CSS3实现逐格动画效果"
description: ""
category: CSS 
tags: [前端, CSS3]
---
{% include JB/setup %}

首先我们有这样一张照片：

{% image css3-stop-animation-background.png class="span10" %} 

照片长度为1184px，高度为75px，共分为24格。

首先我们将DIV大小设置成单格照片的大小，49x75。这样在初始状态下看到的就是第一格的照片。   接下来定义的是最终状态也就是最后一格的状态，通过偏移量背景调整到最后一格。     
最后就是定义动画效果，在下面的设置中时长2秒，动画会无限循环下去。

{% highlight css %}
@keyframes wave {
    to {
        background-position: -1184px 0
    }
}
@-webkit-keyframes wave {
    to {
        background-position: -1184px 0
    }
}
#hahaha {
    margin: 50px auto;
    width: 49px;
    height: 75px;
    background: url('https://raw.github.com/loveky/loveky.github.io/master/assets/images/css3-stop-animation-background.png') 0 0;
    -webkit-animation: wave 2s infinite steps(24);
    animation: 2s wave infinite steps(24);
}
{% endhighlight %}


效果如下:
<iframe width="100%" height="200" src="http://jsfiddle.net/loveky/ecA4N/embedded/result" allowfullscreen="allowfullscreen" frameborder="0">  </iframe>
由于照片是自己剪辑拼成的，所以偏移量不是太精确。。。