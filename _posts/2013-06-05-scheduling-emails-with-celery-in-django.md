---
layout: post
title: "Scheduling emails with celery in Django"
description: ""
category: 
tags: [Celery, Scheduling tasks, Scheduling emails, Periodic tasks, Crontab, Django, RabbitMQ]
---
{% include JB/setup %}

After a long journey with Django, you come to a place where you feel the need to
get some tasks done asynchronously without any supervision of human. Some tasks
need to be scheduled to run once at a particular time or after some time and
some tasks have to be run periodically like
[crontab](http://en.wikipedia.org/wiki/Cron). One of the tasks is
sending emails on specific triggers.

Here at [HackerEarth](http://www.hackerearth.com/)
, one of the major chunk of emails is sent to recruiters and participants after 
a contest is finished or when participant triggers finish-test button. 
Till now we had done this using crontab. But things have changed now and
scaling with such process is time and resource consuming.
Also, looking in to database if there is any task that has to be done with crontab process is not a
good method, atleast for those tasks those have to run only once in the
lifetime.

<br>
####Django-Celery
[Django-Celery](http://docs.celeryproject.org/en/latest/django/index.html)
comes to the rescue here.
Celery gets tasks done asynchronously and also supports scheduling of tasks as well.
Integrating Celery with Django codebase is easy enough, you just need to have some
patience and go through the steps given in the official Celery site.
There are two sides in Celery technology: Broker & Worker. Celery requires a
solution to send and receive messages, usually this comes in the form of a
separate service called a message broker. We use the default broker 
[RabbitMQ](http://www.rabbitmq.com/) to get this done. 
Worker fetches the tasks from the queue at time at which they were scheduled 
to run asynchronously.
You will have to download celery init scripts to run the worker as daemon on
Production. You can get those init scripts from 
[GitHub](https://github.com/celery/celery/tree/master/extra/generic-init.d)
<br>
This is the configuration we used to run celery in our project:
<br>

    # Name of nodes to start
    CELERYD_NODES="w1 w2 w3"

    # Where to chdir at start.
    CELERYD_CHDIR="/hackerearth/"

    # How to call "manage.py celeryd_multi"
    CELERYD_MULTI="$CELERYD_CHDIR/manage.py celeryd_multi"

    # How to call "manage.py celeryctl"
    CELERYCTL="$CELERYD_CHDIR/manage.py celeryctl --settings=settings.hackerearth_settings"

    # Extra arguments to celeryd
    CELERYD_OPTS="--time-limit=300 --concurrency=8"

    # %n will be replaced with the nodename.
    CELERYD_LOG_FILE="/var/log/celery/%n.log"
    CELERYD_PID_FILE="/var/run/celery/%n.pid"

    # Workers should run as an unprivileged user.
    CELERYD_USER="hackerearth"
    CELERYD_GROUP="hackerearth"

    # Name of the projects settings module.
    export DJANGO_SETTINGS_MODULE="settings.hackerearth_settings"

<br>
####Another Problem
After linking triggers to send emails after the contest time is finished or the
participant has finished the test prematurely, all things were working
properly. Now I could easily schedule a task to run asynchronously 
at any time. But I met a problem that there is no method to check if a
particular task has already been scheduled that is assosiated with some Model instance.
This happens when there are more than one triggers for the same task, and it
can easily happen in a fairly complicated system.
To get this done I had to store the task_id with that model instance into
database using generic ContentType. So here is the hack that I
came up with:

<br>
**Generic ModelTask**

This model stores the information of the scheduled task(task_id, name)
and the information of the Model instance to which the task is assossiated.
{% highlight python %}
        from django.contrib.contenttypes import generic
        from django.contrib.contenttypes.models import ContentType
        from django.db import models

        class ModelTask(models.Model):
            """
            For storing all scheduled tasks
            """
            task_id = models.CharField(max_length = 36)
            name = models.CharField(max_length = 200)
            content_type = models.ForeignKey(ContentType)
            object_id = models.PositiveIntegerField()
            content_object = generic.GenericForeignKey('content_type', 'object_id')

            def __unicode__(self):
                return "%s - %s" % (self.name, self.content_object)

            @staticmethod
            def create(async_result, instance):
                return ModelTask.objects.create(task_id=async_result.task_id,
                        name=async_result.task_name, content_object=instance)

            @staticmethod
            def filter(task, instance):
                content_type = ContentType.objects.get_for_model(instance)
                object_id = instance.id
                return ModelTask.objects.filter(content_type=content_type,
                        object_id=object_id, name=task.name)
{% endhighlight %}
<br>
**A custom overridden task decorator 'model_task'**

Overrides the methods : 'apply_async' & 'AsyncResult'
And attaches a new method : 'exists_for'
{% highlight python %}
        import types

        from django.db import models
        from celery import task
        
        from appname.models import ModelTask

        def model_task(*args, **kwargs):
            def dec(func):
                task_dec = task(*args, **kwargs)
                task_instance = task_dec(func)

                def exists_for(self, instance):
                    return ModelTask.filter(self,instance).exists()
                task_instance.exists_for = types.MethodType(exists_for, task_instance)

                def apply_async(self, *args, **kwargs):
                    instance = kwargs.pop('instance',None)
                    async_result = super(type(self), self).apply_async(*args, **kwargs)
                    if instance and not self.exists_for(instance):
                        ModelTask.create(async_result, instance)
                    return async_result
                task_instance.apply_async = types.MethodType(apply_async, task_instance)

                def AsyncResult(self, *args, **kwargs):
                    if args and isinstance(args[0], models.Model) and\
                            self.exists_for(args[0]):
                        task_id = ModelTask.filter(self, args[0])[0].task_id
                        return super(type(self), self).AsyncResult(task_id)
                    else:
                        return super(type(self), self).AsyncResult(*args, **kwargs)
                task_instance.AsyncResult = types.MethodType(AsyncResult, task_instance)

                return task_instance
            return dec
{% endhighlight %}
That's it.

<br>
####The Use Case

**Participation Model**
    
This model contains the information of a User participating in a Event.
{% highlight python %}
        Class Participation(models.Model):
            user = models.ForeignKey(User)
            event = models.ForiegnKey(Event)
            ...
            ...
{% endhighlight %}

<br>
Task for sending email to participant
{% highlight python %}
        @model_task()
        def send_email_on_participation_complete(participation):
            code for sending email
            ...
            ...
{% endhighlight %}            

<br>
Scheduling the task
{% highlight python %}
        duration = calculate_duration_in_seconds(participation)

        # The extra keyword argument 'instance' is necessary as it will create a 
        # ModelTask object.
        send_email_on_participation_complete.apply_async((participation,),
                countdown=duration, instance=participation)
{% endhighlight %}

<br>
Check if the task has already been scheduled assossiated with a participation object
{% highlight python %}
        is_scheduled_before = send_email_on_participation_complete.exists_for(participation)
{% endhighlight %}

<br>
Get the AsyncResult object
{% highlight python %}
        # Returns the async_result object of the scheduled task that is assossiated
        # with given Model instance (participation in our case)
        async_result = send_email_on_participation_complete.AsyncResult(participation)

        # gives the status of the scheduled task : PENDING/STARTED/SUCCESS/FAILURE
        aync_result.status

        # Contains the return value of the task (None in our case)
        async_result.result
 {% endhighlight %}

<br>
All this replaced the cron jobs, custom scripts and some manual tasks with a
robust task (email) scheduling mechanism. This also lays the foundation for
triggering many other types of tasks on top of django-celery architecture set
up by me. And this will certainly make us more efficient and help us to focus
on other core products, while tasks are performed asynchronously and we can
enjoy the awesome weather on a fine day! :)

P.S. I am an undergraduate student at IIT Roorkee.You can reach out to me at
shubham@hackerearth.com for any suggestion, bug or improvement.
You can also find me 
[@ShubhamJain](http://in.linkedin.com/pub/shubham-jain/54/4a/931/).

*Posted by Shubham Jain, Summer Intern 2013 @HackerEarth*
