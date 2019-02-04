---
layout: post
title: Flask by example 10 (Testing the application)
date: 2016-12-12T20:31:12+01:00
excerpt:
      Believe it or not, writing tests can help you structure your application properly which eventually leads to better code.
      Some developers even advise others developers to write tests before the main program, as it helps you envision how your functions/methods are going to look and work before you start writing the main program.
tags: Flask Unit-testing Testing nose
---

### Why testing is important

These are some of the reasons why testing in all it's forms is important:

<ul class="postlist">

  <li>
    It helps developers make modifications and deploy code with more confidence. If all your tests pass for a new feature you just added to your app, you can be sure that the new feature did not break any existing functionality.

    <br><br>
    <image src="/images/flask-10-notsure.png" alt="Not sure" />
  </li>

  <br>

  <li>
    Believe it or not, writing tests can help you structure your application properly which eventually leads to better code.

    Some developers even advise others developers to write tests before the main program, as it helps you envision how your functions/methods are going to look and work before you start writing the main program.
  </li>

  <br>

  <li>
    It speeds up development time; at the beginning of a project the extra time taken to write tests slows the whole development down, but in the long run it eventually makes it faster as all the time that would have been spent debugging errors as a result of a change in codebase would have been discovered before that change was even made.
  </li>

  <br>

  <li>
  There's this "Beast mode" feeling i get anytime my tests pass (especially on Open source projects i contribute to)...Lol
  </li>

</ul>

<br>

To start our tests, we're going to install a test runner for python called [nose](https://nose.readthedocs.io/en/latest/):

{% highlight bash %}
$ pip install nose
{% endhighlight %}

you can use the de-facto `unitTest` module if you want to, but nose comes with a lot of sweet features. I really love the `autodiscover` feature which automatically discovers test files in your application and runs them. the filename just has to conform to this regex `((?:^|[\\b_\\.-])[Tt]est`.

