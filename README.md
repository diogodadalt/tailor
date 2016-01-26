<h1><img width="400" alt="Tailor" src="https://rawgithub.com/zalando/tailor/master/tailor.svg"></h1>

[![Build Status](https://travis-ci.org/zalando/tailor.svg?branch=master)](https://travis-ci.org/zalando/tailor)
[![Test Coverage](https://codeclimate.com/github/zalando/tailor/badges/coverage.svg)](https://codeclimate.com/github/zalando/tailor/coverage)

Tailor is a layout service that uses streams to compose a web page from fragment services.

# Installation

`npm i node-tailor --save`

```javascript
const http = require('http');
const Tailor = require('node-tailor');
const tailor = new Tailor({/* Options */});
const server = http.createServer(tailor.requestHandler);
server.listen(process.env.PORT || 8080);
```

# Options

* `filterHeaders(fragment.attributes, headers)` a function that receives fragment attributes and request headers and should return headers for the fragment. Useful to do some whitelisting of the custom headers. A default draft implementation is in `lib/filter-headers.js`
* `fetchContext(request)` a function that returns a promise of the context, that is an object that maps fragment id to fragment url, to be able to override urls of the fragments on the page, defaults to `Promise.resolve({})`
* `fetchTemplate(request, parseTemplate)` a function that should fetch the template, call `parseTemplate` and return a promise of the result. Useful to implement your own way to retrieve and cache the templates, e.g. from s3. Default implementation `lib/fetch-template.js` streams the template from  the file system
* `fragmentTag` a name of the fragment tag, defaults to `fragment`
* `handledTags` an array of custom tags
* `handleTag` receives a tag or closing tag and serializes it to a string or returns a stream
* `forceSmartPipe(request)` returns a boolean that forces all async fragments in sync mode
* `requestFragment(fragmentAttributes, headers)` a function that returns a promise of request to a fragment server, check the default implementation in `lib/request-fragment`

# Template

Tailor uses [sax](https://github.com/isaacs/sax-js) to parse the template, where it replaces each `fragmentTag` with a stream from the fragment server and `handledTags` with the result of `handleTag` function.

```html
<html>
<head>
    <fragment src="http://assets.domain.com" inline>
</head>
<body>
    <fragment src="http://header.domain.com">
    <fragment src="http://content.domain.com" primary>
    <fragment src="http://footer.domain.com" async>
</body>
</html>
```

In order to initialize the fragments on the frontend, Tailor expects an [AMD](https://github.com/amdjs/amdjs-api/wiki/AMD) loader such as `requirejs` and a special `pipe` function to be defined in the head of the page. Check the `example/templates/index.html` for the draft implementation.

## Fragment attributes

* *id* — optional unique identifier (autogenerated)
* *src* — URL of the fragment
* *primary* — denotes a fragment that sets the response code of the page
* *async* — postpones the fragment until the end of body tag
* *inline* — excludes the div wrapper
* *public* — doesn't send the headers to fragment server

## Fragment server

A fragment is an http(s) server that renders only the part of the page and sets `Link` header to provide urls to CSS and JavaScript resources. Check `example/fragment.js` for the draft implementation.

A JavaScript of the fragment is an AMD module, that exports an `init` function, that will be called with DOM element of the fragment as an argument.

# Events

`Tailor` extends `EventEmitter`, so you can subscribe to events with `tailor.on('eventName', callback)`.

Events may be used for logging and monitoring. Check `perf/benchmark.js` for an example of getting metrics from Tailor.

## Top level events

* Client request received: `start(request)`
* Response started (headers flushed and stream connected to output): `response(request, status, headers)`
* Response ended (with the total size of response): `end(request, contentSize)`
* Template Error: `template:error(request, error)` in case an error fetching or parsing the template
* Context Error: `context:error(request, error)` in case of an error fetching the context
* Primary error: `primary:error(request, fragment.attributes, error)` in case of socket error, timeout, 50x of the primary fragment

## Fragment events

* Request start: `fragment:start(request, fragment.attributes)`
* Response Start when headers received: `fragment:response(request, fragment.attributes, status, headers)`
* Response End (with response size): `fragment:end(request, fragment.attributes, contentSize)`
* Error: `fragment:error(request, fragment.attributes, error)` in case of socket error, timeout, 50x

# Example

To start an example execute `npm run example` and open [http://localhost:8080/index](http://localhost:8080/index).

# Benchmark

To start running benchmark execute `npm run benchmark` and wait for couple of seconds to see the results.
