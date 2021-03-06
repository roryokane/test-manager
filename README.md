Name
====

test-manager/ - A unit test framework for MIT Scheme in the jUnit style.

Synopsys
========

```scheme
(load "test-manager/load.scm")

; This is a test group named simple-stuff
(in-test-group
 simple-stuff

 ; This is one test named arithmetic
 (define-test (arithmetic)
   "Checking that set! and arithmetic work"
   (define foo 5)
   (check (= 5 foo) "Foo should start as five.")
   (set! foo 6)
   (check (= 36 (* foo foo))))

 ; Each of these will become a separate anonymous one-form test
 (define-each-test
   (check (= 4 (+ 2 2)) "Two and two should make four.")
   (check (= 2147483648 (+ 2147483647 1)) "Addition shouldn't overflow."))

 ; Each of these will become a separate anonymous one-form test using check
 (define-each-check
   (= 6 (+ 2 2 2))
   (equal? '(1 2 3) (cons 1 '(2 3))))

 ; This is a test that looks like a REPL interaction
 (define-test (interactive)
   (interaction
    (define foo 5)
    foo
    (produces 5)  ; This compares against the value of the last form
    (set! foo 6)
    (* foo foo)
    (produces 36))))

(run-registered-tests)

; Can run individual groups or tests with
(run-test '(simple-stuff))
(run-test '(simple-stuff arithmetic))
```

Installation
============

Easy as can be: grab the source and put it anywhere you want.  To use,
`load` the toplevel `load.scm` file.

Description
===========

This test framework defines a language for specifying test suites and
a set of commands for running them.  A test suite is a
collection of individual tests grouped into a hierarchy of test
groups.  The test group hierarchy serves to semantically aggregate the
tests, allowing the definition of shared code for set up, tear down,
and surround, and also partitioning the test namespace to avoid
collisions.

The individual tests are ordinary procedures, with some associated
bookkeeping.  A test is considered to pass if it returns normally,
and to fail if it raises some condition that it does not handle
(tests escaping into continuations leads to unspecified behavior).

The framework provides a `check` macro and a library of assertion
procedures that can be invoked in tests and have the desired behavior
of raising an appropriate condition if they fail.  The framework also
provides an `interaction` macro, together with a `produces`
procedure, for simulating read-eval-print interactions, and an
extensible pattern-matching facility for easier testing of the
relevant aspects of a result while ignoring the irrelevant ones.

Defining Test Suites
--------------------

All tests are grouped into a hierarchy of test groups.
At any point in the definition of a test suite, there is an implicit
"current test group", into which tests and subgroups can be added.  By
default, the current test group is the top-level test group, which is
the root of the test group hierarchy.

`(define-test (name) expression ... )`

Define a test named `name` that consists of the given expressions,
and add it to the current test group.  When the test is run, the
expressions will be executed in order, just like the body of any
procedure.  If the test raises any condition that it does not handle,
it is considered to have failed.  If it returns normally, it is
considered to have passed.  Usually, tests will contain uses of the
`check` macro or of assertions from the list below, which raise
appropriate conditions when they fail.  In the spirit of Lisp
docstrings, if the first expression in the test body is a literal
string, that string will be included in the failure report if the test
should fail.

This is the most verbose and most expressive test definition syntax.
The four test definition syntaxes provided below are increasingly
terse syntactic sugar for common usage patterns of this syntax.

`(define-test () expression ... )`

Define an explicitly anonymous test.  I can't see why you would want
to do this, but it is provided for completeness.

`(define-test expression)`

Define a one-expression anonymous test.  The single expression will be
printed in the failure report if the test fails.  This is a special
case of `define-each-test`, below.

`(define-each-test expression ... )`

Define a one-expression anonymous test for each of the given
expressions.  If any of the tests fail, the corresponding expression
will be printed in that test's failure report.

`(define-each-check expression ...)`

Define a one-expression anonymous test for each of the given
expressions by wrapping it in a use of the `check` macro, below.

If you have many simple independent checks you need to make and
you don't want to invent names for each individual one, this is the
test definition syntax for you.

`(in-test-group name expression ... )`

Locate (or create) a test subgroup called `name` in the current test
group.  Then temporarily make this subgroup the current test group,
and execute the expressions in the body of `in-test-group`.  This
groups any tests and further subgroups defined by those expressions
into this test group.  Test groups can nest arbitrarily deep.  Test
groups serve to disambiguate the names of tests, and to group them
semantically.  In particular, should a test fail, the names of the
stack of groups it's in will be displayed along with the test name
itself.

`(define-set-up expression ...)`

Defines a sequence of expressions to be run before every test in
the current test group.  Clobbers any previously defined set up
for this group.

`(define-tear-down expression ...)`

Defines a sequence of expressions to be run after every test in
the current test group.  Clobbers any previously defined tear down
for this group.

`(define-surround expression ...)`

