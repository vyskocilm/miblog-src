---
title: "Go 111 Modules Monorepo and Shared Code"
date: 2018-11-09T19:19:40+02:00
image: "img/gopher-flicker.jpg"
draft: true
---

I recently joined company as a Python developer. However it turns out there is bigger need to evolve their code written in golang (and PHP and Javascript). I am not an expert in go, so it took me a while to figure things out. However I love to learn new things and to solve new puzzles (the real ones, there is nothing more boring to solve artificial ones for me)

In this post I am going to present the way how we moved shared gode to new 1.11 compatible modules. If you want to know more about golang in monorepo, differences between github and gitlab and all the funny things I found, please continue.

# Problem: duplicated code

Move shared code out. Do not repeat yourself. Do not copy code.

Project I am working on is written as a few microservices. There are all the cool and modern stuff above like Dockers, Swarm, distributes, message queue and so. But from developer's point of view the biggest challenge is project structure. As we have joined already existing organization with a monorepo and working CI/CD system around, our proposals to use more smaller repositories were not accepted. There will be so many things needed to rebuild in order to achieve it and no one is going to pay for it.

Because there was no one experienced in golang, devs took the easiest approach copy the shared code around. However that became too confusing as it's impossible to have a track and manually synchronize shared code. As go 1.11 got new [module system](https://github.com/golang/go/wiki/Modules), lets use it.

# Problem: Monorepo structure

I created a small repo mimicking the structure of the one we have.

<script src="https://gist.github.com/vyskocilm/33eb6220e5a32aad08322ce2fd00ee78.js"></script>

This works pretty well, all the microservices are places in a root of monorepo, where shared code lives under `g/` directory. You can build and test shared pieces independently and build and test the service code together with shared code. So far so good ....

# Problem: does not work in gitlab

Monorepo lives in a private instance of gitlab, so let's move the code, change the naming and it should work. Except it does not. Despite the same structure, it can't find the shared code. Fortunately `go build` have `-v` and `-x` command switches, which helped me to track down the problem. Long story short. It turned out that there is a hidden protocol between code hosting and `go build`.

```
curl -s https://github.com/vyskocilm/gazpacho?go-get | grep go-import
 <meta name="go-import" content="github.com/vyskocilm/gazpacho git https://github.com/vyskocilm/gazpacho.git">
```

The `meta` tag `go-import` turned out to be very important factor and it was the case where github and gitlab behaves differently. And as a result was `go build` failed to find shared code. And it turns out that previous incarnation worked somewhat differently than I wanted. I wanted to have shared code, but with independent smaller modules inside. So solution was simple. Add few more `go.mod` files to split the code even more. Now the code works well, except ...

# Problem: change shared code and service code

Internally new module system calls `git` and places results under `~/go/pkg/mod` in somewhat complicated structure. The problem now is

1.  /me adds new method `Reload` to `g/cfg`
2.  /me wants to use it in service code
3.  `go build` clones not yet committed version from master, so fail to build

Initially we have issue two merge requests for each change, but not having an ability to use new code or even worse, inability to fix typos in shared code in less than two requests was extremely unfriendly. Fortunately developers of golang have thought this. It most likely depends on a fact Google has monorepo as well, so they really need it to work as well. Solution for this is [`replace`](https://github.com/golang/go/wiki/Modules#when-should-i-use-the-replace-directive) directive. Here you can trick build system to use different repo. Or the better the different filesystem path. It works for `main` package, so in perfect solves the problem for service code. For shared ones, there is probably no way.

```
# project1-serviceA/go.mod
replace github.com/vyskocilm/gazpacho/g/cfg => ../g/cfg
```

The only one problem is that you can't use new dependencies this way, however this is being worked on and most likely will be fixed for 1.12\. So we can end here ...

# Problem: need to pass gitlab credentials each time

This is huge problem. Devs are going to hate you. Unfortunately golang itself does not provide such method. However as git itself is nowadays one of the most used tools, especially when it comes to CI and fetching the things there, here is the way. Using `insteadOf` you can rewrite git configuration to use ssh transport or to pass credentials

```
# for devs: Use ssh, Luke!
# source: https://gist.github.com/dmitshur/6927554
git config --global url."git@gitlab.example.com:".insteadOf "https://gitlab.example.com"

# For Gitlab CI, pass CI_TOKEN
# gitlab CI worker can access gitlab git
# https://gitlab.com/gitlab-org/gitlab-ce/issues/45610
before_script:
  - git config --global url."https://gitlab-ci-token:${CI_JOB_TOKEN}@gitlab.example.com/".insteadOf "https://gitlab.example.com/"
```

# Problem: there are no more problems

Major takeways

1.  I have discussed working monorepo structure compatible with golang modules
2.  I have talked about ways how to debug issues with `go build` and told you about `?go-get=1` protocol
3.  I presented a way how to push changes to shared and service code in one request
4.  And last but not least, I covered the way how to use `go build` against private git repositories

Feel free to clone [Gazpacho](https://github.com/vyskocilm/gazpacho) repo and give me feedback on how things are working for you.  
[Reddit thread](https://www.reddit.com/r/golang/comments/9viv50/how_i_solved_go_111_modules_private_monorepo_and/)  

> Golang 1.11 modules, monorepo, private gitlab vs github - how this fits together: [https://t.co/OntcaQEGYe](https://t.co/OntcaQEGYe) [#golang](https://twitter.com/hashtag/golang?src=hash&ref_src=twsrc%5Etfw) [#gitlab](https://twitter.com/hashtag/gitlab?src=hash&ref_src=twsrc%5Etfw) [#monorepo](https://twitter.com/hashtag/monorepo?src=hash&ref_src=twsrc%5Etfw) [#git](https://twitter.com/hashtag/git?src=hash&ref_src=twsrc%5Etfw) [#github](https://twitter.com/hashtag/github?src=hash&ref_src=twsrc%5Etfw)
> 
> — Michal Vyskocil (@vyskocilm) [9\. listopadu 2018](https://twitter.com/vyskocilm/status/1060818093948239872?ref_src=twsrc%5Etfw)

Logo by samthor@Flickr: [https://www.flickr.com/photos/samthor/5994939587]
