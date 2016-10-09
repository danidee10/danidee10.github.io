---
layout: post
title: Flask by example (How to build an online polling app)
date: 2016-09-18T22:35:33+01:00
---

Welcome everyone!, This is the beginning of a new series where we'll learn how to use flask by building a simple Voting app.
This series is going to be split into several parts because of the size of the app we're building. At the end of this series we'll have built a polling app, where:

<ul class="postlist">
  <li>Users can login.</li>
  <li>Vote on various topics and leave comments.</li>
  <li>Comments can also be upvoted and downvoted so that the best comments get to the top for other users to see.</li>
</ul>

<br />

We're going to use [Flask](http://flask.pocoo.org) as our backend, [Bootstrap3](http://getbootstrap.com/) for our interface generally and [Reactjs](https://facebook.github.io/react/) for other parts of our UI. the rest is just plain old html and css.

Our polling app is going to be called **Votr**

<br />

#### Why flask?
![Flask](/images/flask.jpg)

Quoting the flask [docs](http://flask.pocoo.org) ***Flask is a microframework for Python based on Werkzeug, Jinja 2 and good intentions***. Most people are quick to dismiss microframeworks like flask as not being powerful compared to django or other 'batteries included' web frameworks which isn't correct. so what does micro mean?

Micro means that the framework was designed to be as minimal as possible and just do what a basic web framework should do, while being extensible without making assumptions and including features that may not be used by everyone. Now What does this mean to you as a developer?, it means that you can build your web app from scratch and then as your need more features for your app you can simply install the required libraries and plugins you need (You can even roll out a new flask plugin and share it with the world if you're feeling adventrous ).

At the end of the day you'll get a better understanding of how different parts of your web app ticks because you actually added those 'parts' yourself (get it? web development one drop at a time). instead of using a web framework for over a year only for you to discover that feature `x` existed like 2 years earlier! and you've been doing things the wrong way.

So if you're a newbie in building web apps with python and you've been frustrated by django, I'll recommend you stick with flask, and then when you're comfortable with it. you can move on to django or another full featured framework. Using flask will help you understand most of the design decision and approaches django uses.

<br />

#### Requirements and setup
To follow this tutorial effectively you should have the following installed:

<ul class="postlist">
  <li><a href="https://www.python.org/downloads/">Python 3x</a></li>
  <li><a href="http://www.virtualenv.org/en/latest/">Virtualenv</a></li>
  <li><a href="http://flask.pocoo.org">Flask</a></li>
</ul>

<br />

 I'll also assume you have basic knowledge of html and css and that you're running Linux (what have you been doing with your life if you aren't).

 Also when i refer to Linux i'm referring to Ubuntu or some other Ubuntu based distro like Linux mint or elementary. user's of other distro's should be able to follow along without any problems. everything should also work on windows, if you're stuck with something on windows, please feel free to drop a comment or google your way out of it if you're ashamed of yourself and feel it's too basic :).

If you're on Linux, you should have python and pip installed installed by default in your distro. On windows make sure you download the python3 `exe` from python.org and tick the option to add python to your path.

Now we're going to install virtualenv and create a new virtual environment where our project and all it's files and dependencies will reside.

### Installing virtualenv

###### Install virtualenv with pip.

{% highlight bash %}
 $ sudo pip install virtualenv
{% endhighlight %}

###### Create a new virtualenv with python3 and activate it, (you can skip the `-p python3` part if you're on windows and you installed python3)

{% highlight bash %}
$ virtualenv -p python3 votr
{% endhighlight %}

###### Activate the virtualenv
{% highlight bash %}
$ cd votr
$ source bin/activate
{% endhighlight %}

###### Your new virtualenv is now installed and activated you should see something like this in your terminal prompt
{% highlight bash %}
(votr) [user@hostname votr]$
{% endhighlight %}

###### Finally we're going to install flask
{% highlight bash %}
$ pip install flask
{% endhighlight %}

#### Everything starts with hello world!
We're going to write a basic flask app that returns 'hello world' when we navigate to `/` in our browser

###### Create the project folder and votr.py
{% highlight bash %}
$ mkdir votr
$ cd votr
$ touch votr.py
{% endhighlight %}

That will create a new directory called `votr` and a file named `votr.py`

###### Open votr.py with favourite text editor and include the following

{% highlight python %}
from flask import Flask

votr = Flask('__name__')

@votr.route('/')
def home():
    return 'hello world'

if __name__ == '__main__':
    votr.run()

{% endhighlight %}

###### Run it from the terminal with:

{% highlight bash %}
$ python3 votr.py
{% endhighlight %}

By default flask runs on port `5000` so open any browser you have an enter the address `localhost:5000` and sure enough `hello world` should be displayed.

That's it! our development environment is all setup with flask installed. Note that we'll have to install other libraries (we haven't installed react yet). but i feel it's best to install every library we need as we go through the series, so we can understand what each library does and handle the quirks and issues that may arise when we get there.

If you have any questions or issues with the installation process. please leave a comment below else I'll see you in the next part where we're going to start writing some code!
