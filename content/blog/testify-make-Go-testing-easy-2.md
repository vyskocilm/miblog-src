---
title: "Testify: make Go testing easy 2/3"
date: 2020-02-03T22:02:59+01:00
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


## Table driven testing

Table driven testing is common and usefull go idiom. And it is good idea to use
with sub tests as well.

{{< highlight go >}}
type testCase struct {
    name string
    input int
    expected int
}

func TestAbs(t *testing.T) {
	require := require.New(t)

    cases := []testCase{
        {
            name: "abs-42",
            input: -42,
            expected: 42,
        },
        {
            name: "abs-1",
            input: -1,
            expected: 1,
        },
    }

    for _, tt := range cases {
        t.Run(tt.name, func(t *testing.T){
	        require.Equalf(tt.expected, Abs(tt.input), "fail in test %s", t.Name())
        })
    }
}
{{< / highlight >}}

To be honest this is an overkill for a simple function like `Abs`. It becomes
handy when dealing with a function with complex input/output data structures
and complex internal states. It is possible to run individual subtests, the
`testing` package deals with all of this well.

One problem remains. Code uses shared instance of `require`. Subtest failure
stops all other tests from run. This is unecessary and can be easilly fixed by
creating new `require` object each time. On the other hand as tests are
supposed to succeed all the time, early failure in a subtest might not be a
problem in all cases.

{{< highlight diff >}}
@@ -29,6 +28,7 @@
 
     for _, tt := range cases {
         t.Run(tt.name, func(t *testing.T){
+               require := require.New(t)
                require.Equalf(tt.expected, Abs(tt.input), "fail in test %s", t.Name())
         })
     }
{{< / highlight >}}

## Interesting testify helpers

While `Equal` is probably the most used function, there are 56 functions
available for `assert` and `require` packages. At least for 1.4.0 version.

{{< highlight sh >}}
go doc github.com/stretchr/testify/assert | grep '^func ' | grep -v '.*f(' | wc -l
56
{{< / highlight >}}

And frankly I doubt there are a lof of developers, which have used them all.
Neither myself. So here is a short list of features I found totally handy.

### Equal methods

Method `Equal` is probably the most commonly used one. It is defined with
`interface{}` type, which is a way how to tell Go compiler that developer is
going to check types during runtime. However `Equal` method checks the type
equality during test run, so it is not possible to pass two same numbers with a
distinct types.


{{< highlight go >}}
func TestEqual(t *testing.T) {
	assert := assert.New(t)
    assert.Equal(42, 42)
    // line below will FAIL
    assert.Equal(int(42), uint(42))
}
{{< / highlight >}}

There is helper `EqualValues` ignoring types

{{< highlight go >}}
func TestEqual(t *testing.T) {
	assert := assert.New(t)
    // and this is OK
    assert.EqualValues(int(42), uint(42))
}
{{< / highlight >}}

Or if it happens there are two variables of different types, however with
identical structure `EqualValues` does the work as well.

{{< highlight go >}}
func TestStructs(t *testing.T) {
	assert := assert.New(t)
    type foo struct {Foo string}
    type bar struct {Foo string}
    aFoo := foo{Foo: "spam"}
    aBar := bar{Foo: "spam"}
    // line below will FAIL
    assert.Equal(aFoo, aBar)
    assert.EqualValues(aFoo, aBar)
}
{{< / highlight >}}

Let's not forget that this is still Go. While something similar can be
done by Ruby or maybe Python libraries, in Go developer must be explicit

{{< highlight go >}}
func TestMagic(t *testing.T) {
	assert := assert.New(t)
    jsn := `{"foo": "spam"}`
    yml := `foo: spam`
    // this is string comparsion and obviously fail
    assert.EqualValues(jsn, yml)
}
{{< / highlight >}}

### JSON/YAML helpers

However! Noth json and yaml are common formats on Internet. Go code usually
receive and send JSONS over HTTP and reads and write YAML files in order to
build or run container, or to build Helm charts to deploy things to Kubernetes.
It is not surprising to have related helpers in `testify`.

{{< highlight go >}}
func TestJSON(t *testing.T) {
	assert := assert.New(t)
    jsn := `{"foo": "spam"}`
    jsn2 := `{
        "foo":
        "spam"
    }`
    assert.JSONEq(jsn, jsn2)
}
{{< / highlight >}}

