---
layout: post
title: "Python Requests Module"
description: "How HackerEarth uses Python Requests to fetch data from various APIs [Tutorial]"
category: 
tags: [Requests, Python, Github, StackOverflow, API]
---
{% include JB/setup %}

One of the most liked feature of the newly launched [HackerEarth profile](http://www.hackerearth.com/about/profile/) is the accounts
connections through which you can boast about your coding activity in various
platforms.

Github and StackOverflow provide their API to pull out various kinds of data.
The API documentation of Github and StackOverflow can be found here.

* Github : [https://developer.github.com/v3/](https://developer.github.com/v3/)
* StackOverflow : [http://api.stackexchange.com/docs](http://api.stackexchange.com/docs)

But what do we use to communicate with these APIs?

Working with `HTTP` is a painful task. Python includes a module called `urllib2`
but working with it can become cumbersome.


[Requests](http://docs.python-requests.org/en/latest/) was written by [Kenneth Reitz](http://www.kennethreitz.org/) which simplies the common use
cases and the tool for HackerEarth to do all the HTTP operations.

Here is a code by @kenneth himself distinguishing urllib2 and requests
<script src="https://gist.github.com/973705.js"></script>

So, this above code clearly distinguishes why we went for the requests module.

###Installation

Installing Requests via pip is fairly simple, just run this in your terminal.  

`$ pip install requests`

###Making your first Request  
  
First of all, you need to import requests

`>>> import requests`

Now let's make a GET requests to get Github's public timeline

`>>> r = requests.get('https://github.com/timeline.json')`  

Now, we have Response object called r using which we can get all the
information.

Requests’ simple API means that all forms of HTTP request are as obvious. For
example, this is how you make an HTTP POST request:

`>>> r = requests.post("http://httpbin.org/post")`

Similarly the other HTTP request types: PUT, DELETE, HEAD and
OPTIONS?

    >>> r = requests.put("http://httpbin.org/put")
    >>> r = requests.delete("http://httpbin.org/delete")
    >>> r = requests.head("http://httpbin.org/get")
    >>> r = requests.options("http://httpbin.org/get")

Now, let's consider the GitHub timeline again:

    >>> import requests
    >>> r = requests.get('https://github.com/timeline.json')
    >>> r.text
    u'[{"created_at":"2014-06-08T20:50:27-07:00","payload":{"sha...

Requests will automatically decode content from the server. The text encoding
guessed by Requests is used when you access r.text. You can find out what
encoding Requests is using, and change it, using the r.encoding property:

    >>> r.encoding
    'utf-8'
    >>> r.encoding = 'ISO-8859-1'

There's an builtin JSON decoder on which we heavily rely on 

    >>> import requests
    >>> r = requests.get('https://github.com/timeline.json')
<script src="https://gist.github.com/973705.js"></script>
    >>> r.json()
    [{u'actor_attributes': {u'name': u'Tin...

###Passing parameters with URLs

You might often need to pass parameters. If you were constructing the URL by
hand, this data would be given as key/value pairs in the URL after a question
mark, e.g. httpbin.org/get?key=val

    >>> payload = {'key1': 'value1', 'key2': 'value2'}
    >>> r = requests.get("http://httpbin.org/get", params=payload)

You can see that the URL has been correctly encoded by printing the URL:

    >>> print(r.url)
    http://httpbin.org/get?key2=value2&key1=value1

Let's take a similar use case from HackerEarth where we get the information
related to the repositories. Pagination in Github says that by default a requests that return
multiple items will be paginated to 30 items. But we can set a custom page size
using `?per_page` parameter.

`'https://api.github.com/user/repos?per_page=100'`

and if you want to specify request a specific page you need to pass the ?page
parameter. The  page numbering is `1-based` and that omitting the `?page` parameter
will return the first page.

So, the URL for requesting 100 items from second page might look like

`'https://api.github.com/user/repos?page=2&per_page=100'`

Let's perform this via requests.

    >>> params = {'page': 2, 'per_page':100}
    >>> r = requests.get('https://api.github.com/user/repo/', params=params)

###Custom Headers

If you’d like to add HTTP headers to a request, simply pass in a dict to the
headers parameter.

For example, we didn’t specify our content-type in the previous example:

    >>> import json
    >>> url = 'https://api.github.com/some/endpoint'
    >>> payload = {'some': 'data'}
    >>> headers = {'content-type': 'application/json'}

    >>> r = requests.post(url, data=json.dumps(payload), headers=headers)

Let's take the earlier example of fetching data from repositories. Most of
the APIs require require access token for requesting data. The access token
needs to be added to HTTP headers.

    >>> headers = {'Authorization':'token %s' % token}
    >>> params  = {'page': 2, 'per_page': 100}
    >>> r = requests.get(url, params=params, headers=headers)

###Response Status Codes

We can check the status codes for the response using:

    >>> r = requests.get('http://httpbin.org/get')
    >>> r.status_code
    200

If we made a bad request like `4XX` or `5XX`, we can raise it with `Response.raise_for_status()`

    >>> bad_r = requests.get('http://httpbin.org/status/404')
    >>> bad_r.status_code
    404

    >>> bad_r.raise_for_status()
    Traceback (most recent call last):
    File "requests/models.py", line 832, in raise_for_status
        raise http_error
    requests.exceptions.HTTPError: 404 Client Error

    Response.raise_for_status() returns None for status_code 200

###Response Headers

We can view the server’s response headers using a Python dictionary:

    >>> r.headers
    {
        'content-encoding': 'gzip',
        'transfer-encoding': 'chunked',
        'connection': 'close',
        'server': 'nginx/1.0.4',
        'x-runtime': '148ms',
        'etag': '"e1ca502697e5c9317743dc078f67693f"',
        'content-type': 'application/json'
    }

    >>> r.headers['Content-Type']
    'application/json'

Let's take the earlier repository example again. Github uses pagination in their API.

    >>> url = 'https://api.github.com/users/sayanchowdhury/repos?page=1&per_page=10'
    >>> r = requests.head(url=url)
    >>> r.headers['link']
    '<https://api.github.com/user/500628/repos?page=2&per_page=10>; rel="next", <https://api.github.com/user/500628/repos?page=8&per_page=10>; rel="last"'

So, we parsed out the next url out of the headers:

    >>> link = r.headers.pop('link').split(',')[0]
    >>> link
    '<https://api.github.com/user/500628/repos?page=2&per_page=10>; rel="next"'
    >>> import re
    >>> url = re.findall(r'<(.*?)>', link)[0]
    'https://api.github.com/user/500628/repos?page=2&per_page=10'
    
But, requests has a intuitive way to do it.

    >>> r.links['next']
    'https://api.github.com/users/500628/repos?page=2&per_page=10'

    >>> r.links['last']
    'https://api.github.com/users/500628/repos?page=6&per_page=10'

###Timeouts

You can tell requests to stop waiting for a response after a given number of seconds with the timeout parameter:

    >>> requests.get('http://github.com', timeout=0.001)
    Traceback (most recent call last):
      File "<stdin>", line 1, in <module>
    requests.exceptions.Timeout: Request timed out.

###Errors and Exception

In case of network problem (e.g. DNS failure, refused connection, etc), Requests will raise a `ConnectionError` exception.  
In the event of the rare invalid HTTP response, Requests will raise an `HTTPError` exception.  
If a request times out, a Timeout exception is raised.  
If a request exceeds the configured number of maximum redirections, a `TooManyRedirects` exception is raised.  
All exceptions that Requests explicitly raises inherit from `requests.exceptions.RequestException`.  

*References - [http://docs.python-requests.org/en/latest/](http://docs.python-requests.org/en/latest/)*

*Posted by [Sayan Chowdhury](http://www.hackerearth.com/users/sayanchowdhury/).
Follow me [@chowdhury_sayan](https://twitter.com/chowdhury_sayan). Write to me at
sayan@hackerearth.com.*
