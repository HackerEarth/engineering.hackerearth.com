---
layout: post
title: "Search @ HackerEarth"
description: ""
category:
tags:
---
{% include JB/setup %}

Even if a website is extremely user friendly but there is no search box, many users fail to understand the site’s navigation or hierarchy. As the whole web is based on search, it becomes one of the most important features of any website both in terms of user design and technology.

At [HackerEarth](www.hackerearth.com) three things which are most likely to be searched for, are:- Users, Problems and Challenges. What if you want to find details of a problem, challenge or users but because of their  large number fail to do so.

So I was given this task of implementing search at HackerEarth. I used Haystack with Elasticsearch Search Engine and mainly went through 3 phases.

- Basic settings of Haystack

- Simple search for users, problems and challenges

- Search relevance

####Basic settings of Haystack and Elasticsearch

Haystack provides modular search for Django. It features a unified, familiar API that allows to plug in different search backends (such as Solr, Elasticsearch, Whoosh, Xapian, etc.) without having to modify your code.
In the settings file:

    INSTALLED_APPS=(
        
       ‘haystack’,
        …,
    )

    HAYSTACK_CONNECTIONS = {

       'default': {
           'ENGINE': 'haystack.backends.elasticsearch_backend.ElasticsearchSearchEngine',
           'URL': 'http://127.0.0.1:9200/', #changes according to your configuration
           'INDEX_NAME': 'haystack',
        },

    }

// This realtime signal processor enable signals. So even if anyone is changing models indices #are updated automatically without anyone having to call update_index

    HAYSTACK_SIGNAL_PROCESSOR = 'haystack.signals.RealtimeSignalProcessor'

####Simple Search
#####Creating Indexes

As I was making global search so I has made different search app and created search_indexes.py there but it can also be created in the app whose model fields have to be indexed or searched.

search_indexes.py

from haystack import indexes

//a lot more imported here

    class UserProfileIndex(indexes.SearchIndex, indexes.Indexable):

        text = indexes.CharField(document=True, use_template=True)
        name = indexes.NgramField(model_attr='get_human_name', boost=1.125)
        username = indexes.NgramField()
        ..
        …
//it contains indexes for other models also
    
        #####python manage.py rebuild_index --settings=’settings file’

#####creates indexes
As I wanted users to be searched by name, username and email

and similarly by multiple fields in challenges and problems also. So I made use_template=True and edited one file search_users_text.txt

         {{ object.name }}
         {{ object.username }}
         {{ object.email }}

This makes a document containing all the fields which have to be searched for and this is the content of text field which is actually searched.


#####Partial Search And Multiple Models

It doesn’t make sense if the search gives results which exactly match with what we search for. Obviously I can’t remember names, usernames or descriptions. So here came the idea of using autocomplete. The autocomplete properly tokenizes the string and checks its existence in all the documents.

Also one more challenge was to search in documents of different models which had fields which had to be prioritised differently in search. I had to narrow the search as search_indexes file were made for a lot of other apps also.

#####SearchQuerySet().autocomplete(kwargs).models(UserProfile, ….)

The other challenge was that filtering should be in such a way that results are OR of different fields but each field should contain AND of all tokens of the query.

So I wrote my custom autocomplete for this:

    def autocomplete(self, **kwargs):

       """

        Must be run against fields that are either ``NgramField`` or

       ``EdgeNgramField``.

       """

       import re

       # Include OR operator results in different fields and AND

       # operator for query bits in one field

       clone = self._clone()

       #query_bits = []

       q = None

       for field_name, query in kwargs.items():

           s = None

           if field_name is "email":

               list_tokens = query.split('@')

           else:

               list_tokens = re.split(';|,|\*|\n| ', query)

           for word in list_tokens:

               if word != '':

                   bit = clone.query.clean(word.strip())

                   kwargs = {

                       field_name: bit,

                   }

                   if s:

                           s = s & SQ(**kwargs)

                   else:

                           s = SQ(**kwargs)

                   #query_bits.append(SQ(**kwargs))

           if q:

               q  = q | s

           else:

               q = s

       # return clone.filter(reduce(operator.__or__, query_bits))

       if q:

               return clone.filter(q)

       else:

               return clone.filter(SQ())

Searching first in name and then usernames and email only when its valid

I was asked to make search in such a way that firstly results that have names matched with the query are shown and then usernames are shown. So I gave this boost attribute to name which makes it more important in search results.

#####name = indexes.NgramField(model_attr='get_human_name', boost=1.125)

Also email should be shown when its an exact match or its valid. So I validate email first

and then send it for search only when its a valid email.

####Search Relevance

The most challenging part of search is bringing relevance to it. What is the use of search if it cannot give the results which are not relevant to the user.

Here are the kind of relevancies I worked upon:-

1. The people you are following should be above in the results.

2. The more you have visited a user, problem or challenge, the higher it should be in your results.

3. The more popular a result is overall, the higher it should be in the results.

4. Time sensitive results. For eg:- if a challenge is going on now or is about to happen in a few days and you are searching for something which is similar to its name then it should be at higher position in the results.

5. Related term frequency:- In all the results there are some terms which occur with the main term (main term which actually had a match with the query), these are called related terms. So I find all the results with this query match and also find top 10 related terms in the results and find their relative frequency. The results should be ordered according to the frequency of the related terms.:w

