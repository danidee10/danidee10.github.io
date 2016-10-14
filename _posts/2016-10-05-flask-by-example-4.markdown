---
layout: post
title: Flask by example 4 (Building an interactive UI with ReactJS)
date: 2016-10-05T22:35:33+01:00
---

From the [previous tutorial]({% post_url 2016-09-24-flask-by-example-3 %}), we designed a landing page and built an authentication system to enable us track our users and the posts they create / vote on.

But our users have no form to add polls, Let's fix that.

We're going to create an interactive user interface where our users can see a live preview of the poll they're creating before it's saved to the database. For this exercise we're going to make use of [React](https://facebook.github.io/react/) which as the docs describe it, is a painless way to create interactive UIs.

*This tutorial includes Live demos of the working code, thanks to [JSFiddle](https://jsfiddle.net/). So you can play with the code without leaving your browser*

### Why React?
There are various reasons on why i chose React, and this list includes "Because Angular is hard and stupid". but let's talk about the important reasons:

<ol>
  <li>
    Because React is easy to learn. It's just pure Javascript (no pseudo language or anything slapped untop). Though you'll encounter <a href="https://facebook.github.io/react/docs/jsx-in-depth.html">JSX</a> frequently, JSX is also very simple and straight forward and if you're already familiar with html or xml, then you'd definitely breeze through JSX.
  </li>
  <br />
  <li>
  It allows you to break down chunks of html into simple re-usable components. You'll see an example of that as we move on.
  </li>
  <br />
  <li>
  Once you know the basics of React, you just need some extras and you can start developing cross platform mobile applications with <a href="https://facebook.github.io/react-native/">React Native</a>
  </li>
</ol>
<br />

### Polling page
We're going to make a copy of our login page and modify it to include a form which would be used to add polls to the site. we're also going to include a preview form that shows the user exactly how a poll would look like as they're creating it.

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

    <title>Create a poll</title>

    {% raw %}
    <!-- Bootstrap core CSS -->
    <link href="{{ url_for('static', filename='css/bootstrap.min.css') }}" rel="stylesheet">

    <!-- React JS -->
    <script src="https://unpkg.com/react@15.3.2/dist/react.js"></script>
    <script src="https://unpkg.com/react-dom@15.3.2/dist/react-dom.js"></script>
    <script src="https://unpkg.com/babel-core@5.8.38/browser.min.js"></script>
    <script type="text/babel" src="{{ url_for('static', filename='js/polls.js') }}"></script>

    <!-- IE10 viewport hack for Surface/desktop Windows 8 bug -->
    <link href="{{ url_for('static', filename='css/ie10-viewport-bug-workaround.css') }}" rel="stylesheet">

    <!-- Custom styles for this template -->
    <link href="{{ url_for('static', filename='css/signin.css') }}" rel="stylesheet" />

    <script src="{{ url_for('static', filename='js/ie-emulation-modes-warning.js') }}"></script>

  </head>

  <body>

    <div class="container">

      <div id="form_container">
        <!--Placeholder for poll creation form -->

      </div>

    </div>

    <!-- IE10 viewport hack for Surface/desktop Windows 8 bug -->
    <script src="{{ url_for('static', filename='js/ie10-viewport-bug-workaround.js') }}"></script>

    {% endraw %}
  </body>
</html>

{% endhighlight %}

We replaced the old login form with a div `poll_form`. This div would serve as the "container" where React would render our components.

we also added required links to enable us use React in our application. which are React, ReactDOM and [Babel](https://babeljs.io/)

Before we proceed, it's good practice to layout the structure of the UI we have in mind. React is all about modular re-usable components. our UI will have the following structure:

![React Layout](/images/react_layout.png)


We're going to have two components, the main component which we'll call `PollForm` is going to contain the form used to create the poll, and the second component called `LivePreview` is a form that shows us how a poll is going to look like during creation.

`LivePreview` is part of the `PollForm` component because the both of them are share the state of different input fields. `PollForm` is the parent component and `LivePreview` is the child component.

`form_container` is just a plain div that houses the two forms.


Create a new javascript file in the static directory called `poll.js`. This is where our react code will live in. We already have a link at the top of `polls.html` that points to it.

{% highlight bash %}
touch static/js/polls.js
{% endhighlight %}


##### polls.js

<script async src="//jsfiddle.net/danidee10/8asnq3uz/2/embed/js,html,css,result/dark/"></script>


Basically, we created the two components we talked about earlier. Looking at `PollForm`, you can see that it contains a form that would be used to save our poll. But if you look at the form closely, you can see that instead of using the `class` attribute as we would do in html we're using `className`.

That is [JSX](https://facebook.github.io/react/docs/jsx-in-depth.html) in action, `JSX` might look similar to html, but it's not. infact it's much more terse (like xml) and simply doesn't allow you do what you want to do. Capitalization matters. (`classname` is not the same as `className`).

 `JSX` eventually translates down to raw Javascript. So you're not required to use JSX with React. without JSX this is how you'd create a simple `<h1>Hello world</h1>`

{% highlight javascript %}
var helloComponent = React.createElement('h1', {}, "Hello world")

ReactDOM.render(helloComponent, document.getElementById('form_container'))
{% endhighlight %}

This doesn't look very pretty and instantly you can see that it's going to become cumbersome to build your interface like this as it becomes more complex.

It's easier to build a web UI in a language that looks and feels like html. This is another reason why i love React and prefer it to form libraries for popular web frameworks like Flask-WTF or django forms.

**Note that React is not a direct replacement for those form libraries and doesn't totally secure your forms. Hackers and malicious users can still bypass your React forms and submit data directly to the server. Always do server side validation alongside React.**

Let's talk about the various methods in `polls.js`:

<u><strong>getInitialState:</strong></u> This is used to setup the initial state of the component. We created two state variables `title` which would be used to store the title of the poll and `options` which is an array that holds the various options for the poll.

For now title and options are static because they contain hardcoded values. If you look at the Result in the JSFiddle demo you should see a new poll. "Alex iwobi vs Dele Alli". with the two football stars as options. Ideally these values would come from `<PollForm />` and once the user changes the value of any field. React will update the UI accordingly with the new data.

`title` and `options` are available to us as properties on an object called `state`. Using `this.state.title` and `this.state.options`. Treat this values as if they're immutable, don't go around doing something like this `this.state.title = 'My Title'`, It could lead to all sorts of problems as your modifications would get overridden by react when you call `setState`.This is the proper way to modify `title` using `setState`:

{% highlight javascript %}
this.setState{'title': 'My Title'}`
{% endhighlight %}

Calling `setState` will cause react to re-render the UI.

<u><strong>LivePreview:</strong></u> This is a little different from `PollForm`. it also a render method like `PollForm` the difference here is function `options` that loops through our state array `options` using map (Similar to python's map. just that it uses a full fledged function and not an anonymous `lambda` function.) and returns radio buttons containing the elements we placed in our options array when we called `getInitialState`.

If you're wondering what `props` (short for properties). They are used to pass data to a component from it's parent. You can think of them as named arguments that we pass to a python function. except that function in this case is a React component. For example from the parent component `<PollForm />` we passed in the title of our poll as `title` to `LivePreview` with `<LivePreview options={this.state.title} />`. inside the render method of `LivePreview` `title` became a property, and we accessed it with `this.props.title`


We've also added some event handlers on our input fields and buttons, we'll implement them and talk about them more in the next section.


### Making the form Reactive

We're going to make the `LivePreview` form inte**Reactive**, so as the user types in the title of his poll, it automatically shows up the the header of our preview form, we also want the user to be able to add an option and see it appear on the preview form immediately.

To achieve this we have to remove the default attributes we set for our state variables earlier and then implement our event handlers properly.

<script async src="//jsfiddle.net/danidee10/5Lex6gtj/9/embed/js,html,css,result/dark/"></script>

Lets talk about the event handlers we just implemented. This is where most of the hardwork is.

`handleTitleChange` and `handleOptionChange` do the same thing for different fields in our form, they update `title` and option respectively as the user types into the field. so at any point in time when we decide to submit the form, we can be sure that our state variables `title` and  `option` have the latest value from the user.

`handlOptionAdd` updates the array of options by appending new items to the array. Notice how we didn't push to the array directly by doing something like this `this.state.options.push({name: this.state.option});`. We're still using `setState` so React is in total control our our application state and knows how to update the UI accordingly. we also cleared the option field in the form by setting option to an empty string.

`handleSubmit` simply prevents the form from submitting by calling `e.preventDefault()`.

We also wrapped the return statement of `options` with a condition to determine if an `option`s name is set before returning a new radio button (this was done to prevent users from nameless options to our preview form by repeatedly clicking on the Add option button).


### Useful React resources and tips
This tutorial was never meant to be a hello world introduction to React JS and if you're still confused about somethings, feel free to lookup the docs, Google and [Stackoverflow](http://stackoverflow.com/questions/tagged/reactjs) for answers to basic React questions.

Here is a list of useful resources and tips that personally helped me grok React faster.

<ol>
  <li>
  Always model your UI down, before you start writing any code. Just like we did. This will help you to have a clear mental picture about your components about who the parent components is and what it's child(ren) is/are
  </li>
  <br />
  <li>
  Buttressing my first point, i found it easier to think about React from top to bottom. Write your component first before writing the React Class.(for example "<PollForm />" should come first before writing "React.createClass" or any functional part)
  </li>
  <br />
  <li>
  Always use <strong>setState</strong> to trigger a UI update. There is another method called <strong>forceUpdate</strong> that forces the UI to re-render itself and it's possible to have a complex interface where you might use it. But if you see yourself relying on <strong>forceUpdate</strong> too much to update the interface, that's a warning sign that something is wrong with your UI design and your componenets aren't really interacting with each other like they should.
  </li>
  <br />
  <li>
  React isn't going anywhere soon. so learn it!. Facebook built react and they use it heavily on <strong>FACEBOOK.COM</strong> which also means that any problem you're likely to encounter with react has already been encountered and solved by facebook which is a plus for you.
  </li>
</ol>

<br />

#### Resources
[React docs](https://facebook)

[Understanding state in React](https://thinkster.io/understanding-react-state)....if you're annoyed because you don't really understand what "state" is and why it's important.

[React Debugger for chrome](https://chrome.google.com/webstore/category/apps). it's also availabe for firefox [here](https://addons.mozilla.org/en-US/firefox/addon/react-devtools/)

[JSX Gotcha's](https://facebook.github.io/react/docs/jsx-gotchas.html)

[9 things every React.js beginner should know](https://camjackson.net/post/9-things-every-reactjs-beginner-should-know)

[Official Video on Debugging React](https://facebook.github.io/react/blog/2014/01/02/react-chrome-developer-tools.html)

[Html to JSX compiler](http://magic.reactjs.net/htmltojsx.htm)...(Extremely useful)


That's all for this part. You've now built an interactive form that allows users preview polls before they save them. The poll form doesn't actually save anything to the database yet. That's what we're going to work on in the next part. we're going to build a RESTful api to handle the submission and retrieval of polls.

The complete code for this tutorial is available on [github](https://github.com/danidee10/Votr), if you have any questions or insight on this article, you can raise an issue there or drop a comment below. If you don't I'll see you in the next part.

Thanks for reading!
