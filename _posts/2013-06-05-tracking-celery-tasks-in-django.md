---
layout: post
title: "Tracking Celery tasks in Django"
description: ""
category: 
tags: [Celery, Scheduling tasks, Periodic tasks, Django]
---
{% include JB/setup %}

Don't you have popcorn and coke? Go get it first!

After a long journey with Django, you come to a place where you feel a need to
get some tasks done asynchronously without any supervision of human. Some tasks
need to be scheduled to run once at a particular time or after some time and
some tasks have to be run periodically like crontab.

Here at [HackerEarth](http://www.hackerearth.com/)
, we send emails to recruiters and participants after 
a contest is finished or when participant triggers finish-test button. 
And only solution we had to get this done was crontab. But things have changed now. 
Looking in to database if there is any task that has to be done with crontab process is not a
good method, atleast for those tasks those have to run only once in the
lifetime.

<br>
####Solution
Tanaaaaa.... Here is the solution to get all these tasks done : 
[Django-Celery](http://docs.celeryproject.org/en/latest/django/index.html).
Integrating Celery with your Django codebase is easy enough, you just need to have some
patience and go through the documentation given in the official Celery site.
You will have to download celery init scripts to run the worker as daemon on
Production. You can get those init scripts from 
[GitHub](https://github.com/celery/celery/tree/master/extra/generic-init.d)
<br>
And you are done !!

<br>
####Another Problem
All things are working properly, you can schedule a task to run asynchronously 
at any time. But I met a problem that there is no method to check if a
particular task has already been scheduled that is assosiated with some Model instance.
To get this done you'll have to store the task_id with that model instance into database.
Thanks Django that there is generic ContentType. So here is the hack that I
came up with:

<br>
- Generic ModelTask

This model stores the information of the scheduled task(task_id, name)
and the information of the Model instance to which the task is assossiated.

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

<br>
- A custom overridden task decorator 'model_task'

Overrides the methods : 'apply_async' & 'AsyncResult'
<br>
And attaches a new method : 'exists_for'

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

That's it.

<br>
####Here is how we use it

- Participation Model
    
This model contains the information of a User participating in a Event.
    
        Class Participation(models.Model):
            user = models.ForeignKey(User)
            event = models.ForiegnKey(Event)
            ...
            ...

<br>
- Task for sending email to participant

        @model_task()
        def send_email_on_participation_complete(participation):
            code for sending email
            ...
            ...

<br>
- Schedule the task
    
        duration = calculate_duration_in_seconds(participation)
        send_email_on_participation_complete.apply_async((participation,),
                countdown=duration, instance=participation)

        # The extra keyword argument 'instance' is necessary as it will create a 
        # ModelTask object.

<br>
- Check if the task has already been scheduled assossiated with a participation object

        boolean_value = send_email_on_participation_complete.exists_for(participation)

<br>
- Get the AsyncResult object

        async_result = send_email_on_participation_complete.AsyncResult(participation)
        # Returns the async_result object of the scheduled task that is assossiated
        # with given Model instance (participation in our case)

        aync_result.status
        # gives the status of the scheduled task : PENDING/STARTED/SUCCESS/FAILURE

        async_result.result
        # Contains the return value of the task (None in our case)

<br>
P.S. I am an undergraduate student at IIT Roorkee. Reach out to me at
shubham@hackerearth.com for any suggestion, bug or improvement.
You can find me 
[@ShubhamJain](http://in.linkedin.com/pub/shubham-jain/54/4a/931/).

*Posted by Shubham Jain, Summer Intern 2013 @HackerEarth*
