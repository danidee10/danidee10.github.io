---
layout: post
title: Flask by example (How to build a Voting app) Part 2
date: 2016-09-19T22:35:33+01:00
---

Welcome to part 2 of the series 'How to build a voting app with flask', in the previous tutorial, we setup our development environment and installed flask. we also wrote a basic hello world application just to see if everything was installed properly.

Now we're going to start building our database, which is the last layer in our voting app. I found this amazing [tool](http://ondras.zarovi.cz/sql/demo/?keyword=default) which i used to visualize how the schema of our database would look like, it's very lightweight and it has no dependencies, it just runs in your browser.

#### Visual representation of our schema
![Database schema](/images/votr.png)

From the image our schema is pretty clear, we have three tables, the *topics* table contains all the various topics we're going to be voting on, for example "***Is Alex iwobi better than Dele Alli?***"

The *options* table are the voting options the users can choose when voting which would be (***Alex iwobi*** or ***Dele Alli***)

Lastly, the *polls* table. this table serves as a *'join'* table (ManyToMany relationship) between *topics* and *options*, this table is needed because of the nature of the voting system where each option can belong to more than one topic at the same time, (we could be comparing ***Dele Alli*** to ***Messi*** at the same time as our first poll). if we don't do this we'll end up creating several ***Dele Alli's*** in our tables for the other topics where his name is used which would defeat the [DRY](https://en.wikipedia.org/wiki/Don%27t_repeat_yourself) principle.

Next step is to convert our schema to code.

#### Flask SQLAlchemy
Since flask is a micro framework, it doesn't provide any special *batteries* out of the box, so the option is left to the programmer to choose the *batteries* he wants for his application. Traditionally, before ORM's (Object Relational mapping) came to make our lives easier, programmers wrote plain SQL and programming languages provided libraries for working with different database (Python for example has libraries for working with sqlite out of the box). You'd typically pass in a valid SQL query to the Database and then have the results returned back to you, stored in an arrays or some container data structure for the programmer to work with, but as applications and data-sets grew larger it became a huge pain to maintain or even look at code that had a lot of SQL running everywhere.

A perfect example is this:

{% highlight php %}
$objQuery = $this->db->query