Create a new file named `api_test` with the following content and place it in the `api` folder (You can place it anywhere you like, it's up to you).

{% highlight python %}

from votr import votr, db, celery
from multiprocessing import Process
import requests
import os
import time
from tasks import close_poll


class Testvotr():

    @classmethod
    def setUpClass(cls):
        votr.config['DEBUG'] = False
        votr.config['TESTING'] = True
        cls.DB_PATH = os.path.join(os.path.dirname(__file__), 'votr_test.db')
        votr.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///{}'.format(cls.DB_PATH)
        celery.conf.update(CELERY_ALWAYS_EAGER=True)
        cls.hostname = 'http://localhost:7000'
        cls.session = requests.Session()

        with votr.app_context():
            db.init_app(votr)
            db.create_all()
            cls.p = Process(target=votr.run, kwargs={'port': 7000})
            cls.p.start()
            time.sleep(2)


    def setUp(self):
        self.poll = {"title": "who's the fastest footballer",
                     "options": ["Hector bellerin", "Gareth Bale", "Arjen robben"],
                     "close_date": 1581556683}

    def test_create_user(self):
        signup_data = {'email': 'admin@gmail.com', 'username': 'Administrator',
                       'password': 'admin'}

        result = requests.post(self.hostname + '/signup', data=signup_data).text

        assert 'Thanks for signing up please login' in result

    def test_login(self):

        # Login data
        data = {'username': 'Administrator', 'password': 'admin'}
        result = self.session.post(self.hostname + '/login', data=data).text

        assert 'Create a poll' in result

    def test_empty_option(self):
        result = requests.post(self.hostname + '/api/polls',
                               json={"title": self.poll['title'],
                                     "options": []}).json()
        assert {'message': 'value for options is empty'} == result

    def test_empty_title(self):
        result = requests.post(self.hostname + '/api/polls',
                               json={"title": "",
                                     "options": self.poll['options']}).json()
        assert {'message': 'value for title is empty'} == result

    def test_new_poll(self):
        result = requests.post(self.hostname + '/api/polls', json=self.poll).json()
        assert {'message': 'Poll was created succesfully'} == result

    def vote(self):
        result = self.session.patch(self.hostname + '/api/poll/vote',
                                    json={'poll_title': self.poll['title'],
                                          'option': self.poll['options'][0]}).json()
        return result

    def test_voting(self):
        result = self.vote()
        assert {'message': 'Thank you for voting'} == result

    def test_voting_twice(self):
        result = self.vote()
        assert {'message': 'Sorry! multiple votes are not allowed'} == result

    def test_zelery_task(self):

        result = close_poll.apply((1, votr.config['SQLALCHEMY_DATABASE_URI'])).get()

        assert 'poll closed succesfully' == result

    @classmethod
    def tearDownClass(cls):
        os.unlink(cls.DB_PATH)
        cls.p.terminate()


{% endhighlight %}


Before the tests are run, we need to setup some basic variables or configuration for our application, this is done by defining the `setUpClass` and/or `setUp` method, the major difference between these two methods is that `setUpClass` is called only once throughout the duration of the tests while `setUp` is called before each test is run.

So use `setUpClass` to set up properties or variables that you want to be accessible to all the test methods in your class which may be too expensive or time-consuming to re-create anytime a new test is about to be run.

Also `setUpClass` is a class method, so it must be decorated with `@classmethod`


<br>

**In the setUpClass method:**

I set `DEBUG` to False to prevent Flask from starting two processes when we're trying to test the application (This is how it's able to detect changes and automatically reload itself when you're running it in debug mode)

While `TESTING` was set to True for better error propagation.

`celery.conf.update(CELERY_ALWAYS_EAGER=True)` with this we don't need to start celery from the command line, the task would be run as if it was a normal function.

We also created a session with `requests.Session`, so we can login to the application and perform some our tests as a logged in user.

At the end of everything, we started a new instance of our application which we would run our tests against.
The small pause (`sleep(2)`) at the end of the method is important, so the flask application has enough time to initialize and listen for connections before we start running our tests, if you don't do this most of the tests won't pass as `requests` will be unable to connect to the application.

{% include article_ads.html %}

Without any explanation, you can guess what the `tearDownClass` method does. There is also another method that wasn't used here `tearDown` which is run at the end of every individual test.

<br>

Now that we've got that out of the way, you can run the tests with:
{% highlight python %}
nostests - v
{% endhighlight %}

nose should be able to discover the file and run it. You should see something like this in the terminal

{% highlight bash %}
(votr) nosetests -v
api.test_api.Testvotr.test_create_user ... ok
api.test_api.Testvotr.test_empty_option ... ok
api.test_api.Testvotr.test_empty_title ... ok
api.test_api.Testvotr.test_login ... ok
api.test_api.Testvotr.test_new_poll ... ok
api.test_api.Testvotr.test_voting ... ok
api.test_api.Testvotr.test_voting_twice ... ok
api.test_api.Testvotr.test_zelery_task ... ok

----------------------------------------------------------------------
Ran 8 tests in 4.731s

OK
{% endhighlight %}


If you're paying attention you might have noticed the weird name of the last test `test_zelery`, this was done intentionally to make the celery test run last. Why?

Because nose and even unitTest run the test alphabetically, which means if we named the test `test_celery` it would have been run first, and at that point we wouldn't have had any poll in our database which means the test would fail.

Technically what we've done isn't unit testing because one of some of our tests depend on each other before they can be run. Tests are supposed to be independent of each other, the order of execution shouldn't matter.

**What advantage is there to this?**

  - *This means that your unit tests can be parallelized (especially if you have a lot of unit tests)*

<br>

**But does that make our tests useless?**

  - *No sometimes you actually want to test like this, and follow a particular process to test something bigger, so it's perfectly fine if that's what you intended.*

<br>

**How can we make this a true unitTest?**

There are two things, we can do:

The first one is to make it a monolithic test and include the code that creates the poll inside the `test_celery` method, but this has another downside, if the code that creates the poll fails or raises an exception for some reason, it would be hard to track down the exact error that made the tests fail.


The second option and the cleanest approach IMO is to move the code that creates the poll into `setUpClass` and have the poll created at the before any test is even run.


**This is the second approach:**

{% highlight python %}

from votr import votr, db, celery
from multiprocessing import Process
import requests
import os
import time
from tasks import close_poll


class Testvotr():

    @classmethod
    def setUpClass(cls):
        votr.config['DEBUG'] = False
        votr.config['TESTING'] = True
        cls.DB_PATH = os.path.join(os.path.dirname(__file__), 'votr_test.db')
        votr.config['SQLALCHEMY_DATABASE_URI'] = 'sqlite:///{}'.format(cls.DB_PATH)
        celery.conf.update(CELERY_ALWAYS_EAGER=True)
        cls.hostname = 'http://localhost:7000'
        cls.session = requests.Session()

        with votr.app_context():
            db.init_app(votr)
            db.create_all()
            cls.p = Process(target=votr.run, kwargs={'port': 7000})
            cls.p.start()
            time.sleep(2)

        # create new poll
        poll = {"title": "Flask vs Django",
                "options": ["Flask", "Django"],
                "close_date": 1581556683}
        requests.post(cls.hostname + '/api/polls', json=poll).json()

        # create new admin user
        signup_data = {'email': 'admin@gmail.com', 'username': 'Administrator',
                       'password': 'admin'}
        requests.post(cls.hostname + '/signup', data=signup_data).text

    def setUp(self):
        self.poll = {"title": "who's the fastest footballer",
                     "options": ["Hector bellerin", "Gareth Bale", "Arjen robben"],
                     "close_date": 1581556683}

    def test_new_user(self):
        signup_data = {'email': 'user@gmail.com', 'username': 'User',
                       'password': 'password'}

        result = requests.post(self.hostname + '/signup', data=signup_data).text

        assert 'Thanks for signing up please login' in result

    def test_login(self):

        # Login data
        data = {'username': 'Administrator', 'password': 'admin'}
        result = self.session.post(self.hostname + '/login', data=data).text

        assert 'Create a poll' in result

    def test_empty_option(self):
        result = requests.post(self.hostname + '/api/polls',
                               json={"title": self.poll['title'],
                                     "options": []}).json()
        assert {'message': 'value for options is empty'} == result

    def test_empty_title(self):
        result = requests.post(self.hostname + '/api/polls',
                               json={"title": "",
                                     "options": self.poll['options']}).json()
        assert {'message': 'value for title is empty'} == result

    def test_new_poll(self):
        result = requests.post(self.hostname + '/api/polls', json=self.poll).json()
        assert {'message': 'Poll was created succesfully'} == result

    def vote(self):
        result = self.session.patch(self.hostname + '/api/poll/vote',
                                    json={'poll_title': self.poll['title'],
                                          'option': self.poll['options'][0]}).json()
        return result

    def test_voting(self):
        result = self.vote()
        assert {'message': 'Thank you for voting'} == result

    def test_voting_twice(self):
        result = self.vote()
        assert {'message': 'Sorry! multiple votes are not allowed'} == result

    def test_celery_task(self):

        result = close_poll.apply((1, votr.config['SQLALCHEMY_DATABASE_URI'])).get()

        assert 'poll closed succesfully' == result

    @classmethod
    def tearDownClass(cls):
        os.unlink(cls.DB_PATH)
        cls.p.terminate()

{% endhighlight %}


Perfect unit tests and no funny names :)

