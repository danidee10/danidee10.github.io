---
layout: post
title: Why Go should be your next language
date: 2018-03-12T22:35:33+01:00
tags: Go Languages
---

Go or "gOlAnG" (It just sounds weird when people call it GoLang. Imagine calling `Python` Pythonlang or `JavaScript` JavaScriptLang).

![golAnG Sponge bob](https://images.complex.com/complex/images/c_limit,w_680/fl_lossy,pg_1,q_auto/bujewhyvyyg08gjksyqh/spongebob)
<figcaption>gOlAnG</figcaption>

<br />

Anyway, it's a programming language created at Google in 2009 by Robert Griesemer, Rob Pike, and Ken Thompson.

Today, the status of Go is a weird one. On one side they’re people that can swear with their life that learning `Go` was one of the best things that happened to them as programmers. On the other hand, you have those that bash it at the slightest opportunity.

Before i learnt Go. I was a little indifferent. Initially, I tried to learn it in January 2017 but I found the Syntax was a bit weird. It also lacked a lot of things i missed in Python and JavaScript (an integrated debugger, Objects etc). To make matters worse i came across a lot of articles that discouraged me from learning `Go`. There's even a [dedicated github project](https://github.com/ksimka/go-is-not-good) of various articles on why "go is not good".

Fast forward 6 months later (After trying `Rust` and not liking it), I decided to give `Go` another shot and this time i actually had something i wanted to do with it. There was an Recruitement portal i had built for a client some years ago with PHP.

My goal “Rewrite it with Go”

### How did it go?

It was very tiring and annoying because i had forgotten almost everything i learnt earlier on [Go Tour](https://tour.golang.org/list) so i had to do a lot of Googling and most times i didn’t really know what i was doing and i was just stitching code together.

But was it rewarding? Yes it was. The main aim of this article is to show you that `Go` is a great language and most articles that bash `Go` were written by people who didn't take their time to understand some of the language design decisions.

*I promise not to talk about Concurrency*

### Single Binary

It was finally time to deploy. I quickly Googled “How to deploy a Go application”. I was expecting the Usual install Language x on the server, setup dependencies, Put Nginx in front of it to handle slow clients etc.

I was astonished when i saw “Just build your application and deploy it as a single binary”. This might not really surprise you if you’re a `C/C++` programmer (or someone who uses compiled languages a lot). Apart from little `C++` I wrote when i was in the university, most of the code i’ve written in my career has a programmer has revolved around Dynamic languages (PHP, Python and JavaScript). You can throw Java in there too.

I ran `go build` locally, and i copied the resulting binary and the website files to the server with `rsync`.

I ran the executable, and boom! if worked!

A lot of people got on the `Go` bandwagon for other reasons like concurrency, speed etc. I was about to become a gopher because of a static binary. Lol

This is a huge advantage especially for simple applications and microservices. `Go` makes it dead simple to deploy applications. You might not even need `Nginx` or `Apache` infront of your web application.

Another cool thing you can do is bundle all your static resources and application code into a single binary. Imagine how cool it would be to send your friend an entire website in a single executable file.

### It’s not as verbose as you’ve been told

It’s a compiled language with static types. So can we really expect it to be as free flowing as python?

A lot of people exaggerate the verbosity of Go. Even Java Programmers haha!

![golAnG Sponge bob](https://images.complex.com/complex/images/c_limit,w_680/fl_lossy,pg_1,q_auto/bujewhyvyyg08gjksyqh/spongebob)
<figcaption>Go is verbose - Java Programmer (Sorry i had to use this picture again)</figcaption>

<br />

Though, i'll admit that error handling can be repetitive sometimes.

Let's write a simple hello world server in `Go` and Python's beloved web framework `Flask`.

With Flask:

{% highlight python %}
from flask import Flask

app = Flask(__name__)


@app.route('/')
def hello_world():
    return 'hello world from Flask'


if __name__ == '__main__':
    app.run("localhost:8000")
{% endhighlight %}


In `Go`:

{% highlight python %}
package main

import (
    "fmt"
    "net/http"
)

func HelloWorld(w http.ResponseWriter, r *http.Request) {
    fmt.Fprintf(w, "Hello world from Go")
}

func main() {
    http.HandleFunc("/", HelloWorld)
    http.ListenAndServe("localhost:8000", nil)
}
{% endhighlight %}

How does that look?

Most of Go’s so called “Verbosity” is due to the fact that it’s compiled with Static types. It'll be unfair to compare it to a dynamic/interpreted language like JavaScript or Python. But compared to Java and C++ It's less verbose.

### It’s really lean at the core

This is another popular criticsm of Go. Many Programmers  complain that it doesn’t have generics, "Standard Objects", OOP Features etc. I didn’t complain about generics but i really wanted `Classes` and `Objects` and all i got was `structs` and `interfaces`.

It turns out that `structs` can be used in places where you'd usually use a `Class` in other languages. `structs` can have methods and they can also inherit behaviour from other Structs (Through composition or "embedding" in the world of Go developers.)

This decision is both a good one and a bad one from the `Go` developers. While `Objects`, `Generics` etc  have their places. Removing them from the Go has it’s advantages. It makes it very easy to learn.

You can learn Go and start building real world applications by just following the [official guide](https://tour.golang.org/list)

As a programming language, there's not really much to it. After learning the basic data types, Getting over the weird syntax in some places, learning how to use Interfaces, channels and goroutines. That's basically it. Most of the new knowledge you'll gain will be from reading other people's code.

Go has just 25 keywords in the language spec.

This is a quote from Rob pike:

> The key point here is our programmers are Googlers, they’re not researchers. They’re typically, fairly ?young, fresh out of school, probably learned Java, maybe learned C or C++, probably learned Python. They’re not capable of understanding a brilliant language but we want to use them to build good software. So, the language that we give them has to be easy for them to understand and easy to adopt. —  Rob Pike

Go's philosophy can be summarized into two words; Performance and ease. To achieve that the language designers had to leave out a lot of things. It doesn't mean `Go` is less powerful.

At the beginning it was difficult, but writing `Go` code really showed me that you can still build meaningful software/applications without a lot of OOP/Functional features present in other languages.

### It beats python

This isn't really about speed or concurrency.

> There should be only one and only one obvious way to do it".

If you're a Python programmer, you should recognize this quote. It's one of the zens of python.

Even though python claims that there’s only one way/one obvious way to do things. That’s not entirely true.
Concurrency in python is a huge mess. They’re a thousand and one ways to do things.

* Threads
* Processes
* SubProcess (Or `os.system` calls)
* concurrent.futures
* gevent, greenlet etc
* asyncio
* cython (Disabling the `GIL`)
* Writing C extensions

This makes it hard and confusing because you're not sure if you're doing something the "right way". This is also something that JavaScript suffers from because of the huge amount of libraries it has.

*Some of these modules are just friendly wrappers around other modules.*

On the other hand, there's usually only one Idiomatic way to achieve something in `Go`. For concurrency it's `goroutines`. For Inheritance it's `composition`. For collection of data and related functions it's `structs`. There's only one way to loop which is a `for` loop (There's no `while`, `foreach` or `do while` loop).

This makes it easy for new programmers to understand `Go` code in a short time because all `Go` code looks the same.

After 5 months of learning `Go`. I decided to look at docker’s source code and i could understand a lot of things. I can’t say the same for python when i looked at twisted’s source code (Everyone suggesting twisted's source code as a guide for "clean python code").

### Tooling

This is another great thing about `Go` and i think it's something the language designers learned by looking at the failures of other programming languages.

Go has a formatting tool called `gofmt` and a linting tool `golint` as part of the language. This means developers don't spend time arguing about Tabs vs Spaces. It's `Tabs` or configuring linters.

JavaScript is notorious for different style guides (Airbnb, standard etc) and linters with each of them having configuration files to enable and disable various checks. This problem is non existent in `Go`. There's only one style and it can be enforced with `gofmt` and `golint`

Go also has a very strict naming convention (with actual consequences). `lowerCase` names are used to refer to variables or `Struct` properties that are private. `Capitalized` names are automatically exported. This eliminates the needs for the `private`, `public` and `protected` keywords found in a lot of Object oriented languages.

The lowercase names combined with special comments allows you to automatically generate documentation with `go doc`

There's also `go test` for testing.

![Two types of people, programmers will know](https://images-cdn.9gag.com/photo/aRQneE2_700b.jpg)
<figcaption>Two types of programmers</figcaption>

This picture is invalid in the `Go` world. There's only one type of `Go` programmer. The `Go` programmer who doesn't put braces on a different line.

### Exceptions

This is a very delicate topic in `Go` and one which programmer's from other languages find weird. Go doesn't have exceptions. There's even an [official explanation](https://blog.golang.org/error-handling-and-go) for it.

This is something that tripped me at first, but after months of writing `Go`, I got used to it. Go's argument is that other languages have abused exceptions to the point that they aren't exceptional.

In python this is true. It's more "pythonic" to ask for forgiveness than permission. consider the following code:

{% highlight python %}
if hasattr(object, 'a'):
    print('object has an a attribute')
else:
   # doesn't have the attribute, handle the error
{% endhighlight %}

{% highlight python %}
try:
    object.b
    print('object has an a attribute')
except AttributeError:
    # doesn't have the attribute, handle the error
{% endhighlight %}

Both of them achieve the same thing, but the second code snippet is more "pythonic". But is a non-existent attribute on a Object really exceptional? That's the direction `Go` is coming from. The only advantage of exceptions is that they can be raised anywhere and caught anywhere in the callstack. In `Go`, the immediate caller of a function is forced to handle the error. It won't bubble up.
 
`Go` has `panic` which at the surface might be similar to `raise` in `python`. but when your code or a library panics in Go. It's basically the "End of the world" for your program because you've encountered an error that cannot be resolved.

[This subreddit](https://www.reddit.com/r/golang/comments/3sfjho/gos_error_handling_is_elegant/) is quite interesting (Make sure you read the blog post first).

### Things don’t blow up easily

Well, this isn't really an advantage/feature of `Go`. It's something that's common to compiled languages with static types. I've mostly programmed in dynamic languages so I really forgot how it feels like to program in a statically typed language.

Look at this python code:

{% highlight python %}
def say_hello():
    print("hello " + 1)
{% endhighlight %}

If you run this code, python won't complain when you try to concatenate a `string` and an `int` (`"hello" + 1`)

But at runtime a `TypeError` exception would be raised.

On the other hand with `Go`, the compiler would never allow you do that. Your code will never compile until you fix it.

*Back to error handling. If you invoke a function that returns an error and you don't handle that error, Your code won't compile. You must handle the error or ignore it. In Dynamic languages you can basically ignore exception handling which increases the probability of your code crashing at runtime.*

When your code finally compiles there's an assurance that It won't fail easily at runtime (Panics can still occur). It also makes it easy to refactor code. If you rename a function or a variable that's used in multiple places, you'll have to hunt them down and change all of them else your code won't compile.

In a dynamic language, the program starts executing and it will only fail when you try to use the non-existent method.

### Fast compiler

Another thing that surprised me about `Go` was the compiler. It's so fast that you might forget that it's a compiler not an interpreter. This makes it easy to quickly iterate when writing code. It also makes it fast to build `Go` software locally on your machine.

`C` and `C++` compilers are notorious for being slow.

There was a time i tried to compile `KDE` and it took over 5 hours. Eventually i had to quit.

Over the weekend, i also compiled `gcc 4.x` and while it was compiling, I cleaned up the house, took my bath and even cooked a meal. Eventually it failed because i ran out of space on `/tmp`. That's a story for another day.

In Comparision, building `docker` (The biggest `Go` program i know) locally on my machine took over an hour. Much longer than I expected. Though, most of the time was spent downloading and cloning stuff.

## Conclusion and Advice

The best advice I can give to you is to:

**Use the standard library.**

If you’re building a web site or a web service with `JavaScript` the first step is to run off to `pypi` or `npm` to look for a web framework and it's associated libraries.

`Go` has learnt a lot from these languages and the standard library is pretty robust for web based tasks. It has libraries for most of the common things you need. A WebServer, A URL router, Templates, JSON and libraries for testing.

The only external library i used was [pq](https://github.com/lib/pq) which was necessary to work with Postgres.

Before you use an third party library make sure you check the standard library to see what if offers.

Finally, make sure you Read the [Go FAQ's](https://golang.org/doc/faq#) it contains a lot of explanations on so many of the design decisions of `Go`.

Don't forget to ***Ignore the Go haters***