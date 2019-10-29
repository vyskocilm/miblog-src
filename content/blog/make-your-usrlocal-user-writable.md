---
title: "Make your /usr/local user writable"
date: 2018-01-21T22:18:29+02:00
draft: false
---

# About

I am the long term user of Linux based operating systems.  And as a C developer mostly working on stuff built by GNU autotools. Therefor I need to quickly build, install and test new versions of the software. I usually make changes to lower layers (like [zeromq/czmq](https://github.com/zeromq/czmq/pull/1833)).

Typical advice for Linux is  to **not** run things as a root!

### What is root

> Linux operating systems are multi user ones. Each user account have own documents and
> can't interfere with others. Except
> root. Root is a super user. It takes it all. Permission checks are not applied
> to this user. Never!

## The solution

The standard installation target for autotools based `make install` is `/usr/local`. This is path can be written by root. There are many ways to do to avoid this problem. You can install using `sudo` helper. You can configure `--prefix` and related variables like `PATH`, `PKG_CONFIG_PATH`, `LD_LIBRARY_PATH` and so.

Or there is way more simpler thing to do

{{< highlight sh >}}
$ sudo chown $myuser: /usr/local
{{< / highlight >}}

This is a tip from [Pieter Hintjens](http://hintjens.com) book [Scalable C](https://hintjens.gitbooks.io/scalable-c/content/chapter1.html). I actually used all three ways, however cannot be more happy with user writable `/usr/local`.

Logo by pxhere: [https://pxhere.com/en/photo/1267185]
