---
layout: post
title: 'Realtime Django Part 5: Building a Chat application RabbitMQ and uWSGI websockets (uWSGI WebSockets)'
date: 2018-01-12T14:30:12+08:00
---

Welcome to the penultimate part of this tutorial, i have a lot of fun writing this tutorial and i hope you have.

In part 4, we were able to acheive our main goal of the tutorial which was to build a web based chat application with django and Vue. But we faced a major issue with scaling the application.

It's not a UI/UX issue that the user can notice immediately but it's really important because if we're an overnight sensation and our app becomes very popular with lot of users talking to their friends It's going to become really slow and before you know it our server will start dropping requests that it can't handle which would leave a very bad impression of us an engineers. So what can we do?

### WebSockets
From the last tutorial, we touched a little on WebSockets basically we know that they're bi-directional channels that stay open and allow the server to communicate instantly with the client and vice-versa. So how do we get this cool stuff into our django application.

Django itself was developed at a different time in our internet history. Back then Websites/Web applications weren't as sophisticated as what we know today. It was basically "Hey i want the articles from the 1st of January 2005" and then the server goes "Don't worry bro I got you" and does some work fetching the articles and then responds "Here are the articles you asked for" and then it goes back to sleep or attend to other users (Closes the socket). You couldn't receive information without asking for it. For example the Web Server couldn't say "Someone in Lagos sent you a message here is the message". You'll need to ask the Web server. "Please let me see all my unread messages" first.

Enough of this annoying examples :-) what we can pull from this is that the communication could only be initiated from one person (The client).

This was really a problem for Django and a lot of python web frameworks because the underlying protocol that governs how python applications should run `WSGI` was tied down to this `request-response` type of communication. Django developers feared the worst because NodeJS hipsters were rubbing it in their faces that django was bad for realtime systems. Something had to be done and it had to be done fast. 

So many people approached the problem from different perspectives. By new frameworks (Torando), Async engines (Gevent and Co), adding support to existing WSGI Servers (`uWSGI`). but nobody tried to modify django directly because it was going to require some drastic changes in the core of django and the `WSGI` protocol itself until someone found a workaround

### django channels

Andrew Goodwin brought websockets to django natively with django-channels. As we speak now it's currently an official product of the django foundation. Meaning it's not going anywhere soon.

Django channels is good but how was he able to build it while maintaining django's compatibility with `WSGI` servers and without changing the core. Well....He compromised.

A new protocol called `ASGI` had to be developed for django-channels It's distinctly different from `WSGI` which means that if you decide to go with django-channels for websockets for an old django application you can say bye bye to your `WSGI` Servers like gunicorn, meinheld and uWSGI any optimizations you made in the past to speed up your application would be lost. django-channels comes with it's own server called `Daphne` It's not that bad but it's relatively young compared to other `WSGI` Servers.

Also using `django-channels` in your existing application glues your application together and makes it harder to scale out to multiple machines. Because you now have one application serving two different purposes. i.e Normal requests and WebSocket requests.
This makes server monitoring tricky and could also result in a performance bottleneck for normal http requests. 

This doesn't mean django-channels is bad. It's actually the future for building realtime systems with django and it's backed by the django software foundation itself. Infact, if you're starting a new django application from scratch and you know you're going to need realtime behaviour i'll advice you to take a look at it.

For existing applications, it can be a real pain to integrate it and performance is going to drop a little when you switch from your existing `WSGI` Server to Daphne.


### uWSGI WebSockets

un-bit (The developers of uWSGI) took a different approach, they decided to integrate WebSockets into the uWSGI Core itslf. uWSGI is a very performant WSGI Web server for python (The Core is written in C).

So this means that you don't need to change your stack to add WebSocket support to Django. Even if you use a different `WSGI` Server like `gunicorn` u just need to `pip install uwsgi` it's that simple.

If you remember the discussion we had earlier in part 3 about RabbitMQ. Remember i told you RabbitMQ is the glue between `uWSGI` and `django`.

We need to create notifications and put them on the RabbitMQ Queue and then through the websockets this messages can be broadcasted directly to thousands of clients effectively.

To ease the process of creating notifications and sending them to RabbitMQ, i've created a third party django library called django-notifs. Go ahead and install it from pip with:

{% highlight python %}
pip install django-notifs
{% endhighlight %}

Make sure you add to your `INSTALLED_APPS`:

{% highlight bash %}
INSTALLED_APPS = (
    'django.contrib.auth',
    ...,
    'rest_framework',
    'rest_framework.authtoken',
    'djoser',

    # Our apps
    'chat',
    'notifications'
)
{% endhighlight %}

