---
layout: post
title: "Introducing CodePlayer"
description: ""
category: 
tags: [HackerEarth, Ace Editor, Django]
---
{% include JB/setup %}

Ever thought of sharing solution of a coding problem in form of a video with someone, to teach them how you implemented the solution. Ofcourse, you have thought about it. But how many good online tools are there to do such thing?

Today, we are announcing the release of HackerEarth's CodePlayer that does the job.

**tl;dr**
Watch a demo code video.

<br>
#### How it works ####
CodePlayer is tightly integrated with code editor in [HackerEarth
site](http://www.hackerearth.com), that is, wherever you will find code editor in our site, there CodePlayer will also be
activated. By choice, we have built it to activate automatically in stealth mode on page load or first keystroke.

Our code editor is built on top of [Ace Editor](http://ace.c9.io). In ace editor, keystrokes or deltas can be captured and applied 
programmatically using [Ace API](http://ace.c9.io/#nav=api). This is where the idea of playing code like a video seemed possible :)

CodePlayer was built in the following line of development:-
* [Setup video](#setupvideo).
* [Recording keystrokes](#record).
* [Playing video](#play).

<br>
#### <a name="setupvideo" style="color:#333333">Setup Video</a>
After the code editor is loaded, an AJAX POST request is made to server to setup video info in Django model.

    /**
     * This function sends ajax request to setup video.
     * This code is made generic to integrate it with code editor anywhere in site.
     * @arg video_obj: contains video info
     * @arg callback: called on success
     * @arg callback_arg_obj: above callback argument 
     */
    function setup_video(video_obj, callback, callback_arg_obj) {
        $.ajax({
            url: '*****',
            type: 'POST',
            data: video_obj,
            dataType: 'json',
            callback: callback,
            callback_arg_obj: callback_arg_obj,
            success: function(response_obj) {
                this.callback(response_obj, this.callback_arg_obj);
            },  
            error: function(err) {
            }   
        }); 
    };
<br>
Video model*(Backend)*

    class CodeVideo(Generic):
        final_code = models.TextField(null=True) # used as thumbnail
        lang = models.CharField(max_length=10)
        last_updated = models.DateTimeField(null=True)
        owner_id = models.PositiveIntegerField(null=True)
        uuid = UUIDField(auto=True) # unique id of video

<br>
#### <a name="record" style="color:#333333">Recording</a>
One of the difficult task in recording keystrokes is to reduce no. of web requests and database insert queries. On an average 
there are 1k keystrokes in a single instance of code editor. Now, for only 100 users on site, there will be 1000 * 100 web 
requests and db insert queries.

Although, [our web servers are capable of handling these many
requests](http://engineering.hackerearth.com/2013/11/21/scaling-python-django-application-apache-mod_wsgi/) but we didn't want to waste resources. So we used batch 
requests in which keystrokes are first grouped locally(in Javascript) and then sent to web servers in batches. Below is the JS code that enqueues keystroke/changeset.

    /**
     * Enqueues changeset in changeset queue.
     */
    this.enqueue_changeset = function(delta, source, timestamp) {
        if(!delta)
            return false;

        var changeset = {
            delta: delta,
            source: source,
            timestamp: new Date().getTime()
        };
        this.changeset_queue.push(changeset);

        return true;
    };

In backend, instead of multiple insert queries, we are using Django's model api
[bulk_create
method](https://docs.djangoproject.com/en/1.4/ref/models/querysets/#django.db.models.query.QuerySet.bulk_create)
for batch insert.

Now, one question is still unanswered. At what intervals, these batch requests are sent? Actually, there is no fixed interval. A 
batch request is sent after an inactivity period of 3 seconds in editor(i.e. user has stopped typing for atleast 3 seconds), AND 
there are still keystrokes left to be sent.

'All changes saved' animation on top-right corner of editor confirms that a batch request is sent successfully.

Often people don't write code continuously. There can be large intervals between successive coding sessions which means video 
length will increase drastically. To tackle this problem, we added a CodeSession model.

    class CodeSession():
        code_video = models.ForeignKey(CodeVideo)
        initial_code = models.TextField()
        # sesson start time 
        start = models.DateTimeField()
        # sesson end time 
        end = models.DateTimeField()


Now, total video length is calculated using formula-
**&Sigma;(session_end_time<sub>i</sub> - session_start_time<sub>i</sub>)**.

Time interval between successive sessions is atleast 2 minutes.

<br>
####<a name="play" style="color:#333333">Playing Video</a>

We decided to launch 'recording' functionality before 'playing' so that we can generate some data and test the scalability of 
the system. And now we had the data, it was a matter of playing it using Ace API.

Each delta/changeset has a timestamp associated with it which is converted to video time according to video length. All these 
deltas are [scheduled](https://developer.mozilla.org/en/docs/Web/API/window.setTimeout) to be applied programmatically(using [applyDeltas](http://ace.c9.io/#nav=api&api=document)) 
according to their video time. This basically, is the play functionality.

    /**
     * Applies deltas/changesets, slides seekbar
     */
    var play_timeout = function(changeset) {
        return function() {
            // Apply delta
            if(changeset.delta) {
                var delta = changeset.delta;
                editor.moveCursorToPosition(delta.range.start);
                var doc = new Document(editor.getValue());
                doc.applyDeltas([delta]);
                editor.setValue(doc.getValue(), 1);
                if(delta.action=='removeText')
                    editor.moveCursorToPosition(delta.range.start);
                else
                    editor.moveCursorToPosition(delta.range.end);
                video_state['cursor_position'] = delta.range.end;
            }
            // Save video state
            video_state['session_index'] = changeset['session_index'];
            video_state['changeset_index'] = changeset['changeset_index'];
            video_state['time'] = changeset['video_time'];
            video_state['code'] = editor.getValue();
            // Slide seekbar to last applied delta time
            seekbar.slider('value', changeset.video_time);
        }
    };


To pause video at any point, all scheduled timeouts are [cleared](https://developer.mozilla.org/en-US/docs/Web/API/window.clearTimeout).

    /**
     * Pauses video.
     */
    this.pause = function() {
        // Clear all previously scheduled play timeouts.
        for(var i=0; i<play_timeout_ids.length; i++) {
            clearTimeout(play_timeout_ids[i]);
        }
        play_timeout_ids = [];
        video_playing = false;
        var play_id = this.player_elements['play_id'];
        show_play_menu_button(play_id);
    };

Changing speed of video was a piece of cake. Say user clicks on 5x, divide timestamp(t) by 5 i.e. t=t/5.

    /**
     * Time after which deltas will be applied.
     */
    var play_after = function(video_time) {
        // Realtive video time
        var r_video_time = video_time-video_time_copy;
        // Convert to milliseconds
        r_video_time *= 1000;
        // Divide by play speed
        r_video_time /= play_speed;
        return r_video_time;
    };


<br>
####Make your own code video
I know you are excited to try out our CodePlayer. Go to
<http://code.hackerearth.com> and start writing code. As soon as the default
code is changed, you will see a 'Replay Code' button. Click on it to watch the
video.


PS: I worked on this project during my winter internship at HackerEarth. I also worked there
as a summer intern. Read my summer internship experience
[here](http://blog.hackerearth.com/2013/07/my-summer-internship-at-hackerearth.html).

PPS: I will be joining the folks at HackerEarth full time after
graduation(expected 2014 :P).

*Posted by Lalit Khattar, Winter Intern 2013. Follow me
[@LalitKhattar](http://twitter.com/LalitKhattar)*

