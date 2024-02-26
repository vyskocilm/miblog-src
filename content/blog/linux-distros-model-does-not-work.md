---
title: "Linux Distros Model Does Not Work"
date: 2024-02-12T11:57:27+01:00
draft: true
---

Links

> consider that Go and Rust were both developed nearly exclusively by people working for firms
https://lists.debian.org/debian-devel/2024/01/msg00308.html

> "I don't care about the plights of farmers; I get my food from a supermarket!"
https://lwn.net/Articles/960161/

>  How about something truly revolutionary and never seen before, like for example a stable ABI and shared libraries?
https://lwn.net/Articles/959916/


https://thephd.dev/binary-banshees-digital-demons-abi-c-c++-help-me-god-please
https://thephd.dev/to-save-c-we-must-save-abi-fixing-c-function-abi
https://thephd.dev/your-c-compiler-and-standard-library-will-not-help-you

https://faultlore.com/blah/c-isnt-a-language/
http://harmful.cat-v.org/software/dynamic-linking/

> Linus Torvalds on why desktop Linux sucks
https://www.youtube.com/watch?v=Pzl1B7nB9Kc

https://tip.golang.org/src/cmd/compile/abi-internal
https://tip.golang.org/doc/asm

http://dave.cheney.net/2016/01/18/cgo-is-not-go
> C doesn’t know anything about Go’s calling convention or growable stacks, so
> a call down to C code must record all the details of the goroutine stack,
> switch to the C stack, and run C code which has no knowledge of how it was
> invoked, or the larger Go runtime in charge of the program.
https://stackoverflow.com/a/37961486/3169754


> The default stack size is platform-dependent and subject to change.
> Currently, it is 2 MiB on all Tier-1 platforms.

https://doc.rust-lang.org/std/thread/#stack-size

While C has 2MB or 4MB (depending on a platform)

https://man7.org/linux/man-pages/man3/pthread_create.3.html


