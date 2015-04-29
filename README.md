# node-tap

A TAP test framework for Node.js.

This is a mix-and-match set of utilities that you can use to write test
harnesses and frameworks that communicate with one another using the
Test Anything Protocol.

It is also a test runner for consuming TAP-generating test scripts,
and a framework for writing such scripts.

## USAGE

Write your tests in JavaScript

```javascript
var tap = require('tap')

// you can test stuff just using the top level object.
// no suites or subtests required.

tap.equal(1, 1, 'check if numbers still work')
tap.notEqual(1, 2, '1 should not equal 2')

// also you can group things into sub-tests.
// Sub-tests will be run in sequential order always,
// so they're great for async things.

tap.test('first stuff', function (t) {
  t.ok(true, 'true is ok')
  t.similar({a: [1,2,3]}, {a: [1,2,3]})
  // call t.end() when you're done
  t.end()
})

// you can specify a 'plan' if you know how many
// tests there will be in advance. Handy when
// order is irrelevant and things happen in parallel.

tap.test('planned test', function (t) {
  t.plan(2)
  setTimeout(function () {
    t.ok(true, 'a timeout')
  })
  setTimeout(function () {
    t.ok(true, 'b timeout')
  })
})

// you can do `var test = require('tap').test` if you like
// it's pre-bound to the root tap object.

var test = require('tap').test

// subtests can have subtests
test('parent', function (t) {
  t.test('child', function (tt) {
    tt.throws(function () {
      throw new Error('fooblz')
    }, {
      message: 'fooblz'
    }, 'throw a fooblz')

    tt.throws(function () { throw 1 }, 'throw whatever')

    tt.end()
  })

  t.end()
})

// thrown errors just fail the current test, so you can
// also use your own assert library if you like.
// Of course, this means it won't be able to print out the
// number of passing asserts, since passes are silent.

test('my favorite assert lib', function (t) {
  var assert = require('assert')
  assert.ok(true, 'true is ok')
  assert.equal(1, 1, 'math works')

  // Since it can't read the plan, using a custom assert lib
  // means that you MUST use t.end()
  t.end()
})

// You can mark tests as 'todo' either using a conf object,
// or simply by omitting the callback.
test('solve halting problem')
test('prove p=np', { todo: true }, function (t) {
  // i guess stuff goes here
  t.fail('traveling salesmen must pack their own bags')
  t.end()
})

// Prefer mocha/rspec/lab style global objects?
// Got you covered.  This is a little experimental,
// patches definitely welcome.
tap.mochaGlobals()
describe('suite ride bro', function () {
  it('should have a wheel', function () {
    assert.ok(thingie.wheel, 'wheel')
  })
  it('can happen async', function (done) {
    setTimeout(function () {
      assert.ok('ok')
      done()
    })
  })
})

// Read on for a complete list of asserts, methods, etc.
```

You can run tests using the `tap` executable.  Put this in your
package.json file:

```json
{
  "scripts": {
    "test": "tap test/*.js"
  }
}
```

and then you can run `npm test` to run your test scripts.

Command line behavior and flags:

```
$ tap -h
Usage:
  tap [options] <files>

Executes all the files and interprets their output as TAP
formatted test result data.

To parse TAP data from stdin, specify "-" as a filename.

Options:

  -c --color                  Force use of colors

  -C --no-color               Force no use of colors

  -R<type> --reporter=<type>  Use the specified reporter.  Defaults to
                              'classic' when colors are in use, or 'tap'
                              when colors are disabled.

  -b --bail                   Bail out on first failure.

  -B --no-bail                Do not bail out on first failure.

                              Available reporters:
                              classic doc dot dump html htmlcov json
                              jsoncov jsonstream landing list markdown
                              min nyan progress silent spec tap xunit

  -gc --expose-gc             Expose the gc() function to Node tests

  --debug                     Run JavaScript tests with node --debug

  --debug-brk                 Run JavaScript tests with node --debug-brk

  --harmony                   Enable all Harmony flags in JavaScript tests

  --strict                    Run JS tests in 'use strict' mode

  -t<n> --timeout=<n>         Time out tests after this many seconds.
                              Defaults to 120, or the value of the
                              TAP_TIMEOUT environment variable.

  -h --help                   print this thing you're looking at

  -v --version                show the version of this program

  --                          Stop parsing flags, and treat any additional
                              command line arguments as filenames.
```

## API

### tap = require('tap')

The root `tap` object is an instance of the Test class with a few
slight modifications.

1. The `plan()` and `test()` methods are pre-bound onto the root
   object, so that you don't have to call them as methods.
2. By default, it pipes to stdout, so running a test directly just
   dumps the TAP data for inspection.
3. Various other things are hung onto it for convenience, since it is
   the main package export.

