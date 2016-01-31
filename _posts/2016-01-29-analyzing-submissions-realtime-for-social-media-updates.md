---
layout: post
title: "Analyzing submissons realtime for social media updates"
description: "Story: How we made a realtime app to automate our marketing"
category:
tags: [Databases, Marketing, Tweets]
---
{% include JB/setup %}

### Objective ###
During Jan 2015, HackerEarth conducted close to 10-12 hiring challenges per month, 5-6 coding challenges and numerous college challenges.
HackerEarth has a decent social media presence and we want to inform our followers about the events on hackerearth.
One of the main objectives was to provide flexibility for marketing team to focus on sophisticated campaigns and automate their simple jobs.

### Design Goals ###
As a first step, We decided to post about our events and its highlights on twitter. Twitter is compartively a very developer-friendly platform, provides APIs to post tweets programmatically and does not treat them any different from user tweets.
We covered event reminders, start/end of contests, who scored first blood (Oh I mean first AC) and leaderboard updates at the end of a contest.
We chose to do it by reading from biggest, meanest table of our database, Submissions.

### Challenges ###
Here is some background on Submissions - <br/>

- It has a casual few million rows and little more than 40 fields, So each row is close to 1KB.<br/>
- Migrations take hours and are a nightmare for any developer (We save money for our crash-the-site fine jar) <br/>
- It is one of the most active tables with high burst rate of writes/reads during any contest. Coincidentally, That is when we want to post tweets. Any minor mishap and
It's return of the deadlocks.

Preventing duplicate tweets by maintaining state is also a challenge. We cannot tweet about first AC submission in an event multiple times.
The pitfall is a poorly coded app will fill the redis with large number of stale keys and raise memory errors, high cpu usage alerts causing the redis to break down (Been there done that). 

### Solution ###
The application made high volume of reads, few writes/updates. So any key/value store will do the job. We chose redis.
By setting key expiry time and less number of keys for single event, We prevented our redis server from crashing.
<br/>
We maintained a key in redis to keep count of submissions for an event. Reading from database was not recommended because it will make a read call per submission during peak time and redis allowed faster reads with lesser network costs than mysql.

The application is an asynchronus worker and payload containing submission id, event slug are passed to it using rabbitmq.
So when the redis key counter hit the magic numbers (1, 100, 500, multiples of 1000), We make a db query and post a tweet.
We also run a cron job to post event reminders.

    def post_challenge_reminder():
        """Task to post challenge reminders.
        """
        now = datetime.now()
        later_1_hour = now + timedelta(hours=1)
        events = Event.objects.filter(
            Q(start__gt=now) & Q(start__lte=later_1_hour))
        events = [event for event in events if is_tweet_allowed_for_event(event)]
        for event in events:
            tweet_grammar = random.choice(grammar.CHALLENGE_REMINDER_FEEDS)
            post_tweet(tweet_grammar, event)

We also used some checks to make sure we are not tweeting about a private event or about a non-live event.

    def is_tweet_allowed(event):
        """Check if tweet is allowed for this event.
        """
        if settings.DEBUG:
            return True

        if event.is_college_event and (event.open or event.only_invited):
            return False

        if not event.start or not event.end:
            return False

        now = datetime.datetime.now()
        if (not event.private and event.published and not event.trashed
                and not event.hidden and
                event.start <= now and event.end >= now):
            return True

        return False


As hackerearth grew, We needed to lower number of tweets in a time interval. We queued our tweet payloads with a delay and posted it in intervals.

### Epilogue ###
Finally, We wrapped it as a valentine's day gift for marketing team and they loved us more since that day.

This was my second intern project a year ago and I would like to thank Virendra who made sure I realized my initial design and code was crappy, and helped me transition into a better engineer.

*Send an email to support@hackerearth.com for any bugs or suggestions.*  
*Posted by [Sreeram Boyapati](https://www.hackerearth.com/@sreeram.boyapati2011)*
