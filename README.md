# mocha.parallel

Speed up your IO bound async specs by running them at the same time. Compatible
with node/iojs, and most versions of mocha.

[![Build Status](https://travis-ci.org/danielstjules/mocha.parallel.svg?branch=master)](https://travis-ci.org/danielstjules/mocha.parallel)

## Installation

```
npm install --save mocha.parallel
```

## Overview

``` javascript
/**
 * Generates a suite for parallel execution of individual specs. While each
 * spec is ran in parallel, specs resolve in series, leading to deterministic
 * output. Compatible with both callbacks and promises. Supports hooks, pending
 * or skipped specs, but not nested suites.
 *
 * @example
 * parallel('setTimeout', function() {
 *   it('test1', function(done) {
 *     setTimeout(done, 500);
 *   });
 *   it('test2', function(done) {
 *     setTimeout(done, 500);
 *   });
 * });
 *
 * @param {string}   name
 * @param {function} fn
 */
```

## Examples

In the examples below, imagine that `setTimeout` is a function that performs
some async IO with the specified delay. This could include requests to your
http server using a module like `supertest` or `request`. Or maybe a headless
browser using `zombie` or `nightmare`.

#### Simple

Rather than taking 1.5s, the specs below run in parallel, completing in just
over 500ms.

``` javascript
var parallel = require('mocha.parallel');
var Promise  = require('bluebird');

parallel('delays', function() {
  it('test1', function(done) {
    setTimeout(done, 500);
  });

  it('test2', function(done) {
    setTimeout(done, 500);
  });

  it('test3', function() {
    return Promise.delay(500);
  });
});
```

```
  delays
    ✓ test1 (500ms)
    ✓ test2
    ✓ test3


  3 passing (512ms)
```

#### Isolation

Individual parallel suites run in series and in isolation from each other.
In the example below, the two specs in suite1 run in parallel, followed by
those in suite2.

``` javascript
var parallel = require('mocha.parallel');

parallel('suite1', function() {
  it('test1', function(done) {
    setTimeout(done, 500);
  });

  it('test2', function(done) {
    setTimeout(done, 500);
  });
});

parallel('suite2', function() {
  it('test1', function(done) {
    setTimeout(done, 500);
  });

  it('test2', function(done) {
    setTimeout(done, 500);
  });
});
```

```
  suite1
    ✓ test1 (503ms)
    ✓ test2

  suite2
    ✓ test1 (505ms)
    ✓ test2


  4 passing (1s)
```

#### Error handling

Uncaught exceptions are associated with the spec that threw them, despite them
all running at the same time. So debugging doesn't need to be too difficult!

``` javascript
var parallel = require('mocha.parallel');

parallel('uncaught', function() {
  it('test1', function(done) {
    setTimeout(done, 500);
  });

  it('test2', function(done) {
    setTimeout(function() {
      // Thrown while test1 is executing
      throw new Error('test');
    }, 100);
  });

  it('test3', function(done) {
    setTimeout(done, 500);
  });
});
```

```
  uncaught
    ✓ test1 (501ms)
    1) test2
    ✓ test3


  2 passing (519ms)
  1 failing

  1) uncaught test2:
     Error: test
      at null._onTimeout (fixtures/uncaughtException.js:11:13)
```

#### Hooks

Hook behavior may not be as intuitive when ran using this library.

``` javascript
var parallel = require('mocha.parallel');
var assert   = require('assert');

parallel('hooks', function() {
  var i = 0;

  beforeEach(function(done) {
    // The beforeEach will be invoked twice,
    // before either spec starts
    setTimeout(function() {
      i++;
      done();
    }, 50);
  });

  it('test1', function(done) {
    // Incremented by 2x beforeEach
    setTimeout(function() {
      assert.equal(i, 2);
      done();
    }, 1000);
  });

  it('test2', function(done) {
    // Incremented by 2x beforeEach
    setTimeout(function() {
      assert.equal(i, 2);
      done();
    }, 1000);
  });
});
```

## Notes

Debugging parallel execution can be more difficult as exceptions may be thrown
from any of the running specs. Also, the use of the word "parallel" is in the
same spirit as other nodejs async control flow libraries, such as
https://github.com/caolan/async#parallel, https://github.com/creationix/step
and https://github.com/tj/co#yieldables This library does not offer true
parallelism using multiple threads/workers/fibers, or by spawning multiple
processes.