### tap.Test

The `Test` class is the main thing you'll be touching when you use
this module.

The most common way to instantiate a `Test` object by calling the
`test` method on the root or any other `Test` object.  The callback
passed to `test(name, fn)` will receive a child `Test` object as its
argument.

A `Test` object is a Readable Stream.  Child tests automatically send
their data to their parent, and the root `require('tap')` object pipes
to stdout by default.  However, you can instantiate a `Test` object
and then pipe it wherever you like.  The only limit is your imagination.

#### t.test(name, [options], [function])

Create a subtest.

If the function is omitted, then it will be marked as a "todo" or
"pending" test.

The options object can include the following fields:

* `todo` Mark this test as a "todo" item, and do not run it.
* `skip` Mark this test as a "skip" item, and do not run it.
* `bailOnFail` Respond to any assertion failure by bailing out.
* `timeout` Time in ms before this test should complete.  (Defaults to
  30000.)

Any additional data will be printed as diagnostic information if the
test fails.

#### t.plan(number)

Specify that a given number of tests are going to be run.

This may only be called *before* running any asserts or child tests.

#### t.end()

Call when tests are done running.  This is not necessary if `t.plan()`
was used.

#### t.bailout([reason])

Pull the proverbial ejector seat.

Use this when things are severely broken, and cannot be reasonably
handled.  Immediately terminates the entire test run.

#### t.passing()

Return true if everything so far is ok.

Note that all assert methods also return `true` if they pass.

#### t.comment(message)

Print the supplied message as a TAP comment.

Note that you can always use `console.error()` for debugging (or
`console.log()` as long as the message doesn't look like TAP formatted
data).

#### t.fail(message, extra)

Emit a failing test point.  This method, and `pass()`, are the basic
building blocks of all fancier assertions.

#### t.pass(message)

Emit a passing test point.  This method, and `fail()`, are the basic
building blocks of all fancier assertions.

### Advanced Usage

These methods are primarily for internal use, but can be handy in some
unusual situations.  If you find yourself using them frequently, you
*may* be Doing It Wrong.  However, if you find them useful, you should
feel perfectly comfortable using them.

#### t.stdin()

Parse standard input as if it was a child test named `/dev/stdin`.

This is primarily for use in the test runner, so that you can do
`some-tap-emitting-program | tap other-file.js - -Rnyan`.

#### t.spawn(command, arguments, [options], [name], [extra])

Sometimes, instead of running a child test directly inline, you might
want to run a TAP producting test as a child process, and treat its
standard output as the TAP stream.

That's what this method does.

It is primarily used by the executable runner, to run all of the
filename arguments provided on the command line.

#### t.addAssert(name, length, fn)

This is used for creating assertion methods on the `Test` class.

It's a little bit advanced, but it's also super handy sometimes.  All
of the assert methods below are created using this function, and it
can be used to create application-specific assertions in your tests.

The name is the method name that will be created.  The length is the
number of arguments the assertion operates on.  (The `message` and
`extra` arguments will alwasy be appended to the end.)

For example, you could have a file at `test/setup.js` that does the
following:

```javascript
var tap = require('tap')

// convenience
if (module === require.main) {
  tap.pass('ok')
  return
}

// Add an assertion that a string is in Title Case
// It takes one argument (the string to be tested)
tap.Test.prototype.addAssert('titleCase', 1, function (str, message, extra) {
  message = message || 'should be in Title Case'
  // the string in Title Case
  // A fancier implementation would avoid capitalizing little words
  // to get `Silence of the Lambs` instead of `Silence Of The Lambs`
  // But whatever, it's just an example.
  var tc = str.toLowerCase().replace(/\b./, function (match) {
    return match.toUpperCase()
  })

  // should always return another assert call, or
  // this.pass(message) or this.fail(message, extra)
  return this.equal(str, tc, message, extra)
})
```

Then in your individual tests, you'd do this:

```javascript
require('./setup.js') // adds the assert
var tap = require('tap')
tap.titleCase('This Passes')
tap.titleCase('however, tHis tOTaLLy faILS')
```

#### t.endAll()

Call the `end()` method on all child tests, and then on this one.

#### t.current()

Return the currently active test.

### Asserts

The `Test` object has a collection of assertion methods, many of which
are given several synonyms for compatibility with other test runners
and the vagaries of human expectations and spelling.  When a synonym
is multi-word in `camelCase` the corresponding lower case and
`snake_case` versions are also created as synonyms.

All assertion methods take optional `message` and `extra` arguments as
the last two params.  The `message` is the name of the test.  The
`extra` argument can contain any arbitrary data about the test, but
the following fields are "special".

