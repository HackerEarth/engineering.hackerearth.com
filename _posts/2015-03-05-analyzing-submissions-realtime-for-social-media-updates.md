---
layout: post
title: "Analyzing submissions realtime for social media updates"
description: "Story: How we made a realtime app to automate our marketing"
category:
tags: [Databases, Marketing, Tweets]
---
{% include JB/setup %}

### Objective ###
In Jan 2015, HackerEarth conducted close to 10-12 hiring challenges per month, 5-6 coding challenges and numerous college challenges.
HackerEarth has a decent social media presence and we want to inform our followers about the events on hackerearth.
One of the main objectives of this project was to provide flexibility for marketing team to focus on sophisticated campaigns and automate their simple jobs.

### Design Goals ###
As a first step, We decided to post about our events and its highlights on twitter.
We covered event reminders, start/end of contests, who scored first AC and leaderboard updates at the end of a contest.
We chose to do it by reading from biggest, meanest table of our database, Submissions.

### Challenges ###
Submissions table is a very large table. An additional query on submissions table during peak time was not favourable. Hence, We did not count the submissions in-place and instead queued them to be processed later.

Preventing duplicate tweets by maintaining state is also a challenge.


### Solution ###
The application made high volume of reads, few writes/updates. So any key/value store will do the job. We chose Redis in lieu of Memcached.

- Redis offers data persistence in the event of node failure. This is very useful to avoid duplicate tweets. For instance, Two different users being credited for first AC submission in an event.
- By setting key expiry time and less number of keys for single event, We prevented our redis server from being overloaded.
- We maintained a key in redis to keep count of submissions for an event. Reading from database was not recommended because it will make a read call per submission during peak time and Redis performed faster reads.

The application is an asynchronous worker and payload containing submission\_id and event\_id are passed to it using Kafka.
So when the redis key counter hit the magic numbers (1, 100, 500, multiples of 1000), The worker makes a db query and posts a tweet.
<br/><br/>
Worker subscribes to a kafka broker on the submission topic to receive payloads regarding it.
Here is the code of the worker.

```python
class ConsumePostTweets(KafkaConsumer):
    def __init__(self):
        routing_key = KafkaConsumer.KAFKA_SUBMISSION_TOPIC
        self.redis_cache = get_redis(
            settings.REDIS_DATA_CONNECTION_URL)
        super(ConsumePostTweets, self).__init__(routing_key)

    def on_message(self, body):
        message = json.loads(body)
        submission_id = message.get('submission_id')
        if submission_id is None:
            autocommit_transaction()
            return
        try:
            message = json.loads(body)
            message.update({"redis_cache": self.redis_cache})
            process_tweets(**message)
            autocommit_transaction()
        except Exception, e:
            log_tags = ["tweets", "queue", "consumer", str(submission_id)]
            tb = traceback.format_exc()
            silo.log(tb, tags=log_tags, type="LiveTweeter")
            autocommit_transaction()
```

We also run a cron job to post event reminders.

```python
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
```

As number of challenges in a time slot increased, We needed to lower number of tweets in a time interval. We queued our tweet payloads with a delay and posted it in intervals.

Here is a screenshot of the app in production.


<img src="/images/tweets_bot_image.png"/>

### Epilogue ###
Finally, We wrapped it as a valentine's day gift for marketing team and they loved us more since that day.

*Send an email to support@hackerearth.com for any bugs or suggestions.*
*Posted by [Sreeram Boyapati](https://www.hackerearth.com/@sreeram.boyapati2011)*
