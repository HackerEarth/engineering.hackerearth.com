---
layout: post
title: "The Realtime Server"
description: ""
category: 
tags: [Tornado, Realtime]
---
{% include JB/setup %}

- - -
This is going to be a long blog post but I promise you will find some interesting
piece of engineering here, so stay till the end.

<br>
####Problem with NowJS
I was told beforehand that I will be primarily working first on writing a realtime server
beside many other things. [Vivek Prakash](http://www.hackerearth.com/users/vivekprakash), co-founder of HackerEarth told me he had himself
written a realtime server [sometime
ago](http://engineering.hackerearth.com/2013/03/12/100000-strong) with [NowJS](https://github.com/Flotype/now), but the problem with it was that it
doesn't scale well beyond 150 online users. There was a problem in file descriptor leak and
NowJS project was abandoned in December, 2012. So there was need of some good alternative
which can handle good amount of simultaneous connections(users). I didn't have to do much research
as Vivek already researched about it. He found [Tornado](ihttp://www.tornadoweb.org/en/stable/) and [Meteor.js](http://meteor.com/) to be good
alternative. Going by order of preference and popularity I choose Tornado.

<br>
####The process
Vivek pretty much explained me how different components of code
submission works. Here is a quick explanation of it. User submits his/her code
and a AJAX request is sent to webserver which further sends submission details
to [RabbitMQ server](http://www.rabbitmq.com/)(a message broker to connect various application components).
Code checker engine(consumer of RabbitMQ here) takes out the submission details
and evaluates the code and submit result back to RabbitMQ. Nowjs server takes it
out from RabbitMQ and finally sends it to the client(browser). Below flowchart
will give you a good idea of how different components are connected.

<img src="/images/flowchart.png" />
Now my job was to replace the last part(NowJS) with Tornado.

<br>
####A basic implementation
Let's code! Now I knew tornado server must read submission results from
RabbitMQ and send it back to submission page('pages' in case user has opened
the same submission problem page in more than one tab). <br>I used [pika
module](https://github.com/pika/pika)
inside Tornado IO loop to connect with RabbitMQ and read messages from it.
A request handler from Tornado api to send and receive messages from client.
On client side I used [HTML5 WebSocket](http://www.html5rocks.com/en/tutorials/websockets/basics/)
to connect to the tornado server. This basic implementation was completed in two days.

#####Front-end
    ...
    function openWebSocketConnection() {
        var ws = new WebSocket(url);

        // On websocket open
        ws.onopen = function() {
            clientInfo = {
                'name': getName()
            }
            clientInfo = JSON.stringify(clientInfo);
            ws.send(clientInfo);
        }

        // On recieve message from server
        ws.onmessage = function(evt) {
            onReceiveMessage(JSON.parse(evt.data));
        };
    }
    ...
#####Back-end
    ...
    class RealtimeWebSocketHandler(websocket.WebSocketHandler):

        def on_message(self, message):
            '''
            Messages recieved from client
            '''
            message = json.loads(message)
            self.name = message['name']
            #listen to submission result messages from rabbitmq
            self.application.pc.add_listener(self)
            ...

        def on_close(self):
            #remove listener
            self.application.pc.remove_listener(self)

        def send(self, message):
            '''
            Send response back to client
            '''
            self.write_message(message)

    class PikaClient(object):

        def connect(self):
            '''
            Connect to RabbitMQ Sever
            '''
            ...
                
        def on_message(self, channel, method, header, body):
            '''
            Submission result messages from RabbitMQ
            '''
            message = json.loads(body)
            self.notify_listeners(message)
            ...

        def notify_listeners(self, message):
            listeners = self.listeners.copy() 

            for listener in listeners:
                #send message to specific listener
                if listener.name==message['name']:
                    listener.send(message)
    ...

<br>
####Testing locally
Everything was working as expected in modern browsers but
when I tested it on IE 7, 8, 9. This was my reaction- “IE sucks man!”.
Ofcourse, IE [doesn't support](http://caniuse.com/websockets) websocket, how it
didn't occur to me. So I was left with only one option to write a fallback
implementation in [long polling(also called comet programming)](http://en.wikipedia.org/wiki/Comet_(programming))
on both client and server side. Wait the problem is not yet solved. Cross domain requests are not supported.
HackerEarth web server and tornado are on different domains. I either have to use
[CORS](http://en.wikipedia.org/wiki/Cross-origin_resource_sharing) long polling
(only supported in major browsers but more secure) or [JSONP](http://en.wikipedia.org/wiki/JSONP)
long polling(supported in every browser but insecure). I eventually used both.
Here is a code snippet:

#####Front-end
    function connectToTornado() {
        // check for browser's websocket support
        if("WebSocket" in window) {
            openWebSocketConnection();
        }
        // fallback to cross origin requests(CORS) long polling
        else if($.support.cors || "XDomainRequest" in window) {
            openCORSLongPollingConnection();
        }
        // fallback to jsonp long polling
        else {
            // setTimeout is used to supress loading sign
            // in some old browsers.
            setTimeout(openJSONPLongPollingConnection, 1);
        }
    }

    function openCORSLongPollingConnection() {
        $.ajax({
            ...
            data: {
                'name':getName()
            },
            dataType: "json",
            success: function(data) {
                onReceiveMessage(data);
                // connect again to recieve new messages
                window.setTimeout(openCORSLongPollingConnection, 0);
            },
            ...
        });
    }

    function openJSONPLongPollingConnection() {
        $.ajax({
            ...
            data: {
                'name':getName(),
                'callback':'success' // **do not delete this line**
            },
            dataType: "jsonp", // must for jsonp
            jsonp: false,
            jsonpCallback: 'success', // **do not delete this line**
            success: function(data) {
                onReceiveMessage(data);
                // connect again to recieve new messages
                window.setTimeout(openJSONPLongPollingConnection, 0);
            },
            ...
        });
    }

#####Back-end
    class RealtimeLongPollingHandler(web.RequestHandler):

        @tornado.web.asynchronous
        def post(self):
            self.transport = 'cors'
            self.name = urllib.unquote(self.request.body.split('=')[1])
            self.application.pc.add_listener(self)

        @tornado.web.asynchronous
        def get(self):
            self.transport = 'jsonp'
            self.name = self.get_argument('name', None)
            self.callback = self.get_argument('callback', None)
            self.application.pc.add_listener(self)

        def on_connection_close(self):
            self.application.pc.remove_listener(self)

        def send(self, message):
            if(self.transport=='jsonp' and self.callback is not None):
                self.finish(self.callback+'('+json.dumps(message)+')')
            elif(self.transport=='cors'):
                self.finish(message)
            self.application.pc.remove_listener(self)

<br>
####What if the client gets disconnected?
On slow internet connections especially with browsers using long polling,
messages sometimes get lost. So I had to create a buffer to store unsent
messages and when client(browser) reconnects, the server will look into
the buffer for latest message and will send it back to client and then again it
will listen for new messages from RabbitMQ. Here is code snippet:

#####Reconnect with tornado
    function reconnectToTornado(callback) {
        // Sleep time is constant after 5 minutes
        if(errorSleepTime<300000)
            errorSleepTime *= 2;
        // callback can be websocket, cors, jsonp function
        window.setTimeout(callback, errorSleepTime);
    }
 
#####Look for unsent messages.
    ...
        newest_message = self.application.pc.unsent_messages.newest(self.name)
        if(newest_message is not None):
            self.send(newest_message)
        else
            self.application.pc.add_listener(self)
    ...

    class UnsentMessageBuffer(object):

        def __init__(self):
            #unsent messages deque with capacity of 1000 objects
            self.messages = collections.deque([], maxlen=1000)

        def add(self, message):
            self.messages.append(message)

        def remove(self, message):
            try:
                self.messages.remove(message)
            except:
                pass

        def newest(self, client_name):
            messages = copy.deepcopy(self.messages)
            newest_message = None
            for message in messages:
                if(message['name']==client_name):
                    if(newest_message is None or message['id']>newest_message['id']):
                        newest_message = message
                    self.remove(message)
            return newest_message
                
That's all! Everything was working fine(atleast on my localhost :P).

<br>
Few days ago, we tested it on development server. And after successfull
testing, it was pushed to production :)

P.S. I am an undergraduate student at IIT Roorkee. You can find me
[@LalitKhattar](https://twitter.com/LalitKhattar)

*Posted by Lalit Khattar, Summer Intern 2013 @HackerEarth*
