---
layout: post
title: Flask by example 8 (Understanding Flask blueprints)
date: 2016-11-20T22:35:33+01:00
excerpt:
        Basically, a flask blueprint is a way for you to organize your flask application into smaller and re-usable applications
        Just like a normal flask application, a blueprint defines a collection of views, templates and static assets.
tags: Flask Blueprints
---


### What are flask blueprints?

Basically, a flask blueprint is a way for you to organize your flask application into smaller and re-usable application

Just like a normal flask application, a blueprint defines a collection of views, templates and static assets.

It should be noted that a blueprint is not a 'plug and play' app, it cannot run on it's own every blueprint must be registered on a real Flask application before it can be used.

### Why should you use blueprints
The main reason why you should use blueprints is to de-couple your application into smaller re-usable components. Like i said earlier in this series it would make perfect sense to move all our api calls into a blueprint.

This makes the code more maintainable and easier to debug.


### How do you create a blueprint
Well, the choice is really up to you, since flask is very liberal and doesn't enforce much conventions on you. There are several ways of creating a blueprint, but personally i prefer to treat blueprints as separate python packages with everything they provide in their own folder. That way, if something goes wrong with our api for example, you already know where to start looking.

Lets create a new package and call it api

**Your application structure should look like this:**

{% highlight python %}
.
├+++ api
│   +++ __init__.py
|   +++ api.py
├── migrations
│   ├── versions
│   ├── alembic.ini
│   ├── env.py
│   ├── README
│   └── script.py.mako
├── static
│   ├── css
│   ├── fonts
│   ├── gifs
│   ├── images
│   └── js
├── templates
│   ├── index.html
│   ├── pollsbackup.html
│   ├── polls.html
│   └── signup.html
├── admin.py
├── config.py
├── LICENSE
├── models.py
├── README.md
├── requirements.txt
├── votr.db
├── votr.py
{% endhighlight %}

**Open `api.py` and include the following**

{% highlight python %}
from models import db, Users, Polls, Topics, Options, UserPolls
from flask import Blueprint, request, jsonify, session

api = Blueprint('api', 'api', url_prefix='/api')


@api.route('/polls', methods=['GET', 'POST'])
# retrieves/adds polls from/to the database
def api_polls():
    if request.method == 'POST':
        # get the poll and save it in the database
        poll = request.get_json()

        # simple validation to check if all values are properly secret
        for key, value in poll.items():
            if not value:
                return jsonify({'message': 'value for {} is empty'.format(key)})

        title = poll['title']
        options_query = lambda option: Options.query.filter(Options.name.like(option))

        options = [Polls(option=Options(name=option))
                   if options_query(option).count() == 0
                   else Polls(option=options_query(option).first()) for option in poll['options']
                   ]

        new_topic = Topics(title=title, options=options)

        db.session.add(new_topic)
        db.session.commit()

        return jsonify({'message': 'Poll was created succesfully'})

    else:
        # it's a GET request, return dict representations of the API
        polls = Topics.query.filter_by(status=1).join(Polls).order_by(Topics.id.desc()).all()
        all_polls = {'Polls':  [poll.to_json() for poll in polls]}

        return jsonify(all_polls)


@api.route('/polls/options')
def api_polls_options():

    all_options = [option.to_json() for option in Options.query.all()]

    return jsonify(all_options)


@api.route('/poll/vote', methods=['PATCH'])
def api_poll_vote():
    poll = request.get_json()

    poll_title, option = (poll['poll_title'], poll['option'])

    join_tables = Polls.query.join(Topics).join(Options)

    # Get topic and username from the database
    topic = Topics.query.filter_by(title=poll_title).first()
    user = Users.query.filter_by(username=session['user']).first()

    # filter options
    option = join_tables.filter(Topics.title.like(poll_title)).filter(Options.name.like(option)).first()

    # check if the user has voted on this poll
    poll_count = UserPolls.query.filter_by(topic_id=topic.id).filter_by(user_id=user.id).count()
    if poll_count > 0:
        return jsonify({'message': 'Sorry! multiple votes are not allowed'})

    if option:
        # record user and poll
        user_poll = UserPolls(topic_id=topic.id, user_id=user.id)
        db.session.add(user_poll)

        # increment vote_count by 1 if the option was found
        option.vote_count += 1
        db.session.commit()

        return jsonify({'message': 'Thank you for voting'})

    return jsonify({'message': 'option or poll was not found please try again'})


@api.route('/poll/<poll_name>')
def api_poll(poll_name):
    poll = Topics.query.filter(Topics.title.like(poll_name)).first()

    return jsonify({'Polls': [poll.to_json()]}) if poll else jsonify({'message': 'poll not found'})

{% endhighlight %}

The `Blueprint` class takes three basic arguments:

The first argument is the blueprints name

The second argument is very important it's the `import_name`. This name has to be set to the name of our package (which is also api) as Flask uses the `import_name` for some internal operations such as locating the template folder of the blueprint and locating various files and objects of the main application from the blueprint. (This ensures that methods like `render_template` and `send_static_files` work properly and give us the actual files that we want)

The third argument is the url prefix of the blueprint. With this we were able to remove the redundancy of prefixing all our api urls with `/api`.


**Let's register the blueprint in votr.py**
{% highlight python %}
from flask import Flask, render_template, request, flash, session, redirect, url_for
from werkzeug.security import generate_password_hash, check_password_hash
from flask_migrate import Migrate
from models import db, Users, Polls, Topics, Options, UserPolls
from flask_admin import Admin
from admin import AdminView, TopicView

# Blueprints
from api.api import api

votr = Flask(__name__)

votr.register_blueprint(api)
{% endhighlight %}

That's all you need to do. Reload the webpage and the application should still function like it did before.

That's all you need to know about blueprints, they're really helpful when you want to refactor your flask application and split it into smaller parts. They help you to add extra routes and functions to your application without chunking the code into a single file.

Join me in the next part where I'll show you how to schedule and run background jobs in Flask with [Celery](http://www.celeryproject.org/).

We're going to use celery to implement a feature where users can set the time they want a poll to stay active. after which the poll would be automatically closed.

Thanks for reading.
