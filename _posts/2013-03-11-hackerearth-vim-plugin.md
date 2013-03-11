---
layout: post
title: "Vim plugin to compile/run code using API"
description: ""
category: 
tags: [Vim, Plugin, HackerEarth API]
---
{% include JB/setup %}

**tl;dr**

Check out <https://github.com/HackerEarth/hackerearth.vim>

- - -

I love Vim editor. And the idea of a plugin to compile and run code from my favorite
code editor sounded exciting. [HackerEarth API](http://developer.hackerearth.com)
made it easier.

Oh man - this is going to be the coolest thing I have ever built. Whoa,
wait a minute. Aren’t you forgetting something? You have never written a
plugin before.

There always comes a time when you have to do something for the first time! ;)


### Where to begin?

Google is the answer :)
After exploring and reading some tutorials, I found out that VimL, also
know as Vim script is the language used to built vim plugins. Here's the
best part. You only need very little knowledge of VimL to be able to write
plugins, if you know Python (or Ruby). I chose Python.

### Why Python?
Because using python gives so much flexibility. Think about using
urllib/httplib/json/vim for accessing some web service that helps editing
in Vim. This is why most of the plugins that work with web services are
usually done in VimL+Python. Also, I am learning Python these days so it
became a more solid reason to use it.

### Let’s write a vim plugin!
Generally vim plugins start with few checks. In this case, it checks for
VimL + Python support and also looks for HackerEarth API client key.
Obviously, if there is some error, plugin should not be loaded.

    " check for python
    if !has('python')
        echoerr "HackerEarth: Plugin needs to be compiled wuth python support."
        finish
    endif

    " check for client key
    if exists("g:HackerEarthApiClientKey")
        let s:client_key = g:HackerEarthApiClientKey
    else
        echoerr "You need to set client key in your vimrc file"
        finish
    endif


As said earlier, Python can easily be integrated with VimL. Python code inside
VimFunction starts after python << EOF and ends before EOF. Here is an example
on how to do it.

    function! s:VimFunction()
    python << EOF
    # write python code here
    import os
    print "Hello World!"
    EOF
    endfunction

### OOPS!
I have written most of the plugin code in Python and I love using Object
Oriented Programming(OOP) so I have used it in here as well.

HackerEarth.vim python code contains three main classes: Api, Argument and
VimInterface.

Api class handles all the Hacker Earth's API related stuff like calling the web
service, setting post data and handling json response. Argument class is
responsible for evaluating command line arguments and it also decides what
should be default argument values. VimInterface is an interesting piece of
code. It basically loads a buffer, appends output and saves it, if users wishes
to do so.

You may be wondering when all these classes gets instantiated. It is done
inside s:HackerEarth function.
In order to call the s:HackerEarth function, some commands are written.

    function! s:HackerEarth(action, ...)
    python << EOF
    action = vim.eval("a:action")
    argslist = vim.eval("a:000")
    args = None if(not argslist) else argslist[0]
    arg = Argument.evalargs(args)
    arg.setaction(action)
    if(not arg.Help):
        api = Api(arg)
        api.call()
    else:
        vim.command("Hhelp")
    EOF
    endfunction

    command! -nargs=? -complete=file Hrun :call <SID>HackerEarth("run", <f-args>)
    command! -nargs=? -complete=file Hcompile :call <SID>HackerEarth("compile", <f-args>)
    command! -nargs=0 Hhelp :call <SID>Hhelp()

### It’s almost finished!
The only part left is to map the above commands to keyboard shortcuts. Ok,
let’s do it.

    map <C-h>r :Hrun<CR>
    map <C-h>c :Hcompile<CR>
    map <C-h>h :Hhelp<CR>

### What's next?
Check out <https://github.com/HackerEarth/hackerearth.vim>

If you want to improve or fix anything, just do it and send us a pull request.
Or if that is not possible for you, send the diff to lalit@hackerearth.com.
Feel free to report any issue or contribute to the github repository.

P.S. I am an undergraduate student at IIT Roorkee, and I will be joining the
folks at HackerEarth in summer for 2 months internship. Follow me
[@LalitKhattar](https://twitter.com/LalitKhattar)


