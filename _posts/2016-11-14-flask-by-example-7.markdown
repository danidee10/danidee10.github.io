---
layout: post
title: Flask by example 7 (Spin up an Admin dashboard quickly and easily with Flask-Admin)
date: 2016-11-14T22:35:33+01:00
---

Welcome to part 7 of this tutorial, in this part we're going to build an admin dashboard for our application which would serve provide basic `CRUD` functionality for the models in our database.

Go ahead and install flask admin from pip

{% highlight shell %}
pip install flask-admin
{% endhighlight %}

It's very easy to use flask-admin, just import the `Admin` class, create a new object out of it, and set the application to our current application

After that you can add different models by "adding" new `ModelView` objects to the application.

This `ModelView` objects would show up as different menu's in our admin dashboard

**Add the following to votr.py**
{% highlight python %}
from flask_admin import Admin
from flask_admin.contrib.sqla import ModelView

admin = Admin(votr, name='Dashboard')
admin.add_view(ModelView(Users, db.session))
{% endhighlight %}

reload the page and open `localhost:5000/admin`

You should see a a blank page that tells you Your "Login was successful" and a plain navbar that looks like this

![flask-7-navbar](/images/flask-7-navbar.png)

You can add your models and play around the the admin. create, delete and modify records as you please


## Adding authentication to flask admin
After playing around with flask admin, you'd have noticed that something very important is missing...**Authentication**

Now, there are several ways of adding authentication to flask admin, but i'm going to show you how to integrate our Authentication system with flask admin.

First of all, it would make more sense to keep all our "admin stuff" in a separate file in order to keep the main application file clean. Let's call that admin file **admin.py**

Flask admin provides a way of "hiding" models from users based on any condition we like, this is very useful in building role-based admin interfaces, For example a cashier might have access to the post transactions but he should not be able to see the total transactions posted for the day (Which contains other entries posted by other cashiers), Only the manager should be able to see that

To add this feature, we have to inherit the `ModelView` class and override two methods

**place the following in the admin.py file**

{% highlight python %}
from flask_admin.contrib.sqla import ModelView
from flask import session, redirect, url_for, request


class AdminView(ModelView):

    def __init__(self, *args, **kwargs):
        super().__init__(*args, **kwargs)
        self.static_folder = 'static'

    def is_accessible(self):
        return session.get('user') == 'Administrator'

    def inaccessible_callback(self, name, **kwargs):
        if not self.is_accessible():
            return redirect(url_for('home', next=request.url))

{% endhighlight %}

The **is_accessible** method determines if a model is accessible or not, based on a particular condition, in our case we only want to show the model to the current user, if they're logged in and their username equals **"Administrator"**

In the **inaccessible_callback** method, we're simply saying if the model is not accessible redirect the user back to the homepage. The next argument is used to take the admin user back to the last page they tried to access.

To make use of this parameter, just change the return statement in your login route to:

{% highlight python %}
return redirect(request.args.get('next') or url_for('home'))
{% endhighlight %}

and the url the login form on the homepage points to:

{% highlight html %}
{% raw %}
<form method="post" action="{{ url_for('login', next=request.args.get('next')) }}">
{% endraw %}
{% endhighlight %}

<br />

To use our `AdminView` class, we have to tell Flask-Admin about it in **votr.py**.

{% highlight python %}
from flask_admin import Admin
from admin import AdminView

votr = Flask(__name__)

# load config from the config file we created earlier
votr.config.from_object('config')

# create the database
db.init_app(votr)
# db.create_all(app=votr)

migrate = Migrate(votr, db, render_as_batch=True)

admin = Admin(votr, name='Dashboard', index_view=AdminView(Topics, db.session, url='/admin', endpoint='admin'))
admin.add_view(AdminView(Users, db.session))
admin.add_view(AdminView(Polls, db.session))
admin.add_view(AdminView(Options, db.session))
{% endhighlight %}

The important line here is:

{% highlight python %}
admin = Admin(votr, name='Dashboard', index_view=AdminView(Topics, db.session, url='/admin', endpoint='admin'))
{% endhighlight %}

specifically the `index_view` argument, we're telling flask admin that we want the **Topics** model to serve as the default view for our application instead of the blank home page and then we also specified that the url for the `Topic` model should be `http[s]://hostname:port/admin` instead of the default `http[s]://hostname:port/admin/topics`

Finally we're setting the `endpoint` to **admin**

That's all!, with this information flask admin is able to build the blueprints with our models (*Internally flask admin uses introspection to get the details about the models passed to it and creates blueprints on the fly*)

