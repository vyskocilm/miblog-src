---
title: "Testify: make Go testing easy 3/3"
date: 2020-02-03T22:24:45+01:00
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

## Test suites

While `testing` documentation encourages table driven tests and subtests, there
are many developers outside, who are used to write [Smalltalk like unit
tests](https://en.wikipedia.org/wiki/SUnit) and this style encourages writing
[test suites](https://en.wikipedia.org/wiki/XUnit#Test_suites).

Package `testify` has a solution in form of `testify/suite`. It comes with
setup/teardown methods for the individual test cases or whole site.
Additionally there are functions to be run before after the test as well.

Below is an example of a suite with a function running before each test case.

{{< highlight go >}}
import "github.com/stretchr/testify/suite"

type absSuite struct {
	suite.Suite
	cases []testCase
}

func (s *absSuite) SetupTest() {
    s.cases = []testCase{
        {name: "abs-42", expected: 42, input: -42},
        {name: "abs-1", expected: 1, input: -1},
    }
}
{{< / highlight >}}

Here we run through initalized test cases. And the test messes up with a shared
resource.

{{< highlight go >}}
func (s *absSuite) TestAbs() {
    for _, tt := range s.cases {
        s.Assert().Equal(tt.expected, Abs(tt.input), "FAIL %s", tt.name)
    }
    s.cases = []testCase{}
    s.Require().Equal(0, len(s.cases))
}
{{< / highlight >}}

It is not a problem at ALL! The `SetupTest` creates new array each time from
scratch. This is an advantage against table driven tests, where each run can
broke shared state, unless the test is not written carefully.

{{< highlight go >}}
func (s *absSuite) TestNoOfCases() {
    s.Require().Equal(2, len(s.cases))
}
{{< / highlight >}}

Up to know the tests from shiny new suite does not run. As typical for a good
Go code, there is a little magic behind the scenes. So integration of
`testify.Suite` with a `testing` is done via new testing function.

{{< highlight go >}}
func TestSuite(t *testing.T) {
    suite.Run(t, new(absSuite))
}
{{< / highlight >}}

> Elementary, my dear Watson.

## Mocks

Code, especially in Go, usually communicates with the rest of the world. There
are database systems, other microservices, operating systems and so. Sometimes
it is easy to simply start the dependency as a part of the unit test and test
against real database.

In other cases the cost of the setup is too complex. It is hard to test error
states, like canceled context, forbidden login or overloaded system.  Then the
solution is to mock (imitate) the third party system as a part of our unit
testing. It generally brings a few advantages.

1. helps decoupling of interfaces and implementation
2. makes failure injection trivial
3. enhances code coverage

Disadvantages are

1. it is way more work
2. devs must pay an attention to keep mock as close simulation to the real world as possible

In general mocks are one of the best tools to test protocols. So here is `abs`
computation to microservices world.

{{< highlight go >}}
type MathService interface {
    Abs(context.Context, int) (int, error)
}
type MathClient struct{
    svc MathService
}
func (c *MathClient) Abs(ctx context.Context, i int) (int, error) {
    return c.svc.Abs(ctx, i)
}
{{< / highlight >}}

Code pretends that one need X machine big cluster each time one want to get
result. So it is impractical to setup, initialize and run it each time one type
`go test`.

Fortunatelly there is `testify/mock` to the rescue.

{{< highlight go >}}
import "github.com/stretchr/testify/mock"
type MathMock struct {
    mock.Mock
}
func (m *MathMock) Abs(ctx context.Context, i int) (int, error) {
    args := m.Called(ctx, i)
    return args.Int(0), args.Error(1)
}
{{< / highlight >}}

And that's it. Now `MathMock` implements necessary interface, so it can be used
in the test. So there is a need to write test case

{{< highlight go >}}
func TestMock(t *testing.T) {
    assert := assert.New(t)
    require := require.New(t)

    ctx := context.Background()
    cli := &MathClient{svc: &MathMock{}}

    i, err := cli.Abs(ctx, -1)
    require.NoError(err)
    assert.Equal(1, i)
}
{{< / highlight >}}

And it fail!

{{< highlight sh >}}
assert: mock: I don't know what to return because the method call was unexpected.
        Either do Mock.On("Abs").Return(...) first, or remove the Abs() call.
        This method was unexpected:
                Abs(*context.emptyCtx,int)
                0: (*context.emptyCtx)(0xc000018188)
                1: -1
        at: [main_test.go:164 main.go:46 main_test.go:175
{{< / highlight >}}

This is because developer must instruct `mock` object to expect mocked function
to be called with a specific arguments. Here comes the `On` method, which
allows to define cases with an arbitrary deep level of a granularity.

{{< highlight go >}}
func TestMock(t *testing.T) {
    assert := assert.New(t)
    require := require.New(t)

    ctx := context.Background()
    mathMock := &MathMock{}
    cli := &MathClient{svc: mathMock}

    mathMock.On("Abs", mock.Anything, mock.Anything).
    Return(1, nil)
    i, err := cli.Abs(ctx, -1)
    require.NoError(err)
    assert.Equal(1, i)
}
{{< / highlight >}}

The problem is it ignores the input parameters at all. Wach call of mocked
`Abs` will return `1, nil`. Better version, which simulates the buggy version
of `Abs` is

{{< highlight go >}}
func TestMock(t *testing.T) {
    assert := assert.New(t)
    require := require.New(t)

    ctx := context.Background()
    mathMock := &MathMock{}
    cli := &MathClient{svc: mathMock}

    mathMock.On("Abs", mock.Anything, -1).
    Return(1, nil)
    mathMock.On("Abs", mock.Anything, mock.Anything).
    Return(-42, nil)
    i, err := cli.Abs(ctx, -1)
    require.NoError(err)
    assert.Equal(1, i)
    i, err = cli.Abs(ctx, -42)
    require.NoError(err)
    assert.Equal(42, i)
}
{{< / highlight >}}

### Called once

`Abs` is quite straightforward function to test. One easily assert
things like number of calls of a mock

{{< highlight go >}}
// main.go
// MathClient has a bug and calls abs twice
func (c *MathClient) Abs(ctx context.Context, i int) (int, error) {
    c.svc.Abs(ctx, i)
	return c.svc.Abs(ctx, i)
}
// main_test.go
// test that function is called ONLY once
    mathMock.On("Abs", mock.Anything, -1).
    Return(1, nil).Once()
{{< / highlight >}}

The same are `Twice` and `Times` methods, which allows to assert number of
calls inside tested functions. Similar method is `Maybe`, which don't fail when
mocked method is not called at all.

### Timeouts

Timeouts is inevitable part of network programming. Testify allows developers to
specify function duration to test the context cancelation. It this example the
API is wrong, because context cancelation should immediatelly return err.
However it would require adding gorutines and context into `MathClient.Abs`
method.

{{< highlight go >}}
func TestMockTimeout(t *testing.T) {
	assert := assert.New(t)
	require := require.New(t)

	ctx := context.Background()
	ctx, cancel := context.WithTimeout(ctx, 250 * time.Millisecond)
    defer cancel()

	mathMock := &MathMock{}
	cli := &MathClient{svc: mathMock}

	mathMock.On("Abs", mock.Anything, -1).
		Return(1, nil).After(1 * time.Second)
	i, err := cli.Abs(ctx, -1)
	require.NoError(err)
    require.NoError(ctx.Err())
	assert.Equal(1, i)
}
{{< / highlight >}}

### Advanced filtering

It can be impractical to list all the possible cases. Or there can
be parts of input, which are constructed in some private functions deep in the
chain and they are better to be ignored.

There is
[MatchedBy](https://pkg.go.dev/github.com/stretchr/testify@v1.4.0/mock?tab=doc#MatchedBy)
function providing argument matching via functions.

{{< highlight go >}}
// main.go
type AbsRequest struct {
    ctx     context.Context
    i       int
    garbage string
}

//main_test.go
    assert := assert.New(t)
    absMatcher := func(r *AbsRequest) bool {
        return assert.EqualValues(r.i, -1)
    }

    mathMock.
        On("Abs", mock.Anything, mock.MatchedBy(absMatcher)).
        Return(1, nil)
{{< / highlight >}}

With `absMatcher` it is

* easy to ignore unecesarry bits of input structures.
* use a function from `assert` package supporting comparsion and diffing of
  arbitrary nested and complicated structures
* if matcher function is a closure, then it can access all
  variables from the outer scope

## All parts

1. [Part1](/blog/testify-make-go-testing-easy/) introduces `assert`, talks briefly abot Python and C and shows basics of `testify`
2. [Part2](/blog/testify-make-go-testing-easy-2/) introduces table driven testing, more helpers like `ElementsMatch` or `JSONeq`
3. [Part3](/blog/testify-make-go-testing-easy-3/) gets more advanced with test suited and mocks

Logo by [CarbonArc@Flickr](https://www.flickr.com/photos/41002268@N03/): https://www.flickr.com/photos/41002268@N03/23958148837
