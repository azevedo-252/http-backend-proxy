#$httpBackend Proxy
[![Build Status](https://travis-ci.org/kbaltrinic/http-backend-proxy.svg?branch=master)](https://travis-ci.org/kbaltrinic/http-backend-proxy)

This is a node module for use with the [AngularJS Protractor][protractor] end-to-end testing framework.  It allows you to make configuration calls to [$httpBackend][httpBackend] from within Protractor itself.  Being able to configuring the mock backend along side the tests that depend upon it improves the modularity, encapsulation and flexibility of your tests.  The proxy allows $httpBackend to live up to its full potential, making it easier to test angular web applications in abstraction from their backend services.

### Credits
The proxy itself is my own work.  However much of the test application and project structure, etc. is based off of [the angular-seed project][angular-seed].  This includes significant parts of this README.

## Getting and Using the Proxy
The proxy can be installed into your project by calling `npm install http-backend-proxy` or better yet, just add `http-backend-proxy` to devDependencies in your `package.json` file.  The current version is 1.1.0.  The sofware is released under the MIT license.

To instantiate an instance of the proxy, require it and create a new instance:
```JavaScript
var HttpBackend = require('http-backend-proxy');
var proxy = new HttpBackend(browser);
```
`browser` is, of course, the protractor browser object.  A configuration object may be passed as a second argument to the constructor.  Available configuration options are discussed further below.

The proxy supports the same interface as $httpBackend so [see its docs][httpBackend] for usage.  The main difference is that all proxied methods return promises in the fashion of most other Protractor methods.  (Exception: see Buffered Mode below.)

See the end-to-end tests in `test/e2e` for some examples of usage.

###Buffered Mode
It is possible to use the proxy in a 'buffered' mode by adding `buffer: true` to the configuration object passed to proxy constructor.  When buffering is enabled, proxied methods return void and do not immediately pass their calls through to the browser. Calling `flush()` will pass all buffered calls through to the browser at once and return a promise that will be fulfilled after all calls have been executed.

Buffering is the generally recommended approach because in most cases you will need to make numersous calls to the proxy to set things up the way you want them for a given spec.  Buffering will reduce the number of remote calls and considerably speed up your test setup.

###Calling Functions and Sharing Data between Tests and Browser
Using the proxy to configure $httpBackend for simple calls is, well, simple:
```JavaScript
proxy.whenGET('/logoff').respond(200);
```
But what if I want to return some big piece of JSON?  If we were coding our test in the browser we might write something like:
```JavaScript
var searchResults = require('./mock-data/searchResults.json');
$httpBackend.whenGET('/search&q=beatles').respond(200, searchResults)
```
This too will work just fine as long as at the time you call `respond()` on the proxy, searchResults resolves to a JSON object (or anything you would expect to find in a JSON object: strings, numbers, regular expressions, arrays, etc.

To get a little fancier, you can even do this:
```JavaScript
proxy.when('GET', /.*/)
    .respond(function(method, url){return [200, 'You called: ' + url];});
```
That will work just fine!  The proxy will serialize the function and send it up to the browser for execution.

But what about:
```JavaScript
var searchResults = require('./mock-data/searchResults.json');

function getQueryTermsFrom(url){  ... implemenation omitted ... };

proxy.when('GET', /search\?q=.*/)
    .respond(function(method, url){
      var term = getQueryTermFrom(url);
      var fixedUpResponse = searchResults.replace(/__TERM__/, term);
      return [200, fixedUpResponse];
    });
```
Now we have a problem.  The proxy can serialize the function just fine and send it to the browser, but when it gets there, `getQueryTermFrom` and `searchResults` are not going to resolve.  This calls for...

####Using the Context Object
The proxy provides a special object called the context object that is intended to help out in theses kinds of situations.  The context object is a property of the proxy, called `context` oddly enough, the value of which will be syncronized between the proxy and the browser-side $httpBackend before any proxied calls are executed on the browser.  With this object in place we can now do the following:

```JavaScript
var searchResults = require('./mock-data/searchResults.json');

proxy.context = {
  searchResults: searchResults,
  getQueryTermsFrom: function (url){  ... implemenation omitted ... };
}

proxy.when('GET', /search\?q=.*/)
    .respond(function(method, url){
      var term = $httpBackend.context.getQueryTermFrom(url);
      var fixedUpResponse = $httpBackend.context.searchResults.replace(/__TERM__/, term);
      return [200, fixedUpResponse];
    });
```

*Hint:* If we rename our in-test proxy from `proxy` to `$httpBackend`, our tests will more easily get through any linting we may have set up.

When buffering is turned off, the context object is synchronized with the browser before every call to `respond()`.  When buffering is turned on, this syncronization occurs as part of the call to `flush()`.  This is important, because is it the value of the context object at the time of syncronization, not at the point when `respond()` is called on the proxy, that determines its value in the browser.  More precisely, since values in the context object may not even be evaluated in the browser until an HTTP call occurs, the value at the time of the HTTP call will be the value of the object as of the most recent prior syncronization.  Said another way, there is no closure mechanism in play here.

####Configuring the Context Object
If for some reason you don't like your context object being called 'context' (or more importantly, if it should turn out to conflict with a future property of $httpBackend), it can be renamed by adding `contextField: "a-new-field-name"` to the configuration object that the proxy is constructed with.  Passing a value of `false` will disable the context passing mechanism all together.

###Resetting the Mock
The underlying $httpBackend mock does not support resetting the set of configured calls.  So there is no way to do this through the proxy either.  The simplest solution is to use `browser.get()` to reload your page.  This of course resets the entire application state, not just that of the $httpBackend.  Doing so may not seem ideal but if used wisely will give you good test isolation as well resetting the proxy.

###Configuring the App-Under-Test
Somewhere in the application-under-test, you need to add the following line.

```JavaScript
window.$httpBackend = $httpBackend;
```
The `module.run()` block of your top-level application module is probably the best place to put this.  This exposes $httpBackend globally so that it becomes accessible to the proxy.  _If anyone has ideas on how to avoid needing this, please let me know._

## The Test-Harness Application

The angular application that makes up most of this repository is simply a test harness for the proxy.  It enables testing the proxy by way of Protractor.  However it can also be used to manualy tests the 'hard-coded' $httpBackend configuration set up in `app.js`.  Manual testing can usefull for debugging problems during proxy development.

###Setup

#### Install Dependencies

There are three kinds of dependencies in this project: tools, angular framework code and bootstrap css.  The tools help us manage and test the application.

* We get the tools we depend upon via `npm`, the [node package manager][npm].
* We get the angular code via `bower`, a [client-side code package manager][bower].
* I added `bootstrap.css` as a hardcoded dependency under `app/css` just to make the test harness look nice. :-)  Stricktly speaking it is not needed.

`npm` is preconfigured to automatically run `bower` so you can simply do:

```
npm install
```

Behind the scenes this will also call `bower install`.  You should find that you have two new
folders in your project.

* `node_modules` - contains the npm packages for the tools we need
* `bower_components` - contains the angular framework files

#### Run the Test Harness

The project is preconfigured with a simple development web server.  The simplest way to start
this server is:

```
npm start
```

Now browse to the app at `http://localhost:8000/app/index.html`.


### Directory Layout

    app/                --> all of the files to be used in production
      css/              --> css files
        app.css         --> default stylesheet
        bootstrap.css   --> bootstrap stylesheet
      index.html        --> that test app's single page and view.
      js/               --> javascript files
        app.js          --> the application's angular code

    test/               --> test config and source files
      protractor-conf.js    --> config file for running e2e tests with Protractor
      e2e/                  --> end-to-end specs
        ...
      lib/
        http-backend-proxy.js  ==>This is the proxy libary itself.
      unit/                 --> unit level specs/tests
        ...

## Testing

There are two kinds of tests in the project: Unit tests and end-to-end tests.

### Running the Unit Tests

The project contains unit tests for testing the proxy behavior in abstraction from Protractor and Selenium.  These are written in [Jasmine][jasmine] and run using the [Jasmine-Node][jasmine-node] test runner.

* the unit tests are found in `test/unit/`.

The easiest way to run the unit tests is to use the supplied npm script:

```
npm test
```

This script will start the Jasmine-Node test runner to execute the unit tests.

If you are actually doing development work on the proxy itself (Thank you!) and want to run the tests continuously as you work, use the following command line.


```
npm run test-watch
```

This will watch both the `test/unit` and `test/lib` directories.

### Running the End-to-End Testing

The project comes with end-to-end tests, again written in [Jasmine][jasmine]. These tests
are run using Protractor.  They also demonstrate how to actually use the proxy to manipulate $httpBackend in your tests.  I recommend taking a look.

* the configuration is found at `test/protractor-conf.js`
* the end-to-end tests are found in `test/e2e/`

To run them, first start the test harness.

```
npm start
```

In addition, since Protractor is built upon WebDriver we need to install this.  The angular-seed
project comes with a predefined script to do this:

```
npm run update-webdriver
```

This will download and install the latest version of the stand-alone WebDriver tool.

Once you have ensured that the development web server hosting the test harness is up and running
and WebDriver is updated, you can run the end-to-end tests using the supplied npm script:

```
npm run e2e
```

This script will execute the end-to-end tests against the application being hosted on the
development server.

##Feedback and Support
To provide feedback or get support, please [post an issue][issues] on the  GitHub project.

[angular-seed]: https://github.com/angular/angular-seed
[httpBackend]: http://docs.angularjs.org/api/ngMockE2E/service/$httpBackend
[bower]: http://bower.io
[npm]: https://www.npmjs.org/
[node]: http://nodejs.org
[protractor]: https://github.com/angular/protractor
[jasmine]: http://pivotal.github.com/jasmine/
[jasmine-node]: https://github.com/mhevery/jasmine-node
[http-server]: https://github.com/nodeapps/http-server
[issues]: https://github.com/kbaltrinic/http-backend-proxy/issues