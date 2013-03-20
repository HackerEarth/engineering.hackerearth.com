---
layout: post
title: "HackerEarth Technology Stack"
description: "Description of technology used by us in building HackerEarth."
category: 
tags: [HackerEarth, Technology, Python, Django, CodeFactory, node.js]
---
{% include JB/setup %}

*Originally posted as [Quora answer](http://www.quora.com/HackerEarth/What-technology-does-X-use-What-is-Hackerearth-technology-stack/answer/Vivek-Prakash-2)*.

<br>
This might take a while, so go grab some popcorn. You are going to enjoy this :)

At [HackerEarth](http://www.hackerearth.com), we deeply believe in open-source.
Why not, our roots are in
there. We use open-source software, hack it according to our needs and create
something amazing out of that. At the same time, we don't fear writing a
seemingly complex project from scratch and turn it into beautiful piece of
code. And we have done that so many times in a very short span.

<br>
Our application backend is primarily in Python/Django. We modified
django-allauth for some of our custom needs. For example, it allows you to
login on all sites -  [HackerEarth](http://www.hackerearth.com),
[MyCareerStack](http://learn.hackerearth.com), and
[CodeTable](http://code.hackerearth.com) with the same
login credentials. We modified the django-threadedcomments for spam control
(they can really eat you inside!) and enhanced moderator permissions. We wrote
our own generic newsfeed system from scratch which can be plugged with any
information schema to generate feeds. For example, the feed that you see on
MyCareerStack and the recent submissions that you see on HackerEarth, they all
come filtered from the same core engine. This feed engine will form one of the
core of the upcoming platform and there are some interesting work being done in
there - like faster filtering with advanced algorithms, implementation of
relevant feed system, and other exciting stuffs. We wrote a notification engine
on top of newsfeed system which generates notifications for you. These are part
of MyCareerStack for now but as we integrate everything, they will be core of
the whole product. We also wrote a generic poll application which tracks all
the actions of a user on any item. For example, upvote/downvote on a question,
like/dislike on a tutorial, etc. are all powered and regulated by the same
application. These user actions can be easily integrated with any object
model e.g. programming problem with small snippets of code. As this
application develops, we will make it open-source eventually. We did some
nice hack with the avatar application to make the image loads faster from
Amazon S3. Similarly, we messed around with apache-solr to customize some
of the backends involved in searching.

<br>
And all this is just a fraction. Currently there are around 60 django
applications written in a very generic and modular way which communicate
with each other to power everything.

<br>
Wait, I haven't told you yet about the amazing stuffs. We wrote the
code-checker from scratch, and it's not the usual college project. It is
known as [CodeFactory Engine] (http://engineering.hackerearth.com/2013/03/12/100000-strong/)
internally and forms the core of HackerEarth.
The core-engine is written in C and the server is written in C++ using
Apache Thrift. It's a very robust client-server architecture with
auto-scaling and auto-deployment which gives result of each testcase in
real-time. We have built an [API](http://developer.hackerearth.com) around it
and we are going to build some
amazing things on top of it. [Vim plugin to compile/run code using API] (http://engineering.hackerearth.com/2013/03/11/hackerearth-vim-plugin/) is
just a start. We wrote a realtime server using node.js & nowjs which pushes
data to your browser as it gets updated or changed live. The result you get
in realtime on submission of program is powered by this server.

<br>
In the start of January 2013, we undertook one major task which was not
needed very much but we knew it will be essential very soon. We wanted to
reduce the page load time using pipelining techniques, similar to
[BigPipe: Pipelining web pages for high performance](http://www.facebook.com/note.php?note_id=389414033919).
But there wasn't anything out
there which we could have been directly integrated. This led to our native
implementation of bigpipe, which is still in a very nascent stage. But it
brought wonders to the page load time, reducing them by almost half.
Another major contribution was of memcached, we integrated it at view level
and core level throughout the site. Most of the things are cached, there
are systems in place which invalidates and updates them as the data
changes. Remember the hit-miss algorithm from Comp. Science 101 ? We did
that for webpages!

<br>
One weekend in October 2012, we had built the
[CodeTable](http://code.hackerearth.com/) to test our
CodeFactory server. But then we integrated collaborative coding in it and
it went on to become a full-fledged product in itself. It's widely used by
someone who wants to hack on some code instantly and share it with others
in realtime and doesn't bother about installing compilers/interpreters
locally.

<br>
At the frontend, we write [Stylus](http://learnboost.github.com/stylus/),
JavaScript and jQuery. We have written
custom jQuery plugins and JavaScript functions to work easily with lots of
often required tasks. For example, we wrote a generic plugin for lazy
scrolling, and it powers all the pages with infinite scrolling like
newsfeed, recent activity, user submission list, etc.   

<br>
All the sites, everything is part of one big project. This keeps us sane
and allows us to have greater control. Having said that, as the code base
grows over 100,000 lines of code and there are multiple servers running,
then there are totally different sets of challenges. There are 5 different
servers running right now - apache server, codeFactory server, realtime
server, collaboration server, and search server. The apache servers
(webserver) and CodeFactory servers are running on few different EC2
instances at any given moment. We wrote custom auto-deployment scripts,
builder & developer tools to make the life easy and improve the
productivity of our own. We have been using Git from day zero to manage all
the source code and have written some nice hooks and wrappers on top of it
to abstract a few tasks.

<br>
And we have done all this with just two of us working full time since
October 2012, and that too with so many things going around with us. I am
still a college student! Two others joined to work remotely from January
and their contribution has been significant too. And you might acknowledge
from all this that we don't fear anything, we deploy fast, we fail fast and
are growing aggressively. We still work the way we used to work in college
- enjoying life each and every day. We are expanding the team in summer and
are going to release tons of cool products that you will love and
programmers will love. But most importantly, that will disrupt the way tech
recruitment is done in India.

<br>
And all this has not been for nothing. Our user base have increased
significantly in past few months. Many companies have become our customers.
And we are working resolutely towards solving the problem we have to set
out to - i.e. to connect smart programmers to awesome product companies
coming out of India!

<br>
If all this sounds interesting and exciting to you, let's have a chat.
Email me at vivek@hackerearth.com, and I will be as enthusiastic to talk to
you as you will be! You can also find me
[@vivekprakash](https://twitter.com/vivekprakash).

*Posted by Vivek Prakash, Co-founder - HackerEarth*
