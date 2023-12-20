---
title: "Fearless BDD for Go"
date: 2023-12-20T21:46:14+01:00
image: "blog/fearless-bdd-for-go/Go_gopher_run.png"
draft: false
---

I like the Go's built-in support for
[testing](https://go.dev/doc/tutorial/add-a-test), especially
[table-driven](https://go.dev/wiki/TableDrivenTests) ones. At the same time I
always have a problem with how to structure the test properly. It is easy to
write something, but it is hard to read, adapt or improve the test in the
future.

Ruby developers know their [RSpec](https://rspec.info/), which enforces a test
structure. Then there is the
[Given-When-Then](https://en.wikipedia.org/wiki/Given-When-Then) naming
convention introduced by Cucumber or Selenium tests. Let us explore the
advantages and find out the simple way on how to write well structured tests in
idiomatic Go. Only standard library needed.

# The good

 1. Helps the developer focus on the test rather than (re)inventing the naming and the structure again.
 1. Makes tests more consistent
 1. Helps to write the independent test cases.

Last point is particularly useful in Go, where any test or a subtest can be
marked as suitable for a parallel execution using `t.Parallel()`

# The list

Go has no shortage of BDD styled libraries. [Awesome
Go](https://github.com/avelino/awesome-go#testing) lists a few

  1. [biff](https://github.com/fulldump/biff) - Bifurcation testing framework, BDD compatible.
  1. [gherkingen](https://github.com/hedhyw/gherkingen) - BDD boilerplate generator and framework.
  1. [ginkgo](https://onsi.github.io/ginkgo/) - BDD Testing Framework for Go.
  1. [GoConvey](https://github.com/smartystreets/goconvey/) - BDD-style framework with web UI and live reload.
  1. [godog](https://github.com/cucumber/godog) - Cucumber BDD framework for Go.
  1. [gogiven](https://github.com/corbym/gogiven) - YATSPEC-like BDD testing framework for Go.
  1. [GoSpec](https://github.com/orfjackal/gospec) - BDD-style testing framework for the Go programming language.
  1. [gospecify](https://github.com/stesla/gospecify) - This provides a BDD syntax for testing your Go code. It should be familiar to anybody who has used libraries such as rspec.

# Alien looking code

While I do appreciate all the efforts of all open-source authors, I have to
admit that I have a particular taste about how good Go code should look like. I
am sure every listed library is and can be used in many projects. So the
following critique is meant to explain why I offer another alternative.

My biggest problem is that in order to make the tests as concise as possible,
the result does not look like a Go code at all. The following example is from
[the Writing Specs](https://onsi.github.io/ginkgo/#writing-specs) chapter of
Ginkgo documentation.

```go
var _ = Describe("Books", func() {
  var book *books.Book

  BeforeEach(func() {
    book = &books.Book{
      Title: "Les Miserables",
      Author: "Victor Hugo",
      Pages: 2783,
    }
    Expect(book.IsValid()).To(BeTrue())
  })
```

Godog is an another example. A casual reading of the example gave me no clue to how
this worked. I would have been very confused if I had been asked to modify it.

```go
package main

import "github.com/cucumber/godog"

func iEat(arg1 int) error {
    return godog.ErrPending
}

func thereAreGodogs(arg1 int) error {
    return godog.ErrPending
}

func thereShouldBeRemaining(arg1 int) error {
    return godog.ErrPending
}

func InitializeScenario(ctx *godog.ScenarioContext) {
    ctx.Step(`^there are (\d+) godogs$`, thereAreGodogs)
    ctx.Step(`^I eat (\d+)$`, iEat)
    ctx.Step(
        `^there should be (\d+) remaining$`,
        thereShouldBeRemaining,
    )
}
```

Also, these libraries tends to hide `*t` as much as possible and insist
on using their own assert methods.

And last, but not least. They increase the level of indentation a lot. This is
a particular problem for Go, as you can see

```go
TestMe(t *testing.T) {
    t.Parallel()
    for _, tt := range tests {
        tt := tt
        t.Run(t.name, func(t *testing.T) {
            t.Parallel()
            // code starts at 3rd level
        })
    }
}
```

# Fearless BDD style testing

> “Talk is cheap. Show me the code.”
> ― Linus Torvalds

Let us rewrite a [rspec.info](https://rspec.info) test case from Ruby into
idiomatic Go using nothing just a standard library. First the `Bowling` struct.

```go
package main

type Bowling struct {
    score int
}
func NewBowling() *Bowling {
    return &Bowling{}
}
func (b *Bowling) Hit(pins int) {
    b.score += pins
}
func (b *Bowling) Strike() {
    panic("not implemented")
}
func (b *Bowling) Spare() {
    panic("not implemented")
}
func (b Bowling) Score() int {
    return b.score
}
```

And a test is simple - [see Go playground](https://go.dev/play/p/yf1rKtbr7CZ).

```go
package main

import "testing"

func times(n int, fn func()) {
    for i:= 0; i != n; i++ {
        fn()
    }
}

func TestBowling(t *testing.T) {
    t.Parallel()
    type given struct {
        strikes int
        spares int
    }
    type when struct {}
    type then struct {
        score int
    }

    tests := []struct {
        scenario string
        given given
        when when
        then then
    }  {
        {
            scenario: "no strikes, no spares",
            given: given{strikes: 0, spares: 0},
            then: then{score: 40},
        },
    }

    for _, tt := range tests {
        tt := tt
        t.Run(tt.scenario, func(t *testing.T) {
            t.Parallel()
            // given
            bowling := NewBowling()
            times(tt.given.strikes, func() {bowling.Strike()})
            times(tt.given.spares, func() {bowling.Spare()})
            times(
                10-tt.given.strikes-tt.given.spares,
                func() {bowling.Pins(4)},
            )
            // when
            score := bowling.Score()
            // then
            if score != tt.then.score {
                t.Errorf("got %q, want %q", score, tt.then.score)
            }
        })
    }
}
```

And running it is no surprise.

```
=== RUN   TestBowling
=== PAUSE TestBowling
=== CONT  TestBowling
=== RUN   TestBowling/no_strikes,_no_spares
=== PAUSE TestBowling/no_strikes,_no_spares
=== CONT  TestBowling/no_strikes,_no_spares
--- PASS: TestBowling (0.00s)
    --- PASS: TestBowling/no_strikes,_no_spares (0.00s)
PASS

Program exited.
```

Personally like this format more and more. It has nice advantages over all
the other approaches.

 1. Idiomatic Go.
 1. Easy to add more test cases.
 1. The start of each test can be copied and customized.
 1. Does not require a 3rd party library.
 1. That makes it compatible with and 3rd party library.
 1. Swaps the indentation for selector depth (`tt.phase.attribute`).
 1. Scales well from unit up to integration tests.
 1. Compatible with any keywords. Having a `describe`, `context`, `it`, `expect`
    like Ruby has is no problem.

It is longer than Ruby version and slightly more verbose than Go team's concise
approach.

```go
var flagtests = []struct {
    in  string
    out string
}{
    {"%a", "[%a]"},
    {"%-a", "[%-a]"},
```

However I like the clarity and the _idiomaticity_ of the result. Which is always
a nice quality to have when working with a code.