* `todo` Set to boolean `true` or a String to mark this as pending
* `skip` Set to boolean `true` or a String to mark this as skipped
* `at` Generated by the framework.  The location where the assertion
  was called.  Do not set.
* `stack` Generated by the framework.  The stack trace to the point
  where the assertion was called.

#### ok(obj, message, extra)

Verifies that the object is truthy.

Synonyms: `t.true`, `t.assert`

#### notOk(obj, message, extra)

Verifies that the object is not truthy.

Synonyms: `t.false`, `t.assertNot`

#### error(obj, message, extra)

If the object is an error, then the assertion fails.

Note: if an error is encountered unexpectedly, it's often better to
simply throw it.  The Test object will handle this as a failure.

Synonyms: `t.ifErr`, `t.ifError`

#### throws(fn, [expectedError], message, extra)

Expect the function to throw an error.  If an expected error is
provided, then also verify that the thrown error matches the expected
error.

Synonyms: `t.throw`

#### doesNotThrow(fn, message, extra)

Verify that the provided function does not throw.

Note: if an error is encountered unexpectedly, it's often better to
simply throw it.  The Test object will handle this as a failure.

Synonyms: `t.notThrow`

#### equal(found, wanted, message, extra)

Verify that the object found is exactly the same (that is, `===`) to
the object that is wanted.

Synonyms: `t.equals`, `t.isEqual`, `t.is`, `t.strictEqual`,
`t.strictEquals`, `t.strictIs`, `t.isStrict`, `t.isStrictly`

#### notEqual(found, notWanted, message, extra)

Inverse of `equal()`.

Verify that the object found is not exactly the same (that is, `!==`) as
the object that is wanted.

Synonyms: `t.inequal`, `t.notEqual`, `t.notEquals`,
`t.notStrictEqual`, `t.notStrictEquals`, `t.isNotEqual`, `t.isNot`,
`t.doesNotEqual`, `t.isInequal`

#### same(found, wanted, message, extra)

Verify that the found object is deeply equivalent to the wanted
object.  Use non-strict equality for scalars (ie, `==`).

Synonyms: `t.equivalent`, `t.looseEqual`, `t.looseEquals`,
`t.deepEqual`, `t.deepEquals`, `t.isLoose`, `t.looseIs`

#### notSame(found, notWanted, message, extra)

Inverse of `same()`.

Verify that the found object is not deeply equivalent to the
unwanted object.  Uses non-strict inequality (ie, `!=`) for scalars.

Synonyms: `t.inequivalent`, `t.looseInequal`, `t.notDeep`,
`t.deepInequal`, `t.notLoose`, `t.looseNot`

#### strictSame(found, wanted, message, extra)

Strict version of `same()`.

Verify that the found object is deeply equivalent to the wanted
object.  Use strict equality for scalars (ie, `===`).

Synonyms: `t.strictEquivalent`, `t.strictDeepEqual`, `t.sameStrict`,
`t.deepIs`, `t.isDeeply`, `t.isDeep`, `t.strictDeepEquals`

#### strictNotSame(found, notWanted, message, extra)

Inverse of `strictSame()`.

Verify that the found object is not deeply equivalent to the unwanted
object.  Use strict equality for scalars (ie, `===`).

Synonyms: `t.strictInequivalent`, `t.strictDeepInequal`,
`t.notSameStrict`, `t.deepNot`, `t.notDeeply`, `t.strictDeepInequals`,
`t.notStrictSame`

#### match(found, pattern, message, extra)

Verify that the found object matches the pattern provided.

If pattern is a regular expression, and found is a string, then verify
that the string matches the pattern.

If pattern is an object, the verify that the found object contains all
the keys in the pattern, and that all of the found object's values
match the corresponding fields in the pattern.

This is useful when you want to verify that an object has a certain
set of required fields, but additional fields are ok.


Synonyms: `t.has`, `t.hasFields`, `t.matches`, `t.similar`, `t.like`,
`t.isLike`, `t.includes`, `t.include`

#### notMatch(found, pattern, message, extra)

Interse of `match()`

Verify that the found object does not match the pattern provided.

Synonyms: `t.dissimilar`, `t.unsimilar`, `t.notSimilar`, `t.unlike`,
`t.isUnlike`, `t.notLike`, `t.isNotLike`, `t.doesNotHave`,
`t.isNotSimilar`, `t.isDissimilar`

#### type(object, type, message, extra)

Verify that the object is of the type provided.

Type can be a string that matches the `typeof` value of the object, or
the string name of any constructor in the object's prototype chain, or
a constructor function in the object's prototype chain.

For example, all the following will pass:

```javascript
t.type(new Date(), 'object')
t.type(new Date(), 'Date')
t.type(new Date(), Date)
```

Synonyms: `t.isa`, `t.isA`
