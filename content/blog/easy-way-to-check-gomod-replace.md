---
title: "Easy way to check go.mod replace via Linux namespaces"
date: 2019-11-04T10:26:04+01:00
image: "img/broken-chain.png"
draft: false
---

The common problem for monorepo and go modules is that the following

> It is really easy to miss the go.mod replace directive

In a following text we will learn more about Linux, namespaces and container
technologies like Docker or Podman.

## The problem

While `go build` is awesome tool, doing a lot of things in the background has
sometimes consequences. And the typical consequences is when we build the
software from an environment without network access. It should be **every**
build environment.

Both [https://opensuse](openSUSE) and [https://getfedora.org][Fedora]
disallow network access from their build workers. So that every build dependency
must be correctly packaged and installed as a part of build process. And I am
sure others Linux distributions have the same policy.

Similar case is a build of Docker image, where we don't (and don't want to)
deploy SSH keys to access private git instance.

```
Step 4/9 : RUN go build
go: example.org/project/foo/bar@v0.0.0-20191028170754-6e9b624fadee requires
        example.org/project/foo/spamm@v0.0.0-20191028170754-6e9b624fadee:
        reading https://api.example.org/2.0/repositories/projects/foo?fields=scm: 403 Forbidden
```

There are a few ways for developer to solve it

## Turn off networking

It is harsh method and I won't be happy to have my audiostreams interrupted.
Sadly it is truly multiplatform way.

## Unshare

Linux kernel has a concept of
[namespaces](https://en.wikipedia.org/wiki/Linux_namespaces). They simply allow
to virtualize several kernel resources and to pretend to userspace they are
running on their own computer. Namespaces are key concept of all container
technologies like Docker, Podman, LXC and others.

System allows you to manipulate with namespaces is via
[https://github.com/karelzak/util-linux/blob/master/sys-utils/unshare.1](unshare)
command and system call. This way one can create private network namespace,
which is not configured by default and can't connect to anything.

```
curl --silent example.com | wc -l
46
unshare -r -n curl example.com
curl: (7) Couldn't connect to server
```

Command does following

1. `-r, --map-root-user` maps current user and group id to root. Not that this
   is fully secure, because from non-namespaced point of view is this still the
   same user.
2. `-n, --net` create own empty network namespace

So running `unshare -r -n go build` will ensure that `go build` does not have
access to network at all.

## Containers

As I have mentioned namespaces are key for container technologies. And it is
not surprising that both [https://www.docker.com/](Docker) and
[https://podman.io](Podman), and most likely all others, can do the same.

```
$ docker run alpine:latest ping -c 1 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
64 bytes from 8.8.8.8: seq=0 ttl=54 time=1.426 ms

--- 8.8.8.8 ping statistics ---
1 packets transmitted, 1 packets received, 0% packet loss
round-trip min/avg/max = 1.426/1.426/1.426 ms
```

With network none

```
$ docker run --network=none alpine:latest ping -c1 8.8.8.8
PING 8.8.8.8 (8.8.8.8): 56 data bytes
ping: sendto: Network unreachable
```

Logo by Clker-Free-Vector-Images-3736@Pixbay: [https://pixabay.com/vectors/chain-broken-link-freedom-297842/]
