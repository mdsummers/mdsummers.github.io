---
title: "Ansible on FreeBSD: Faster setup"
date:  2016-08-18 19:30:00
description: "Digging into performance bottlenecks when using Ansible under FreeBSD"
keywords: ansible, freebsd, bsd, ulimit, limits, setup, module, slow, fd, file descriptor, truss, cProfile, fork, exec
---

In a stock FreeBSD install Ansible's "setup" task can take a really long time. Testing against a dual xeon with 256GB of memory I observed the task consistently taking over 15 seconds to complete. Compare this to a 2-core Ubuntu 16.04 vm taking a couple of seconds. Something is very wrong!

In the single jail test I have my `hosts` file as follows:

~~~nohighlight
my_single_jail ansible_connection=jail ansible_python_interpreter=/usr/local/bin/python
~~~

and I make sure to keep the generated ansible script on the target host when running the setup module in isolation.

~~~nohighlight
sudo ANSIBLE_KEEP_REMOTE_FILES=1 ansible -m setup -i hosts my_single_jail
~~~

The remote file, a script, can be found under `/root/.ansible` on the target host (`ansible_connection=jail` requires Ansible be run as root rather than becoming root with something like `sudo` or `doas`).

Running the script under [`truss`](https://www.freebsd.org/cgi/man.cgi?truss) and following forks processes gives some interesting results...
~~~nohighlight
truss -f /usr/local/bin/python .ansible/tmp/ansible-tmp-1471479743.01-38328629651633/setup
~~~

as the output is filled with close syscalls against ascending file descriptors:

~~~nohighlight
8975: close(117158)	ERR#9 'Bad file descriptor'
8975: close(117159)	ERR#9 'Bad file descriptor'
8975: close(117160)	ERR#9 'Bad file descriptor'
8975: close(117161)	ERR#9 'Bad file descriptor'
8975: close(117162)	ERR#9 'Bad file descriptor'
8975: close(117163)	ERR#9 'Bad file descriptor'
8975: close(117164)	ERR#9 'Bad file descriptor'
8975: close(117165)	ERR#9 'Bad file descriptor'
8975: close(117166)	ERR#9 'Bad file descriptor'
8975: close(117167)	ERR#9 'Bad file descriptor'
8975: close(117168)	ERR#9 'Bad file descriptor'
8975: close(117169)	ERR#9 'Bad file descriptor'
8975: close(117170)	ERR#9 'Bad file descriptor'
~~~

On my testbed this number grow into the millions and took a few minutes before my SIGINT was able to stop the process.

But what code is causing this? Fortunately python ships with a [great module](https://docs.python.org/2/library/profile.html#module-cProfile) that allows us to profile the execution of a script by function.

~~~nohighlight
python -m cProfile -s cumtime .ansible/tmp/ansible-tmp-1471479743.01-38328629651633/setup
~~~

Running this we can see that most of the cumulative time of execution is caught up running subprocesses:

~~~nohighlight
setup:64(<module>)
setup:131(main)
setup:81(run_setup)
setup:5154(ansible_facts)
setup:1890(run_command)
subprocess.py:650(__init__)
subprocess.py:1195(_execute_child)
~~~

~~~nohighlight
$ pkg list python27 | grep subprocess.py
/usr/local/lib/python2.7/subprocess.py
...
~~~

Running a subprocess requires fork'ing the python process and exec'ing the new command. After the fork a certain amount of tidying up of preparation is done in the new environment pre-exec. Part of this means closing any inherited file descriptors that are not required.

~~~python
if close_fds:
    self._close_fds(but=errpipe_write)
~~~

What does this function do?

~~~python
def _close_fds(self, but):
    if hasattr(os, 'closerange'):
        os.closerange(3, but)
        os.closerange(but + 1, MAXFD)
~~~

It closes all file descriptors that aren't the error pipe, upto `MAXFD` which is defined above as

~~~python
try:
    MAXFD = os.sysconf("SC_OPEN_MAX")
except:
    MAXFD = 256
~~~

What does that `sysconf` evaluate to on our system?

~~~nohighlight
$ python
>>> import os
>>> os.sysconf("SC_OPEN_MAX")
7546230
~~~

That's 7 million wasted syscalls everytime we try to run a subprocess.

~~~nohighlight
$ ulimit -n
7546230
~~~

Yes, the limits for maxfiles are maximum by default. Let's fix it:

~~~nohighlight
limits -n 1024 /usr/local/bin/python .ansible/tmp/ansible-tmp-1471479743.01-38328629651633/setup
~~~

The setup code now completes in under a second. How do we fix this for actual ansible runs?

### Solution 1

**Only works on Ansible < 2.1**

The [BSD Support](http://docs.ansible.com/ansible/intro_bsd.html) page on Ansible's site notes that the ansible_python_interpreter host_var should be set to `/usr/local/bin/python`. We have to go one step further to include the maxfiles limit:

~~~nohighlight
ansible_python_interpreter="limits -n 1024 /usr/local/bin/python"
~~~

This is broken in Ansible 2.1 as per [this bug report](https://github.com/ansible/ansible/issues/15635). A patch was submitted and merged in, but it only fixes the case where `/usr/bin/env <command>` is used.

### Solution 2

We can make a custom wrapper for python, that applies the limit and runs python.

~~~bash
#!/bin/sh
exec limits -n 1024 /usr/local/bin/python "$@"
~~~

Save this, make it executable and refer to it in `host_vars`:

~~~nohighlight
ansible_python_interpreter="/usr/local/bin/pythonwrapper"
~~~

This requires additional complexity on a jail setup, as all of the jails must have a copy of this wrapper available.

### Solution 3

Alter limits for the user running ansible (root for me) under `/etc/login.conf` and run `cap_mkdb /etc/login.conf` to update the login class database.

## Aside: Why are the file limits so high?

The FreeBSD handbook section on [tuning kernel limits](https://www.freebsd.org/doc/handbook/configtuning-kernel-limits.html) covers the `kern.maxfiles` sysctl:

~~~nohighlight
The read-only sysctl(8) variable kern.maxusers is automatically sized at boot based on the amount of memory available in the system
~~~

The beefier the box is, the slower it will run Ansible's setup without modifications.

One of the fantastic things about FreeBSD is that the source code for the system can typically be found under `/usr/src`. The code that determines `maxfiles` and `maxfilesperproc` can be found under `sys/kern/subr_param.c`.