Making an example for `YAMLeq` would be trivial, so lets reimplement the magic test with a real check

{{< highlight go >}}
import "github.com/ghodss/yaml"

func y2j(yml string) (string, error){
	ret, err := yaml.YAMLToJSON([]byte(yml))
    if err != nil {
        return "", err
    }
    return string(ret), nil
}

func TestNoMagic(t *testing.T) {
	assert := assert.New(t)
	require := require.New(t)
	jsn := `{"foo": "spam"}`
	yml, err := y2j(`foo: spam`)
    require.NoError(err)
	assert.JSONEq(jsn, yml)
}
{{< / highlight >}}

Ignoring the tiny helper converting the yaml to json format, the test is longer
only beause of explicit error check. And error checks are usually done by
`require` as there is no reason to continue.

### Error/NoError

The most common usage of `Error` method is with a `require` package. If there is
a fail in operation supposed to not fail, then there is usually no reason to
continue with a test. And vice versa. It is handy to halt the test in a case of
unexpected failure during execution.

{{< highlight go >}}
    resp, err := http.Get(url)
    require.NoError(err)
{{< / highlight >}}


{{< highlight go >}}
// main.go
func Err() error {
	return fmt.Errorf("bind: already in use")
}
// main_test.go
func TestErr(t *testing.T) {
	assert := assert.New(t)
	require := require.New(t)
	err := Err()
	require.Error(err)
	assert.Equal(
		fmt.Errorf("bind: already in use"),
		err)
}
{{< / highlight >}}

The check `NoError` is simply the same, only checks the case `err == nil`.


{{< highlight go >}}
// main.go
func NoErr() error {
	return nil
}
// main_test.go
func TestNoErr(t *testing.T) {
	require := require.New(t)
	err := NoErr()
	require.NoError(err)
}
{{< / highlight >}}

## ElementsMatch

Sometimes developer do not care about an order of items in JSON array. Most
likelly it depends on some B-tree magic of used SQL engine table index. All
developer care about is that the right values are in place. How many times
there is an unecessary sort in place just to make unit test pass? Not
anymore with a `testify` and `ElementsMatch` helper.

{{< highlight go >}}
func TestNoOrder(t *testing.T) {
	assert := assert.New(t)
	expected := `{"data": [1, 2, 3]}`
	actual := `{"data": [2, 1, 3]}`
	assert.JSONEq(expected, actual)
}
{{< / highlight >}}


Of course `JSONEq` can't help. It works for native data only.

{{< highlight go >}}
func TestNoOrder2(t *testing.T) {
	assert := assert.New(t)
    expected := []int{1, 2, 3}
    actual := []int{2, 1, 3}
	assert.ElementsMatch(expected, actual)
}
{{< / highlight >}}

## Panics

In go panic is something one should use carefully. At the same time it can be
the only one way to return an error from an interface, which does not make that
possible. Consider following totally fabricated example. There is standard
interface for math functions as they typically do not return an error. And
consider there is a special ultra fast implementation. Which unfortunatelly
does work for a subset of inputs. The only way to deal with it and still
implement the interface is to panic for invalid inputs.

And `NotPanics` and `Panics` methods will make sure we can test such functions
well.

{{< highlight go >}}
type AbsComputer interface {
    Abs(i int) int
}

func UberFastAbs(i int) int {
    if i == -1 {
        return 1
    }
    panic(i)
}

// main_test.go
func TestUberFastAbs(t *testing.T) {
	assert := assert.New(t)
    assert.NotPanics(func (){UberFastAbs(-1)})
    assert.Panics(func() {UberFastAbs(42)})
}
{{< / highlight >}}

## All parts

1. [Part1](/blog/testify-make-go-testing-easy/) introduces `assert`, talks briefly abot Python and C and shows basics of `testify`
2. [Part2](/blog/testify-make-go-testing-easy-2/) introduces table driven testing, more helpers like `ElementsMatch` or `JSONeq`
3. [Part3](/blog/testify-make-go-testing-easy-3/) gets more advanced with test suited and mocks

Logo by [CarbonArc@Flickr](https://www.flickr.com/photos/41002268@N03/): https://www.flickr.com/photos/41002268@N03/23958148837
