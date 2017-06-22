---
layout: post
title: "Integrating partner APIs with minimal effort"
description: "Comprehensive guide on how to integrate multiple API providers with minimal repetitive effort"
category:
tags: [API, REST, Python, Django, Tastypie]
---
{% include JB/setup %}

Problem
--------
HackerEarth being an assessment platform, naturally needs to be accessible from third party ATS vendors. To integrate into the these vendors we have to implement the API documented by each one of them. But each vendor has a different API structure, the data format is different (JSON, XML, HTML etc.), the data structure is different. That means we have to implement for each of the vendors from scratch. This is time consuming and also lot more things to maintain. The simpler approach would be writing a single universal set of API that covers all use cases and also somehow can cater to different data format and structures.

For simplicity, in this post we will take example of only one endpoint i.e. inviting a candidate to a test.

To invite a candidate through HackerEarth's Recruit API -

{% highlight python %}
CLIENT_ID ="xxxxxx.api.hackerearth.com"
CLIENT_SECRET = "xxxxxxxxxxxxxx"
TEST_ID = 53

payload = {
    'client_id': CLIENT_ID,
    'client_secret': CLIENT_SECRET,
    'test_id': TEST_ID,
    'emails': ['foo@bar.com', 'alice@bob.com', 'yet@another.email'],
    'send_email': False,
    'extra_parameters': {
        'candidate_ids': {
            'foo@bar.com': 1234522,
            'alice@bob.com': 'alpha123',
            'yet@another.email': 'letters'
        },
        'candidate_names': {
            'foo@bar.com': 'Foo Bar',
            'alice@bob.com': 'Alice Bob'
        },
        'redirect_urls': {
            'foo@bar.com': 'http://www.redirecturl.com/',
            'alice@bob.com': 'http://www.redirecturl2.com/'
        },
        'report_callback_urls': {
            'foo@bar.com': 'http://www.callbackurl.com/',
            'alice@bob.com': 'http://www.callbackurl2.com/'
        }
    }
}
requests.post("https://api.hackerearth.com/recruiter/v1/tests/invite/", data=json.dumps(payload))
{% endhighlight %}

To do the same request through Greenhouse ATS -

{% highlight python %}
auth = HTTPBasicAuth(username=<greenhouse_api_key>, password=<greenhouse_api_secret>)
headers = {'Content-Type': 'application/json'}
payload = json.dumps({
    "partner_test_id": 53,
    "candidate": {
        "first_name": "Foo",
        "last_name": "Bar",
        "email": "foo@bar.com",
    }
})
requests.post("api.hackerearth.com/greenhouse/v1/events/invite/", data=payload, headers=headers, auth=auth)

{% endhighlight %}

Here two different APIs are doing the same thing but the interface is same. But these two were implemented separately. This means double the code double the effort to maintain.

Solution
---------
We divided the whole flow into independent components, and picked what could be re-used for e.g. authentication/authorization, serialization/deserialization, data computation etc., and made interfaces for components that can not be re-used - so that those could be implemented for each of the ATS vendors.

<img style="display: block; margin: 0 auto;" src="/images/partner_api_diagram.png" alt="Architecture" title="Depiction of the Partner API architecture"/>

Using this architecture, notice how the code remains same except the url.

HackerEarth API

{% highlight python %}
CLIENT_ID ="xxxxxx.api.hackerearth.com"
CLIENT_SECRET = "xxxxxxxxxxxxxx"
TEST_ID = 53
payload = {
    'client_id': CLIENT_ID,
    'client_secret': CLIENT_SECRET,
    'test_id': TEST_ID,
    'emails': ['foo@bar.com', 'alice@bob.com', 'yet@another.email'],
    'send_email': False,
    'extra_parameters': {
        'candidate_ids': {
            'foo@bar.com': 1234522,
            'alice@bob.com': 'alpha123',
            'yet@another.email': 'letters'
        },
        'candidate_names': {
            'foo@bar.com': 'Foo Bar',
            'alice@bob.com': 'Alice Bob'
        },
        'redirect_urls': {
            'foo@bar.com': 'http://www.redirecturl.com/',
            'alice@bob.com': 'http://www.redirecturl2.com/'
        },
        'report_callback_urls': {
            'foo@bar.com': 'http://www.callbackurl.com/',
            'alice@bob.com': 'http://www.callbackurl2.com/'
        }
    }
}
requests.post("https://api.hackerearth.com/partner/hackerearth/events/invite/", data=json.dumps(payload))
{% endhighlight %}

Through Greenhouse

{% highlight python %}
auth = HTTPBasicAuth(username=<greenhouse_api_key>, password=<greenhouse_api_secret>)
headers = {'Content-Type': 'application/json'}
payload = json.dumps({
    "partner_test_id": 53,
    "candidate": {
        "first_name": "Foo",
        "last_name": "Bar",
        "email": "foo@bar.com",
    }
})
requests.post("api.hackerearth.com/partner/greenhouse/v1/events/invite/", data=payload, headers=headers, auth=auth)
{% endhighlight %}

Now, these two calls although look quite different, are being served through same code. You may be wondering how the two requests which are significantly different, are being processed by the same API on backend. Here is how each request is processed at each step -

Deserialize
------------
Deserialization is the process of converting data received over the network into a format consumable by the system. The deserialization is done by a serializer class, which is selected according to the partner's request datatype. This information is stored in the database alongwith the other configuration options like response datatype, authentication info etc.

