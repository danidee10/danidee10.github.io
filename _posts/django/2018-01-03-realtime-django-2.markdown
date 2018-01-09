---
layout: post
title: 'Realtime Django Part 2: Building a Chat application RabbitMQ and uWSGI websockets (Authentication and User Management)'
date: 2018-01-03T22:35:33+01:00
---

Welcome to the second part of this series.

We're going to kick off chatire by Implementing User Management and Authentication so users can create accounts and login.

Thanks to Django's excellent and Vibrant community, most of the work has been done for us. Hence we're going to be using a third-party django library called djoser

Go ahead and install it with:

{% highlight bash %}
pip install djangorestframework
pip install djoser
{% endhighlight %}

Basically djoser is an authentication app built untop django rest framework to add more functionality to django rest framework's inbuilt authentication.

It provides us with url endpoints that we can call from our Vue later on.


### Configuring djoser
We're going with the simplest setup possible for djoser. Include the following in your `INSTALLED_APPS`

{% highlight bash %}
INSTALLED_APPS = (
    'django.contrib.auth',
    ...,
    'rest_framework',
    'rest_framework.authtoken',
    'djoser',
)
{% endhighlight %}

Configure urls.py:

{% highlight bash %}
urlpatterns = [
    ...,
    path('auth/', include('djoser.urls')),
    path('auth/', include('djoser.urls.authtoken')),
]
{% endhighlight %}

Finally, Add `rest_framework.authentication.TokenAuthentication` to Django REST Framework authentication strategies tuple:

{% highlight python %}
REST_FRAMEWORK = {
    'DEFAULT_AUTHENTICATION_CLASSES': (
        'rest_framework.authentication.TokenAuthentication',
        (...)
    ),
}
{% endhighlight %}

Lastly, run the migrations again with `python manage.py migrate`

That's it! We can create a user by posting data to `http://127.0.0.1:8000/auth/users/create/`

{% highlight bash %}
curl -X POST http://127.0.0.1:8000/auth/users/create/ --data 'username=danidee&password=mypassword'

{"email":"","username":"danidee","id":1}
{% endhighlight %}

Voila! we have a new user. Check out the djoser docs for the list of available endpoints and how to use them http://djoser.readthedocs.io/en/latest/base_endpoints.html

# Vue

Vue is a JavaScript Framework for building Reactive interfaces. Even though i'm a big fan of React (Due to React-native), I prefer Vue for developing JavaScript applications.

It's very easy to get Started and unlike React it doesn't use JSX (You can if you want to) and it has a Vibrant community with lot of plugins and tutorials available on the internet.

Let's start a new Vue project:

{% highlight bash %}
npm install -g vue-cli
{% endhighlight %}

Let's scaffold a new project based on the webpack template with `vue-cli`

{% highlight bash %}
vue init webpack chatire-frontend
{% endhighlight %}

***NoTE: Make Sure you install vue-router***

This might be the right time to grab a coffee or a quick snack because this might take sometime depending on your network speed. Our ISP's are pretty terrible in Nigeria lol.

After that navigate to the new vue app and run the dev server with `npm run dev` if everything went well you should see this when you access `localhost:8080`

<img />

Talk about the folder arrangement

Create two new components one for the main chat screen callled `Chat.vue` and another for User Authentication and Signup we'll call that `UserAuth.vue`

Ideally what we want is to conditionally display the components based on the Login status of the user. If the User is Authenticated, we want to display the Chat component else we want them to either signup or login which means we'll display the `UserAuth` Component.

We can do this by creating a implementing a global navigation guard on the router. Edit the router's `index.js` file to include the following

{% highlight JavaScript %}
import Vue from 'vue'
import Router from 'vue-router'
import Chat from '@/components/Chat'
import UserAuth from '@/components/UserAuth'

Vue.use(Router)

const router = new Router({
  routes: [
    {
      path: '/chat',
      name: 'Chat',
      component: Chat
    },

    {
      path: '/auth',
      name: 'UserAuth',
      component: UserAuth
    }
  ]
})

router.beforeEach((to, from, next) => {
  if (sessionStorage.getItem('authToken') !== null || to.path === '/auth') {
    next()
  }
  } else {
    next('/auth')
  }
})

export default router
{% endhighlight %}

The `beforeEach` guard is called before a navigating to any route in our application (Global guard) if a token is stored in the `sessionStorage` we allow the navigation to proceed by calling `next()` else we redirect to the auth component.

No matter the route a user navigates to in our application this function will check if the user has an auth token and redirect them appropraitely.

### Design the Login/Signup page

I've built a simple Login/Signup with Bootstrap 4 tabs, this is the content of `UserAuth.vue`

