---
layout: post
title: Python’s subprocess and pipes
tags:
  - python
  - unix
  - pipes
---
# {{ page.title }}

{% highlight python %}
import subprocess

dump_proc = subprocess.Popen(["pg_dump", "-d dbname", "-h localhost"],
    stdout=subprocess.PIPE)
bzip_proc = subprocess.Popen(["bzip2"], stdin=dump_proc.stdout,
    stdout=open("db_backup.bz2", "wb"))
{% endhighlight %}
