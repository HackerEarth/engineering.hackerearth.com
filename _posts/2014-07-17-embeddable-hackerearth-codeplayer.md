---
layout: post
title: "Embeddable HackerEarth CodePlayer"
description: "Six months back we launched CodePlayer as an initiative to help programmers share their knowledge in form of video and also provide a useful tool for recruiters to analyze thought process of candidates.
"
category:
tags: [HackerEarth, Ace Editor, Django, CodePlayer]
---
{% include JB/setup %}

Six months back we launched [CodePlayer](http://engineering.hackerearth.com/2014/01/21/introducing-codeplayer/) as an initiative to help programmers share their knowledge in form of video and also provide a useful tool for recruiters to analyze thought process of candidates.

Until yesterday, you could watch code video only on hackerearth site. Now, you can add any code video(with public url) to your website/blog using our embed code video feature.

### How to embed code video ###

<ul>
<li>First, you need to create a video. You can do so by writing code in <a href="http://code.hackerearth.com" target="_blank">CodeTable</a> or try <a href="http://code.hackerearth.com/code/play/41dc84fbad4d49ac9d91593890b6d534/">solving a problem</a> on HackerEarth.</li>
<li>As soon as you start writing code, a play code button appears which will take you to CodePlayer page. Click on 'Embed' tab below to get the embed code required for adding same video to your website/blog. You can configure the size and theme of video too.</li>
<img src="/images/embed-code.png" title="Sample image"/>
<li>Copy & Paste the highlighted embed code into your website/blog html page and you are done(easy as a piece of cake).</li>
<li>Hurrah! You can see some code action in your website.</li>
</ul>

### A working demo ###

Below I have added [Introducing CodePlayer](http://code.hackerearth.com/code/play/41dc84fbad4d49ac9d91593890b6d534/) video along with its embed code.

<iframe src="http://code.hackerearth.com/code/play/embed/41dc84fbad4d49ac9d91593890b6d534/?theme=dark" width="640" height="360" frameborder="0" allowtransparency="true" scrolling="no"></iframe>

    <!-- This is the code that renders above video -->
    <iframe src="http://code.hackerearth.com/code/play/embed/41dc84fbad4d49ac9d91593890b6d534/?theme=dark" width="640" height="360" frameborder="0" allowtransparency="true" scrolling="no"></iframe> 

### More features ###

The codeplayer page has few more features like editing video description only if you are owner of that video, number of views, and sharing on social media. We will be adding alot more useful features in time. And, if you have anything constructive to say let us know!

PS: This feature has helped us easily integrate codeplayer in recruiter panel. We will be launching that feature for recruiters soon.

* Posted by [Lalit Khattar](http://www.hackerearth.com/users/lalitkhattar/). Follow me [@LalitKhattar](http://twitter.com/LalitKhattar)*