{% highlight html %}
<template>
  <div class="container">
    <h1 class="text-center">Welcome to the Chatire!</h1>
    <div id="auth-container" class="row">
      <div class="col-sm-4 offset-sm-4">
        <ul class="nav nav-tabs nav-justified" id="myTab" role="tablist">
          <li class="nav-item">
            <a class="nav-link active" id="signup-tab" data-toggle="tab" href="#signup" role="tab" aria-controls="signup" aria-selected="true">Sign Up</a>
          </li>
          <li class="nav-item">
            <a class="nav-link" id="signin-tab" data-toggle="tab" href="#signin" role="tab" aria-controls="signin" aria-selected="false">Sign In</a>
          </li>
        </ul>

        <div class="tab-content" id="myTabContent">

          <div class="tab-pane fade show active" id="signup" role="tabpanel" aria-labelledby="signin-tab">
            <form @submit.prevent="signUp">
              <div class="form-group">
                <input v-model="email" type="email" class="form-control" id="email" placeholder="Email Address" required>
              </div>
              <div class="form-row">
                <div class="form-group col-md-6">
                  <input v-model="username" type="text" class="form-control" id="username" placeholder="Username" required>
                </div>
                <div class="form-group col-md-6">
                  <input v-model="password" type="password" class="form-control" id="password" placeholder="Password" required>
                </div>
              </div>
              <div class="form-group">
                <div class="form-check">
                  <input class="form-check-input" type="checkbox" id="toc" required>
                  <label class="form-check-label" for="gridCheck">
                    Accept terms and Conditions
                  </label>
                </div>
              </div>
              <button type="submit" class="btn btn-block btn-primary">Sign up</button>
            </form>
          </div>

          <div class="tab-pane fade" id="signin" role="tabpanel" aria-labelledby="signin-tab">
            <form @submit.prevent="signIn">
              <div class="form-group">
                <input v-model="username" type="text" class="form-control" id="username" placeholder="Username" required>
              </div>
              <div class="form-group">
                <input v-model="password" type="password" class="form-control" id="password" placeholder="Password" required>
              </div>
              <button type="submit" class="btn btn-block btn-primary">Sign in</button>
            </form>
          </div>
          
        </div>
      </div>
    </div>
  </div>
</template>

<script>

  const $ = window.jQuery // JQuery

  export default {

    data () {
      return {
        email: '', username: '', password: ''
      }
    }

  }
</script>

<style scoped>
  #auth-container {
    margin-top: 50px;
  }

  .tab-content {
    padding-top: 20px;
  }
</style>
{% endhighlight %}

In the snippet above i've used `v-model` for two way data binding on all the input fields. Which means whatever is typed in those fields can be accessed on the JavaScript side with `this.field_name`.

Also i've created event listeners on both forms (`@submit.prevent`) that'll listen to the form submit event of each form and call the methods specified. We haven't implemented those methods yet.

