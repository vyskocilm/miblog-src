---
title: "Limiting Your Gorutines"
date: 2018-10-10T19:11:24+02:00
image: "img/gopher-flicker.jpg"
draft: false
---

# Limiting your goroutines

How to properly implement goroutine pool in golang with input and output channel(s). Text expects knowledge about [golang](https://golang.org/) and about [Go Concurency](https://tour.golang.org/concurrency/1). That means I am not going to explain what goroutine, channel or defer actually are.

## Intro

Gorutines are some kind of lightweight threads managed by userspace golang runtime. They save memory and context switches. Therefor it is possible to spawn literally [Millions!](https://rcoh.me/posts/why-you-can-have-a-million-go-routines-but-only-1000-java-threads/) of them. However as BenPar^W Voltaire said

![With great power comes great responsibility](https://media.licdn.com/media/gcrc/dms/image/C4E12AQFoCyvG3y_XEA/article-cover_image-shrink_600_2000/0?e=1544659200&v=beta&t=nNlZk3EjA5HecFvdl52P0qsWxmeuBbLq81ESKDa04ms)  
[Source: linkedin](https://www.linkedin.com/pulse/quote-great-power-comes-responsibility-voltaire-mba-bsee/)

It is not always practical to spawn your task million times. For example when you need to crawl web pages. It's good idea to NOT do it millions of times at the same time. On the other hand this is EXACTLY the kind of task golang and its goroutines shines at. The way of solving it is called **pool**, or workers, task queue, whatever you want. The idea is the same, we have a huge number N of tasks, which will be distributed to smaller number P of goroutines. Each goroutine will read next input from the channel, do the work and return result on a channel. This way we can parameterize the number of concurrent tasks we run. We can measure speed of execution using different numbers. Or we can ensure our code can deal with huge input without crashing the world :-)

## Deadlocks everywhere

The situation looks simple. Golang is an advanced language with builtin support for goroutines and concurrency. One just need to search the internet a bit to find an example. Knowing very little about golang, I naturally came to [StackOverflow](https://stackoverflow.com/questions/29342701/write-to-same-channel-with-multiple-goroutines) to get the examples of READing data from the go channel. And the [code](https://play.golang.org/p/ESq9he_WzS) looked simple and printed all numbers. That means we can **distribute** data from one channel and **consume** them from go routines. And golang runtime will make the magic behind to make this happen.

So let's add writing to the an another output channel. We will read it from main thread and everything should work! Something like [Worker pools @gobyexample.com](https://gobyexample.com/worker-pools).

And BAM! Code deadlock!

![deadlock](http://nikolar.com/wp-content/uploads/2013/09/trafficDeadlock.jpg)  
[Source: nikolar.com](http://nikolar.com/tag/deadlock/)

Program crashed and never print anything. Panic mode started. To make the long story short. Here is the key

> You MUST read from channel you're writing into, or BAD things will happen.

My code was structured this way

1.  After some init, create P goroutines and execute them
2.  Write data to input channel
3.  Read results from output channel

But program stopped in part 2, so reading of the data never happened, which blocked the send part inside goroutines, ... there is no better word than **deadlock** to describe the situation.

## The solution

Fortunately golang provides a simple way to fix it. _Offload the second part to goroutine_. We can read from output channel immediately from main thread. Full code available on [https://play.golang.org](https://play.golang.org/p/hTHhgZy-KvV) or below

<script src="https://gist.github.com/vyskocilm/d0c1a2f57a9b0821c00f31190c167781.js"></script>

Logo by samthor@Flickr: [https://www.flickr.com/photos/samthor/5994939587]