{% highlight python %}
# Get RecruitApiPartner object from request.
partner_name = kwargs['partner_name']
In first case partner_name will hackerearth and in second case it will be greenhouse.
#Get partner object from db.
partner = RecruitApiPartner.objects.get(partner_name=partner_name)
The partner object stores all the configuration settings related to each partner i.e. request/response datatype, authentication info etc.
# Get request datatype from partner object
request_datatype = partner.request_datatype
# Choose serializer based on request datatype
serializer = get_serilizer(request_datatype)
# Call deserializer
derserialized_data = serializer.deserialize(request_data)
{% endhighlight %}

Authenticate request
----------------------
Current method of authentication for present partners is api key/secret with pair being present either in the Authentication header or the request body. The partner object has two relevant attributes - auth_type and auth_mapping. auth_type indicates where to find the key/secret pair i.e. either header or request body. auth_mapping stores name of field for auth key and secret. For e.g.

{% highlight python %}
auth_type = 'form'
auth_mapping = {'api_key': 'client_id', 'api_secret': 'client_secret'}
{% endhighlight %}
This auth_mapping means that the api_key is present in the client_id field in request body and similarly api_secret in client_secret field of request body. Once we get the api_key and api_secret, this data is passed to a common authentication class.

{% highlight python %}
# Authenticate request
is_authenticated = self._meta.authentication.is_authenticated(
      request, deserialized_req_data, partner.auth_type, partner.auth_mapping)
if not is_authenticated:
    raise ImmediateHttpResponse(
        http.HttpUnauthorized(
            content='Unauthenticated', content_type='plain/text'))
{% endhighlight %}

Decode
-------
Decoding is the process of converting deserialized request data to a common format understandable by the internal service. Internal service is the internal API which does the actual work. The decoder is implemented as a method of Codec class which inherits from BaseClass.

{% highlight python %}
# Codec interface
class BaseCodec(object):
    def _get_encoder(self, api_name):
        """Get encoder callable."""
        encoder_map = {
            'create_invite': self._create_invite_encoder,
        }
        encoder = encoder_map[api_name]
        return encoder

    def _get_decoder(self, api_name):
        """Get decoder callable."""
        decoder_map = {
            'create_invite': self._create_invite_decoder,
        }
        decoder = decoder_map[api_name]
        return decoder

    def encode(self, api_name, data):
        encoder = self._get_encoder(api_name)
        return encoder(data)

    def decode(self, api_name, data, headers):
        decoder = self._get_decoder(api_name)
        return decoder(data, headers)
{% endhighlight %}

BaseCodec is implemented for each partner. In this case for HackerEarth and Greenhouse, both will implement _create_invite_decoder and _create_invite_encoder. These methods will receive the deserialized request data and convert that data into a dictionary with keys that are understandable by the service. This way the service can operate on uniform inputs without having knowledge about different partners.

{% highlight python %}
# Decode data
CodecClass = get_codec_class(partner_name)
codec_obj = CodecClass()
headers = request.META
apiname = 'create_invite'
decoded_request_data = codec_obj.decode(apiname, deserialized_req_data, headers)
{% endhighlight %}

Call internal service
-----------------------
Internal service is the common functionality moved into an API that is only accessible internally. It operates on uniform input without any knowledge of any partners. This way we don't have to duplicate the functionality for each partner. In this case, the service for create_invite api will be called with decoded request data as input.

{% highlight python %}
# Call the internal service to do the actual work of inviting candidate
apiname = 'create_invite'
service = get_service(apiname)
res_data = service(**{'request_data': decoded_request_data})
{% endhighlight %}

Encode
-------
The response from service is in internal format. During the encoding process the labels are changed to what is expected by the partner. For e.g. after calling the service to invite a candidate, HackerEarth API expects following response -

{% highlight json %}
{
  "candidates_already_completed": [],
  "emessage": [],
  "invites_sent_count": 2,
  "mcode": "SUCCESS",
  "invalid_emails": [],
  "message": "Request successful",
  "extra_parameters": {
      "invite_urls": {
          "test@test.com": "https://www.hackerearth.com/your-test-name-here/?login=6fc3af861erb3f5f85baf1bb031ad07b",
          "foo@bar.com": "https://www.hackerearth.com/your-test-name-here/?login=a313381c02458298196219bba2d9e04e"
       }
    },
  "candidates_already_invited": [],
  "ecode": []
}
{% endhighlight %}

While Greenhouse expects following response
{% highlight json %}
{
    "partner_interview_id": <unique session id>
}
{% endhighlight %}

It is the encoder's job to convert the response from service to this data format. Encoder is part of the Codec class which has been explained in the Decoder section. So, the code to encode looks like -

{% highlight python %}
# Convert response data into external format
encoded_res_data = codec_obj.encode(apiname, res_data)
{% endhighlight %}

Serialize
----------
Serializer converts the encoded data into a format suitable for transmitting over the network. The format depends upon the partner. In this case HackerEarth and Greenhouse accept JSON, so a JSON serializer does the work, however for some other partner it may be XML or YAML etc.

{% highlight python %}
# Serialize encoded data
res_datatype = partner.res_datatype
serializer = get_serializer(res_datatype)
serialized_res_data = serializer.serialize(encoded_res_data)
{% endhighlight %}

Conclusion
-----------
Using this structure we were able to integrate any third party partner without implementing an API from scratch. Now the whole process was reduced to making an entry into the database, implementing a codec, and a serializer(in case serializer of the datatype is not already implemented, which is going to be a small set).
