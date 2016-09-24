---
layout: post
title: Flask by example (How to build a Voting app) Part 3
date: 2016-09-22T22:35:33+01:00
---

Welcome to part 3 of 'How to build a voting app with flask', the last part was all about setting up our database with SQLAlchemy, if you missed it check it out [here]({% post_url 2016-09-19-flask-by-example-2 %}).

In this part, we're going to design a landing page for our application and an authentication system to allow our users create and vote on topics.


#### Landing page
We're going to use one of [bootstrap's](https://getbootstrap.com) demo pages and customize it a little to suit our needs, before we do that we're going to create some new folders to hold our templates and static files (css, images and javascript).

###### create the required folders
{% highlight bash %}
mkdir -p templates static/css static/images static/js
{% endhighlight %}

###### Download the webpage from bootstrap
{% highlight bash %}
wget -pk http://getbootstrap.com/examples/jumbotron-narrow/
{% endhighlight %}

This would create a folder named `getbootstrap.com` that contains the template and all the required assets for it to render properly.

###### Move the templates and the assets to the right folder so votr can pick them up.
{% highlight bash %}
$ mv getbootstrap.com/examples/jumbotron-narrow/index.html templates
$ mv getbootstrap.com/examples/jumbotron-narrow/jumbotron-narrow.css static/css
$ mv getbootstrap.com/dist/css/bootstrap.min.css static/css
$ mv getbootstrap.com/dist/fonts static
$ mv getbootstrap.com/assets/css/ie10-viewport-bug-workaround.css static/css
$ mv getbootstrap.com/assets/js/* static/js
$ rm getbootstrap.com -r
{% endhighlight %}

I also dowloaded the sample login page from bootstrap [here](https://getbootstrap.com/examples/signin/) and renamed the template to `signup.html`. Repeat the previous steps above, but don't forget to rename the login template from `index.html` to `signup.html`. If you did this correctly your layout should look like this.

{% highlight bash %}
.
├── config.py
├── LICENSE
├── models.py
├── README.md
├── static
│   ├── css
│   │   ├── bootstrap.min.css
│   │   ├── ie10-viewport-bug-workaround.css
│   │   ├── jumbotron-narrow.css
│   │   └── signin.css
│   ├── fonts
│   │   ├── glyphicons-halflings-regular.eot
│   │   ├── glyphicons-halflings-regular.eot?
│   │   ├── glyphicons-halflings-regular.svg
│   │   ├── glyphicons-halflings-regular.ttf
│   │   ├── glyphicons-halflings-regular.woff
│   │   └── glyphicons-halflings-regular.woff2
│   ├── images
│   │   ├── background.jpg
│   │   └── logo.png
│   └── js
│       ├── ie10-viewport-bug-workaround.js
│       └── ie-emulation-modes-warning.js
├── templates
│   ├── index.html
│   └── signup.html
├── votr.db
└── votr.py
{% endhighlight %}

I also included some custom images for the logo and landing page, you can find them in the github [repo](https://github.com/danidee10/Votr) for votr.

We now have a new folder `templates` that contains our html files and a `static` folder and it's subfolders which holds our static files. by default this is where flask looks for templates and static files.

### Rendering templates

flask provides a `render_template` method that allows us to display a template, when a user visits a particular url route, flask doesn't map url's to views like django, the Flask object has a `route` decorator that you can use to decorate a function that will be executed when a user visits a particular url on our website. we're going to make use of that decorator to create a route and render the corresponding template.

{% highlight python %}
from flask import Flask, render_template
from models import db

votr = Flask('__name__')

# load config from the config file we created earlier
votr.config.from_object('config')

# create the database
db.init_app(votr)
db.create_all(app=votr)

@votr.route('/')
def home():
    return render_template('index.html')

@votr.route('/signup')
def signup():
    return render_template('signup.html')

if __name__ == '__main__':
    votr.run()

{% endhighlight %}

If you visit `localhost:5000/signup` in your browser, you should see the `signup.html` rendered in your browser

### User Registraton and Authentication

To register new users to our application, you need a new model to store their details, we'll call that model `Users`

###### Create the User model

{% highlight python %}
# Model to store user details
class Users(Base):
    email = db.Column(db.String(100), unique=True)
    username = db.Column(db.String(50), unique=True)
    password = db.Column(db.String(200))
{% endhighlight %}

The email and username fields are unique to avoid duplicate users in our database.
You might be wondering why the password field is 200 characters long.

We did this because when we hash a user's password we don't want the resulting hash to be truncated by the database, if it gets truncated then we won't be able to authenticate our users as the password they submit (which would also be hashed and compared to the hash in the database) would never match.

### Sign them up
Import the new `Users` model you just created in `votr.py` and use it in your `signup` route.

{% highlight python %}
from flask import Flask, render_template, request, flash, redirect, url_for
from werkzeug.security import generate_password_hash, check_password_hash
from models import db, Users

votr = Flask('__name__')

# load config from the config file we created earlier
votr.config.from_object('config')

# create the database
db.init_app(votr)
db.create_all(app=votr)

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

if __name__ == '__main__':
    votr.run()

{% endhighlight %}

Woah! that's a lot of imports at the top of the file, Let's discuss each of them and their functions:


  **request:** The request object allows the client (users and their browsers) to interact with your application (server). It enables you to get data from the user easily, for example the form details from the form the user submits is stored in the `request` object as a dictionary called `form` that we can access. You can also access the type of request the user made to your route through `request.method`.

  **NOTE:** Take care when you're trying to access values submitted in the form by the user, if you try to access a field that doesn't exist and you don't catch the resulting `keyError` exception, the request will become invalid and flask will fail with a `400 bad request`. for example if you try to get the username of a user which you gave a name of  `username` in your html template with `request.form['USERNAME']` instead of `request.form['username']`.


  **flash:** The name is a little misleading (personally i would've preferred `send_message`). It doesn't flash (show a popup) message to the user as you would expect. what it actually does is store a message at the end of the request and them makes it accessible **ONLY AND ONLY** on the next request.

  It will be discarded after that, so don't try to store messages or data with it. Instead we can use it to pass a message to the next page we're redirecting to, in our templates, all the `flash` messages can be accessed by calling  `get_flashed_messages()`.

  **redirect:** It simply redirects to another page.

  **url_for:** This prevents us from hardcoding the `homepage` route as `/` when calling `redirect`, if you hardcode it, then what happens when you decide to change the route of `home` from `/` to `/index`. this would break all existing links to the homepage in your application, you'll have to go round and change all the hardcoded url's to new url you've specified.


  **generate_password_hash and check_password_hash**: are helpers from [werkzeug](http://werkzeug.pocoo.org/) for hashing and comparing a string with a password hash respectively. The hash is a one way salted hash, meaning we cannot decrypt it back. want to read more about hashing and the meaning of a salted hash read this [Quora](https://www.quora.com/What-does-it-mean-to-add-a-salt-to-a-password-hash) post.

  Security is a very important aspect in web applications and other applications generally (even in our daily life), it's best to use already made solutions that have been tested and trusted by thousands of developers.

  Never use own encryption or hash algorithm no matter how smart and efficient you think it is. You're not a cryptographer, you're a programmer. Allow the cryptographers do their job. Cryptography is a very broad study that include lots of mathematics, computer science and ***dark magic*** which several people have devoted their entire lives to study. you simply don't know more than them.


### User Login

We're going to add a new route `/login` that accepts `POST` requests only and redirects back to the login page with a specific message if a login attempt was successful or not.

{% highlight python %}
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
{% endhighlight %}

### Logout

To log users out, we'll simply perform a check to see if they currently logged in. If they are remove their username from the session and redirect them back to the homepage with a message that they successfully logged out.

{% highlight python %}
@votr.route('/logout')
def logout():
    if 'user' in session:
        session.pop('user')

        flash('We hope to see you again!')

    return redirect(url_for('home'))
{% endhighlight %}


### Jinja2 and front end design
That's all for our simple authentication system but we haven't really talked about the front-end of our application, which is very important to our users, we're going to keep our user interface as simple as possible without discarding most of the functionality that we set out to accomplish at the beginning of this tutorial.

By default flask uses [jinja2](http://jinja.pocoo.org/docs/dev/) as the template engine. `Jinja2` and other python templating language allow us to execute python code in html, reuse templates by inheritance and to generally make working with html easier with flask. Nothing actually stops us from doing this.

{% highlight python %}
@votr.route('/')
def home():
    return """
           <html>
             <head>
             <link rel=\"stylesheet\" href=\"/static/css/style.css\" />
             </head>

             <body>

               <h1>Hello world</h1>

             </body>
           </html>
           """
{% endhighlight %}

You could get killed if other flask developers see this!. On a more serious note, this has a lot of downsides. You'll have to escape all the double quotes, poor handling of urls and links and you can't re-use similar templates by inheritance, i can go on and on but it's very obvious that is just ugly and messy and would be hard to debug because you wouldn't even have html templates! No text editor would be able to work with this ***stringified*** html.

I'll advise you to skim through the [jinja2](http://jinja.pocoo.org/docs/dev/) docs to get a feel of how it works and looks before moving on to the next section.

### Landing page template

Edit the default bootstrap narrow-jumbotron template you downloaded earlier to include a login form. The new template is also going to have a signup button that routes to the `signup.html` template.

{% highlight html %}
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">

    <meta name="description" content="">
    <meta name="author" content="">
    <link rel="icon" href="http://getbootstrap.com/favicon.ico">

    <title>Votr!</title>

    <!-- Bootstrap core CSS -->
    <link href="static/css/bootstrap.min.css" rel="stylesheet">

    <!-- IE10 viewport hack for Surface/desktop Windows 8 bug -->
    <link href="static/css/ie10-viewport-bug-workaround.css" rel="stylesheet">

    <!-- Custom styles for this template -->
    <link href="static/css/jumbotron-narrow.css" rel="stylesheet">

    <script src="static/js/ie-emulation-modes-warning.js"></script>
  </head>

  <body>

    <div class="container">
      <div class="header clearfix">
        <nav>
          <ul class="nav nav-pills pull-right">
            {% raw %}
            {% if session.get('user') %}
            <li class="welcome-message">Hey {{ session['user']}}!</li>
              <li role="presentation"><a href="{{ url_for('logout') }}">Logout</a></li>
            {% endif %}
            {% endraw %}
            <li role="presentation"><a href="/polls">Polls</a></li>
            <li role="presentation"><a href="https://github.com/danidee10/Votr">Github</a></li>
          </ul>
        </nav>
        <a href="/"><img src="static/images/logo.png" /></a>
      </div>

      <div class="jumbotron">
        <h1>It's easier with Votr!</h1>
        <p class="lead">Create free online polls today with Votr</p>

        {% raw %}
        {% if not session.get('user') %}
        <p><a class="btn btn-lg btn-success" href="{{ url_for('signup') }}" role="button">Sign up today</a></p>
        {% else %}
        <p><a class="btn btn-lg btn-success" href="/polls" role="button">Create a poll</a></p>
        {% endif %}
        {% endraw %}
      </div>

      <div class="row marketing">
        <div class="col-lg-6">
          <h3>Popular votes</h3>
          <p>It's so easy to use Votr, just create an account and you can start
          creating polls for the world to see!</p>
        </div>

        <div class="col-lg-6">
          <h3 class="form-header">Login</h3>
          {% raw %}
          <form method="post" action="{{ url_for('login') }}">
          {% with messages = get_flashed_messages(with_categories=true) %}
              {% if messages %}
                  {% for category, message in messages %}
                  <p class="{{ category }}">{{ message }}</p>
                  {% endfor %}
              {% endif %}
          {% endwith %}
          {% endraw %}
            <div class="form-group has-success">
              <input type="text" class="form-control" name="username" placeholder="Username" />
            </div>

            <div class="form-group has-success">
              <input type="password" class="form-control" name="password" placeholder="Password" />
            </div>

            <button type="submit" name="submit" class="btn btn-success">Submit</button>
          </form>
        </div>
      </div>

      <footer class="footer">
        <p>&copy; 2016 Votr</p>
      </footer>

    <!-- IE10 viewport hack for Surface/desktop Windows 8 bug -->
    <script src="static/js/ie10-viewport-bug-workaround.js"></script>
  </body>
</html>

{% endhighlight %}

At the top of the file, we can display different menu items for users if they're logged in or not, using Jinja2's {% raw %}`{% if %}` and `{% else %}` tags {% endraw %}. Also the session variable we used in `votr.py` is made available to us automatically by flask in all our templates, so we don't need to pass the `session` object around.

We also used the same technique for menu items to display a `"signup"` or `"create a poll"` button depending on the login state of the user.

Earlier we talked about `flash` messages and their use. Just above the login form we're displaying all the flashed messages if anyone is present, you can use a context manager **Wrong!!!**

The with statement in jinja2 is not equivalent to python's `with` statement. it doesn't do any cleanup action when you exit the block and the variables inside the {% raw %}`{% with %}`{% endraw %} block are not available outside it.

This is exactly what this user was thinking about when he asked this question on [stackoverflow](http://stackoverflow.com/questions/6432355/variable-defined-with-with-statement-available-outside-of-with-block).

It simply sets a variable and creates a new scope, so all the variables used inside the {% raw %}`{% with %}`{% endraw %} block  won't be accessible outside that block.

###### something like this in pure python.

{% highlight python %}
def numcount():
    for num in [1, 2, 3, 4]:
        print(num)

print(num) # num is only accessible inside the function
{% endhighlight %}

when you call `get_flashed_messages()` with `with_categories` set to `true` it allows you to access the class of the message that was flashed. Based on that, we can define a css class that displays different colours of text depending on the class of message that was flashed (Yeah you guessed right!, green for success and red for errors, and probably yellow for warnings).

Jinja2 doesn't actually mirror python as a language in all it's constructs, some statements might be slightly different from their counterparts in pure python, so watch out for this if something doesn't work as you expect it to work.

### Signup template

Our `signup` template is very basic, we'll only ask the users for their email, username and their password once (We'll work on our forms later with `ReactJS`). There's also no form of validation going on. Except you consider the unique email and username fields we set back in our `models.py` file.

{% highlight html %}
<!DOCTYPE html>
<html lang="en">
  <head>
    <meta charset="utf-8">
    <meta http-equiv="X-UA-Compatible" content="IE=edge">
    <meta name="viewport" content="width=device-width, initial-scale=1">
    <!-- The above 3 meta tags *must* come first in the head; any other head content must come *after* these tags -->
    <meta name="description" content="">
    <meta name="author" content="">
    <link rel="icon" href="http://getbootstrap.com/favicon.ico">

    <title>Signup To Votr!</title>

    <!-- Bootstrap core CSS -->
    <link href="static/css/bootstrap.min.css" rel="stylesheet">

    <!-- IE10 viewport hack for Surface/desktop Windows 8 bug -->
    <link href="static/css/ie10-viewport-bug-workaround.css" rel="stylesheet">

    <!-- Custom styles for this template -->
    <link href="/static/css/signin.css" rel="stylesheet" />

    <script src="static/js/ie-emulation-modes-warning.js"></script>

  </head>

  <body>

    <div class="container">

        <form class="form-signin" method="post" action="{% raw %}{{ url_for('signup') }}{% endraw %}">
        <h2 class="form-signin-heading" align="center">Sign up</h2>
        <div class="form-group has-success">
          <label for="email" class="sr-only">Email address</label>
          <input type="email" id="email" name="email" class="form-control" placeholder="Email address" required autofocus>
        </div>

        <div class="form-group has-success">
          <label for="username" class="sr-only">Username</label>
          <input type="text" id="username" name="username" class="form-control" placeholder="Username" required autofocus>
        </div>

        <div class="form-group has-success">
          <label for="password" class="sr-only">Password</label>
          <input type="password" id="password" name="password" class="form-control" placeholder="Password" required>
        </div>  
        <button class="btn btn-lg btn-success btn-block" type="submit">Sign up</button>
      </form>

    </div> <!-- /container -->


    <!-- IE10 viewport hack for Surface/desktop Windows 8 bug -->
    <script src="static/js/ie10-viewport-bug-workaround.js"></script>
  </body>
</html>

{% endhighlight %}


#### Handling URL's and Static files the flask way

If you've been looking at the links to our templates, you should have noticed that we're again using `url_for`. we could just hardcode our url's like the way we did to polls with `<a href="/polls" />Polls</a>` and get away with it, but what if a new programmer comes into our fictional company and decides to change the `polls` route form `/polls` to `/votr_polls`, and then forgets to change the links in the html template.

You'll end up with a broken link, and unless you have good behavioural testing, the same broken link is going to end up in your production environment.

Using `url_for` ensures that when a route is changed, a template is moved to another directory or we deploy our application on another machine, our links will never break.

This same re-usability applies for static files and assets like images and css, we want to make sure our links never break if we rename our `static` folder to something else, or change their location, the syntax for linking to a static file is similar to that of the template url's. You only have to specify the filename of the asset. an example is the bootstrap css link in our homepage.


{% highlight html %}
<link href="{% raw %}{{ url_for('static' filename='/css/bootstrap.min.css'}}{% endraw %}" rel="stylesheet">
{% endhighlight %}


You have to repeat the same procedure and change all the links to equivalent flask url's. This process could get very tiring and become a huge pain especially if you have a lot of templates with a lot of static links in them.

I created an open source python library called [Staticfy](https://github.com/danidee10/Staticfy) to automate this process.

To Use Staticfy, clone it in the root directory for votr

{% highlight bash %}
$ git clone https://github.com/danidee10/Staticfy.git && cd Staticfy
{% endhighlight %}

Run `staticfy.py` on the templates folder

{% highlight bash %}
$ python3 staticfy.py ../templates                                  
staticfied ../templates/signup.html ==> staticfy/signup.html

staticfied ../templates/index.html ==> staticfy/index.html
{% endhighlight %}

It should create a folder called `staticfy` containing two new files (with the same name as our original templates), whose static links have been converted to their flask equivalents.

copy the new files over to replace the old files.

{% highlight bash %}
cp -i staticfy/* ../templates
{% endhighlight %}

Thats it!. All your static files have been taken care of and linked properly, reload the page and the design should stay the same if everything went well.

### Summary
We've now built an authentication system for our users, and we we're able to display different information based on the login status of the user. We also implemented a one way hash to make sure our passwords are stored securely in our database.

**NOTE:** Our authentication system is no where near secure. We are still vulnerable to [CSRF](https://www.owasp.org/index.php/Cross-Site_Request_Forgery_(CSRF)) attacks, and we're still passing information around as plain text.

They're several flask extensions that are better at handling user management and authentication in a more secure way than what we've done. a popular one in the flask community is [Flask-Login](https://flask-login.readthedocs.io/en/latest/) we also have the full featured [Flask-security](https://pythonhosted.org/Flask-Security/) which integrates a lot of flask addons together including Flask login to provide robust user management for flask applications.

There is also no way for our users to recover their password, if they forget it.

In the next part we're going to design the poll form, and the actual voting form with `ReactJS`. If you don't know about `ReactJS`, It would be wise to look at the ReactJS [docs](https://facebook.github.io/react/) now to know why we're using it in the first place instead of `html` and pure `javascript` or `JQuery`.

All the custom css and the complete source code for this project can be found on [github](https://github.com/danidee10/Staticfy), feel free to fork it and play with it.

Thanks for reading, if you found Staticfy useful don't forget to give it a star on github. Otherwise raise an issue to contribute and improve it's development. Drop your comments if you have any question or insights about this part.