<br>

### Test Coverage

*Now another question arises, how do you know that your application is well tested and that your unit tests reach all the important parts of your code?*

There is a nice python module that does that and more **`Coverage.py`**

It can be installed from pip with:

{% highlight bash %}
$ pip install coverage
{% endhighlight %}

nose makes it easy to use coverage, just run the tests with the `--with-coverage` flag

{% highlight bash %}
nosetests -v --with-coverage
{% endhighlight %}

At the end of the tests you should see a nice output that shows you the number of statements, the number of lines missed and the percentage covered, at the bottom there is also a overall summary of these information.

When i ran this, i got a coverage of **37%**, you should get something similar or better if you wrote some extra tests or even worse if you added a lot of untested code.

<br>

Coverage.py even provides an option for displaying the test results as html:

{% highlight bash %}
coverage html
{% endhighlight %}

a new folder called `htmlcov` should be created in the working directory, inside the folder you'll find html files of all the python files in your project, our `api.py` file should be named `api_api_py.html ` Open it up, you should see something like this:

![api_coverage](/images/flask-10-api_coverage.png)

hmmm **18%** coverage pretty poor right? Yeah, this poor result boils down to the fact that we didn't actually test the methods (call them in our tests), we only tested the routes which in turn called the required methods. You'll even notice that the Lines containing `@api.route` are all highlighted with green.

**There is no recommended percentage for coverage**, so don't get carried away by the numbers, a high coverage percentage means the code is well tested, but most times people get carried away and turn their code into a horrible dissected mess just because they want to get better coverage results.

Imagine us doing this, so we can increase our test coverage by exposing a new function called `poll_vote`:

{% highlight python %}

# actual voting method
def poll_vote(poll):
    poll_title, option = (poll['poll_title'], poll['option'])

    join_tables = Polls.query.join(Topics).join(Options)
    .......

# method for routing only
@api.route('/poll/vote', methods=['PATCH'])
def api_poll_vote():
    poll = request.get_json()

    # call true vote method
    poll_vote(poll)

{% endhighlight %}


Don't do this, First of all the method isn't re-usable, so it shouldn't even be exposed to the outside world, it's more appropriate to make it an inner method of `api_poll_vote` (if you really want to make it new function). If you continue like this, you'll end up having a lot of localized functions splattered everywhere that eventually make the code harder to read that it was before.

**Don't sacrifice readability for better test coverage**

<br>

Not every part of your code needs to be tested, some parts are simply irrelevant or are handled by an underlying library. It would make no sense to write tests for SQLAlchemy in your code.

If you noticed, coverage.py ran tests for a whole lot of files we didn't care about like `config.py` and some libraries that we made use of e.g `SQLAlchemy` and even `requests`!. These extra reports reduced the overall coverage percentage of our code, so you'll probably want to exclude them.

Excluding those unimportant files, the actual code coverage at the time of writing is [57.50%](https://codecov.io/gh/danidee10/Votr) and that's quite decent IMO.

You can exclude files and directories with `coverage.py` but that's out of the scope of this tutorial, The [Coverage docs](http://coverage.readthedocs.io/en/coverage-4.0.3/index.html#) has more information about that.


Aside unit tests, they're still a lot of tests to be carried out on the application:

  - * Load testing
  - * Functional testing
  - * Acceptance testing etc

<br>

but unit tests are at the top of the pile, they're one of the most important tests carried out in the Life cycle of a software project and are mostly written by you the developer who understands the inner-workings of the code.

Non developers or those that aren't familiar with the codebase can still write tests for your code with a tool library [Cucumber](https://cucumber.io/)


***It should be noted that Flask has an [inbuilt testing client](http://flask.pocoo.org/docs/0.11/testing/), i decided to use requests because it's easier to get json responses from the api and i also wanted to test against a real instance of the application.***



In the next part, we're going to round of the series by deploying our application to Heroku and instead of SQLite, we're going to use PostgreSQL.


Thanks for reading.

Don't forget to share this part if you enjoyed it. Cheers!