Flask admin should be protected from others whose username doesn't equal **Administrator**. Our username field is actually unique, so this means that we only have one admin user.

If you want to add other admin users, you can add more usernames to check for in the `is_accessible` method or better still add a boolean field to your user model to determine if the user is an admin user or not. I'll leave you to your imagination here

<br />

### Protecting the polls from multiple votes
In the last part, we talked about prevent users from voting multiple times, to do this we'll have to add a new model to track a user and the polls he has voted on, and then on every request `/api/vote` we can simply check if he's voted on that poll before and then abort the request with a custom message or allow the request like we've always done

**Add a new model called UserPolls in votr.py**

{% highlight python %}
class UserPolls(Base):

    topic_id = db.Column(db.Integer, db.ForeignKey('topics.id'))
    user_id = db.Column(db.Integer, db.ForeignKey('users.id'))

    topics = db.relationship('Topics', foreign_keys=[topic_id],
                             backref=db.backref('voted_on_by', lazy='dynamic'))

    users = db.relationship('Users', foreign_keys=[user_id],
                            backref=db.backref('voted_on', lazy='dynamic'))
{% endhighlight %}

after doing that run the [migrations and upgrade the database]({% post_url 2016-10-09-flask-by-example-5 %})



**IMPORTANT: before running the migrations, make sure you comment out the `db.create_all` in `votr.py` if you don't SQLAlchemy would create the new table from you and Alembic won't be able to detect any change in the database schema**


**Modify the /api/vote endpoint to incorporate the new changes**

{% highlight python %}
@votr.route('/api/poll/vote', methods=['PATCH'])
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
{% endhighlight %}

That's all, a user shouldn't be able to vote on a poll more than once, if they try that they should see a popup in their browser telling them that multiple votes are not allowed

<br />

### Customizing Flask-Admin
Out of the box, flask admin gives us a pretty usable interface and reasonable defaults, but that doesn't mean it's not flexible. flask admin customize it and add extra behaviour, flask admin allows us make different enhancements ranging from ui enhancements (Search boxes, filters) to behavioural enhancements.

We're going to add some new features to the admin page

<ul class="postlist">

  <li>A search box to search for poll by their title</li>

  <li>A filter to show opened/closed polls</li>

  <li>Sort the Topics by the date they were created</li>

  <li>Add a filter on the status field so we can see all open or closed polls at any point in time</li>

  <li>Change the default date format to something more readable</li>

  <li>Display the total vote count for each topic and make the topics sortable with it</li>

</ul>

<br />

The first four options added easily

{% highlight python%}
class TopicView(AdminView):

    def __init__(self, *args, **kwargs):
        super(TopicView, self).__init__(*args, **kwargs)

    column_list = ('title', 'date_created', 'date_modified', 'status')
    column_searchable_list = ('title',)
    column_default_sort = ('date_created', True)
    column_filters = ('status',)
{% endhighlight %}

**<u>column_list:</u>** This is used to list the various columns in the order you want them

**<u>column_searchable_list:</u>** Columns that you want to be searchable in the model

**<u>column_default_sort:</u>** The column that the view should be sorted with by default (when the view is loaded for the first time). The second parameter `True` tells flask-admin to sort it in descending order

**<u>column_filters:</u>** List of columns that can be used to filter

Notice that we created a new class, so let's change the `index_view` argument for the `Admin` object in **votr.py** to use our new class

{% highlight python %}
# new top level imports
from admin import AdminView, TopicView


admin = Admin(votr, name='Dashboard', index_view=TopicView(Topics, db.session, url='/admin', endpoint='admin'))
{% endhighlight %}


That's it, reload the admin page and you should see the new changes in effect

![flask-7-customize1](/images/flask-7-customize1.png)

With some simple variables, we've been able to add some customization to our admin interface.

The last two customizations are not as straight-forward, but they aren't too complex either

The following code is used to change the date format

{% highlight python %}
from flask_admin.contrib.sqla import ModelView
from flask import session, redirect, url_for, request
from flask_admin.model import typefmt
from datetime import datetime

class AdminView(ModelView):

    def __init__(self, *args, **kwargs):
        super(AdminView, self).__init__(*args, **kwargs)
        self.static_folder = 'static'

        self.column_formatters = dict(typefmt.BASE_FORMATTERS)
        self.column_formatters.update({
                type(None): typefmt.null_formatter,
                datetime: self.date_format
            })

        self.column_type_formatters = self.column_formatters

    def date_format(self, view, value):
          return value.strftime('%B-%m-%Y %I:%M:%p')
{% endhighlight %}

We put it in the constructor of `AdminView` because we want to re-use this date format in all our models including `Topics` which already inherits from `AdminView`

