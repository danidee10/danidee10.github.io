---
layout: post
title: 'Realtime Django Part 1: Building a Chat application RabbitMQ and uWSGI websockets (Introduction and Setup)'
date: 2018-01-01T22:35:33+01:00
---

It's been a while since i published an article. Just like the Flask by Example series i did earlier i'm back with another series this time on Django.

For this series we're going to build a Chat application. In my own words a "Mini slack" where two users can chat with themselves. To keep it brief and simple we're focusing on text messages only no fancy code snippets or images but if you follow this article to the end, you should have enough knowledge to extend and add new functionality if you desire.

## Requirements

To get the most out of this series, you should be familiar with Django (At least you've gone through the Getting started and Poll app guides).

You Should also be Comfortable writing JavaScript and know a thing or two about VueJS. We aren't building a complicated Vue SPA but if you have no experience with Vue. You can check it out now to see what it's all about.

With that out of the way, i'll do my best to breakdown and explain any sections that i feel are complex or have a lot of Magic going on.

If you don't understand anything, the comments are wide-open, just ask a question and i'll do my best to answer.

### Libraries involved

Python:

- django
- django rest framework
- djoser
- django-notifs
- pika

<br>

JavaScript:

- Vue
- vue-cli
- vue-router

<br>

### What are you going to learn ?
The main goal of this tutorial is to teach you about websockets and how you can integrate them with your django application(s).

That's not all, while we're at that, you'll also learn a lot about VueJS


## Installation

***Before you start make sure you create a virtual environment activate it***

Go ahead and install django with

{% highlight bash %}
pip install django
{% endhighlight %}

after that let's start a new project called Chatire.

{% highlight bash %}
django-admin startproject chatire
{% endhighlight %}

run the migrations

{% highlight bash %}
python manage.py migrate
{% endhighlight %}

Finally, start the django development server with:

{% highlight bash %}
python manage.py runserver
{% endhighlight %}

If everything works properly you should see this:

<img />

That wraps it up for this part, in the next part we'll start by implementing user registration and authentication with djoser.

See you in the next part.