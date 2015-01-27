---
layout: post
title: "Making django-threadedcomments more powerful"
description: "Comments have become an integral part of our website. They are integrated
almost everywhere-challenge, practice problem page etc. and soon will be added to few more pages"
category:
tags: [Django, threadedcomments, Realtime, Pusher]
---
{% include JB/setup %}

In this post, I am going to briefly descibe the challenges we faced while building a powerful
commenting system.


Comments have become an integral part of our website. They are integrated almost everywhere - 
challenge, practice problem page etc. and soon will be added to our new products. We have
been working on adding new features to it for past 2-3 months to make it more usable.
Here is what we did:-

* [Pluggable architecture](#pluggable)
* [Ajaxifying comments](#ajaxify)
* [Realtime sync](#realtime)
* [Tagging people](#tagging)

### From the beginning</a> ###
Our commenting system is built using an open source django app, named [django-threadedcomments](https://github.com/HonzaKral/django-threadedcomments).
In threadedcomments, commenters can reply both to the original item, and reply to other comments as well.
This open source app best suited our requirements in terms of UX, hence we decided to use it. But later we realised
it was not powerful enough, and we decided to add our own features in it.

### <a name="pluggable" style="color:#333333">Pluggable architecture</a> ###
We added django-threadedcommnets in a pluggable architecture form. This lets us integrate it anywhere on our
website easily without writing the same code again and again.

Below is the snippet which we can include in any django template to add comments.

    <div id="comments-{{"{{model.get_content_type.id"}}}}-{{"{{model.id"}}}}" class="pagelet-inview standard-margin comments" ajax="{{"{% url render_comments model.id model.get_content_type.id"}} %}" target="comments-{{"{{model.get_content_type.id"}}}}-{{"{{model.id"}}}}"></div>

Above single line of code renders complete comment list(including reply form) using [bigpipe](https://www.facebook.com/notes/facebook-engineering/bigpipe-pipelining-web-pages-for-high-performance/389414033919). Isn't it cool?

One more reason I am calling our commenting system pluggable is that we can easily override most of the comments display logic,
show comments differently on different pages. For example, comments appearing on a [note](http://www.hackerearth.com/notes/getting-started-with-the-sport-of-programming/) and [problem](http://www.hackerearth.com/problem/algorithm/big-p-and-math-15/) page needs to be shown differently
based on the logic 'who is the moderator of the content'. This couldn't have been possible without django template [block tag](https://docs.djangoproject.com/en/1.7/ref/templates/builtins/#block).

*comments/base/list.html*

{% highlight html %}
<!-- This blocks handles logic to determine if user is a normal user or a moderator -->
{{"{% block userextrainfo"}} %}
<!-- Override this block in your app -->
{{"{% endblock"}} %}
{% endhighlight %}

*comments/problem/list.html*

{% highlight html %}
{{"{% block userextrainfo"}} %}
<!-- A moderator can be problem owner, tester, editorialist or event admin -->
{{"{% endblock"}} %}
{% endhighlight %}

*comments/notes/list.html*

{% highlight html %}
{{"{% block userextrainfo"}} %}
<!-- A moderator can be note owner -->
{{"{% endblock"}} %}
{% endhighlight %}

### <a name="ajaxify" style="color:#333333">Ajaxifying comments</a> ###
This was the most challenging task because of the way django-threadedcomments is built. Special thanks to
[Virendra](http://hck.re/virendra) for taking the initiative and finding an easy to implement solution.

Posting a comment via AJAX request was realtively easy compared to deleting it because of comment's threaded nature.
Whenever a comment is deleted, we first determine if that comment has atleast a single child which is not deleted.
Based on that logic we decide the format in which deleted comment will be shown to user. If you didn't understand a word
of what I wrote above, look at the images below.

*Initial comments*

<img src="/images/comments.png"/>

*After deleting comment 2, child comment 2.1 should be visible*

<img src="/images/comment_parent_deleted.png"/>

*After deleting comment 2.1, delete complete tree*

<img src="/images/comment_child_deleted.png"/>


We implemented BFS algorithm to handle all the scenarios and corner cases.

{% highlight python %}
class ThreadedComments(Comment):
  """
  ThreadedComment model
  """

  def _child_exists(self):
    """
    Returns boolean
    Implemets BFS to check if comment obj has
    atleast one child which is not removed.
    Uses cache to avoid using BFS everytime.
    """
    key = self.get_child_exists_key()
    is_child = cache.get(key, None)
    if is_child is None:
        queue = deque()
        queue.append(self)
        while len(queue):
            comment = queue.popleft()
            children_exists, childs = self.get_child_exists_and_childs(comment)
            if children_exists:
                is_child = True
                break
            else:
                for child in childs:
                    queue.append(child)

        if is_child is None:
            is_child = False

        cache.set(key, is_child, CACHE_TIME_MONTH)
    return is_child
  child_exists = property(_child_exists)
{% endhighlight %}

### <a name="realtime" style="color:#333333">Realtime sync</a> ###
After ajaxifying comments, we decide to put the cherry on top. Making comments appear in realtime
was not easy at all. First, our [realtime server](http://engineering.hackerearth.com/2013/05/31/the-robust-realtime-server/) was not scaling very well due to alot of traffic
these days. So we started looking for alternatives. [Pusher](http://pusher.com) seemed to be a reliable option untill our
own realtime server is up and running.

Below is generic python code for pushing data to pusher via rabbitmq:

{% highlight python %}
class PusherClient(BaseClient):
    def __init__(self):
        routing_key = PUSHER_ROUTING_KEY
        retry(super(PusherClient, self).__init__, routing_key)

    def call(self, message):
        retry(super(PusherClient, self)._call, message)

class PusherWorker(ConsumeQueue):
    """
    Push data to pusher service
    """

    def on_message(self, body):
        message = json.loads(body)
        channel = message.get('channel', None)
        event = message.get('event', None)
        data = message.get('data', '')
        if channel is not None and event is not None:
            pusher_instance = get_pusher_instance()
            # Socket id to exclude
            if data:
                socket_id = data.get('socket_id', None)      
            else:                                            
                socket_id = None
            if socket_id:
                pusher_instance[channel].trigger(event, data,
socket_id)
            else:
                pusher_instance[channel].trigger(event, data)
{% endhighlight %}

Pusher is great for broadcasting messages in realtime but it has some drawbacks. It doesn't have a
scalable presence system, means it's difficult to store more than 100 clients info on their servers.
Thus making it difficult to write complex logic on client side.

Javascript code to post/delete comment
{% highlight javascript %}
function subscribeComment(channel_name) {
    var pusher = get_pusher_instance();
    if(pusher) {
      var channel = pusher.subscribe(channel_name);
      channel.bind('comment_added', function(data) {
        var comment_html = data.html;
        var parent_comment_id = data.parent_id;
        addComment(parent_comment_id, comment_html);
        /* Some hacks to decide whether to keep reply, PM, delete link */
      });
      channel.bind('comment_removed', function(data) {
        // Comment id to be delete
        var comment_id = data.comment_id;
        var has_child = data.has_child;
        deleteComment(comment_id, has_child);
      });
    }
}
{% endhighlight %}

### <a name="tagging" style="color:#333333">Tagging people</a> ###
It does exactly what it says, that means you can now tag people in comments and they will be notified by email.
Checkout the screenshot below.

*Tagging people using @*

<img src="/images/comment_tagging.png"/>

<br>
*Comment posted after tagging*

<img src="/images/comment_posted.png"/>

I worked on this feature in our very first internal hackathon. I tried to make it as generic as possible by binding event
handler on 'mentionable' class.

{% highlight html%}
<textarea ajax="/search/AJAX/search/search_users/" rows="10" result-div-id="search-users-dropdown" name="comment" id="id_comment" cols="40" class="mentionable"></textarea>
{% endhighlight %}

{% highlight javascript %}
$('.mentionable').live('keyup', function(e) {
    var url = $(this).attr('ajax');
    var result_div_id = $(this).attr('result-div-id');
    var name_str = 'developer'; // will be made generic
    var val = $(this).val();
    var cursorPos = $(this).prop('selectionStart');
    var result_div = $('#' + result_div_id);
    val = val.substr(0, cursorPos);

    $.ajax({
       url: url,
       type: 'GET',
       data: {'q': q},
       id: $.now(),
    }).done(function(data, method) {
       if(method==='success') {
           var r_time = this.id;
       } else {
           var r_time = $.now();
       }
       var data_time = result_div.attr('timestamp');
       if(data_time===undefined || data_time<r_time) {
           var html = $.trim(data.html);
           result_div.html(html);
           if(html.length>0) {
               result_div.show();
           }
           result_div.attr('timestamp', r_time);
       }
    }).fail(function() {
    });
});
{% endhighlight %}

In backend we are querying from graph search db.

### Till the end ###
There are still alot of improvements like UI changes etc. in pipeline which will be executed soon.
If you have some suggestions, do let us know.

Hope these improvements have made comments on HackerEarth more usable.

*Posted by [Lalit Khattar](http://www.hackerearth.com/users/lalitkhattar/). Follow me
[@LalitKhattar](http://twitter.com/LalitKhattar)*
