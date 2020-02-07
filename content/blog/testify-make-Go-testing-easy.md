---
title: "Testify: make Go testing easy 1/3"
date: 2020-02-03T22:02:55+01:00
image: "img/test-reset.jpg"
draft: false
---

While [Go](https://golang.org) comes with [go
test](https://golang.org/cmd/go/#hdr-Test_packages) and a
[testing](https://pkg.go.dev/testing?tab=doc) package by default, the
experience can be better with [testify](https://github.com/stretchr/testify)
package. Git hub page introduces it as Go code (golang) set of packages that
provide many tools for testifying that your code will behave as you intend.

By the end I found the resulting article as too long (~25 minutes) to read, so
I split it into

1. [Part1](/blog/testify-make-go-testing-easy/) introduces `assert`, talks briefly abot Python and C and shows basics of `testify`
2. [Part2](/blog/testify-make-go-testing-easy-2/) introduces table driven testing, more helpers like `ElementsMatch` or `JSONeq`
2. [Part3](/blog/testify-make-go-testing-easy-3/) gets more advanced with test suited and mocks

## Standard approach

Example as it comes with a [testing](https://pkg.go.dev/testing?tab=doc) package from the standard library.

```go
func TestAbs(t *testing.T) {
    got := Abs(-1)
    if got != 1 {
        t.Errorf("Abs(-1) = %d; want 1", got)
    }
}
```

And reader will notice two things.

1. Unlike `assert` based tests, you test for value you do not want to get. Which becomes confusing for readers and writers.
2. There is no support for generating expected output, so you have to manually develop and mix the test itself with error output.

Following function `Abs` is buggy.

```go
// main.go
func Abs(i int) int {
    if i == -1 {
        return 1
    }
    return i
}
// main_test.go
func TestAbs(t *testing.T) {
	got := Abs(-1)
	if got != 1 {
		t.Errorf("Abs(-1) = %d; want 1", got)
	}
	got = Abs(-42)
	if got != 42 {
		t.Errorf("Abs(-42) = %d; want 42", got)
	}
}
```

```bash
go test -v
=== RUN   TestAbs
--- FAIL: TestAbs (0.00s)
    main_test.go:12: Abs(-42) = -42; want 42
FAIL
exit status 1
FAIL	_/home/mvyskocil/projects/vyskocilm/miblog-src/miblog/X	0.002s
```

## Unit testing in Python

Interesting approach implemented by [py.test](https://pytest.org/en/latest/)
tool for Python language. Despite the fact Python has a builtin
[unittest](https://docs.python.org/3/library/unittest.html) module, `py.test`
is as popular tool. It does the work by overloading standard
[assert](https://docs.python.org/3/reference/lexical_analysis.html#keywords)
keyword.

```python
def Abs(i: int) -> int:
    if i == -1:
        return 1
    return i

def test_Abs():
    assert Abs(-1) == 1
    assert Abs(-42) == 42
```

As you can see, where even trivial test is on 4 lines of go, usage of `assert`
and `py.test` command makes tests crispy and clean. It can be hardly done in
less lines.

```bash
============================= test session starts ==============================
platform linux -- Python 3.7.3, pytest-4.6.9, py-1.8.1, pluggy-0.13.1
rootdir: /home/mvyskocil/X
collected 1 item

main.py F                                                                [100%]

=================================== FAILURES ===================================
___________________________________ test_Abs ___________________________________

    def test_Abs():
        assert Abs(-1) == 1
>       assert Abs(-42) == 42
E       assert -42 == 42
E        +  where -42 = Abs(-42)

main.py:8: AssertionError
=========================== 1 failed in 0.03 seconds ===========================
```

While the output can be seen as unecessary verbose, it is far from truth. It
gives as much detailed information as possible in a case of failure. Which
includes expression which failed, expected and got values and `py.test` can
automatically compare nested structures and print diff as well.

## A bit of C

C is not the language seen as role model for others. It did not evolved a lot
since 70s. However there is
[zproject](https://github.com/zeromq/zproject/blob/master/zproject_class.gsl#L238)
of zeromq project. It is powerfull generator of C based projects and amongs all
others things, it provides [unit testing
support](https://github.com/zeromq/zproject/blob/master/zproject_class.gsl#L238)
as a part of the project. Tests are mandatory for ZeroMQ projects, so once cant
find a code like
[czmq/src/zactor.c](https://github.com/zeromq/czmq/blob/master/src/zactor.c#L336)

```c
void
zactor_test (bool verbose)
{
    printf (" * zactor: ");

    //  @selftest
    zactor_t *actor = zactor_new (echo_actor, "Hello, World");
    assert (actor);
    zstr_sendx (actor, "ECHO", "This is a string", NULL);
    char *string = zstr_recv (actor);
    assert (streq (string, "This is a string"));
    freen (string);
    zactor_destroy (&actor);
```

That means if you can have nice and crispy clean unit tests in C. There is
hardly a reason for not having it in Go.

## Testify

Sadly as Go does not have a concept of
[assert](<https://en.wikipedia.org/wiki/Assertion_(software_development>) even
simple tests are hard to read. And what is worse, there is nothing
supporting the diff to exactly now what was wrong in a case of failure.

And here comes [stretchr/testify](https://github.com/stretchr/testify). It is
popular library bringing `py.test` like capabilities to Go.

<p>
<a href="https://github.com/stretchr/testify/watchers/"><img src="https://img.shields.io/github/watchers/stretchr/testify.svg?style=social&amp;label=Watcher&amp;maxAge=2592000" alt="GitHub watchers" style="display:inline; float:inline-start;" />
</a>
<a href="https://github.com/stretchr/testify/stargazers/"><img src="https://img.shields.io/github/stars/stretchr/testify.svg?style=social&amp;label=Star&amp;maxAge=2592000" alt="GitHub stars" style="display:inline;" /></a>
<a href="https://github.com/stretchr/testify/network/"><img src="https://img.shields.io/github/forks/stretchr/testify.svg?style=social&amp;label=Fork&amp;maxAge=2592000" alt="GitHub forks" style="display; inline"/></a>
</p>

So this is how test looks like.

```go
import "github.com/stretchr/testify/assert"
func TestAbs(t *testing.T) {
    assert.Equal(t, 1, Abs(-1))
    assert.Equal(t, 42, Abs(-42))
}
```

Or we can create `assert.Assertion` structure, which is typically named
`assert` to avoid putting `t` pointer everywhere. In most of cases hiding name
for the module level import is considered as a bad practice. For `assert` and
`require` is is fine and intended usage.

```go
import "github.com/stretchr/testify/assert"
func TestAbs(t *testing.T) {
    assert := assert.New(t)
    assert.Equal(1, Abs(-1))
    assert.Equal(42, Abs(-42))
}
```

And this is the run

```bash
go test -v
=== RUN   TestAbs
--- FAIL: TestAbs (0.00s)
    main_test.go:11: 
        	Error Trace:	main_test.go:11
        	Error:      	Not equal: 
        	            	expected: 4
        	            	actual  : -42
        	Test:       	TestAbs
FAIL
exit status 1
FAIL	X	0.003s
```

## Error messages

Many methods ends with `f` so they will print extra error messages in a case of failure.

```go
	assert.Equalf(4, Abs(-42), "fail in test %s", t.Name())
```

This becomes handy with a next topic.

```shell
        	            	expected: 4
        	            	actual  : -42
        	Test:       	TestAbs
        	Messages:   	fail in test TestAbs
```

## Subtests

Go `testing` package allows you to run subtests inside your testing function.
This has similar properties to test cases, where your tests can share common
objects created in the begginning of each test.

```go
import "github.com/stretchr/testify/require"

func TestAbs(t *testing.T) {
	require := require.New(t)
	t.Run("abs-42", func(t *testing.T){require.Equalf(4, Abs(-42), "fail in test %s", t.Name())})
    t.Run("abs-1", func(t *testing.T){require.Equal(1, Abs(-1))})
}
```

```bash
go test -v
=== RUN   TestAbs
=== RUN   TestAbs/abs-42
--- FAIL: TestAbs (0.00s)
    main_test.go:10: 
        	Error Trace:	main_test.go:10
        	Error:      	Not equal: 
        	            	expected: 4
        	            	actual  : -42
        	Test:       	TestAbs
        	Messages:   	fail in test TestAbs/abs-42
    --- FAIL: TestAbs/abs-42 (0.00s)
        testing.go:864: test executed panic(nil) or runtime.Goexit: subtest may have called FailNow on a parent test
FAIL
exit status 1
FAIL	X	0.002s
```

There are two sub tests inside `TestAbs`

* `TestAbs/abs-42`
* `TestAbs/abs-1`

The advantage is that we can use `-run` argument of `go test`. Which means that
run and explore specifically broken test and don't be distracted by the
succesfull runs.

Then there is another change in code.

```diff
- import "github.com/stretchr/testify/assert"
+ import "github.com/stretchr/testify/require"
```

All the methods of `assert` package mark a test failure and let test to continue.
On the other hand `require` will immediatelly stop the execution. It is the
same behavioral difference between
[Fail](https://pkg.go.dev/testing?tab=doc#T.Fail) and
[FailNow](https://pkg.go.dev/testing?tab=doc#T.FailNow) methods of a `t`
object.

It illustrates the fact that shared resources can affects other tests.

## End

This part introduced

* the concept of `assert` and the usage in languages others than Go
* basic concepts of `testify` package
* how it seamlesly integrates with Go own `testing` package

## All parts

1. [Part1](/blog/testify-make-go-testing-easy/) introduces `assert`, talks briefly abot Python and C and shows basics of `testify`
2. [Part2](/blog/testify-make-go-testing-easy-2/) introduces table driven testing, more helpers like `ElementsMatch` or `JSONeq`
3. [Part3](/blog/testify-make-go-testing-easy-3/) gets more advanced with test suited and mocks

Logo by [CarbonArc@Flickr](https://www.flickr.com/photos/41002268@N03/): https://www.flickr.com/photos/41002268@N03/23958148837
