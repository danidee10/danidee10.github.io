---
layout: post
title: 'Realtime Django Part 6: Build a Chat application RabbitMQ and uWSGI websockets (Extras)'
date: 2018-03-12T22:35:33+01:00
tags: python vue JavaScript
---

This part is going to be about improving the Chat application, I might not take my time to explain some things but if there's anything that's not clear enough; Feel free to ask a question.

## Frontend Improvements

### Implementing a loading screen

If you paid close attention, you'll notice that before we determine if we should join a chat session or not. An empty chat page is displayed (Since it's the default view of the component).

We don't want users to see this. We want to show them a beautiful "loading page" and then redirect then display the appropriate view.

To do this, we need to add a new "loading" state to our component

{% highlight JavaScript %}
data () {
  return {
    loading: true,
    messages: [],
    message: '',
    sessionStarted: false
  }
},
{% endhighlight %}

By default it's set to true which means the chat interface is still loading (In the background we're trying to join a chat session)

Next, we need to set the change modify the conditions in the template (Some content has been collapsed). We now have a new block that's displayed when the other conditions fail. `disqus.svg` is a loader that i downloaded from [loader.io](https://loader.io) You can download it from the [Github repo](https://github.com/danidee10/Chatire/blob/master/chatire-frontend/src/assets/disqus.svg). You'll see it soon.

{% highlight html %}
<template>
  <div class="container">
    <div class="row">
      <div class="col-sm-6 offset-3">

        <div v-if="!loading && sessionStarted" id="chat-container" class="card">
          ...
        </div>

        <div v-else-if="!loading && !sessionStarted">
          ...
        </div>

        <div v-else>
          <div class="loading">
            <img src="../assets/disqus.svg" />
            <h4>Loading...</h4>
          </div>
        </div>
      </div>
    </div>
  </div>
</template>
{% endhighlight %}

If you save and reload the application, you should see the loading screen (At least the "Loading..." text) forever!

![realtime django 6.1](../../../images/django/realtime-django/realtime-django-6.1.png)
<figcaption>The disqus logo is actually animated</figcaption>

We need to tell our component that the we've started a new session by setting the "loading" state to false which makes one of the previous conditions true. We can hook that into the `fetchChatSession` method.

{% highlight JavaScript %}
fetchChatSessionHistory () {
  $.get(`http://127.0.0.1:8000/api/chats/${this.$route.params.uri}/messages/`, (data) => {
    this.messages = data.messages
    setTimeout(() => { this.loading = false }, 2000)
  })
},
{% endhighlight %}

Instead of simply setting `this.loading = false` we used the `setTimeout` function to delay it for 2 seconds. I did this so we can see the loading screen. Without this, it might just flash away since the operation is pretty fast.

If you're doing something intensive in the background you'll want to get rid of `setTimeout` and display your content to the user as soon as possible.

At the end of the `created` hook, do the same

{% highlight JavaScript %}
created () {
  ...

  setTimeout(() => { this.loading = false }, 2000)
},
{% endhighlight %}

That's all for this section, I'll leave you to figure out the CSS.

### Automatically scrolling to the bottom of the screen when messages exceed the window height

You'll also notice that the chat window doesn't scroll the bottom once our messages exceeds the height of the containing div. That's really annoying. Imagine if you had to scroll on your mobile phone to read the latest message from Whatsapp. Let's fix it.

We can quickly fix this by hooking into the `updated` life cycle. It's called after a Component's state has been updated and it's been re-rendered.

{%highlight JavaScript%}
updated () {
  // Scroll to bottom of Chat window
  const chatBody = this.$refs.chatBody
  if (chatBody) {
    chatBody.scrollTop = chatBody.scrollHeight
  }
},
{% endhighlight %}

PAY ATTENTION TO THIS!

*If you try to scroll too early for example in `mounted`, the `ref` might not be available because it's in an `if block`, the ref is only set when the page has finished loading and we've started a new session.*

*The first condition that's true when the page is mounted is the "loading" block (i.e v-else) when `mounted` is called, the ref is not set in that block. MOUNTED IS ONLY CALLED ONCE. The $refs Object will eventually be updated by Vue, but by then you won't be able to call it again because MOUNTED is only called once and it won't be called again when $refs is updated*

