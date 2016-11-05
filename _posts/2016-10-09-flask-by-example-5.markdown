---
layout: post
title: Flask by example 5 (How to build a simple REST API)
date: 2016-10-05T22:35:33+01:00
---

Welcome to part 5. In this part I'm going to show you how to build your own RESTful API with flask

The API would serve as the middle-man between the client (browser) and the web server and it's the means by which data would be sent across both parties. We're going to be passing the data along as JSON (Javascript object Notation).

At the end of this tutorial you'd have created an API that receives data as JSON, parses it and stores the information in the database, and also sends JSON back to the client whenever it's queried.

The API would also be robust enough to respond with useful error messages (as JSON) whenever the client sends incomplete data to the server.

This is an overview on the structure of our application and the important technology/library used by each component.

From the diagram you can see that the API is the one who runs the show and provides a means for the other components to communicate with each other, including external services.

![flask part 5](/images/flask-5-overview.png)

## WHY DO WE NEED AN API
I'm not going to bore you out with all the numerous reasons to use an API, because you've probably read them online somewhere if you've not here are [some](http://sproutsocial.com/insights/what-is-an-api/).

The main reason why need an API is because client-side Javascript libraries like ReactJS cannot communicate directly with your database (which resides on the server) directly. Though Server side Javascript like NodeJS can do that.

Currently this is how our form declaration looks with React

#### With React
{% highlight html %}
<form id="poll_form" className="form-signin" onSubmit={this.handleSubmit}>
{% endhighlight %}

without React we have this

#### Without React
{% highlight html %}
<form id="poll_form" className="form-signin" method="POST" action="/polls">
{% endhighlight %}

When the `handleSubmit` function is eventually called a lot of "DOM changing" processes would have gone on in the other methods `handleOptionAdd`, `handleOptionChange` and `handleTitleChange` (Adding options and setting the Title of the poll) that the DOM wouldn't know about.

 I urge you to experiment and  submit the form you created in the previous part without using the `onSubmit` handler and see what happens when you print `request.form` in flask.

So in summary an API serves as a common interface for the client and the server to talk to each other reliably.

Enough of the talk let's build an API.

*The full code of some of some files won't be pasted, Only the important part will be pasted so if you want to see the full source code at any point, feel free to checkout the github [repo](https://github.com/danidee10/Votr)*



## Building the API

Our API is going to perform two basic functions at this stage:

<ul class="postlist">
  <li>Receive data as a valid JSON string, parse it and store the information in the database</li>

  <li>Return data in as JSON back to our client who then decides on what he wants to do with the data. We're simply providing the raw data back and not forcing the presentation of the data on the client. we don't care about what the client wants to do with the data he could be using it for stastics and data analysis or to build an alternate version of our polling application</li>

</ul>

<br />

## JSON STRUCTURE FOR GET REQUESTS

{% highlight javascript %}
{
	"Polls": [{
			"title": "poll1",
			"options": [{
				"name": "option1",
				"vote_count": "45"
			}, {
				"name": "option2",
				"vote_count": "16"

			}],
			"status": "open"

		},

		{
			"title": "poll2",
			"options": [{
				"name": "option2",
				"vote_count": "12"
			}, {
				"name": "option2",
				"vote_count": "18"

			}],
			"status": "closed"
		}
	]
}
{% endhighlight %}

This is how we want to return polls back to the client when they make a request to *get some or all the polls* in our database. Let's call this structure our *dream API*

If the JSON structure looks scary to you, just think of it as a collection of python dictionaries and lists and the structure should become clearer to you.

