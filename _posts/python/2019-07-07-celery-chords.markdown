---
layout: post
title: Run Chord callbacks in Celery (4.3.0) regardless of the exit status of it's tasks
date: 2019-07-07T22:35:33+01:00
---

You're probably here because you've stumbled upon a problem with Celery chords. The Chord callback function isn't executed if one of the tasks in the Chord's body fails.

This sucks because in most cases, The callback's execution confirms that all the tasks have been executed by the workers.

We can quickly reproduce this problem in a small python file (I'm using the Redis backend with docker, but you can use any backend that supports chords)

Start a new `Redis` container with:

{% highlight bash %}

docker run -p 6379:6379 redis

{% endhighlight %}

Create a new file called `test_celery.py`

{% highlight python %}

from celery import Celery, group


redis_url = "redis://localhost:6379"
app = Celery('test_celery', backend=redis_url, broker=redis_url)


@app.task
def good_task():
    return 'Good task'


@app.task
def other_good_task():
    return 'Other good task'


@app.task
def bad_task():
    raise Exception('Bad Task!!!')


@app.task
def tasks_completed(results):
    print('Tasks completed', results)


if __name__ == '__main__':
    # A chained group of tasks is automatically Upgraded to a Chord
    (group(good_task.s(), bad_task.s(), other_good_task.s()) | tasks_completed.s()).delay()

{% endhighlight %}

Let's run the worker file with:

{% highlight bash %}celery worker -A test_celery --loglevel=info{% endhighlight %}

Now, execute the chord:

{% highlight bash %}python test_celery.py{% endhighlight %}

If you look at the logs you'll see the exception raised in the `bad_task` Task but the callback function (i.e `tasks_completed`) isn't executed.

{% highlight bash %}

line 352, in _unpack_chord_result
    raise ChordError('Dependency {0} raised {1!r}'.format(tid, retval))
celery.exceptions.ChordError: Dependency 4d39d906-0118-4984-b4a6-68fa97f220b4 raised Exception('Bad Task!!!')
[2019-07-06 18:51:03,297: INFO/ForkPoolWorker-1] Task test_celery.other_good_task[ee3d4a1c-7232-4344-8703-1ff750bf9023] succeeded in 0.005841678001161199s: 'Other good task'

{% endhighlight %}

The Recommendation from the Celery team is to make your tasks are bullet proof. But this is almost impossible especially when your tasks are communicate with external API's or depend on external resources.

In celery <= 3.1 There used to be a setting called `CELERY_CHORD_PROPAGATES` to control this behaviour but it was removed and this is all I could find concerning the removal:

> ***CELERY_CHORD_PROPAGATES*** *setting will eventually be removed, as it was just added to be backward compatible with the old undefined behavior, which was just accidental and never considered to be a feature.*

Anyway, to achieve this we have to monkey patch the redis backend to return a custom value instead of raising a `ChordError`.

*The trackeback already gives us a clue on where to look (Line 352)*

{% highlight python %}

import celery
from celery import Celery, group, states
from celery.backends.redis import RedisBackend


def patch_celery():
    """Patch the redis backend."""
    def _unpack_chord_result(
        self, tup, decode,
        EXCEPTION_STATES=states.EXCEPTION_STATES,
        PROPAGATE_STATES=states.PROPAGATE_STATES,
    ):
        _, tid, state, retval = decode(tup)

        if state in EXCEPTION_STATES:
            retval = self.exception_to_python(retval)
        if state in PROPAGATE_STATES:
            # retval is an Exception
            return '{}: {}'.format(retval.__class__.__name__, str(retval))

        return retval

    celery.backends.redis.RedisBackend._unpack_chord_result = _unpack_chord_result

    return celery

{% endhighlight %}

`patch_celery` is a factory function that returns a patched version of Celery's Redis backend. Instead of raising an exception, we return the Exception class name alongside the exception body.

[View the original source code](https://github.com/celery/celery/blob/master/celery/backends/redis.py#L345-L354)

Putting everything together, we have this:

{% highlight python %}

import celery
from celery import Celery, group, states
from celery.backends.redis import RedisBackend


def patch_celery():
    """Patch redis backend."""
    def _unpack_chord_result(
        self, tup, decode,
        EXCEPTION_STATES=states.EXCEPTION_STATES,
        PROPAGATE_STATES=states.PROPAGATE_STATES,
    ):
        _, tid, state, retval = decode(tup)

        if state in EXCEPTION_STATES:
            retval = self.exception_to_python(retval)
        if state in PROPAGATE_STATES:
            # retval is an Exception
            return '{}: {}'.format(retval.__class__.__name__, str(retval))

        return retval

    celery.backends.redis.RedisBackend._unpack_chord_result = _unpack_chord_result

    return celery

redis_url = "redis://localhost:6379"
app = patch_celery().Celery('test_celery', backend=redis_url, broker=redis_url)


@app.task
def good_task():
    return 'Good task'


@app.task
def other_good_task():
    return 'Other good task'


@app.task
def bad_task():
    raise Exception('Bad Task!!!')


@app.task
def tasks_completed(results):
    print('Tasks completed', results)


if __name__ == '__main__':
    # A chained group of tasks is automatically Upgraded to a Chord
    (group(good_task.s(), bad_task.s(), other_good_task.s()) | tasks_completed.s()).delay()

{% endhighlight %}

Now restart the celery worker process and run

{% highlight python %}python test_celery.py{% endhighlight %} again

and you should see something like this in the logs:

{% highlight bash %}
raise Exception('Bad Task!!!')
Exception: Bad Task!!!
[2019-07-06 19:08:28,909: INFO/MainProcess] Received task: test_celery.other_good_task[8a13fc4f-ca80-44a8-b8b6-d0433e34166b]  
[2019-07-06 19:08:28,923: INFO/ForkPoolWorker-1] Task test_celery.other_good_task[8a13fc4f-ca80-44a8-b8b6-d0433e34166b] succeeded in 0.009584861996700056s: 'Other good task'
[2019-07-06 19:08:28,923: INFO/MainProcess] Received task: test_celery.tasks_completed[d3cbd175-c477-4eaa-b008-eb87efe9dbc1]  
[2019-07-06 19:08:28,925: WARNING/ForkPoolWorker-2] Tasks completed
[2019-07-06 19:08:28,925: WARNING/ForkPoolWorker-2] ['Good task', 'Exception: Bad Task!!!', 'Other good task']
[2019-07-06 19:08:28,926: INFO/ForkPoolWorker-2] Task test_celery.tasks_completed[d3cbd175-c477-4eaa-b008-eb87efe9dbc1] succeeded in 0.0014016950008226559s: None
{% endhighlight %}

We can see that the exception is still raised but now, `tasks_completed` is executed and we also see the Exception returned as one of the results!

{% highlight python %}[2019-07-06 19:08:28,925: WARNING/ForkPoolWorker-2] ['Good task', 'Exception: Bad Task!!!', 'Other good task']{% endhighlight %}

You can also do some interesting things with the task id. Like retrieving the function name, retrying the failed task etc. Hopefully, `CELERY_CHORD_PROPAGATES` makes it's way back to Celery.
