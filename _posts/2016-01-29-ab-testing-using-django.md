---
layout: post
title: "A/B testing using Django"
description: "How to implement a basic A/B testing framework in Django."
category:
tags: [Django, A/B Testing, decorators]
---
{% include JB/setup %}

Whenever we roll out an improvement on our platform, we at HackerEarth love to conduct A/B tests on the improvement to understand which iteration helps our users more in using the platform in a better way. Since the available third party libraries did not quite meet our needs, we wrote our own A/B testing framework in Django. In this post we will share a few insights as to how we accomplished this.

### The basics ###

A lot of products, especially on web, use a method called **A/B testing** or *split testing* to quantify how well a new page or layout performs as compared to the old one. The crux of the method is to show layout ‘A’ to a certain set or bucket of users and layout ‘B’ to another set of users. The next step is to track user actions leading to certain milestones, which would provide critical data regarding the ‘effectiveness’ of both the pages or layouts.

Before we began writing code for the framework, we made a list of all the things that we wanted the framework to do -

 - *Route users to multiple views (with different templates)*
 - *Route users to a single view with different templates*
 - *Make the views/templates stick for users*
 - *A/B test visitors who do not have an account on HackerEarth (anonymous users)* 
 - *Sticky views/templates for anonymous users as well*
 - *Support for A/A/B or A/B/C/D…./n/ testing (just for the heck of it!)*
 - *Analytics*

We went out to grab some pizza and beer, and when we got back we came up with this wire-frame -
  
<br />
### A/B for Views ###
<img src="/images/A_B_Views.png"/>
<br /><br /><br /><br />
### A/B for Templates ###
<img src="/images/A_B_Templates.png"/>
<br /><br />


### Getting the logic right ###

To begin with, we had to categorize our users into buckets. So all our users were assigned a bucket number ranging from 1 to 120. This numbering is not strict and the range can be arbitrary or as per your needs. Next, we defined two constants - the first one specifies which view a user is routed to, and the second one specifies the fallback or primary view. The tuples in the first constant are the bucket numbers assigned to users. The primary view in the second constant will be used when we do not want to A/B test on anonymous users. 

```python
AB_TEST = {
        tuple(xrange(1,61)): 'example_app.views.view_a',
        tuple(xrange(61,121)): 'example_app.views.view_b',
    }

AB_TEST_PRIMARY = 'example_app.views.view_a'
```

Next we wrote two decorators which we could wrap around views - one for handling views and the other for handling templates. In the first scenario, the decorator would take a dictionary of views i.e. the first constant that we defined, a primary view i.e. the second constant, and a boolean value which specifies if anonymous users should be A/B tested as well.

Here’s what the decorator essentially does for logged in users -

 - *Get the user’s bucket number*
 - *Check which view is assigned to that bucket number*
 - *Return the corresponding view*

The flow is a bit different in case of anonymous users. If we do not want to perform A/B testing on anonymous users, then we just return the primary or fallback view that we had defined earlier. However, if we want to include anonymous users in the A/B tests, we need a couple of extra things to begin with -

 - *Set a unique cookie for the user which is independent of the session*
 - *A simple and fast key-value pair storage e.g. Redis*

Once we have these things in place, here’s what we need to do -

 - *Get the user’s unique cookie*
 - *Check if a key exists in redis for that cookie value*
 - *If a key is found, get the value of the key and return it*
 - *If no key is found, choose a view randomly from the view dictionary*
 - *Set a key in redis corresponding to the user with the chosen view as value*
 - *Return the chosen view*

Now, the A/B will work perfectly for anonymous users as well. Once an anonymous user gets routed to one of the views, that view will stick for him or her.

### Let's dive into code ###

An example for the view decorator is given below -

```python
"""
Decorator to A/B test different views.
Args:
    primary_view:       Fallback view.
    anon_sticky:        Determines whether A/B testing should be performed on   
                        anonymous users as well.
    view_dict:          A dictionary of views(as string) with buckets as keys.
"""
def ab_views(
        primary_view=None,
        anon_sticky=False,
        view_dict={}):
    def decorator(f):
        @wraps(f)
        def _ab_views(request, *args, **kwargs):
            # if you want to do something with the dict returned
            # by the view, you can do it here.
            # ctx = f(request, *args, **kwargs)
            view = None
            try:
                if user_is_logged_in():
                    view = _get_view(request, f, view_dict, primary_view)
                else:
                    redis = initialize_redis_obj()
                    view = _get_view_anonymous(request, redis, f, view_dict,
                            primary_view, anon_sticky)
            except:
                view = primary_view
            view = str_to_func(view)
            return view(request, *args, **kwargs)

        def _get_view(request, f, view_dict, primary_view):
            bucket = get_user_bucket(request)
            view = get_view_for_bucket(bucket)
            return view

        def _get_view_anonymous(request, redis, f, view_dict,
                primary_view, anon_sticky):
            view = None
            if anon_sticky:
                cookie = get_cookie_from_request(request)
                if cookie:
                    view = get_value_from_redis(cookie)
                else:
                    view = random.choice(view_dict.values())
                    set_cookie_value_in_redis(cookie)
            else:
                view = primary_view
            return view

        return _ab_views
    return decorator
```
The noteworthy piece of code here is the function str*_*to*_*func(). This returns a view object from a view path (string).

```python
def str_to_func (func_string):
    func = None
    func_string_splitted = func_string.split('.')
    module_name = '.'.join(func_string_splitted[:-1])
    function_name = func_string_splitted[-1]
    module = import_module(module_name)
    if module and function_name:
        func = getattr(module, function_name)
    return func
```

We can write another decorator for A/B testing multiple templates using the same view in a similar way. Instead of passing a view dictionary, pass a template dictionary and return a template.
    
### Putting things together ###

Now, let’s assume that we have already written the ‘A’ and ‘B’ views which are to be A/B tested. Let’s call them ‘view_a’ and ‘view_b’. To get the entire thing working, we will write a new view. Let’s call this view as ‘view_ab’. We will wrap this view with one of the decorators we wrote above and create a new url to point to this new view. You may refer to the code snippet below -

```python
@ab_views(
        primary_view=AB_TEST_PRIMARY,
        anon_sticky=True,
        view_dict=AB_TEST,
        )
def view_ab(request):
    ctx = {}
    return ctx
```

*Just for the sake of convenience we require that this new view returns a dictionary.*

Finally, we need to integrate analytics into this framework so that we have quantifiable data regarding the performance or effectiveness of both the views or layouts. We decided to use mixpanel at the JavaScript end to track user behaviour on these pages. You can also use any analytics or event tracking tool out there for this purpose.

This is just one of the ways you can do A/B testing using Django. You can always take this basic framework and improve it or add new features.

*P.S. : If you want to experiment with an A/A/B or A/B/C testing, all you need to do is change the first constant that we defined i.e. AB_TEST*

Feel free to comment below or ping us at [support@hackerearth.com](mailto:support@hackerearth.com) if you have any suggestions!  

*Posted by [Arindam Mani Das](http://hck.re/arindam/).*  
*Follow me [@arindammanidas](http://twitter.com/arindammanidas)*

