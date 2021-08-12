---
author: Ashu Deshwal
layout: post
title: "How to set a React Component or dom element as a background image"
description: "My internship experience at HackerEarth"
category:
tags: [React, Frontend, Html]
---

{% include JB/setup %}

During my internship at HackerEarth, I faced an interesting problem. This blog is about that and how I solved it.

## Problem: To set a background image to the textarea element.

My initial impression on seeing the design was that it would be easy. I thought it's a image but after exploring the code I came to know that it’s not an image that we have to show as background instead, it’s a **React Component**.
So now what? To solve this problem we need to think from scratch.

## Solution:

First, we will think about how to do it and then will implement it step by step.

### 1. Think From Scratch

<iframe  height="500" width="680"  src="https://codepen.io/thestl/embed/preview/bGVeama?default-tabs=css%2Cresult&amp;height=600&amp;host=https%3A%2F%2Fcodepen.io&amp;referrer=https%3A%2F%2Fmedium.com%2F%40ashudeshwal999%2Fcss-hack-how-to-set-a-react-component-or-dom-element-as-a-background-image-7b2628662544&amp;slug-hash=bGVeama"></iframe>

In this example, as you can see the content in the body element overlap the background-image. To solve this problem we just need to overlap the React Component with the textarea.

### 2. Implementation

Create a textarea and React Component which we are going to use. The text area is to the left and to the right is the React Component which we are going to set as a background image in the textarea element.

<iframe src="https://cdn.embedly.com/widgets/media.html?src=https%3A%2F%2Fcodesandbox.io%2Fembed%2Fcranky-pascal-3wd16%3Ffile%3D%2Fsrc%2FApp.js&amp;display_name=CodeSandbox&amp;url=https%3A%2F%2Fcodesandbox.io%2Fs%2Fcranky-pascal-3wd16%3Ffile%3D%2Fsrc%2FApp.js&amp;image=https%3A%2F%2Fcodesandbox.io%2Fapi%2Fv1%2Fsandboxes%2F3wd16%2Fscreenshot.png&amp;key=a19fcc184b9711e1b4764040d3dc5c07&amp;type=text%2Fhtml&amp;schema=codesandbox" height="500" width="680"></iframe>

<br/>
#### Steps:

1.Create a div which wrap textarea and React Component. Set the div position: relative and React Component position: absolute, top: 0 & left: 0.

{% highlight html %}
{% raw %}
<div className="editor-container"> // position: relative
  <textarea className="editor"/>
  <BackGround /> // position: absolute; top: 0; left: 0;
</div>
{% endraw %}
{% endhighlight %}

2.To overlap textarea on React component we need to set React Component z-index: -1.

{% highlight css %}
{% raw %}
.bg-img {  
  position: absolute;  
  top: 0;  
  left: 0;  
  width: 250px;  
  font-family: monospace;
  text-align: center;  
  color: rgba(0, 0, 0, 0.29);  
  z-index: -1;
}
{% endraw %}
{% endhighlight %}

As you can see we have a **problem** now. React Component is below the textarea and we are not able to see.

3.To solve the problem we need to make the textarea background transparent. But then if you click on the React Component you won’t be able to edit the textarea. To solve this set pointer-events: none.

{% highlight css %}
{% raw %}
.editor {
  height: 400px;
  width: 400px;  
  background: rgba(0, 0, 0, 0); // background transparent
}
.bg-img {  
  position: absolute;  
  top: 0;  
  left: 0;  
  width: 250px;  
  font-family: monospace;
  text-align: center;  
  color: rgba(0, 0, 0, 0.29);  
  z-index: -1;
  pointer-events: none;  // not to react on pointer events
}
{% endraw %}
{% endhighlight %}

4.(Optional) Set the position of React Component at the centre.

{% highlight css %}
{% raw %}
.bg-img {
  position: absolute;
  width: 250px;
  font-family: monospace;
  text-align: center;
  color: rgba(0, 0, 0, 0.29);
  z-index: -1;
  pointer-events: none;  
  top: 50%;
  left: 50%;
  transform: translate(-50%, -50%);
}
{% endraw %}
{% endhighlight %}

## End Result
<br />

<iframe src="https://codesandbox.io/embed/hopeful-brook-1ysf9?file=%2Fsrc%2Fstyles.css&amp;referrer=https%3A%2F%2Fmedium.com%2F%40ashudeshwal999%2Fcss-hack-how-to-set-a-react-component-or-dom-element-as-a-background-image-7b2628662544" height="500" width="680"></iframe>

<br/>
_Posted by [Ashu Deshwal](https://www.linkedin.com/in/ashu-deshwal/){:target="\_blank"}, Frontend Engineer_
