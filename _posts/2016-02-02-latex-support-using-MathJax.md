---
layout: post
title: Beautiful Math Symbols
description: HackerEarth now renders math symbols using LaTex macro using
MathJax javascript library.
category: front-end
tags:['latex', 'javascript', 'front end', 'HackerEarth']
---
{% include JB/setup %}

### Introduction:

By nature HackerEarth has so many programming puzzles. These puzzles bound to have mathematical equations, statements and symbols. Problem Setters often requested us to add support for Latex.

### Implementation:
The obvious and easy solution for this is to implement Latex support for individual pages. But that’s not scalable and maintainable. Then we came with up with a solution where we didn’t have to write custom code for every page. We examined the site and figured out three type of rendering that happens in the browser. Before going to solutions case by case, let’s first understand how MathJax, the library we use for [typesetting](https://en.wikipedia.org/wiki/Typesetting) Latex, works.

Everything in MathJax works asynchronously. After initializing MathJax, it executes a set of tasks and then typesets the queued content. Everything in MathJax works asynchronously using queues. Mathjax [docs](http://docs.mathjax.org/en/latest/start.html) explains it in detail.

**Types of rendering:**

 - Rendering of preview section in editor
 - Static content rendering (synchronous)
 - Rendering of dynamic content via Ajax (Asynchronous)

Read through rest of the article for this categorization to make sense.

### Editor:

We use pagedown editor throughout our site. It has a preview section which displays the converted markdown content. Now this should also display typesetted Latex content.

![Latex in pagedown editor](https://d320jcjashajb2.cloudfront.net/media/uploads/89bea86.png)

Pagedown editor has feature called hooks. This is a mechanism for plugging external text processors in between various steps of markdown processing. We have written a hook for typesetting Latex macros. This is chained to the hooks at the end, after all the markdown processing is completed. This hook takes markdown processed content as input and spits out Latex typesetted content.

```javascript
function renderLatex(text) {
    var invisible_div = document.createElement("div");
    invisible_div.style.cssText = "display:hidden";
    attr = document.createAttribute("id");
    attr.value = "mathjax_text";

    invisible_div.setAttributeNode(attr);
    invisible_div.innerHTML = text;
    document.body.appendChild(invisible_div);

    elem = document.getElementById("mathjax_text");
    MathJax.Hub.Queue(["Typeset", MathJax.Hub, elem]);

    child_node = document.body.removeChild(elem);
    return child_node.innerHTML;
}
```

This function creates an invisible div, sets the input text the to this div, queues this div in MathJax queue. Then MathJax typesets the div.

We are doing this because, MathJax work asynchronously. It doesn’t takes input text and spits out typeset text, like normal functions do. Hence this workaround. When you’re working with MathJax this is not a workaround. This is the ideal way.

### Webpages:
The content that’s written in the editor is saved in the database and shown in the web pages. And this content displayed in the browser in two different way.

 1. Synchronous
 2. Asynchronous aka Ajax

The way MathJax works, we can only typeset the Latex content only after it has loaded into the page. Once the page loaded, we typeset the whole page. But when the user interacts with the page, content keep changing. New content is added to page via ajax. It is really inefficient to typeset the whole page every time a bit of content changes. Sometimes it even causes errors.

**Synchronous:**

This bit is easy. After the page loads completely, ask MathJax to render the all the latex content in the page. This can be achieved by following piece of code.

```javascript
<script type="text/javascript">
    window.addEventListener("load", function() {
                MathJax.Hub.Queue(["Typeset", MathJax.Hub]);
            });
</script>
```

**Asynchronous:**

In this case, instead of queueing the whole page for typesetting, we queue only the divs that are modified, for typesetting. Luckily(rather pragmatically) we use very few set of javascript functions throughout the site for fetching ajax content. All we have to do is to modify these function so that they will queue the modified divs for typesetting.

This is the function for typesetting divs.

```javascript
function latexifyAjaxDiv(div) {
    setTimeout(function() {
        MathJax.Hub.Queue(["Typeset", MathJax.Hub, div[0]]);
    }, 10);
}
```

This function is called in ajax utility functions, right after setting content to the divs.

This approach makes almost all the pages in our site Latex compatible.