You should also note that we can define flask admin variables as class variables like we did for the first four customizations or as instance variables like we're doing  with `self.column_type_formatters`. so if you don't need it in the constructor `column_type_formatters` is just fine

`column_formatters` expects a dictionary of Object types as the key and the corresponding display value we want as the value

`typefmt.BASE_FORMATTERS` provides sane defaults for the various types of field formatters our model should have this list includes formatters for `lists`, `dicts`, `bools` and other python data types.

We simply updated the dictionary with our own custom format for datetime objects and left the rest to flask admin


To accomplish the last improvement, We have to make use of a feature in SQLAlchemy called [hybrid_property](http://docs.sqlalchemy.org/en/latest/orm/extensions/hybrid.html)

A `hybrid_property` can be thought of as a computable column. At the database layer the column doesn't exist but we can use it ***almost*** like we use other columns in our model.

Let's modify the `Topics` model in the **models.py** file

{% highlight python %}
# new top level imports
from sqlalchemy.ext.hybrid import hybrid_property
from sqlalchemy import select, func

# Model for poll topics
class Topics(Base):
    title = db.Column(db.String(500))
    status = db.Column(db.Boolean, default=1)  # to mark poll as open or closed
    create_uid = db.Column(db.ForeignKey('users.id'))

    created_by = db.relationship('Users', foreign_keys=[create_uid],
                                 backref=db.backref('user_polls', lazy='dynamic'))

    # user friendly way to display the object
    def __repr__(self):
        return self.title

    # returns dictionary that can easily be jsonified
    def to_json(self):
        return {
                'title': self.title,
                'options': [{'name': option.option.name, 'vote_count': option.vote_count}
                            for option in self.options.all()],
                'status': self.status,
                'total_vote_count': self.total_vote_count
            }

    @hybrid_property
    def total_vote_count(self, total=0):
        for option in self.options.all():
            total += option.vote_count

        return total

    @total_vote_count.expression
    def total_vote_count(cls):
        return select([func.sum(Polls.vote_count)]).where(Polls.topic_id == cls.id)
{% endhighlight %}

The `hybrid_property` decorator tells SQLAlchemy that we want to use the return value of `total_vote_count` as a column of the model not just a plain property of the class

The key part of the hybrid_property is the the decorator `@total_vote_count.expression`. Depending on how complex the `hybrid_property` is and how we compute the values, we might need a `hybrid_property` expression (*For simple computed values based on fields of the model it's not necessary, flask-admin can figure out how to sort the column*)

The expression must return `SQL` that would be used by flask-admin to sort the column properly, if you don't need the sorting you can just leave out the expression part.

So this this case we're returning SQL that's similar to this:

{% highlight sql %}
SELECT sum(polls.vote_count) AS sum_1 FROM polls, topics WHERE polls.topic_id = topics.id
{% endhighlight %}

So with that information, flask-admin would be able to sort the total_vote_count column with it's total votes

Don't forget to add this to the `TopicView` class in `admin.py`

{% highlight python %}
column_sortable_list = ('total_vote_count',)
{% endhighlight %}

and modify `colum_list` to show the new column

{% highlight python %}
column_list = ('title', 'date_created', 'date_modified', 'total_vote_count', 'status')
{% endhighlight %}

We also modified the `to_json` method of the model to use the new `total_vote_count` property


<br />
We've come to the end of this part, this post was a gentle introduction to flask-admin and what you can do with it.
you also saw how you can restrict each user to a single vote on a poll.

Everything about flask admin is customizable, to keep this tutorial brief i didn't talk about customizing the templates but if you're wondering how "customizable" flask admin templates are here are two screenshots of an admin dashboard built with flask admin.


![flask admin dashboard](http://mrjoes.github.io/shared/posts/flask-admin-120/cb1.jpg)

![flask admin dashboard2](http://mrjoes.github.io/shared/posts/flask-admin-120/cb3.jpg)

There's a so much to explore. [Redis CLI](https://github.com/mrjoes/flask-admin/tree/master/examples/rediscli) is even integrated with flask admin!.

***Now that you know about flask admin, try to resist the urge to use it for your general/public user's admin dashboard. If you know 101% of what you're doing, there's actually nothing wrong with using it, but have it at the back of your head that you'll probably spend more time customizing flask admin compared to the time you'll spend rolling your own admin dashboard from scratch.***

***You also run the risk of users gaining access or seeing things they aren't supposed to see.***

***Flask admin is more suitable for admins or users that you can trust***

In the next part, we're going to talk about flask blueprints, when and why you should use them in your flask application. see you there!
