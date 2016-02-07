---
title: "Node.js - child process timeouts"
date:  2015-10-27 23:24:00
description: "Some notes on timeouts in the child_process module for node.js"
keywords: node.js, node, child, process, timeouts, gotcha, milliseconds, seconds, javascript
---

One of the options when `exec`ing a child process is `timeout`. Setting this means that a kill signal will automatically be sent after the given time period.

~~~javascript
var cp = require('child_process');

cp.execFile('/usr/bin/env', ['sleep', '10'], {
  timeout: 1000
}, function (err, stdout, stderr) {
  console.log(err);
});
~~~

Running this gives the following:

~~~
$ node timeout.js
{ [Error: Command failed: ] killed: true, code: null, signal: 'SIGTERM' }
~~~

Note that the parameter `timeout` takes a value in milliseconds. Setting the timeout to 1 (plenty of time if the unit was in seconds) can lead to unpredictable results, as the process is signaled at creation.

When creating a child process from a setuid binary, node is unable to send a kill signal.

~~~javascript
var cp = require('child_process');

cp.execFile('/usr/bin/sudo', ['sleep', '5'], {
  timeout: 10
}, function (err, stdout, stderr) {
  console.log(err);
});
~~~
~~~
$ node timeout2.js
{ [Error: kill EPERM] code: 'EPERM', errno: 'EPERM', syscall: 'kill' }
$ man 2 kill | grep -A 4 EPERM
[EPERM]     The sending process is not the super-user and its
            effective user id does not match the effective user-id
            of the receiving process.  When signaling a process
            group, this error is returned if any members of the
            group could not be signaled.
$
~~~
The callback executes when the timeout kicks in, with the error message that the kill syscall failed with errno EPERM. However, the `node` process stays alive until the `sleep` ends.

Strangely, if the `console.log` is swapped for a `throw`, no output is ever seen.

~~~javascript
var cp = require('child_process');

cp.execFile('/usr/bin/sudo', ['sleep', '5'], {
  timeout: 10
}, function (err, stdout, stderr) {
  throw(err);
});
~~~

~~~nohighlight
$ time node timeout3.js

real	0m5.122s
user	0m0.079s
sys	0m0.038s
$
~~~
