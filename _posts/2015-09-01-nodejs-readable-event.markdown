---
title: "Node.js and the readable event"
date:  2015-09-01 20:53:00
description: "A look at the readable event and the alternative paused mode of reading streams in node.js"
keywords: tags, node.js , node, event, streams, paused mode, flowing mode, javascript
---

When it comes to readable streams in node.js, perhaps the most common idiom is the following (using standard input as an example):

~~~javascript
var rebuilt = '';

process.stdin.on('data', function (chunk) {
  rebuilt += chunk;
});

process.stdin.on('end', function () {
  console.log(rebuilt);
});
~~~

That is, using one or more `data` events and their associated chunks to rebuild the entire input. This uses the **flowing mode** of readable streams, basically relaying the stream as fast as possible. The `data` event brings with it the data chunk, and something must be done with it.

This isn't the only way of handling readable streams though. There is also **paused mode**, which can used equivalently using the `readable` event:

~~~javascript
var rebuilt = ''

process.stdin.on('readable', function () {
  var chunk = process.stdin.read();
  if (chunk !== null) {
    rebuilt += chunk;
  }
});

process.stdin.on('end', function () {
 console.log(rebuilt);
});
~~~

With `readable`, the event highlights the presence of bytes on the internal buffer, but does nothing in itself to remove them. That must be done with a call to `read()`. An argument can be given as the number of bytes to take off the internal buffer, otherwise all are taken.

Taking one byte at a time:

~~~javascript
process.stdin.setEncoding('utf8');

process.stdin.on('readable', function () {
  console.log('readable event');
  var char;
  while ((char = process.stdin.read(1)) !== null) {
    console.log('char: ' + char);
  }
});

process.stdin.on('end', function () {
  console.log('end');
});
~~~

~~~
$ echo test | node readable3.js
readable event
char: t
char: e
char: s
char: t
char:

readable event
end $
~~~

After the entire buffer is read, any new input will cause a new `readable` event to be emitted. However, if the buffer is not emptied, and no new data is added to the buffer, no new event will be emitted. This has the effect of causing node.js with stdin as a tty to exit, rather than wait for more input.

~~~javascript
process.stdin.on('readable', function () {
  console.log('Event: readable');
  var firstByte = process.stdin.read(1);
  console.log('First byte of buffer is "%s"', firstByte);
});

process.stdin.on('end', function () {
  console.log('Event: end');
});
~~~

Running interactively:
~~~
$ node firstByte.js
Event: readable
First byte of buffer is "null"
test
Event: readable
First byte of buffer is "t"
~~~
Whereas, if only newline characters are sent to the script, it will continue to receive events and wait on input.

This is not the case when stdin is coming from another source.

~~~
$ { while :; do echo test; sleep 5; done } | node firstByte.js
Event: readable
First byte of buffer is "t"
Event: readable
First byte of buffer is "e"
Event: readable
First byte of buffer is "s"
Event: readable
First byte of buffer is "t"
...
~~~

In the above test, the delay between readable events was roughly 10 seconds. When using `cat` I could see that the events were being generated every second newline (corresponding to two 5 second sleep loops). Presumably there are other conditions that will cause an additional `readable` event to be generated?

### See also
* [Stack Overflow: What are the differences between readable and data event of process.stdin stream](http://stackoverflow.com/questions/26174308/what-are-the-differences-between-readable-and-data-event-of-process-stdin-stream)
* [Stream - Node.js Manual & Documentation](https://nodejs.org/api/stream.html)
