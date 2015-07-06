---
title:  "Initial commit"
date:   2015-07-06 11:42:00
description: Fun with Jekyll
---

In the pursuit of an ad-free blog I stumbled upon [Github pages](https://pages.github.com/). Another stellar offering from Github I thought. After a brief jaunt on [Jekyll Themes](http://jekyllthemes.org) I found the cleanest, most minimalist theme I could - [Cactus](https://github.com/koenbok/Cactus).

I see this blog as a good platform to give answers to some of the problems that, when googled, lead to a stack overflow posted years ago with no answers. Either that or things that I found interesting for one reason or another.

Token code highlighting test:

{% highlight bash linenos %}
# print listening tcp ports

# Linux
netstat -plnt
ss -plnt

# OS X (does this work on FreeBSD?)
lsof -nP | grep LISTEN
{% endhighlight %}

### See also
* [Jekyll docs](http://jekyllrb.com)
* [`highlight.js`](https://highlightjs.org/)
