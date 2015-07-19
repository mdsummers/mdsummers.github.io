---
title: "Keeping and organising notes"
date:  2015-07-19 20:51:00
description: Using the today command to help keep notes
---

I'm a strong believer in taking notes. I find that writing something down increases my chances of committing the item to memory. Failing that, it's always possible to refer back to the notes at a later time.

Working a tech job for a number of years, primarily under Linux, I was unable to find a decent established notes solution. At the same time, there were often a number of files I would generate on a particular day that weren't full documents. They may have been assets ready to be uploaded to a jira ticket or graphs rendered out from stored metrics. At any rate, these are files that may be referred to later, but are only useful when bound to a particular day.

I've started writing a [script](https://github.com/mdsummers/today_cmd) called `today` which should help to provide a solution for some of the above issues.

{% highlight text %}
$ today
/Users/matt/today/2015-07-15
{% endhighlight %}

### Write or amend a set of notes for today
Opens a file called `notes.md` under today's directory using the default editor.
{% highlight text %}
today notes
{% endhighlight %}

### Change directory to today
{% highlight text %}
today cd
{% endhighlight %}

### See more usage information
{% highlight text %}
today help
{% endhighlight %}

## See also
* [Github repo](https://github.com/mdsummers/today_cmd)