You can also check the validity of this JSON and any JSON string online with [JSON lint](http://jsonlint.com/)

Moving forward, we're going to separate our API routes from our main application routes by prefixing the url with `/api`

*hint: We're goint to refactor the whole codebase at the end of this series and make our API into a flask blueprint. if you don't know what a blueprint is in flask this might be a good time to [check it out](http://flask.pocoo.org/docs/0.11/blueprints/)*



###### Now edit the `votr.py` file to include a new route `/api/polls`

{% highlight python %}
@votr.route('/api/polls', methods=['GET', 'POST'])
# retrieves/adds polls from/to the database
def api_polls():
    if request.method == 'POST':
        # get the poll and save it in the database
        poll = request.get_json()

        return "The title of the poll is {} and the options are {} and {}".format(poll['title'], *poll['options'])

    else:
        # query the db and return all the polls as json
        all_polls = {}

        # get all the topics in the database
        topics = Topics.query.all()
        for topic in topics:
            # for each topic get the all polls that are it's children
            all_polls[topic.title] = {'options': [poll.option.name for poll in Polls.query.filter_by(topic=topic)]}

        #print(jsonify(json_list=Topics.query.join(Polls).all()))
        print(all_polls)

        return jsonify(all_polls)
{% endhighlight %}


In the POST request, we aren't really doing anything, we just get the JSON the user sent with `request.get_json()` and then return the details back to show us that our API works. request.get_json() is a helper method provided that flask that gets the JSON data from the request and converts it to a python dictionary for us. It's similar to calling json.loads on a JSON string.

Earlier, our aim was to return a JSON string containing the poll and various details associated with it. This is exactly what we implemented in the GET request. To acheive that we queried the database for all the existing Topics and then for each topic we fetched the associated polls (`Polls.query.filter_by(topic=topic)`), after that we appended the results in the format we wanted and then used another flask helper function called `jsonify` to convert the string to JSON. jsonify is also similar to python's json.dumps but it does more for you behind the scenes that pure json.dumps won't do.

Okay our should work properly but we could still improve it by leveraging the power of SQLAlchemy and using an SQL join statement *But how do you use a join statement when you're not writing raw SQL...Simple use SQLAlchemy*

##### Let's fire up a terminal and play around with SQLAlchemy
{% highlight bash %}
>>> from votr import db, votr, Topics, Polls
>>> db.app = votr
>>> db.init_app(votr)
>>> Topics.query.join(Polls).all()
[Which side is going to win the EPL this season, Whos better liverpool or city]

{% endhighlight %}

*If for some reason you have an empty database (your query returns a blank result), here's a github [gist](https://gist.github.com/danidee10/a597bbc92dbb8b5add6eea768b1bc18b) to quickly create all the records in the database at this stage. Just run each line of code in the file in a python interactive shell*

Neat! we were able to fetch all the topics and their associated polls in a single query. Though you're seeing only the title of the Topic (that's `__repr__` in action). those two items are actually SQLAlchemy objects that contain all the data you need, you just need to know how to extract them from the object.

Another amazing thing about SQLAlchemy is that we can see the query it generated for us behind the scenes

{% highlight python %}
>>> str(Topics.query.join(Polls))     
'SELECT topics.id AS topics_id, topics.date_created AS topics_date_created, topics.date_modified AS topics_date_modified, topics.title AS topics_title \nFROM topics JOIN polls ON topics.id = polls.topic_id'
{% endhighlight %}

That shows us the General SQL statment it's going to generate for us, but we can go a step further and see the specific query for the SQL dialect of a specific database.

{% highlight python %}
>>> from sqlalchemy.dialects import postgresql
>>> ddd = Topics.query.join(Polls)
>>> print(ddd.statement.compile(dialect=postgresql.dialect()))
SELECT topics.id, topics.date_created, topics.date_modified, topics.title
FROM topics JOIN polls ON topics.id = polls.topic_id
>>> from sqlalchemy.dialects import sqlite
>>> print(ddd.statement.compile(dialect=sqlite.dialect()))
SELECT topics.id, topics.date_created, topics.date_modified, topics.title
FROM topics JOIN polls ON topics.id = polls.topic_id

{% endhighlight %}

For the experienced database admins this could be pretty useful, as ORM's don't generate the most efficient SQL query sometimes (but there's no need for premature optimization at this level). And as a beginner you could also learn more about SQL as a languge. by simply writing complex queries in SQLAlchemy and inspecting the generated SQL.

Okay, so how do we convert this complex query object into a dictionary which can be jsonified.

To do that we're going to define a method called `to_json` in the `Topics` Model that abstracts this functionality and simply gives us a custom dictionary representation of the object.

Update the `Topics` model and add a `to_json` method

{% highlight python %}
# returns dictionary that can easily be jsonified
def to_json(self):
    return {
            'title': self.title,
            'options':
                [{'name': option.option.name, 'vote_count': option.vote_count}
                    for option in self.options.all()]
        }
{% endhighlight %}


Now we can simply call `to_json` on any `Topic` to get a custom `dict` representation, which leaves us with a simple list comprehension

{% highlight bash %}
>>> [poll.to_json() for poll in polls]
[{'title': 'Which side is going to win the EPL this season', 'options': [{'name': 'Arsenal', 'vote_count': None}, {'name': 'Spurs', 'vote_count': None}]}, {'title': 'Whos better liverpool or city', 'options': [{'name': 'Liverpool FC', 'vote_count': None}, {'name': 'Manchester city', 'vote_count': None}]}]
>>>

{% endhighlight %}


### Wait? what about the poll state
This is the kind of JSON that our API will send back to us when we query it currently.

{% highlight javascript %}
{
	"Polls": [{
			"title": "poll1",
			"options": [{
				"name": "option1",
				"vote_count": "45"
			}, {
				"name": "option2",
				"vote_count": "16"

			}]

		},

		{
			"title": "poll2",
			"options": [{
				"name": "option2",
				"vote_count": "12"
			}, {
				"name": "option2",
				"vote_count": "18"

			}]
		}
	]
}
{% endhighlight %}

What happened to our dream API, weren't we supposed to have the status of the poll included.

Yes we were but we can't, because well...our database design is broken. Yes you read that right it's broken. The status of the poll is supposed to be a column on the `Topics` model, so once we set a Topic as closed all options (actually `Poll` objects in our models) associated with it are automatically closed (because their parent was). Thereby marking the poll as closed.

But you went ahead and *brilliantly* put it in the `Polls` table (actually i misled you and congrats! you fell for it).

So after cursing yourself and me, how are you going to fix this mistake. Yes you're interacting with your database with python through SQLAlchemy and you can easily cut and paste the status line into the `Topics` model. But that's not going to work.

SQLAlchemy on it's own can only alter the schema of our database by adding new models, but we've found ourself in a situation where we need to delete a column from a table and then add create that column in another table. (altering two tables).

SQLAlchemy and other ORM's are only an abstraction not a direct replacement for SQL, so at the end of the day you're still working with a database and SQL.

The database on it's own part, knows nothing about python so it wouldn't know when we made any change to the python file containing the models. The `models.py` needs a way need a way to tell the database *hey one of the us just got updated. you need to update yourself to match our current state*

This is where [Flask-Migrate](https://flask-migrate.readthedocs.io/en/latest/) comes to the rescue and says step aside guys I'll settle this.

### Flask Migrate
Quoting the [docs](https://flask-migrate.readthedocs.io/en/latest/)

*Flask-Migrate is an extension that handles SQLAlchemy database migrations for Flask applications using <strong>Alembic</strong>. The database operations are made available through the Flask command-line interface or through the Flask-Script extension.*

So what exactly is a migration?

Migrations (also known as ‘schema evolution’ or ‘mutations’) are simply a way of changing your database schema from one version into another.

Carrying out database migrations manually is very problematic, sure you could use a tool like [phpMyAdmin](https://www.phpmyadmin.net/) or [pgAdmin](https://www.pgadmin.org/)  to alter your tables, but picture this scenario - You've done a lot of migrations manually with a graphical tool or from the command line and then you realized that your initial database design was better than the one you have currently, so you decide to go three steps back to the previous state of your database.

 How do you go back when you have no previous history of your schema at that point in time? If you don't have a good memory or you didn't document that information down, you're screwed. You either have to design it again or design that table or tables(s) again depending on the size of your database and how much you deviated from that migration

This is what Flask Migrate (and other solutions for other web frameworks) are good at, with Flask migrate you can easily switch between older and newer versions of your database state at anytime. It's just like a mini `git` for your database.

It also makes it easier for you to switch databases, you could design and develop on an SQLite database database, and then deploy the application to a PostgreSQL database.


###### Let's go on and install flask migrate from pip
{% highlight bash %}
pip install flask_migrate
{% endhighlight %}


Edit `votr.py` and import the `Migrate` class from `flask_migrate`

This is how `votr.py` looks like now

{% highlight python %}
from flask import Flask, render_template, request, flash, session, redirect, url_for, jsonify
from werkzeug.security import generate_password_hash, check_password_hash
from flask_migrate import Migrate
from models import db, Users, Polls, Topics

votr = Flask(__name__)

# load config from the config file we created earlier
votr.config.from_object('config')

# create the database
db.init_app(votr)
db.create_all(app=votr)

migrate = Migrate(votr, db)


@votr.route('/')
def home():
    return render_template('index.html')


@votr.route('/signup', methods=['GET', 'POST'])
def signup():
    if request.method == 'POST':

        # get the user details from the form
        email = request.form['email']
        username = request.form['username']
        password = request.form['password']

        # hash the password
        password = generate_password_hash(password)

        user = Users(email=email, username=username, password=password)

        db.session.add(user)
        db.session.commit()

        flash('Thanks for signing up please login')

        return redirect(url_for('home'))

    # it's a GET request, just render the template
    return render_template('signup.html')


@votr.route('/login', methods=['POST'])
def login():
    # we don't need to check the request type as flask will raise a bad request
    # error if a request aside from POST is made to this url

    username = request.form['username']
    password = request.form['password']

    # search the database for the User
    user = Users.query.filter_by(username=username).first()

    if user:
        password_hash = user.password

        if check_password_hash(password_hash, password):
            # The hash matches the password in the database log the user in
            session['user'] = username

            flash('Login was succesfull')
    else:
        # user wasn't found in the database
        flash('Username or password is incorrect please try again', 'error')

    return redirect(url_for('home'))


@votr.route('/logout')
def logout():
    if 'user' in session:
        session.pop('user')

        flash('We hope to see you again!')

    return redirect(url_for('home'))


@votr.route('/polls', methods=['GET'])
def polls():
    return render_template('polls.html')


@votr.route('/api/polls', methods=['GET', 'POST'])
# retrieves/adds polls from/to the database
def api_polls():
    if request.method == 'POST':
        # get the poll and save it in the database
        poll = request.get_json()

        return "The title of the poll is {} and the options are {} and {}".format(poll['title'], *poll['options'])

    else:
        # return dict representation of our API
        polls = Topics.query.join(Polls).all()
        all_polls = [poll.to_json() for poll in polls]

        return jsonify(all_polls)

{% endhighlight %}

Did you notice that at the end of the file we don't have the `if __name__ == "__main__"` condition again? we don't need that statement again because there is an alternate way to run flask applications (From Flask 0.11). This new method is more flexible and enables us to easily run the new migration commands Flask Migrate has exposed to us.

So you wont be running your application with `python3 votr.py` anymore instead you'll use the `flask run` command, but for that to work properly we need to setup some basic info that tells flask how to find our app and run it.

from your terminal type in the following

{% highlight bash %}
export FLASK_APP=votr.py
export FLASK_DEBUG=1   
{% endhighlight %}

This set's the flask app to votr and then turns on debug mode

*To prevent yourself from typing this anytime you want to work on votr (possibly after a reboot) just add those two lines
to your activate script for your virtualenv. located in the bin directory in your virtualenv's root folder.

Anytime you run `source/bin/activate` the correct app and the debug mode would be set.*



To run the initial database migration, use this:
{% highlight bash %}
flask db init
{% endhighlight %}

You'll be instructed to edit the alembic file in the migrations folder. (You don't really need to, for a simple database like yours)

Then generate the initial migration script with:

{% highlight bash %}
flask db migrate
{% endhighlight %}

After doing this you can now make the required change to the `Topics` model and also add the status field to the `to_json` method (We also added a default value of `1` to the field so any poll that created is automatically opened for voting)

{% highlight python %}
# Model for poll topics
class Topics(Base):
    title = db.Column(db.String(500))
    status = db.Column(db.Boolean, default=1) # to mark poll as open or closed should be under title not polls

    # user friendly way to display the object
    def __repr__(self):
        return self.title

    # returns dictionary that can easily be jsonified
    def to_json(self):
        return {
                'title': self.title,
                'options':
                    [{'name': option.option.name, 'vote_count': option.vote_count}
                        for option in self.options.all()],
                'status': self.status
            }
{% endhighlight %}

Now run

{% highlight bash %}
flask db migrate
flask db upgrade
{% endhighlight %}

You should oops!

`sqlalchemy.exc.OperationalError: (sqlite3.OperationalError) near "DROP": syntax error [SQL: 'ALTER TABLE polls DROP COLUMN status']`

Don't worry this problem isn't because you did something wrong it's because SQLite doesn't support [SQL alter statements](https://www.sqlite.org/lang_altertable.html) fully. Gladly there is a workaroud in alembic > 0.7 that simulates this alter statements:

<ul class="postlist">

  <li>Renaming the table</li>

  <li>Creating a new table with the old name of the previous table with the new column</li>

  <li>Copying data over from the old table to the new one</li>

  <li>And finally dropping the old table</li>

</ul>


This procedure is called a *Batch Migration* and it's not perfect in alembic and most times you'll still need to edit the migrations file yourself to make it work. but don't panic I'll hold your hand as we go.

*Alembic is what actually handles the database migration, Flask Migrate is just a library that provides better integration with flask*


Edit `votr.py` and include the new parameter `render_as_batch`, this tells alembic to use batch migrations from now on

{% highlight python %}
migrate = Migrate(votr, db, render_as_batch=True)
{% endhighlight %}

Setting `render_as_batch` to  `True` should prevent this issue from happening in subsequent migrations, but we didn't apply this fix before the initial migration, so we have to manually edit the migration file and fix that.

Go the the `migrations/versions` folder and you should see a python file with a random number as the name (mine was `347f4ec5eb5e_.py`, yours would be different), this is the migration script that was automatically generated by alembic. The migration script tells alembic exactly what to do when you run `flask upgrade`. A new one is generated anytime you change your models and run `flask migrate`.

If you're familiar with alembic, you could easily write one yourself.


Open the file and Edit the upgrade function to look like this:

{% highlight python %}
def upgrade():
    ### commands auto generated by Alembic - please adjust! ###
    with op.batch_alter_table("polls") as batch_op:
        batch_op.drop_column('status')
    op.add_column('topics', sa.Column('status', sa.Boolean(), default=1))
    ### end Alembic commands ###
{% endhighlight %}

We aren't going to touch the `downgrade` function because we have no plans of going back to the previous schema of our database before the migration

After doing all this you should be able to run `flask db upgrade` without any issues.

Hurray! you just did your first database migration!

In the next section we're going to learn about some extra commands in the flask cli.

### Flask CLI
Normally you'd start a python interactive shell by typing `python` in the terminal.

Since Flask 0.11 the flask shell and some new commands where added to make working with flask easier.

Now you can start an interactive python shell with:

{% highlight bash %}
flask shell
{% endhighlight %}


The `flask shell` command starts up an interactive Python shell like you'd expect but it also sets up the correct application context and some variables for you. Now You don't need to import `votr` anymore it's available to you as variable called `app`

Let's update the previous polls we created earlier and make them all open for voting.

{% highlight bash %}
>>> for topic in Topics.query.all():
...     topic.status = 1
...
>>> from votr import db
>>> db.session.commit()

{% endhighlight %}

That's it. All our existing polls are now open for voting

You can now run the application with

{% highlight bash %}
$ flask run
{% endhighlight %}

### Testing the API
To test our API we need to be able to make `requests` to it. Python already has an excellent module that has the same name `requests`. you can also use `curl` if you're familiar with it.


Install `requests` with

{% highlight bash %}
pip install requests
{% endhighlight  %}

Based on what we discussed earlier about SQLAlchemy and Join statements, this is how our `/api/polls` route looks like like now:

{% highlight python %}
@votr.route('/api/polls', methods=['GET', 'POST'])
def api_polls():
    if request.method == 'POST':
        # get the poll and save it in the database
        poll = request.get_json()

        # simple validation to check if all values are properly secret
        for key, value in poll.items():
            if not value:
                return jsonify({'error': 'value for {} is empty'.format(key)})

        title = poll['title']
        options_query = lambda option : Options.query.filter(Options.name.like(option))

        options = [Polls(option=Options(name=option)) for option in poll['options']
                  ]

        new_topic = Topics(title=title, options=options)

        db.session.add(new_topic)
        db.session.commit()

        return jsonify({'message': 'Poll was created succesfully'})

    else:
        # it's a GET request, return dict representations of the API
        polls = Topics.query.join(Polls).all()
        all_polls = {'Polls':  [poll.to_json() for poll in polls]}

        return jsonify(all_polls)
{% endhighlight %}


###### Open another Python shell, import the requests module and make a get request to the url:

{% highlight bash %}
>>> import requests
>>> requests.get('http://localhost:5000/api/polls')
<Response [200]>
{'Polls': [{'status': None, 'options': [{'name': 'Arsenal', 'vote_count': None}, {'name': 'Spurs', 'vote_count': None}], 'title': 'Which side is going to win the EPL this season'}, {'status': None, 'options': [{'name': 'Liverpool FC', 'vote_count': None}, {'name': 'Manchester city', 'vote_count': None}], 'title': 'Whos better liverpool or city'}]}
<class 'list'>

{% endhighlight %}

Nice we got what we expected from our API.

The requests module even has a nifty `json` method that converts the JSON string in the response to a python dictionary.

Let's make another request, this time a POST requests with incomplete info, to test if our validation for POST requests work, and if we can eventually create a new poll when we supply the right information.

{% highlight python %}
>>> requests.post('http://localhost:5000/api/polls', json={"title": "who's the fastest footballer","options": []}).json()
{'error': 'value for options is empty'}
>>> requests.post('http://localhost:5000/api/polls', json={"title": "","options": ["Gareth Bale"]}).json()
{'error': 'value for title is empty'}
>>> requests.post('http://localhost:5000/api/polls', json={"title": "who's the fastest footballer","options": ["Hector bellerin", "Gareth Bale", "Arjen robben"]}).json()
{'message': 'poll was created succesfully'}

{% endhighlight %}


It worked! we tested our validation by supplying invalid JSON and we got a JSON response back telling us our request failed because a particular field was empty.

and then we finally gave it the right data and it succesfully created the poll. Let's make a GET request to be sure the poll got saved in the database.

{% highlight python %}
>>> requests.get('http://localhost:5000/api/polls').json()
{'Polls': [{'options': [{'name': 'Arsenal', 'vote_count': 3}, {'name': 'Spurs', 'vote_count': 2}], 'title': 'Which side is going to win the EPL this season', 'status': True, {'options': [{'name': 'Liverpool FC', 'vote_count': 0}, {'name': 'Manchester city', 'vote_count': 0}], 'title': 'Whos better liverpool or city', 'status': True}, {'options': [{'name': 'Hector bellerin', 'vote_count': 0}, {'name': 'Gareth Bale', 'vote_count': 0}, {'name': 'Arjen robben', 'vote_count': 0}, {'name': 'Hector bellerin', 'vote_count': 0}, {'name': 'Gareth Bale', 'vote_count': 0}, {'name': 'Arjen robben', 'vote_count': 0}], 'title': "who's the fastest footballer", 'status': True}]}
>>>

{% endhighlight %}

Congratulations! you've just built a fully functional RESTful API, that accepts JSON and also sends JSON data back when it's queried

You've now provided a universal means for any client to talk to your application...The client could be a mobile app, a Desktop app or another website!


### Another Migration
Now we want to make sure duplicate options don't get stored in our database, to do that we're going to make the `name` field in the `Options` table column unique, and run another migration.

Change the field declaration in the `Options` model to this


{% highlight python %}
# Model for poll options
class Options(Base):
    name = db.Column(db.String(200), unique=True)
{% endhighlight %}


Now run:
{% highlight bash %}
$ flask db migrate
{%endhighlight%}


Edit the new migrations file and change the `upgrade` method to

{% highlight python %}
def upgrade():
    ### commands auto generated by Alembic - please adjust! ###
    with op.batch_alter_table('options', schema=None) as batch_op:
        batch_op.create_unique_constraint('unique_user_name', ['name'])

    ### end Alembic commands ###
{% endhighlight %}

This seems like a lot work for something simple, but don't get it wrong. Alembic isn't as painful to work with as this. We don't have to edit the migrations file for every migration we do (*All these issues are as a result of us using SQLite and batch migrations*), if you use a more robust database like MySQL or PostgreSQL you'll hardly come across migration issues like this.

But it's advised to always inspect the migration file before you do any migration because sometimes alembic doesn't detect all the changes made in a model.



### Link existing options
We've finally set a unique constraint on our `Options` table so duplicate entries won't be allowed.

Resend the previous poll we just created

{% highlight python %}
>>> requests.post('http://localhost:5000/api/polls', json={"title": "who's the fastest footballer","options": ["Hector bellerin", "Gareth Bale", "Arjen robben"]}).json()
{% endhighlight %}

You should get a `JSONDecodeError`, because we didn't return any valid JSON back TO inform us that a certain option in the poll already existed. Rather SQLAlchemy raised an exception. switch back to the flask terminal and you should see this:

{% highlight python %}
sqlalchemy.exc.IntegrityError: (sqlite3.IntegrityError) UNIQUE constraint failed: options.name [SQL: 'INSERT INTO options (date_created, date_modified, name) VALUES (CURRENT_TIMESTAMP, CURRENT_TIMESTAMP, ?)'] [parameters: ('Hector bellerin',)]
{% endhighlight %}

This means our unique constraint worked!

But instead of returning a messsage back saying *xxxx poll already existed* it's better to link that existing option to the new polls. For example "Gareth bale" is a valid option for The "Fastest player in the world" and "The best footballer in the world" (hopefully when Messi and Cristiano Ronaldo retire).

We should only return a message that an option already existed when we're trying to create an option directly not when it's part of a poll. (Our API doesn't support that feature currenty).

So if an option exists in the database instead of failing with an `IntegrityError` or returning a message back, let's link the old option with a new poll to be created.

###### We only need to change one thing, the dictionary comprehension that creates Poll objects from Option objects

{% highlight python %}
options = [Polls(option=Options(name=option))
              if options_query(option).count() == 0
              else Polls(option=options_query(option).first()) for option in poll['options']
          ]
{% endhighlight %}

The list comprehension reads as follows *Make a Poll object out of an option if that option doesn't exist in the database (`count == 0`), if exists in the database (`else`) then get that option and make a Poll object from it.*

Make another post request and it should tell us the Poll was inserted successfully, even though "Gareth Bale" was part of the options

{% highlight python %}
>>> requests.post('http://localhost:5000/api/polls', json={"title": "who's the best footballer on the planet","options": ["Lionel Messi", "Gareth Bale", "Cristiano Ronaldo", "Alexis sanchez"]}).json()
{'message': 'Poll was created succesfully'}
{% endhighlight %}

and we should get a json string telling us that it was created successfully

Lastly we need to build a new API endpoint that returns a list of all the options we have in our database, so as our users are trying to create a poll. They can see the list of all the options in our database in a drop down

Add a new endpoint to the `votr.py`

{% highlight python %}
@votr.route('/api/polls/options')
def api_polls_options():

    all_options = [option.to_json() for option in Options.query.all()]

    return jsonify(all_options)
{% endhighlight %}

Now add the `to_json` method on the Options model

{% highlight python %}
# Model for poll options
class Options(Base):
    name = db.Column(db.String(200), unique=True)

    def __repr__(self):
        return self.name

    def to_json(self):
        return {
                'id': uuid.uuid4() # Generates a random uuid
                'name': self.name
        }

{% endhighlight %}

instead of sending the database id of the record back to the user we sent back a uuid (Universally unique identification), a randomly generated number instead. Why did we do this?

It's for security reasons. You should expose as little information about the inner workings and structure of your application as possible (though this makes it harder to accomplish some tasks) to make it difficult for hackers to know how your application works and prevent attacks. Your application might be secure in all the common areas where web applications can be exploited but you might be using a design pattern itself that endangers your application. If hackers know you're using this so called *vulnerable* design pattern, they can easily adopt another approach and hack your web app or gain access to information they weren't supposed to see.


Let's test it

{% highlight python %}
>>> requests.get('http://localhost:5000/api/polls/options').json()
[{'id': '5f4ceb02-bafa-4910-a156-9c3972a1d123', 'name': 'Arsenal'}, {'id': 'dba2948f-f9a1-49d9-ac70-6102b90ffa52', 'name': 'Spurs'}, {'id': '8dadd3d6-f723-4af9-a6b6-503ef85f41ca', 'name': 'Liverpool FC'}, {'id': 'a314955c-3453-4491-a14a-193bdf80db79', 'name': 'Manchester city'}, {'id': '867f16a2-52dc-4a77-84d1-9f8e52a7728b', 'name': 'Hector bellerin'}, {'id': '2ec9e28a-bb66-49ab-9cdf-da5d9f10c076', 'name': 'Gareth Bale'}, {'id': '4d0a25a5-e2f1-4505-87bc-aeca23db6597', 'name': 'Arjen robben'}, {'id': '0b0e85f8-ac58-4c64-9c4d-de890cb92f41', 'name': 'Lionel Messi'}, {'id': '9bc0acae-fa15-42db-8d28-52a27757d6c0', 'name': 'Cristiano Ronaldo'}, {'id': '40b6b016-53c2-421d-8bfa-6b6ee9d14c4b', 'name': 'Alexis sanchez'}]
>>>

{% endhighlight %}


What's left is to make sure our users can vote

To achieve this we'll make another api endpoint and this time send a `PATCH` request. a PATCH request unlike post is used to update existing information and not to create a new one entirely

{% highlight python %}

@votr.route('/api/poll/vote', methods=['PATCH'])
def api_poll_vote():

    poll = request.get_json()

    poll_title, option = (poll['poll_title'], poll['option'])

    join_tables = Polls.query.join(Topics).join(Options)
    # filter options
    option = join_tables.filter(Topics.title.like(poll_title)).filter(Options.name.like(option)).first()

    # increment vote_count by 1 if the option was found
    if option:
        option.vote_count += 1
        db.session.commit()

        return jsonify({'message': 'Thank you for voting'})

    return jsonify({'message': 'option or poll was not found please try again'})

{% endhighlight %}


{% highlight python %}
>>> requests.patch('http://localhost:5000/api/poll/vote', json={"poll_title": "who's the fastest footballer", "option": "Hector bellerin"}).json()
{'message': 'Thank you for voting'}
>>> requests.patch('http://localhost:5000/api/poll/vote', json={"poll_title": "who's the fastest footballer", "option": "Mertesacker"}).json()
{'message': 'option or poll was not found please try again'}
>>>

{% endhighlight %}


## Conclusion
Well, well well that's it for this part, if you followed this tutorial through the hoops of annoying database migrations with SQLite. Then you should take sometime and congratulate yourself because you've just built your first REST API with flask, expanded your knowledge about SQLAlchemy and also learnt a little about alembic and how to fix migrations that go wrong.

Our API isn't complete yet. There is no link between the users and the polls they created or a way to track and prevent users from voting twice. Infact our API currently has no link to our Authentication system. This tutorial was meant to be a gentle introduction to building API's with flask, so we're still going to integrate our API with our auth system later on.

In the next part of this series, we're going to re-visit ReactJS and hook our UI to all the API calls we've been making with the requests module.

If you don't have any questions or observations, I'll see you in the next part.
