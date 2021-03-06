---
title: Python daemons via SSH makes it hang
date: 2011-05-23 00:00:00 Z
tags:
- python
- daemons
- ssh
layout: post
---

# {{ page.title }}

First, the reason for the SSH problem is described [here](http://www.snailbook.com/faq/background-jobs.auto.html). Second, it makes sense, but… is there an alternative other than getting rid of the daemon output? I mean, I would appreciate if I could see if the daemon was started properly without having to run two commands (ex: start and then status).

OK, let’s start from the begining. I wrote a daemon in python this week and then a "runner" in shell script, which simply sets the environment variables before running the daemon itself. Everything works fine if I log in the server and run the script there, manually. But then I was asked to implement a command in the [fabfile](http://fabfile.org/) of the project, so other developers would not be required to ssh into the server and run the command there, they could just run locally the following:

{% highlight bash %}
$ fab daemon:start
{% endhighlight %}

…and having it saying if it succeed or not:

{% highlight bash %}
$ fab daemon:start
out: Starting...
out: PID: 12345
{% endhighlight %}

By the way, what fab really does is to connect in the server and run a set of commands defined in a file `fabfile.py`. In my case, the steps for `daemon:start` are:

{% highlight bash %}
cd ~project/project/ && ./scripts/daemon.sh start
{% endhighlight %}

So, the command fab is really running is something like this:

{% highlight bash %}
ssh user@host "cd ~project/project/ && ./scripts/daemon.sh start"
{% endhighlight %}

If I try to run this command by myself, SSH will hang (same way does fab). Obviously, the problem was the way I tried to run the script. After reading about the [SSH problem](http://www.snailbook.com/faq/background-jobs.auto.html), I changed the command above to redirect `stdout` to `/dev/null` and I tried again:

{% highlight bash %}
ssh user@host "cd ~project/project/ && \
    ./scripts/daemon.sh start >/dev/null"
{% endhighlight %}

It worked, but it lost the output and there’s no way to know if the command really worked, unless a `status` command is ran next. Bummer!

I had to know what was going on, and the first step I took was reviewing the forking process in the daemon code. As you may know, when the subject comes to daemonizing things, [Richard Steven’s tutorial](http://www.steve.org.uk/Reference/Unix/faq_2.html#SEC16) is _the reference_. After reading these steps, I realized that I missed the following:

<pre>6. close() fds 0, 1, and 2.</pre>

Uh! Certainly it is the problem, let’s try closing `stdout` after the two forks, i.e, close the `stdout` in the *final* daemon process:

{% highlight python %}
...
sys.stdout.close()
...
{% endhighlight %}

I ran it again, no luck, still hanging. WTF? After a lot of attempts, debugging and mainly [research](http://code.activestate.com/recipes/186101-really-closing-stdin-stdout-stderr/), I finally found the culprit -> python! (whoa?).

Python sees `stdout`, `stderr` and `stdin` as special files, so when you close it the way above, python will simply *mark* these files as closed, but will not really close them. You could check it by yourself:

{% highlight python %}
>>> import os, sys
>>> sys.stderr.close() # so we can ignore the exceptions raised
>>> print "Hello"
Hello
>>> sys.stdout.close()
>>> print "Hello"
>>> os.write(1, "Hello\n")
Hello
>>> os.close(1)
>>> os.write(1, "Hello\n")
>>>
{% endhighlight %}

As you can see in the second `print`, it seems like python really closed `stdout`, but the next attempt to write in `stdout`, through `os.write`, confirms that it didn’t happen. To close `stdout` effectively, you need to do both things: 1) `sys.stdout.close();` 2) `os.close(1)`. By the way, `1` is the file descriptor number associated with `stdout`, as defined by the standard; the call to `os.close` is a _system call_, which means this is gonna be sent to the kernel itself.

So, here’s where I found the solution for starting a python daemon via SSH without losing its output.

{% highlight python %}
pid = os.fork()
if pid:
    sys.exit(0)
os.setsid()
os.umask(0)
pid = os.fork()
if pid:
    sys.exit(0)
print "Started:", os.getpid()
# don't reuse stdout from parent processes, so SSH doesn't hang
sys.stdout.close()
os.close(1)
{% endhighlight %}

Conclusion: give [python-daemon](http://pypi.python.org/pypi/python-daemon) a try.
