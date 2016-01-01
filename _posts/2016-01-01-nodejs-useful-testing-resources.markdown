---
title: "A useful testing resource for node.js"
date:  2016-01-01 11:53:00
description: "An article on various testing modules"
keywords: node.js, node, test, framework, unit, resource, nock, supertest, javascript
---
I recently got to adding a test suite to an established module I had developed for work. The following resource was especially helpful in pointing me in the right direction:

[Unit testing, the Clock way](http://www.clock.co.uk/blog/tools-for-unit-testing-and-quality-assurance-in-node-js)

Briefly outlined, the modules I started using:
* [Mocha](https://mochajs.org/) - test framework
* [should.js](https://github.com/shouldjs/should.js) - assertion library
* [Nock](https://github.com/pgte/nock) - http request interception and mocking
* [supertest](https://github.com/visionmedia/supertest) - http server endpoint testing
* [Istanbul](https://github.com/gotwarlost/istanbul) - a code coverage tool
* [jshint](https://github.com/jshint/jshint/) - javascript linting
* [JSCS](https://github.com/mdevils/node-jscs) - code style enforcing

Next step - a continuous integration server that plays well with gitlab merge requests.