Since we're using bootstrap and `jQuery` We've also defined a variable `$` that points to `jQuery` We're going to make use of `jQuery`'s ajax method soon. Feel free to use another ajax library like Axios (It's pretty popular among Vue users) if you want to decouple your application from `jQuery`.

Don't forget to include bootstrap's CSS and JavaScript in the main `index.html` page.

{% highlight html %}
<!DOCTYPE html>
<html>
  <head>
    <meta charset="utf-8">
    <meta name="viewport" content="width=device-width,initial-scale=1.0">
    <link rel="stylesheet" href="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-beta.3/css/bootstrap.min.css" integrity="sha384-Zug+QiDoJOrZ5t4lssLdxGhVrurbmBWopoEl+M6BdEfwnCJZtKxi1KgxUyJq13dy" crossorigin="anonymous">

    <style>
      .nav-tabs .nav-item.show .nav-link, .nav-tabs .nav-link.active {
        outline: none;
      }
    </style>

    <title>chatire-frontend</title>
  </head>
  <body>
    <div id="app"></div>
    <!-- built files will be auto injected -->

    <script src="https://code.jquery.com/jquery-3.2.1.min.js" integrity="sha384-KJ3o2DKtIkvYIK3UENzmM7KCkRr/rE9/Qpg6aAZGJwFDMVNA/GpGFF93hXpG5KkN" crossorigin="anonymous"></script>
    <script src="https://cdnjs.cloudflare.com/ajax/libs/popper.js/1.12.9/umd/popper.min.js" integrity="sha384-ApNbgh9B+Y1QKtv3Rn7W3mgPxhU9K/ScQsAP7hUibX39j7fakFPskvXusvfa0b4Q" crossorigin="anonymous"></script>
    <script src="https://maxcdn.bootstrapcdn.com/bootstrap/4.0.0-beta.3/js/bootstrap.min.js" integrity="sha384-a5N7Y/aK3qNeh15eJKGWxsqtnX/wWdSZSKp+81YjTmS15nvnvxKHuzaWwXHDli+4" crossorigin="anonymous"></script>
  </body>
</html>
{% endhighlight %}

If you've done everything correctly, the page should look like this:

<img />

### Let's Grab our token

It's time to get the token from our django server, for the signup process, we want to sign the user up and then redirect them to the Chat route.

To acheive that, we have to implement the `signUp` and `signIn` methods we specified earlier:

{% highlight JavaScript %}
methods: {
  signUp () {
    $.post('http://localhost:8000/auth/users/create/', this.$data, (data) => {
      alert("Your account has been created. You will be signed in automatically")
      this.signIn()
    })
    .fail((response) => {
      alert(response.responseText)
    })
  },

  signIn () {
    const credentials = {username: this.username, password: this.password}

    $.post('http://localhost:8000/auth/token/create/', credentials, (data) => {
      sessionStorage.setItem('authToken', data.auth_token)
      sessionStorage.setItem('username', this.username)
      this.$router.push('/chat')
    })
    .fail((response) => {
      alert(response.responseText)
    })
  }
}
{% endhighlight %}


Now try and submit the form. Oops! It failed with:

**Cross-Origin Request Blocked: The Same Origin Policy disallows reading the remote resource at http://localhost:8000/auth/users/create. (Reason: CORS header ‘Access-Control-Allow-Origin’ missing).**

### CORS

Quoting the mozilla developer's site:

Cross-Origin Resource Sharing (CORS) is a mechanism that uses additional HTTP headers to let a user agent gain permission to access selected resources from a server on a different origin (domain) than the site currently in use. A user agent makes a cross-origin HTTP request when it requests a resource from a different domain, protocol, or port than the one from which the current document originated.

Basically it's a security mechanicsm that prevents a website from making a `XmlHttpRequest` to another unless explicitly allowed.

In our case, even though both webservers are running on localhost, due to the fact that they're on different ports (8080 and 8000) they're seen as different domains.

For the domains to match the protocol (http or https) must be the same and the ports must also be the same.

So how do we enable CORS in our django application? There is an excellent third-party app that handles all that for us called `django-cors-header` let's install it from pip

{% highlight bash %}
pip install django-cors-header
{% endhighlight %}

add it to your `INSTALLED_APPS`

{% highlight python %}
INSTALLED_APPS = [
    'django.contrib.admin',
    'django.contrib.auth',
    'django.contrib.contenttypes',
    'django.contrib.sessions',
    'django.contrib.messages',
    'django.contrib.staticfiles',

    # Custom Apps
    'rest_framework',
    'rest_framework.authtoken',
    'corsheaders',
    'djoser'
]
{% endhighlight %}

Include the middleware, (Make sure it comes before `django.middleware.common.CommonMiddleware`)

{% highlight python %}
MIDDLEWARE = [
    'django.middleware.security.SecurityMiddleware',
    'django.contrib.sessions.middleware.SessionMiddleware',

    'corsheaders.middleware.CorsMiddleware',

    'django.middleware.common.CommonMiddleware',
    'django.middleware.csrf.CsrfViewMiddleware',
    'django.contrib.auth.middleware.AuthenticationMiddleware',
    'django.contrib.messages.middleware.MessageMiddleware',
    'django.middleware.clickjacking.XFrameOptionsMiddleware',
]
{% endhighlight %}

Finally set `CORS_ORIGIN_ALLOW_ALL = True`

***Note that this enables CORS for all domains. This is fine for development but when you're in production you only want to allow certain domain(s) this can be controlled with:***

`CORS_ORIGIN_WHITELIST`

Read the django-cors-header docs to find out about other options

<br />

After doing this, you should be able to create an account and sign in immediately

### Logout
Since we're using a `sessionStorage` to logout simply close the browser or open a new tab. To actually persist the token between browser restarts you can switch to `localStorage`. It has the same api as the `sessionStorage` so you're just changing "session" to "local" really.

Then you can create a function that removes the token from the storage you decide to stick with by calling

{% highlight JavaScript %}
sessionStorage.removeItem('authToken')
{% endhighlight %}

### Recap
That's all for this part, Our goal was to built a simple User Managment and auth system. We started out by installing djoser which is an excellent third party django app that provides authentication `REST` endpoints that we can call from our Vue frontend.

We also went through how we can call those endpoints from Vue.

On the road to victory, we were stopped by `CORS` and we briefly talked about why it exists. Eventually we saw how we could enable `CORS` for our django app with `django-cors-header`.

In the next part, we'll build the chat interface and more importantly, we'll learn about websockets and how you can easily integrate them into your django application. Adios! and see you in the next part.