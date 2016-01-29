---
layout: post
title: "Logging Javascript errors in production"
description: "Implementation of errorception javascript logger on HackerEarth"
category:
tags: [JavaScript, Logger, Error, Logging, HackerEarth, Errorception, Fastly]
---
{% include JB/setup %}

We had implemented a javascript logger to capture the pesky issues our users faced and had problems when using our site. When we came up with the idea, we thought it would be just a five minute job where we would just have to add a snippet and we would be done, but as we all know, nothing is simple when it comes to real world production issues.

After analyzing a lot of loggers, we decided to use **errorception** as our javascript logger. The reason was simple, it was simple in its approach and it provided the data we needed. This was the easy part, next came the part of integration, and yes there was a snippet which we just had to paste in our base javascript file, but one thing we had forgotten, CORS.

We host our static files on S3 and they are served via fastly, now because the static files are served through another domain, the error tracking snippet can’t log the errors on those files because window.onerror does not return the necessary stack trace and information and hence, comes CORS.

Many of you may have heard about CORS, it is a mechanism that allows restricted resources (e.g. fonts) on a web page to be requested from another domain outside the domain from which the resource originated. For security reasons, browsers restrict cross-origin HTTP requests initiated from within scripts. The only way to solve this and get errors to be posted to errorception was to allow CORS in the header of the static files.

After some research (errorception also has a good blog on it), we finally managed to have the error logging implemented. Below are the steps we had to go through:
* * *
### Configuring S3 ###
S3 has this unnecessarily complicated "CORS configuration" that you need to create. Here's the steps to get that right:

* Log into your AWS S3 console, select your bucket, and select "Properties". S3 CORS configurations seem to apply at the level of the bucket, and not the file. I have no clue why.
* Expand the "Permissions" pane, and click on "Add CORS configuration" or "Edit CORS configuration" depending on what you see.
* You should already be provided with a default permission configuration XML. If not, use the following XML to get started.

```XML
<?xml version="1.0" encoding="UTF-8"?>
<CORSConfiguration xmlns="http://s3.amazonaws.com/doc/2006-03-01/">
    <CORSRule>
        <AllowedOrigin>*</AllowedOrigin>
        <AllowedMethod>GET</AllowedMethod>
        <MaxAgeSeconds>3000</MaxAgeSeconds>
        <AllowedHeader>Authorization</AllowedHeader>
    </CORSRule>
</CORSConfiguration>
```

You should look at Amazon's docs to see what this configuration means.

* Make sure you add <?xml ?> declaration, if you omit these, Amazon will fail silently, showing you a happy looking green tick!

* Once you've saved the configuration, give it a couple of minutes.

* Test if everything's looking right. You could use a tool like curl to specify the additional headers needed for a "correct" CORS request:

```
$ curl -sI -H "Origin: example.com" -H "Access-Control-Request-Method: GET" https://s3.amazonaws.com/bucket/script.js

HTTP/1.1 200 OK

Date: Wed, 05 Nov 2014 13:37:20 GMT

Access-Control-Allow-Origin: *

Access-Control-Allow-Methods: GET

Access-Control-Max-Age: 3000

Vary: Origin, Access-Control-Request-Headers, Access-Control-Request-Method

Cache-Control: max-age=604800, public

...snip...
```
You should see the "Access-Control-Allow-Origin: *" header, and the "Vary: Origin" header in the output. If you do, you're golden.
* * *
### Configuring Fastly ###

For configuring fastly, we just had to follow a few simple steps and then we were done:

* Set up a custom HTTP header via the Content pane for your service.

* Then, create a custom header with the following information that adds the required "Access-Control-Allow-Origin" header for all requests.

```
Name            -   CORS S3 Allow
Type/Action     -   Cache       Set
Destination     -   http.Access-Control-Allow-Origin
Source          -   "*"
Ignore if Set   -   No
Priority        -   10
```

* Finally test it out:
Running the command **_curl -I your-hostname.com/path/to/resource_** should include similar information to the following in your header:

```
Access-Control-Allow-Origin: http://your-hostname.tld
Access-Control-Allow-Methods: GET
Access-Control-Expose-Headers: Content-Length, Connection, Date...
```
Once we had the "Access-Control-Allow-Origin" set up for both **S3** and **Fastly**, we had one major thing left to do, to add **_crossorigin=’anonymous’_** to all our script tags. For this we used a simple regex to modify the existing script tags in all the files:

```
Find:       (<script)(.*?)(STATIC_URL)(.*?)(.js)(.*?)("|')(>)

Replace:    $1$2$3$4$5$6$7 crossorigin=’anonymous’$8
```
After this we just bumped the version of all js files (to clear cache) and then we were done. We finally had the js logger implemented.

This helped us to identify the javascript issues in realtime and make the user experience better across all browsers.

<center>And as always, *Happy Coding!*</center>


* * *

References:  
[http://blog.errorception.com/2012/12/catching-cross-domain-js-errors.html](http://blog.errorception.com/2012/12/catching-cross-domain-js-errors.html)  
[http://blog.errorception.com/2014/11/enabling-cors-on-amazon-cloudfront-with.html](http://blog.errorception.com/2014/11/enabling-cors-on-amazon-cloudfront-with.html)  
[https://docs.fastly.com/guides/performance-tuning/enabling-cross-origin-resource-sharing](https://docs.fastly.com/guides/performance-tuning/enabling-cross-origin-resource-sharing)

* * *

*Send an email to support@hackerearth.com regarding any bugs or suggestions.*  
*Posted by [Shivindera Singh](https://www.hackerearth.com/@shivindera).*