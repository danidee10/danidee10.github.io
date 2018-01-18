---
layout: post
title: 'Realtime Django Part 1: Build a Chat application with django, RabbitMQ and Vue.js (Introduction and Setup)'
date: 2018-01-01T22:35:33+01:00
tags: django vue rabbitmq
excerpt: The main goal of this tutorial is to teach you about WebSockets and how you can integrate them with your django application(s).
---

![Realtime Django 1.1](../../../images/django/realtime-django/realtime-django-1.1.png)


## Prerequisites

This tutorial is not an Introduction to `Django` or `Vue`.

To get the most out of it, you should be familiar with Django. It's expected that you've gone through the [Getting Started guide](https://docs.djangoproject.com/en/2.0/intro/overview/) and the [Polling app](https://docs.djangoproject.com/en/2.0/intro/tutorial01/).

You Should also be Comfortable writing JavaScript and know the basics of `Vue.js`. We are not building a complicated Vue SPA but if you're not familiar with Vue, You can check it out the [documentation](https://vuejs.org) and get familiar with it's principles and API's.

If have development experience with React or Angular you shouldn't have problems understanding Vue because it draws a lot of things from both of them (Especially React).

During the course of this tutorial i'll do my best to breakdown and explain any sections that i feel are complex or have a lot of Magic going on.

If you don't understand anything, the comments are wide-open. Feel free to reach out.

<br />

### Libraries involved

**Python:**

- [django 2.0](https://www.djangoproject.com/)
- [djoser](http://djoser.readthedocs.io)
- [django-notifs](https://github.com/danidee10/django-notifs)
- [pika](https://pika.readthedocs.io)
- [uWSGI](https://uwsgi-docs.readthedocs.io/en/latest/)

<br />

**JavaScript**:

- [Vue.js](https://vuejs.org/)
- [vue-cli](https://github.com/vuejs/vue-cli)
- [vue-router](https://router.vuejs.org/)

### What are you going to learn?

- The main goal of this tutorial is to teach you about WebSockets and how you can integrate them with your django application(s).

- While we walk through this tutorial you'll also learn how to build your own ["mini pusher"](https://pusher.com) using `RabbitMQ` to broadcast messages in realtime to multiple clients.

- That's not all, while we're at it, We'll build a simple token based auth system and you'll see how you can connect a Vue.js SPA to a django backend with `django-rest-framework`. There are a lot of Vue tutorials online but most of them are based on `Laravel` or `NodeJS`.

Before we proceed, this is a glimpse of what we'll build:

![Realtime Django 1.2](../../../images/django/realtime-django/realtime-django-1.2.gif)

I call the application chatire but you can call it whatever you want to.

## Installation

***Before you start make sure you create a virtual environment activate it***

Go ahead and install django with:

{% highlight bash %}
$ pip install django
{% endhighlight %}

after that let's start a new project called `chatire`.

{% highlight bash %}
$ django-admin startproject chatire
{% endhighlight %}

run the migrations

{% highlight bash %}
$ python manage.py migrate
{% endhighlight %}

Finally, start the django development server with:

{% highlight bash %}
$ python manage.py runserver
{% endhighlight %}

If everything worked you should see this:

![Realtime Django 1.2](../../../images/django/realtime-django/realtime-django-1.2.png)
<figcaption>The Django 2.0 landing page looks really nice!</figcaption>

<br />

That wraps it up for this part, in the next part we'll implement user registration and authentication with djoser.

<br />

[Continue reading Realtime Django Part 2: Authentication and User Managment]({{ '2018/01/03/realtime-django-2.html' | relative_url }})