*If you decide to scroll as soon as you recieve a message in `onMessage`, The page would appear not to scroll because Vue wouldn't have re-rendered the component hence `chatBody.scrollHeight` won't be updated. (The message hasn't been appended to the div)*

`updated` is the perfect place because we can be sure that the new message has been added to the body and our `chatBody` ref is available.

*We're doing an extra Check for the `chatBody` ref because if a user is trying to start a chat session, the ref wouldn't be set (It's only set when a user tries to join a chat session). This might be a good time to refactor the "Start Chatting" screen into a different component*.

If you don't understand what I've explained above, ask a question and I'll gladly answer.

### Play sounds when the notification window is not focused

This is a common feature is a lot of web based chat apps. If you're on a different tab and you receive a message, a sound (and sometimes a browser notification) would be played to alert you and draw your focus back to it. Let's see how we can add that to our chat app.

`HTML5` and `JavaScript` make it easy for us to play audio. Let's add a new "notification" property to our component state

{% highlight JavaScript %}
data () {
  return {
    loading: true,
    messages: [],
    message: '',
    notification: new Audio('../../static/plucky.ogg'),
    sessionStarted: false
  }
},
{% endhighlight %}

*We placed the audio file in the static folder, so it can be served directly. If you place in in assets, Vue router will think it's a browser request and it'll redirect you to the auth page. This behaviour is documented [here](https://vuejs-templates.github.io/webpack/static.html). You can get the plucky.ogg file from the [Github Repo](https://github.com/danidee10/Chatire/blob/master/chatire-frontend/static/plucky.ogg)*

In your `onMessage` method, add the following:

{% highlight JavaScript %}
onMessage (event) {
  const message = JSON.parse(event.data)
  this.messages.push(message)

  if (!document.hasFocus()) {
    this.notification.play()
  }
},
{% endhighlight %}

This simply checks if the current tab (document) is not focused, If it's not then we'll play the sound.

<br />

That's it for the frontend and UI/UX Improvements.

## Backend Improvements

### Sending the messages directly through the WebSocket instead of the API

### JSON web tokens with `djsoer`

If you have some experience with tokens and authentication you might be asking yourself "What's the expiry date of the tokens obtained from `djoser`". It turns out that by default there's no expiry date. We're putting our users at risk because if an attacker obtains the token, that means they'll have lifetime access to a user's account and [`localStorage` and `sessionStorage` aren't the best place to store sensitive information](https://dev.to/rdegges/please-stop-using-local-storage-1i04) (Which includes tokens)

If an account is compromised, there should be a way to "expire" the token and make it Invalid.

To enable `JWT`'s in `djoser` we need to install an extension for django rest framework (Remember `djoser` is built around django rest framework)

{% highlight bash %}
$ pip install djangorestframework-jwt
{% endhighlight %}


Then we need to update our application's settings. Go to your application settings and update the `REST_FRAMEWORK` setting:

{% highlight python %}
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework_jwt.authentication.JSONWebTokenAuthentication',
    )
}
{% endhighlight %}

update `urls.py`

{% highlight python %}
urlpatterns = [
    path('admin/', admin.site.urls),

    # Custom URL's
    path('auth/', include('djoser.urls')),
    path('auth/', include('djoser.urls.jwt')),
    path('api/', include('chat.urls'))
]
{% endhighlight %}

Run `python manage.py runserver`. If everything went well the server should start without any errors.

Now use curl and run this (With a valid username and password in your database)

{% highlight bash %}
curl -X POST http://127.0.0.1:8000/auth/jwt/create/ --data 'username=username&password=password'
{"token":"eyJ0eXAiOiJKV1QiLCJhbGciOiJIUzI1NiJ9.eyJ1c2VyX2lkIjoxLCJ1c2VybmFtZSI6ImRhbmlkZWUiLCJleHAiOjE1MjE2NjIwOTgsImVtYWlsIjoib3NhZXRpbmRhbmllbEBnbWFpbC5jb20ifQ.l1eqoUgz5Nh9UEAz_OI_Xcr1Dvcjozo3HlkNh7qkC-0"}%
{% endhighlight %}

*Notice that we're using a different endpoint `/jwt/create` instead of `/token/create`*

You should get a token back. You can take that token to [JWT.io](https://jwt.io) and look inside, below is an image of what is contained in my own token.

![realtime django 6.2](../../../images/django/realtime-django/realtime-django-6.2.png)
<figcaption>Don't store secrets in JWT's</figcaption>

You can view all the information inside the JWT. The Goal of JSON web tokens is not to store sensitive information (Like credit card details, addresses etc). It's to allow us transfer information securely between the client and the server.

If an attacker alters this JWT by modifying it's data when he sends it to the server the JWT verification will fail because he signed it with a different key. ***YOU MUST PROTECT YOUR SECRET KEY AT ALL COST!***

Now that we have that out of the way, we need to update the `signIn` method in Vue to use the new endpoint.

{% highlight JavaScript %}
signIn () {
  const credentials = {username: this.username, password: this.password}

  $.post('http://localhost:8000/auth/jwt/create/', credentials, (data) => {
    sessionStorage.setItem('authToken', data.token)
    ...
}
{% endhighlight %}

We also need to change the `Authorization` header from `Token` to `JWT` in the `Chat.vue` component

{% highlight JavaScript %}
created () {
  this.username = sessionStorage.getItem('username')

  // Setup headers for all requests
  $.ajaxSetup({
    headers: {
      'Authorization': `JWT ${sessionStorage.getItem('authToken')}`
    }
  })

  ...
}
{% endhighlight %}

That's all. We've switched our auth scheme to JWT and we can still chat as Usual.

### Refreshing the token

If you continue using the chat application, after sometime you might notice that messages aren't going through because the token has expired.

This is the response you'll get from the API.

`{"detail":"Signature has expired.}`

Telling us that the token has expired. The default expiry time is 300 Seconds (5 minutes). This can be increased by setting:

`JWT_EXPIRATION_DELTA` to a valid datetime e.g `datetime.timedelta(minutes=30)` for 30 minutes. But we're still left with the problem of expired tokens. We can increase the expiry time/make the token active forever but that's not ideal because if an attacker obtains the token, they'll have lifetime access to a user's account.

We can also store the username and password and initiate a login but that's a NO NO!. You could get away with it in a mobile app or server side environment, but it's still a bad idea.

The best way is to have shortlived tokens and refresh them (obtain a new token). We can easily incorporate that into our Vue app by creating a new method that's called at a specific interval (Before the token expires).

The endpoint for refreshing tokens in `djoser` is `/jwt/refresh` but that's very easy to guess. We can change it to something more obscure to make it difficult to discover. *This is security by Obfuscation.* We're just making it harder for attackers to find the endpoint for refreshing tokens. Even if they use a bruteforcing software like [DirBuster](https://www.owasp.org/index.php/Category:OWASP_DirBuster_Project) It can take them weeks (Even months) to get the link.

The first thing to do is to enable the refresh functionality by setting

{% highlight python %}
JWT_ALLOW_REFRESH = True
{% endhighlight %}

Now update `urls.py` to include the link:

{% highlight python %}
from django.contrib import admin
from django.urls import path, include

from rest_framework_jwt.views import verify_jwt_token

from chat.views import raise_404

urlpatterns = [
    path('admin/', admin.site.urls),

    # Custom URL's
    path('auth/', include('djoser.urls')),

    # disable the old endpoint (Order is important)
    path('auth/jwt/refresh/', raise_404),

    # Register the new URL under an ambigous name
    path('this/is/hard/to/find/', verify_jwt_token),

    path('auth/', include('djoser.urls.jwt')),
    path('api/', include('chat.urls'))
]
{% endhighlight %}

In the `raise_404` view, we simply raise a `Http404` exception.

{% highlight python %}
from django.http import Http404


def raise_404(request):
   """Raise a 404 Error."""
   raise Http404
{% endhighlight %}

Finally, we need to add the token refresh logic to `Chat.vue`

{% highlight JavaScript %}
refreshToken () {
  const data = {token: sessionStorage.getItem('authToken')}

  $.post('http://127.0.0.1:8000/this/is/hard/to/find/', data, (response) => {
    sessionStorage.setItem('authToken', response.token)
  })
}
{% endhighlight %}

In the `created` hook register the function to be called every 240 seconds (4 minutes)

{% highlight JavaScript %}
created () {
  ...

  // Refresh the JWT every 240 Seconds (4 minutes)
  setInterval(this.refreshToken, 240000)
},
{% endhighlight %}

That's it! Your chat shouldn't get interrupted because the token is silently refreshed in the background.

If you need to customize JSON web tokens in django rest framework, please take a look at the options available in the [documentation](https://getblimp.github.io/django-rest-framework-jwt/#additional-settings)

That's the end of this tutorial. The real end :-)
