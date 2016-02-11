---
layout: post
title: "Sending emails to our half million and growing user community"
description: "Email infrastructure at HackerEarth."
category:
tags: [Email-Infrastructure,MySQL, MongoDB, RabbitMQ]
---
{% include JB/setup %}

At hackerearth we send emails keep our users updated on upcoming challenges and
their activities, for example, when a user successfully solves a problem,
receives test-invitation, updates on user comments. Basically whenever
it’s appropriate.

### Architecture ###
It’s takes lot of computational power to send emails in such a large quantities
synchronously. So we implemented an asynchronous architecture to send emails.

Here’s brief overview of how the architecture:

Step 1: Construct an email and save the serialized email object in database.<br/>
Step 2: Queue the metadata for later consumption.<br/>
Step 3: Consume the metadata, recreate the email object and deliver.


The diagram below shows high level architecture of emailing system. The solid
line represent the data flow between different components. The dotted line
represents the communications.
Hackerearth email infrastructure consists of mysql database, mongo database,
rabbitmq queues.


<br />
<img src="/images/Hackerearth-Blog-Email.jpg"/>
<br /><br />


### Journey Of Email ###

**Step 1 - Construct email:**

There are two different email by construct

1. Text-Plain text emails
2. Html-Emails with rich interface using html elements. These emails are made
using django templates

API used by hackerearth developers for sending email -

```python
    send_email(ctx, template, subject, from_email, html=False, async=True,
                **kwargs)
```
The above API creates Sendgrid Mail object, serializes and saves it in the
db with some additional data.

A piece of code similar to the bit shown below is used to create sendgrid
Mail object

```python

    import sendgrid

    sg = sendgrid.SendGridClient('YOUR_SENDGRID_API_KEY')

    message = sendgrid.Mail()
    message.add_to('John Doe <john@email.com>')
    message.set_subject('Example')
    message.set_html('Body')
    message.set_text('Body')
    message.set_from('Doe John <doe@email.com>')
    status, msg = sg.send(message)
```


The below models is used for storing the serialized mail object and
additional data.

```python
    class Message():
            # The actual data - a pickled sendgrid.Mail object
            message_data = models.TextField()
            when_added = models.DateTimeField(default=datetime.now, db_index=True)
```


After constructing and saving the email object in the database, metadata is
queued in the rabbitmq queues. Following section explains this in detail.

Note: send_email() API can send synchronous emails. Switch the flag ‘async’
to False to send synchronous emails. This will bypass all the asynchronous
architecture and directly delivers the emails to inbox. But this is used to
send extremely important emails, for example, infrastructure monitoring,
alarms, and for monitoring email infrastructure itself.

**Step 2 - Queue the metadata:**

Not all emails have same importance in terms of delivery time. So, we have
created multiple queues to reduce waiting time in queue for important
mails.

1. High priority queue
2. Medium priority queue
3. Low priority queue

It’s up to the application developer to decide the importance of the email
and queue it in appropriate queue.

We queue the following metadata in the queue as a json object:
```python
{‘message_id’: 123}
```

**Step 3 - Reconstruct and deliver:**

We run delivery workers, which consume the metadata from queues,
reconstruct email object and deliver it.

These workers consumes the messages from rabbitmq queues and fetches the
message object from Message model(explained in the section above),
deserializes the data to reconstruct the sendgrid Mail object.

We run different number of workers depending on the volume of emails in
each queue. 

This is where if we implement final checks which help us to make decision
whether to deliver the email or not. For example, if the email id is
blacklisted, if the emails non-zero number of receivers. This helps to
distribute the load(?)

After request is sent to sendgrid for delivering the email, these email
objects are logged into a mongodb to maintain the history of delivered
emails.

###A/B Test In Emails###

Million emails requires optimization to improve user experience. This is done
through A/B tests on emails type. We can test emails for subject and content
variations. Every user on hackearth is assigned a bucket number to ensure
emails are consistent during the experiment .
Every A/B experiment is defined as dictionary mapped constants with all the
information.

Here is one example of an A/B test with subject variation.

```python
"""
    EMAIL_VERSION_A_B
    format of writing A/B test
    key: test_email_type_version_number
    value: email_dict


    format for email_dict
    keys: tuple(user_buckets)
    values: category, subject, template
"""

EMAIL_VERSION_A_B = {
                     'A_B_TEST_1': {
                     tuple(user_bucket_numbers):{'a_b_category': 'email_category_v1',
                                                 'subject': 'Hello hackerearth',
                                                 'template': 'emails/email.html'
                                                },
                     tuple(user_bucket_numbers):{'a_b_category': 'email_category_v2',
                                                 'subject': 'Welcome hackerearth',
                                                 'template': 'emails/email.html'
                                                }
                     }}
```


New Experiments must update ```EMAIL_VERSION_A_B``` with experiment data.
Information from ```EMAIL_VERSION_A_B``` is used to update the
key word arguments of hackerearth sending email API(send_email).
The category is propagated to update the category of sendgrid Mail object.
Categories are used to see the variations in open rate and click rate for different A/B
experiments.



Feel free to comment below or ping us at [support@hackerearth.com](mailto:support@hackerearth.com) if you have any suggestions!

*Posted by [Kaushik Kumar](https://www.hackerearth.com/@kkaushik24).*

*Thanks to [Pradeepkumar Gayam](http://hck.re/in3xes/) for improving it*
