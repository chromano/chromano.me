---
title: Python’s subprocess and pipes
date: 2011-04-04 00:00:00 Z
tags:
- python
- unix
- pipes
layout: post
---

# {{ page.title }}

{% highlight python %}
import subprocess

dump_proc = subprocess.Popen(["pg_dump", "-d dbname", "-h localhost"],
    stdout=subprocess.PIPE)
bzip_proc = subprocess.Popen(["bzip2"], stdin=dump_proc.stdout,
    stdout=open("db_backup.bz2", "wb"))
{% endhighlight %}