Defines a sequence of expressions to be run surrounding every test in
the current test group.  Inside the `define-surround`, the identifier
`run-test` is bound to a nullary procedure that actually runs the
test.  The test will get run as many times as you call `run-test`, so
you can run each test under several conditions (or accidentally not
run it at all if you forget to call `run-test`).  Clobbers any
previously defined surround for this group.

`(define-group-set-up expression ...)`

Defines a sequence of expressions to be run once before running any
test in the current test group.  Clobbers any previously defined group
set up for this group.

`(define-group-tear-down expression ...)`

Defines a sequence of expressions to be run once after running all
tests in the current test group.  Clobbers any previously defined
group tear down for this group.

`(define-group-surround expression ...)`

Defines a sequence of expressions to be run once surrounding running
the tests in the current test group.  Inside the
`define-group-surround`, the identifier `run-test` is bound to a
nullary procedure that actually runs the tests in this group.
Clobbers any previously defined group surround for this group.

Running Test Suites
-------------------

The following procedures are provided for running test suites:

`(run-test name-stack)`

Looks up the test indicated by name-stack in the current test group,
runs it, and prints a report of the results.  Returns the number of
tests that did not pass.  An empty list for a name stack indicates the
whole group, a singleton list indicates that immediate descendant, a
two-element list indicates a descendant of a descendant, etc.  For
example, `(run-test '(simple-stuff arithmetic))` would run the first
test defined in the example at the top of this page.

`(run-registered-tests)`

Runs all tests registered so far, and prints a report of the results.
Returns the number of tests that did not pass.  This could have been
defined as `(run-test '())`.

`(run-tests-and-exit)`

Runs tests as with `run-registered-tests` and *exits Scheme*.  The
exit status is the number of tests that did not pass.  This is
intended as an entry point for commandline test suite execution, as
from a shell script or a Makefile.

`(clear-registered-tests!)`

Unregister all tests.  Useful when loading and reloading test suites
interactively.  For more elaborate test structure manipulation
facilities, see also `test-group.scm`.

Checks
------

The `check` macro is the main mechanism for asking tests to actually
test something:

`(check expression [message])`

Executes the expression, and passes iff that expression returns a true
value (to wit, not `#f`).  If the expression returns `#f`, constructs a
failure report from the expression, the message if any, and the values
of the immediate subexpressions of the expression.

`check` is a macro so that it can examine the expression provided and
construct a useful failure report if the expression does not return a
true value.  Specifically, the failure report includes the expression
itself, as well as the values that all subexpressions (except the
first) of that expression evaluated to.  For example,

``` scheme
 (check (< (+ 2 5) (* 3 2)))
```

fails and reports

```
 Form      : (< (+ 2 5) (* 3 2))
 Arg values: (7 6)
```

so you can see right away both what failed, and, to some degree, what
the problem was.

In the event that the failure report generated by `check` itself is
inadequate, `check` also accepts an optional second argument that is
interpreted as a user-supplied message to be added to the failure
report.  The message can be either a string, or an arbitrary object
that will be coerced to a string by `display`, or a promise (as
created by `delay`), which will be forced and the result coerced to a
string.  The latter is useful for checks with dynamically computed
messages, because that computation will then only be performed if the
test actually fails, and in general for doing some computation at
check failure time.

Interactions
------------

The style of interactively fooling with a piece of code at the
read-eval-print loop differs from the style of writing unit tests for
a piece of code and running them.  One notable difference is that at
the REPL you write some expression and examine its return value to see
whether it was what you expected, whereas when writing a unit test you
write a check form that contains both the expression under test and
the criterion you expect it to satisfy.  In order to decrease the
impedance mismatch between these two ways of verifying what a program
does, `test-manager` provides the procedure `produces`, which
retroactively checks the last return value, and the macro
`interaction`, which enables `produces` to work inside a unit test.

`(produces pattern)`

Checks that the return value of the previous evaluated expression
matches (via `generic-match`, below) the provided pattern.  This
works at the REPL via the REPL history, and also works inside a use of
the `interaction` macro.

`(interation form ...)`

Tracks the return values of each `form` and makes them available for
use with `produces`.  For an example, see the last test in the
synopsis.

Pattern Matching
----------------

The user-extensible pattern matching facility is the generic procedure
`generic-match`.  This procedure is generic in the sense of the
Scheme Object System provided with MIT Scheme.  It can be used in
tests directly, and is automatically invoked by `produces` above, and
`assert-match` and `assert-no-match` below.

`(generic-match pattern object)`

Returns `#t` iff the given object matches the given pattern.  The
meaning of "matches" is user-extensible by adding methods to this
generic procedure.  By default compares whether the pattern is
`equal?` to the object, but also see provided methods below.

`(generic-match pattern-string string)`

If the pattern and the object are strings, interprets the pattern
as a regular expression and matches it against the object.

`(generic-match pattern-pair pair)`

