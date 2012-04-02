Fibrous
=======

Easily mix asynchronous and synchronous programming styles in node.js.

Install
-------

    npm install fibrous

Fibrous requires node version 0.6.x or greater.

Usage
-----

Would you rather write this:

```javascript
var updateUser = function(id, attributes, callback) {
  User.findOne(id, function (err, user) {
    if (err) return callback(err);
    
    user.set(attributes);
    user.save(function(err, updated) {
      if (err) return callback(err);

      callback(null, updated);
    });
  });
});
```

Or this, which behaves identically to calling code?

```javascript
var updateUser = fibrous(function(id, attributes) {
  user = User.sync.findOne(id);
  user.set(attributes);
  return user.sync.save();
});
```

Or even better, with [coffeescript](coffeescript.org):

```coffeescript
updateUser = fibrous (id, attributes) ->
  user = User.sync.findOne(id)
  user.set(attributes)
  user.sync.save()
```

### Without Fibrous

Using standard node callback-style APIs without fibrous, we write 
(from [the fs docs](http://nodejs.org/docs/v0.6.14/api/fs.html#fs_fs_readfile_filename_encoding_callback)):

```javascript
fs.readFile('/etc/passwd', function (err, data) {
  if (err) throw err;
  console.log(data);
});
```

### Using sync

Using fibrous, we write:

```javascript
data = fs.sync.readFile('/etc/passwd');
console.log(data);
```

### Using future

This is the same as writing:

```javascript
future = fs.future.readFile('/etc/passwd');
fibrous.wait(future);
data = future.get();
console.log(data);
```

### Waiting for Multiple Futures

Or for multiple files read asynchronously:

```javascript
futures = []
futures.push(fs.future.readFile('/etc/passwd'));
futures.push(fs.futures.readFile('/etc/hosts'));
fibrous.wait(futures);                               // or fibrous.wait(future1, future2,...)
console.log(futures[0].get());
console.log(futures[1].get())
```

Note that `fs.sync.readFile` is **not** the same as `fs.readFileSync`. The
latter blocks while the former allows the process to continue while
waiting for the file read to complete.

Fibrous Requires a Fiber for sync and wait
--------------------------------

Fibrous uses [node-fibers](https://github.com/laverdet/node-fibers)
behind the scenes.

`fibrous.wait` and `fibrous.sync` (which uses `wait`
internally) require that they are called within a fiber. Fibrous
provides two easy ways to do this.

### 1. fibrous Function Wrapper

Pass any function to `fibrous` and it returns a function that
conforms to standard node async APIs with a callback as the last
argument that expects `err` as the first argument and the function
result as the second. Any exception thrown will be passed to the
callback as an error.

```javascript
var asynFunc = fibrous(function() {
  return fs.sync.readFile('/etc/passwd');
});
```

is functionally equivalent to:

```javascript
var asyncFunc = function(callback) {
  fs.readFile('/etc/passwd', function(err, data) {
    if (err) return callback(err);

    callback(null, data);
  });
}
```

With coffeescript, the fibrous version is even cleaner:

```coffeescript
asyncFunc = fibrous ->
  fs.sync.readFile('/etc/passwd')
```

`fibrous` ensures that the passed function is
running in an existing fiber (from higher up the call stack) or will
create a new fiber if one does not already exist.

### 2. Connect Middleware

Fibrous provides [connect](http://www.senchalabs.org/connect/)
middleware that ensures that every request runs in a fiber.
If you are using [express](http://expressjs.com/), you'll
want to use this middleware.

```javascript
var express = require('express');
var fibrous = require('fibrous');

var app = express.createServer();

app.use(fibrous.middleware);

app.get('/', function(req, res){
    data = fs.sync.readFile('./index.html', 'utf8');
    res.send(data);
});
```

Error Handling / Exceptions
---------------------------

In the above examples, if `readFile` produces an error, the fibrous versions
(both `sync` and `wait`) will throw an exception. Additionally, the stack
trace will include the stack of the calling code unlike exceptions
typically thrown from within callback.


Testing
-------

Fibrous provides a test helper for [jasmine-node](https://github.com/mhevery/jasmine-node) 
that ensures that `beforeEach`, `it`, and `afterEach` run in a fiber.
Require it in your shared `spec_helper` file or in the spec files where
you want to use fibrous.

```javascript
require('fibrous/lib/jasmine_spec_helper');

describe('My Spec', function() {
  
  it('tests something asynchronous', function() {
    data = fs.sync.readFile('/etc/password');
    expect(data.length).toBeGreaterThan(0);
  });
});
```

If an asynchronous method called through fibrous returns an error, the
spec helper will fail the spec.


Console
-------

Fibrous can make it easier to work with asynchronous methods in the
console. It's not convenient to create a fiber to run console commands with `sync`, but
using `future` is still easier than constructing callbacks in the
console.

```
> fs = require('fs');
> require('fibrous');
> data = fs.future.readFile('/etc/passwd', 'utf8');
> data.get()
```

Behind The Scenes
-----------------

Fibrous uses the `Future` implementation from [node-fibers](https://github.com/laverdet/node-fibers).
`fibrous.wait` is a convenience passthrough to `Fiber.wait`.

Fibrous mixes `future` and `sync` into `Function.prototype` so you can
use them directly as in:

```javascript
readFile = require('fs').readFile;
data = readFile.sync('/etc/passwd');
```

Fibrous adds `future` and `sync` to `Object.prototype` correctly so they
are not enumerable.

These proxy methods also ignore all getters, even those that may
return functions. If you need to call a getter with fibrous that returns an
asynchronous function, you can do:

```javascript
func = obj.getter
func.future.call(obj, args)
```

### Disclaimer

We know that many object to libraries that mix in to Object.prototype
and Function.prototype. If that's how you feel, then fibrous is probably
not for you. We've been careful to mix in 'right' so that we don't
change property enumeration and find that the benefits of having sync
and future available without explicitly wrapping objects or functions
are worth the philosophical tradeoffs.

Gotchas
-------

The first time you call `sync` or `future` on an object, it builds the sync
and future proxies so if you add a method to the object later, it will
not be proxied.

Contributors
------------

* Randy Puro [rpuro](https://github.com/randypuro)
* Alon Salant [asalant](https://github.com/asalant)
* Bob Zoller [bobzoller](https://github.com/bobzoller)

License
-------

(The MIT License)

Copyright (c) 2012 Good Eggs, Inc.

Permission is hereby granted, free of charge, to any person obtaining a copy of this software and associated documentation files (the 'Software'), to deal in the Software without restriction, including without limitation the rights to use, copy, modify, merge, publish, distribute, sublicense, and/or sell copies of the Software, and to permit persons to whom the Software is furnished to do so, subject to the following conditions:

The above copyright notice and this permission notice shall be included in all copies or substantial portions of the Software.

THE SOFTWARE IS PROVIDED 'AS IS', WITHOUT WARRANTY OF ANY KIND, EXPRESS OR IMPLIED, INCLUDING BUT NOT LIMITED TO THE WARRANTIES OF MERCHANTABILITY, FITNESS FOR A PARTICULAR PURPOSE AND NONINFRINGEMENT. IN NO EVENT SHALL THE AUTHORS OR COPYRIGHT HOLDERS BE LIABLE FOR ANY CLAIM, DAMAGES OR OTHER LIABILITY, WHETHER IN AN ACTION OF CONTRACT, TORT OR OTHERWISE, ARISING FROM, OUT OF OR IN CONNECTION WITH THE SOFTWARE OR THE USE OR OTHER DEALINGS IN THE SOFTWARE.