Installing `django-notifs` install pika which is a python library for connecting to RabbitMQ (Since RabbitMQ is written in Erlang not python).

Run the migrations with `python manage.py migrate`

Finally we're going to install `RabbitMQ` The instructions vary for different operating systems, so head on to the website and find out how to install it for your machine.


Now that we have our packages installed (Make sure the `RabbitMQ` server is running, if not you'll get errors) let's integrate notifications into our applications. With the inbuilt signals from `django-notifs` it's very straightforward

open `views.py` and update the `ChatSessionMessageView` view:

{% highlight python %}
from notifications.signals import notify


class ChatSessionMessageView(APIView):
    ...

    def post(self, request, *args, **kwargs):
        """create a new message in a chat session."""
        uri = kwargs['uri']
        message = request.data['message']

        user = request.user
        chat_session = ChatSession.objects.get(uri=uri)

        chat_session_message = ChatSessionMessage.objects.create(
            user=user, chat_session=chat_session, message=message
        )

        notif_args = {
            'source': user,
            'source_display_name': user.get_full_name(),
            'category': 'chat', 'action': 'Sent',
            'obj': chat_session_message.id,
            'short_description': 'You a new message', 'silent': True,
            'extra_data': {'uri': chat_session.uri}
        }
        notify.send(sender=self.__class__, **notif_args)

        return Response ({
            'status': 'SUCCESS', 'uri': chat_session.uri, 'message': message,
            'user': deserialize_user(user)
        })
{% endhighlight %}

Just before we return a response to the user, we send the notify signal with arguments, Most of the arguments are self explanatory.

The `silent` parameter means the notification won't be persisted to the database. We're using django-notifs as an event emitter we can also pass arbitrary notification data in the `extra_data` argument as a dictionary.


### Notification channels

django-notifs makes use of `channels` to deliver messages. This means that you can write your own custom channel to deliver messages via emails, SMS(Via twilio for example), infact anything you can think of. It even comes with an inbuilt websocket channel but that won't suffice for our case because it's a user to user channel. But we want to broadcast information to multiple users at the same. This pattern of communication is called Pub/Sub (Publish Subscribe) and RabbitMQ has support for this: time which is supported by RabbitMQ.

We'll have to create a RabbitMQ `exchange`, an `exchange` is a channel that receives messages from a producer (Our application) and then broadcasts it to multiple queues. There are four different types of `exchanges` namely direct, topic, headers and fanout. We'll focus on using the `fanout` exchange it's the simplest to understand and fits our use case perfectly.

This is an illustration from the RabbitMQ docs on how the fanout exchange works:

![RabbitMQ Fanout](https://www.rabbitmq.com/img/tutorials/python-three-overall.png)

For more info about the different exchanges and how they can be used check out the RabbitMQ docs


To implement the Pub/Sub pattern we'll need to write our own delivery channel.

It's quite simple. Create a new file called `channels.py`

{% highlight python %}
"""Notification channels for django-notifs."""

from json import dumps

import pika

from notifications.channels import BaseNotificationChannel


class BroadCastWebSocketChannel(BaseNotificationChannel):
    """Fanout notification for RabbitMQ."""

    def _connect(self):
        """Connect to the RabbitMQ server."""
        connection = pika.BlockingConnection(
            pika.ConnectionParameters(host='localhost')
        )
        channel = connection.channel()

        return connection, channel

    def construct_message(self):
        """Construct the message to be sent."""
        extra_data = self.notification_kwargs['extra_data']

        return dumps(extra_data['message'])

    def notify(self, message):
        """put the message of the RabbitMQ queue."""
        connection, channel = self._connect()

        uri = self.notification_kwargs['extra_data']['uri']

        channel.exchange_declare(exchange=uri, exchange_type='fanout')
        channel.basic_publish(exchange=uri, routing_key='', body=message)

        connection.close()
{% endhighlight %}

We set the exchange name as the `uri` of the chat session.

We also dumped the chat message as a dictionary. We'll need all the details about the message on the client side not just the actual message.

Try and send a message and it should create a new RabbitMQ exhange based on the uri for the chat session.

To see the list of exchanges we have (for *nix systems):

{% highlight bash %}
rabbitmqctl list_exchanges
Listing exchanges
amq.match	headers
amq.direct	direct
amq.rabbitmq.log	topic
amq.rabbitmq.trace	topic
amq.topic	topic
	direct
amq.fanout	fanout
amq.headers	headers
fe662fd9de834fc	fanout  # our Exchange
{% endhighlight %}

You can see a lot of exchanges that RabbitMQ uses internally but there's our exchange at the bottom.

Now we're going to dynamically create queues and bind them to the exchange we created earlier so they can receive messages.

Create a new file called `websocket.py`.

{% highlight python %}
"""Receive messages over from RabbitMQ and send them over the websocket."""

import pika


connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
)
channel = connection.channel()

channel.exchange_declare(
    exchange='fe662fd9de834fc', exchange_type='fanout'
)

# exclusive means the queue should be deleted once the connection is closed
result = channel.queue_declare(exclusive=True)
queue_name = result.method.queue  # random queue name generated by RabbitMQ

channel.queue_bind(exchange='fe662fd9de834fc', queue=queue_name)

print('listening for messages...')

while True:
    for method_frame, _, body in channel.consume(queue_name):
        try:
            print(body)
        except OSError as error:
            print(error)
        else:
            # acknowledge the message
            channel.basic_ack(method_frame.delivery_tag)
{% endhighlight %}

Once again we connect to `RabbitMQ` using `pika` and declare the exchange again.

Declaring an exchange (or queue) multiple times has no adverse effects, If the exchange didn't exist beforehand `RabbitMQ` creates it if it does it simply does nothing.

The most important line in this file now is:

`channel.queue_bind(exchange='fe662fd9de834fc', queue=queue_name)`

This binds the queue to the exchange. It's more like "Hey exchange, i'm interested in the messages you receive. Please send them to me."

The `queue_name` is generated randomly by `RabbitMQ` because we called `queue_declare` without passing in a name.

There are different ways of consuming messages off a channel. You can use callbacks or consume manually with a `for loop` we went with the second option. We also have some unnecessary exception handling at this point but don't worry you'll see why

`channel.basic_ack(method_frame.delivery_tag)` acknowledges that the client succesfully received the message and it can be removed from the queue. Without acknowledging the message, it would remain on the queue.

For more info about method frames and the different types of frames check out the RabbitMQ docs.


Go ahead and start the run the `websocket` file with:

{% highlight bash %}
$ python websocket.py
listening for messages...
{% endhighlight %}

Now go back to the chat UI and send some messages. I sent `"hello world"` and `"how are you doing"` and this was the output.

{% highlight bash %}
listening for messages...
b'{"user": {"id": 1, "username": "danidee", "email": "", "first_name": "", "last_name": ""}, "message": "Hello world"}'
b'{"user": {"id": 12, "username": "daniel", "email": "", "first_name": "", "last_name": ""}, "message": "How are you doing"}'
{% endhighlight %}

Open up a new terminal and run the websocket file. You should be able to see new messages as they come in ***REAL TIME***. You can run more instances of the websocket file and they should all print incoming messages in their terminals.

Nice work! Now there's only one thing left sending the messages directly to the user.


### Where is the websocket?
The websockets are included in the core `uwsgi` python object. First of all install `uWSGI` if you haven't already.

{% highlight python %}
$ pip install uwsgi
{% endhighlight %}

We're going to make some modifications to the `websocket.py` file

{% highlight python %}
"""Receive messages over from RabbitMQ and send them over the websocket."""

import sys

import pika
import uwsgi


def application(env, start_response):
    """Setup the Websocket Server and read messages off the queue."""
    connection = pika.BlockingConnection(
    pika.ConnectionParameters(host='localhost')
    )
    channel = connection.channel()

    exchange = env['PATH_INFO'].replace('/', '')

    channel.exchange_declare(
        exchange=exchange, exchange_type='fanout'
    )

    # exclusive means the queue should be deleted once the connection is closed
    result = channel.queue_declare(exclusive=True)
    queue_name = result.method.queue  # random queue name generated by RabbitMQ

    channel.queue_bind(exchange=exchange, queue=queue_name)

    uwsgi.websocket_handshake(
        env['HTTP_SEC_WEBSOCKET_KEY'],
        env.get('HTTP_ORIGIN', '')
    )

    def keepalive():
        """Keep the websocket connection alive (called every 30 seconds)."""
        print('PING/PONG...')
        try:
            uwsgi.websocket_recv_nb()
            connection.add_timeout(30, keepalive)
        except OSError as error:
            print(error)
            sys.exit(1)  # Kill process and force uWSGI to Respawn

    keepalive()

    while True:
        for method_frame, _, body in channel.consume(queue_name):
            try:
                uwsgi.websocket_send(body)
            except OSError as error:
                print(error)
                sys.exit(1)  # Force uWSGI to Respawn
            else:
                # acknowledge the message
                channel.basic_ack(method_frame.delivery_tag)
{% endhighlight %}

The `uwsgi` websocket api is very simple. We're using just three methods

<ul>
  <li>
    <strong>uwsgi.websocket_handshake:</strong> The handshake is the bridge from HTTP to WS protocol. In the handshake This method establishes tries to connect the client and the server together, if it fails for any reason an exception would be raised.
  </li>

  <li>
    <strong>uwsgi.websocket_recv_nb:</strong> This method is actually deceptive and misleading (no pun intended) because the full name is `websocket receive non blocking` but it doesn't only receive messages in a non-blocking manner it also helps to maintain the connection with the client by sending a `ping/pong`.

    The keep alive function calls this method every 30 seconds, without that the client might disconnect the connection (Typically after a minute of inactivity)
  </li>

  <li>
    <strong>uwsgi.websocket_send:</strong> You don't need a sooth-sayer to tell you about this one :-) Though the reason we need the error handler, is in case the connection is closed and we try to send a message `uwsgi.websocket_send` would raise an `OSError` and we'll kill the process and `uWSGI` will take care of restarting it for us. Also, the `else` block would never run which means we won't acknowledge the message and it would stay on the queue.
    
    The next time we enter the for loop and call `channel.consume` would send the unacknowledged message plus any new messages in the queue. Which means we'll never miss any message due to network connectivity.
  </li>
</ul>

<br />

Did you notice that the exchange `uri` is no longer hardcoded? Instead we get the exchange name from the connection URL which would require our clients to connect to a URL like this:

`http://websocket-server/<uri>`

Don't worry if this doesn't make any sense to you when we finally connect our Vue frontend to the WebSocket server it'll clear a lot of things up.


### Connecting to the WebSocket with JavaScript

It's impossible to talk about WebSockets in the context of a web application without JavaScript. Most modern browsers already support WebSockets so we don't need to install any library.

{% highlight html %}
<script>

const $ = window.jQuery

export default {
  ...

  created () {
    this.username = sessionStorage.getItem('username')

    // Setup headers for all requests
    $.ajaxSetup({
      headers: {
        'Authorization': `Token ${sessionStorage.getItem('authToken')}`
      }
    })

    if (this.$route.params.uri) {
      this.joinChatSession()
    }

    this.connectToWebSocket()
  },

  methods: {
    ...

    postMessage (event) {
      const data = {message: this.message}

      $.post(`http://localhost:8000/api/chats/${this.$route.params.uri}/messages/`, data, (data) => {
        this.message = '' // clear the message after sending
      })
      .fail((response) => {
        alert(response.responseText)
      })
    },

    joinChatSession () {
      ...
    },

    fetchChatSessionHistory () {
     ...
    },

    connectToWebSocket () {
      const websocket = new WebSocket(`ws://localhost:8081/${this.$route.params.uri}`)
      websocket.onopen = this.onOpen
      websocket.onclose = this.onClose
      websocket.onmessage = this.onMessage
      websocket.onerror = this.onError
    },

    onOpen (event) {
      console.log('Connection opened.', event.data)
    },

    onClose (event) {
      console.log('Connection closed.', event.data)

      // Try and Reconnect after five seconds
      setTimeout(this.connectToWebSocket, 5000)
    },

    onMessage (event) {
      const message = JSON.parse(event.data)
      this.messages.push(message)
    },

    onError (event) {
      alert('An error occured:', event.data)
    }
  }
}
</script>
{% endhighlight %}

Now start the `uWSGI` Server on port 8081

{% highlight bash %}
$ uwsgi --http :8081 --module websocket --master --processes 4
{% endhighlight %}


Hooray! Hooray! Hooray!

There's still a problem. open 3 more tabs (5 active clients). The last client wouldn't be able to connect because the 4 processes we specified are tied down by other clients.

*NOTE: That's a reason why we need to kill stuck processes immediately with `sys.exit(1)` because the process may not really be stuck, the user might have intentionally left the chat room and it would take uWSGI sometime to figure out that the client has disconnected before closing the connection on the server.

The --master option invokes the master process which monitors dead processes and restarts them without this, the process would die and never be restarted and it would continue until the last process dies and uWSGI exits.
*

so why is this happening? i thought you said WebSockets scale?


### I thought you said WebSockets Scale

Yes i said it in my head and wrote it yes. The reason our current setup won't scale is simply because of the way we're running `uWSGI`; with processes and threads like a normal python `WSGI` server runs.

An Easy solution would be to increase the processes but that can only take us far probably a few hundred users or less depending on your server's resources. Surely there has to be a better way


### Asynchronous IO and Concurrency

Asynchronous IO deserves an entire article on it's own and if you don't have the slightest idea about what it is i'll advice you to go crazy on google and read every article you can find.

But basically the idea behind `AsyncIO` or simply `async` is when we have multiple IO bound tasks that we need to run during the said IO Operation (In our case, sending and receiving messages), instead of the process sitting idle waiting for the process to complete it can quickly switch to another IO bound task and so on.

This switching is relatively quicker than process context switching because everything is run in one single process so there's no copying of variables/xxxx synchronization etc etc explain more...xxxx No time is spent waiting idly.

This Simple process is what makes `NodeJS` excel so much at IO bound applications because at it's core it's Asynchronous you don't need to install any special libraries or learn anything new.

This is not the case for python, When Python was developed most computers had only one processor so really NodeJS is benefitted from being a more modern language. But Thankfully we're not left out in the dust as python programmers if you want to bring async to python they're several libraries (even an official one `asyncio`) to help you out. Though i must admit it's not as natural and smooth as writing async code in JavaScript.

`uWSGI` itself supports various concurrency models `asyncio` being one of them but we're going to use `gevent`.

From my experience i found `gevent` works better compared to `asyncio` and `greenlet`. `Gevent` also has a lot of methods like `monkey-patching` (Which is really useful) and `gevent.select` for waiting for IO Events.

Install it from pip

{% highlight bash %}
$ pip install gevent
{% endhighlight %}

Now you just need to start the `uWSGI` WebSocket Server like this:

{% highlight bash %}
$ uwsgi --http :8081 --gevent 2 --module websocket --gevent-monkey-patch --master
{% endhighlight %}

First we're starting with 2 gevent threads and a single process, that means we can only handle two clients reasonably (it flunctuates between 3 and 4 clients randomly but most times only 2 clients receive messages reliably), when more clients try to join in you'll get a warning in the `uWSGI` saying:

{% highlight bash %}
async queue is full !!!
{% endhighlight %}

There are two ways you can scale up, the easiest way is to increase the number of gevent threads (Actually greenlet) that are run. If we change the `uWSGI` startup code to this

{% highlight bash %}
$ uwsgi --http :8081 --gevent 100 --module websocket --gevent-monkey-patch --master
{% endhighlight %}

Boom! just like that we can handle 100 concurrent users.

What about having multiple processes?

That's the second way of scaling up, you can have multiple processes let's start the `uWSGI` server with 4 processes

{% highlight bash %}
$ uwsgi --http :8081 --gevent 100 --module websocket --gevent-monkey-patch --master --processes 4
{% endhighlight %}

4 processes * 100 --gevent threads that's 400 Concurrent users already!

Depending on your server's specs and configuration you can increase the number of processes and gevent threads but before doing this make sure you profile and monitor your application's performance because at a certain stage increasing numbers will cause degraded performance.

`uWSGI` features a python package (installable from pip) called `uwsgitop` that can be used for monitoring it. But that's out of the scope of this tutorial. Maybe i'll write about it in the future


### Scaling out to multiple servers

At a certain point, we'll max out our server's resources and we'll need to scale out to multiple servers. Since our websocket server isn't coupled to our application. That's fairly easy we can easily load balance multiple servers (Each running multiple `uWSGI` processes and gevent threads) behind Nginx.

And before you know it, you'll be handling thousands of connections easily.

You can also apply the same clustering and loadbalancing technique to RabbitMQ when you need to scale out (though not with Nginx since RabbitMQ doesn't use HTTP).


Apart from our main aim of building a chat application, one of my silent goals was to show you that `uWSGI` is not "just a WSGI server" it's way more than that.

It has a lot of goodies which can be overwhelming if you're just getting started. It even features a standalone server mode which means it doesn't need to be run behind Webservers like Apache and Nginx. (For small web applications i'll never recommend it for a service with high traffic).

<br />

Well well well, that's all for this tutorial i hope you've accquired some new skills as you tagged along.

As a bonus, i'm going to make one last part that'll briefly cover a lot of things and improvements i couldn't talk about in other parts for the sake of not derailing the tutorial. below are the things i'll cover in the next part:

#### Frontend

Automatically scrolling to the bottom of the screen when messages exceed the window height.

Creating a config file to store variables.

#### Backend

JSON web tokens with `djsoer`.

<br>

Thanks for reading and see you there!