If the pattern and the object are pairs, recursively matches their
`car`s and `cdr`s against each other.

`(generic-match pattern-vector vector)`

If the pattern and the object are vectors, recursively matches their
components against each other elementwise.

`(generic-match x y)`

If the pattern and the object are inexact numbers, checks them for
equality, and then then checks whether the object rounded to five
or twelve significant digits equals the pattern.  For example, `(generic-match
1.4142 (sqrt 2))` returns `#t`, as does
`(generic-match 1.4142135623730951 (sqrt 2))`.

Assertions
----------

The following assertion procedures are provided for situations where
`check` being a macro makes it unwieldy.  The `message` arguments to
the assertions are user-specified messages to print to the output if
the given assertion fails.  The `assert-proc` assertion requires a
message argument because it cannot construct a useful output without
one, and because it is not really meant for extensive direct use.  The
message is optional for the other assertions because they can say
something at least mildly informative even without a user-supplied
message.  In any case, the message arguments are treated the same way
as by `check`: they can be strings, objects to be `display`ed, or
promises to be `force`d.

`(assert-proc message proc)`

Passes iff the given procedure, invoked with no arguments, returns a
true value.  On failure, arranges for the given `message` to appear in
the failure report.  This is a primitive assertion in whose terms
other assertions are defined.

`(assert-true thing [message])`

Passes iff the given value is a true value (to wit, not `#f`).

`(assert-false thing [message])`

Passes iff the given value is a false value (to wit, `#f`).

`(assert-equal expected actual [message])`
 Likewise `assert-eqv`, `assert-eq`, and `assert-=`

Passes iff the given `actual` value is equivalent, according to the
corresponding predicate, to the `expected` value.  Produces a
reasonably helpful message on failure, and includes the optional
`message` argument in it if present.  When in doubt, use
`assert-equal` to compare most things; use `assert-=` to compare
exact numbers like integers; and use `assert-in-delta`, below, for
inexact numbers like floating points.

`assert-equals, assert=`

Are aliases for `assert-equal` and `assert-=`, respectively.

`(assert-equivalent predicate [pred-name])`

This is intended as a tool for building custom assertions.  Returns an
assertion procedure that compares an expected and an actual value with
the given predicate and produces a reasonable failure message.
`assert-equal` and company could have been defined in terms of
`assert-equivalent` as, for example, `(define assert-equal
(assert-equivalent equal? "equal?"))`.

`assert-<`, `assert->`, `assert-<=`, and `assert->=`

Like `assert-=`, but with a different comparator.  In particular, these
aren't equivalence relations, so the order of arguments matters.

`(assert-matches pattern object [message])`

Passes iff the given object matches the given pattern, per
`generic-match`.

`(assert-no-match pattern object [message])`

Passes iff the given object does not match the given pattern, likewise
per `generic-match`.

`(assert-in-delta expected actual delta [message])`

Passes iff the given `actual` value differs, in absolute value, from
the given `expected` value by no more than `delta`.  Use this in
preference to `assert-=` to check sameness of inexact numerical
values.

Portability
===========

I originally started this project with MIT Scheme and Guile in mind as
target Scheme implementations.  That aim met with success through
version 1.1, but as of version 1.2 I dropped explicit support for the
Guile port.  I have left all the portability code intact; the vast
majority of the documented features should work in Guile.  Also, since
this software has been two-Scheme for much of its life, I expect it
should not be hard to port to other Schemes.

The specific things that I know do not work in Guile are: `produces`
does not work in the Guile REPL (though it does still work inside
`interaction`) which rather defeats its purpose; `generic-match` is
not actually a generic procedure in Guile (though that could
presumably be fixed by one who knew Guile's generic procedure
facilities); and `check` does not accept a message argument in Guile.

Bugs
====

The language for explicit test group handling is ill-specified and
undocumented (peruse `test-group.scm` if interested).

The framework is inclined to catch all conditions raised by any test,
so this tends to break code that depends on the execution of condition
handlers that are set up outside the framework (for example, one way
to make vectors of procedures be applicable objects (returning the
vector of results of applying all the procedures) would be to install
a global handler for the "inapplicable object" condition that tries
the above tweak and restarts with that answer if it works).
`test-manager` contains a hack to work around this problem for this
case, but this ugliness may lead to other bugs that I don't know
about.

Author
======

Alexey Radul, axch@mit.edu

License
=======

Copyright 2007-2008 Alexey Radul.

This file is part of Test Manager.

Test Manager is free software; you can redistribute it and/or modify
it under the terms of the GNU General Public License as published by
the Free Software Foundation, either version 3 of the License, or
(at your option) any later version.

Test Manager is distributed in the hope that it will be useful,
but WITHOUT ANY WARRANTY; without even the implied warranty of
MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the
GNU General Public License for more details.

You should have received a copy of the GNU General Public License
along with Test Manager.  If not, see <http://www.gnu.org/licenses/>.