( "SELECT rd.*, ((rd.rd_numberofrooms) - (select sum(rn.reservation_numberofrooms) as count_reserve_room from reservation as rn WHERE rn.reservation_rd_id = rd.rd_id AND (str_to_date('$data_Check_in_date','%d-%m-%y') BETWEEN str_to_date(rn.reservation_check_in_date,'%d-%m-%y') AND str_to_date(rn.reservation_check_out_date,'%d-%m-%y') OR str_to_date('$data_Check_out_date','%d-%m-%y') BETWEEN str_to_date(rn.reservation_check_in_date,'%d-%m-%y') AND str_to_date(rn.reservation_check_out_date,'%d-%m-%y') OR str_to_date('$data_Check_in_date','%d-%m-%y') <= str_to_date(rn.reservation_check_in_date,'%d-%m-%y') AND str_to_date('$data_Check_out_date','%d-%m-%y') ) )) as reserve

FROM room_details rd LEFT JOIN reservation rn ON rd.rd_id = rn.reservation_rd_id WHERE NOT EXISTS

(

SELECT rn.* FROM reservation rn WHERE rn.reservation_rd_id = rd.rd_id

AND (str_to_date('$data_Check_in_date','%d-%m-%y') BETWEEN str_to_date(rn.reservation_check_in_date,'%d-%m-%y') AND str_to_date(rn.reservation_check_out_date,'%d-%m-%y') OR str_to_date('$data_Check_out_date','%d-%m-%y') BETWEEN str_to_date(rn.reservation_check_in_date,'%d-%m-%y') AND str_to_date(rn.reservation_check_out_date,'%d-%m-%y') OR str_to_date('$data_Check_in_date','%d-%m-%y') <= str_to_date(rn.reservation_check_in_date,'%d-%m-%y') AND str_to_date('$data_Check_out_date','%d-%m-%y') >= str_to_date(rn.reservation_check_out_date,'%d-%m-%y'))

AND (rd.rd_numberofrooms <= (select sum(rn.reservation_numberofrooms) as count_reserve_room from reservation as rn WHERE rn.reservation_rd_id = rd.rd_id AND (str_to_date('$data_Check_in_date','%d-%m-%y') BETWEEN str_to_date(rn.reservation_check_in_date,'%d-%m-%y') AND str_to_date(rn.reservation_check_out_date,'%d-%m-%y') OR str_to_date('$data_Check_out_date','%d-%m-%y') BETWEEN str_to_date(rn.reservation_check_in_date,'%d-%m-%y') AND str_to_date(rn.reservation_check_out_date,'%d-%m-%y') OR str_to_date('$data_Check_in_date','%d-%m-%y') <= str_to_date(rn.reservation_check_in_date,'%d-%m-%y') AND str_to_date('$data_Check_out_date','%d-%m-%y') ) ) )

)

");

{% endhighlight %}

Imagine having hundreds of this everywhere in your web app. it's bound to happen as your data sets and your application grows.

Some people got fed up and decided to put another layer of abstraction to make working with SQL easier for themselves and everyone else. so basically ORM is a technique in programming that allows you to interact with your database in an object oriented fashion. something like this:

{% highlight python %}
>>> dele_alli = Option(name='Delle Alli', club='Spurs')
>>> dele_alli.club
>>>'Spurs'
>>> dele_alli.save()
{% endhighlight %}

[SQLAlchemy](http://www.sqlalchemy.org/) is a very mature ORM that's very popular in the python community, there are other [ORM's]https://www.fullstackpython.com/object-relational-mappers-orms.html for python, but we'll stick with SQLAlchemy due to it's sheer popularity and excellent documentation.

[Flask SQLAlchemy](flask-sqlalchemy.pocoo.org/2.1/) is a flask extension that makes it easier for us to work with SQLAlchemy directly in flask (You can use plain SQLAlchemy without the flask extension if you want to). we're going to install it now with `pip`.

###### Make sure your virtualenv is activated
{% highlight bash %}
$ pip install flask_sqlalchemy
{% endhighlight %}

We're now ready to start building our database with SQLAlchemy. If you've read the flask docs you should know that all our code can live in one file, but we won't be stupid just because we were given the license to be stupid :). so we're going to seperate files that'll only contain code for a common thing they achieve.

###### Create a new file called `models.py` in the `votr` directory
{% highlight bash %}
$ touch models.py
{% endhighlight %}

###### Open up your favourite text-editor and type in the following
{% highlight python %}
from flask_sqlalchemy import SQLAlchemy

# create a new SQLAlchemy object
db = SQLAlchemy()

# Base model that for other models to inherit from
class Base(db.Model):
    __abstract__ = True

    id = db.Column(db.Integer, primary_key=True, autoincrement=True)
    date_created = db.Column(db.DateTime, default=db.func.current_timestamp())
    date_modified = db.Column(db.DateTime, default=db.func.current_timestamp(),
            onupdate=db.func.current_timestamp())

# Model for poll topics
class Topics(Base):
    title = db.Column(db.String(500))

# Model for poll options
class Options(Base):
    name = db.Column(db.String(200))

# Polls model to connect topics and options together
class Polls(Base):

    # Columns declaration
    topic_id = db.Column(db.Integer, db.ForeignKey('topics.id'))
    option_id = db.Column(db.Integer, db.ForeignKey('options.id'))
    vote_count = db.Column(db.Integer)
    status = db.Column(db.Boolean) # to mark poll as open or closed

    # Relationship declaration (makes it easier for us to access the polls model
    # from the other models it's related to)
    topic = db.relationship('Topics', foreign_keys=[topic_id],
            backref=db.backref('options', lazy='dynamic'))
    option = db.relationship('Options',foreign_keys=[option_id])

    def __repr__(self):
        # a user friendly way to view our objects in the terminal
        return self.option.name
{% endhighlight %}

To properly use `flask_sqlalchemy`, we have to create a configuration file which tells flask the type of database we're making use of and how to connect it, we can also use our configuration file to store site-wide information (Like the password to our admin account!). we're going to use sqlite as the database for votr because it's lightweight and easy to setup.

But seriously never store passwords or sensitive information of any kind in a plain file or unencrypted in your database. Never!

###### Create a new file `config.py` and include the following

{% highlight python %}
#configuration file for votr
import os

DB_PATH = os.path.join(os.path.dirname(__file__), 'votr.db')
SECRET_KEY = 'development key' # keep this key secret during production
SQLALCHEMY_DATABASE_URI = 'sqlite:///{}'.format(DB_PATH)
SQLALCHEMY_TRACK_MODIFICATIONS = False
DEBUG = True
{% endhighlight %}

To find out more about the configuration parameters you can use for sqlalchemy you can read the docs [here](http://flask-sqlalchemy.pocoo.org/2.1/config/)
For now we'll just keep everything simple.

###### Edit the `votr.py` file we created earlier to include the following:

{% highlight python %}
from flask import Flask
from models import db

votr = Flask('__name__')

# load config from the config file we created earlier
votr.config.from_object('config')

# initialize and create the database
db.init_app(votr)
db.create_all(app=votr)

@votr.route('/')
def home():
    return 'hello world'

if __name__ == '__main__':
    votr.run()

{% endhighlight %}

We imported the `db` we created in our `models.py` file, and called `db.init_app(votr)` and `db.create_all(app=votr)` to bind sqlalchemy to our flask app and create the database tables we specified in our `models.py` file. Note that `db.create_all` doesn't actually create our database, it would simply create the required tables we specified in our `models.py` file on an existing database (Remember our `config.py` file where we specified the location of our database).

We're going to create that db file now.

###### If you have python installed properly you should have sqlite3, if for some reason you don't have it, you can install it with:

{% highlight bash %}
$ sudo apt-get install sqlite3
{% endhighlight %}

###### Now lets create our database
{% highlight bash %}
$ sqlite3
SQLite version 3.14.2 2016-09-12 18:50:49
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> .save votr.db
sqlite>
{% endhighlight %}

###### Press Ctrl + D or Ctrl + C to exit.

###### This creates a new db called `votr.db` in the `votr` directory, now run the `votr.py` with python.

{% highlight bash %}
$ python3 votr.py
{% endhighlight %}

###### This should create all the models we specified in our `models.py` file as tables in our database, we can confirm that by using sqlite3 again from the terminal.

{% highlight bash %}
$ sqlite3
SQLite version 3.14.2 2016-09-12 18:50:49
Enter ".help" for usage hints.
Connected to a transient in-memory database.
Use ".open FILENAME" to reopen on a persistent database.
sqlite> .open votr.db
sqlite> .tables
options  polls    topics
{% endhighlight %}

when we run `.tables`, we can see the three tables we specified in our `models.py` file.

Let's play with SQLalchemy a little.

###### Open a new python shell

{% highlight python %}
>>> from votr import db, votr
>>> from models import Options, Topics, Polls
>>> topic = Topics(title='Which side is going to win the EPL this season')
>>> arsenal = Options(name='Arsenal')     
>>> spurs = Options(name='Spurs')                   
>>> poll = Polls(topic=topic, option=arsenal)
>>> poll1 = Polls(topic=topic, option=spurs)  
>>> topic.options.all()
[Arsenal, Spurs]
>>> poll.vote_count = 3
>>> poll1.vote_count = 2
'Arsenal'
>>> for option in topic.options.all():
...     print(option, option.vote_count)
...
Arsenal 3
Spurs 2
{% endhighlight %}

From the example above, we're already interacting with our database without writing any SQL!, we created a topic, two poll objects, and the `backref` we created earlier in our `models.py` file, made it possible for us to access all the options under our topic. by simply calling `topics.options.all()`.

Note that everything we've done isn't actually stored in our database yet, we'll have to add it to the SQLalchemy session and then commit `commit()` before it would be written to our database. it would be a disaster if everything we experimented with is automatically commited to our database by default.

{% highlight python %}
>>> db.app = votr
>>> db.init_app(votr)
>>> db.create_all()
>>> db.session.add(arsenal)
>>> db.session.commit(arsenal)
>>> db.session.commit()
{% endhighlight %}

###### If you're still reeling in confusion here's another example to make you understand better:

{% highlight bash %}
>>> city = Options(name='Manchester city')
>>> liverpool = Options(name='Liverpool FC')
>>> liverpool = Polls(option=liverpool)
>>> city = Polls(option=city)
>>> new_topic = Topics(title='Whos better liverpool or city', options=[liverpool, city])
>>> new_topic.options.all()
[Liverpool FC, Manchester city]
>>> liverpool.topic
<models.Topics object at 0x7f797135ef28>
>>> liverpool.topic.name
"Who's better liverpool or city"
>>>
{% endhighlight %}

This shows that you can pass in a list of options to a topic, thanks to `backref` again!

Well that's it for this part!
We were able to set up our database and get a feel of how SQLalchemy works, in the next part we're going to build the front-end for our users, so they can easily add polls to votr easily.

SQLAlchemy can be quite confusing sometimes, but if you stick with it and just take things one step at a time it's only going to get easier. To see more examples of long SQL queries checkout this [quora](https://www.quora.com/Whats-the-most-complex-SQL-query-you-ever-wrote) post.

If you have any questions or insights please use the comment box below. See you in the next part!
