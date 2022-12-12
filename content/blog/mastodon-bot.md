---
title: "@hintjensquotes@botsin.space: a server-less Mastodon bot"
date: 2022-12-11T20:04:14+01:00
image: "img/hintjens-logo.jpg"
draft: false
---

Why and how I built the simply server-less
[@hintjensquotes@botsin.space](https://botsin.space/@hintjensquotes) bot. And
my tribute to Pieter.

![bot's main page](hintjensquotes.png)

Recently I setup the Mastodon bot serving quotes of [Pieter
Hintjens](http://hintjens.com). Pieter was one of the most influential man I
ever met. And I do consider his ideas as brilliant, though-provoking, sometimes
counter intuitive and typically in a sharp opposition to industry practices.
His list of achievements is amazing, he was a president of [Foundation for a
Free Information
Infrastructure](https://en.wikipedia.org/wiki/Foundation_for_a_Free_Information_Infrastructure)
fighting the software patents in Europe. He acted as CEO of
[imatix](https://imatix-legacy.github.io/), where they developed many things
including [Xitami web server](https://en.wikipedia.org/wiki/Xitami), [GSL code
generator](https://github.com/imatix/gsl) and
[OpemAMQ](http://web.archive.org/web/20161202232557/http://www.openamq.org/)
AMQP 0.9 server.

From recent past he has famous for
[ZeroMQ](https://en.wikipedia.org/wiki/ZeroMQ) project he co-founded. He was a
force behind the community pushing a lot of former imatix experience to his
open source project, namely high level bindings for libzmq
[czmq](https://github.com/zeromq/czmq), C/C++ project skeleton generator,
[zproject](https://github.com/zeromq/zproject), client/server generator
[zproto](https://github.com/zeromq/zproto) and so.

![Pieter](pieter.jpg)

Sadly as he suffers the terminal phase of a cancer, he took euthanasia in
October 4th 2016. Despite his friends took over websites to maintain his
legacy, the digital footprint of a person is typically dying too. Domains got
expired, links are no longer valid. Or as it happened recently, the Wikidot,
the engine behind his former homepage was destroyed by hackers and turned
offline.

Pieter spent most of his professional life building distributed messaging
applications and have a great passion for the open source. Therefor the
Mastodon Fediverse looks like the best place to publish his words giving them
an after life. I am pretty sure he would love the idea and [@hintjensquotes@botsin.space][https://botsin.space/@hintjensquotes bot].

That would be the why part. The how is fortunately simple. All in all what it
does is to provide Pieters's quotes, so having it running on top cluster on top
of Hadoop on top of RabbitMQ on top of Kafka sending stuff via CORBA or JMS
would not be in the spirit.


> The Lazy Perfectionist
>
> Never design anything that's not a precise minimal answer to a problem
> we can identify and have to solve. (Pieter Hintjens)

[The Lazy Perfectionist and other Patterns](http://hintjens.com/blog:22)

> Engineers and designers love to make stuff and decoration, and this
> inevitably leads to complexity. It is crucial to have a '"stop mechanism"', a
> way to set short, hard deadlines that force people to make smaller, simpler
> answers to just the most crucial problems.

[How to Design Perfect (Software) Products](http://hintjens.com/blog:19)

And a last, but not least

> Use Git and Github
> 
> I am not going todignify this with a non-zero number. If you aren't
> using git and github.com then you are already making excuses for doing
> it wrong. It is not about fashion or groupthink. Github (the company)
> know how to make Good Code, and their tools reflect this.

[Ten Rules for Good code](http://hintjens.com/blog:96)

Note that the github advice was written in 2015, eons before Microsoft takeover
and Pieter got [worried](http://hintjens.com/blog:111) about it in a meantime.
There are alternatives like [gitlab](https://gitlab.com) or
[codeberg](https://codeberg.org), [SourceHut](https://sourcehut.org/).

But the requirements from a guru were clean

 * simple
 * using git(hub)

Whole bot is then a single 170 lines long program in Go:
[main.go](https://github.com/vyskocilm/hintjensquotes/blob/main/main.go#L151).
Go is compiled language, but `go run` command and the focus on a compilation
speed, makes it suitable for a scripting work. And since Go is statically typed
language, yet a simple, it is great alternative to traditional shell, Python,
Perl or Ruby.

[Quotes](https://github.com/vyskocilm/hintjensquotes/blob/main/quotes.yaml)
themselves are in a plain yaml file, which is
[embeded](https://pkg.go.dev/embed) into Go variable during a build. {retty
neat feature simplifying the code even more.

Bot will then choose a random number capped by number of quotes, traverse the
list of articles and quotes to find a proper one and send it to Mastodon. The
most challenging part was to find a way how to actually post the status,
however fellow Python devs posted quite some
[howtos](https://www.ayush.nz/2018/09/tweet-toot-building-a-bot-for-mastodon-using-python).

TL;DR; is create an account on botsin.space, in development create an application, copy the access token to your application code and that's it.

![Mastodon application setup](mastodon-application.png)

And the serverless part? I did not want to setup some machine, or to deploy the code into my Turris router and make it running from there. As serverless means to use other's server, the Github and [Github Actions](https://github.com/vyskocilm/hintjensquotes/blob/main/.github/workflows/daily.yml) was the best place to ensure bot actually runs daily.

![Github Action daily.yml](gha-daily.png)

And that's all folks
