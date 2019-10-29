---
title: "Speeding Your Ci With Go Mod Download Again"
date: 2019-02-13T22:56:28+02:00
image: "img/gopher-flicker.jpg"
draft: true
---

Running `go mod download` slows down each CI run a lot. More dependencies increases time and it wastes the bandwidth again and again. As we do use special CI containers for the task, then go modules can be downloaded during container build phase and CI build will benefit from it. There are many articles about this topic. However most of them discuss one go module per repository or Docker image. As I wrote recently, we do use [monorepo with many go modules](
https://vyskocilm.github.io/blog/go-111-modules-monorepo-and-shared-code). So here is the short howto article how to squash all your go.mod files to one to simplify container build. And speedup your CI of course.

## The problem

The problem was that `go.mod` files are scattered around git repo and `go mod download` expects exactly one file to be used. There were two things comes into my mind

1. regenerate directory structure as a part of Docker file and copy `go.mod` and `go.sum` files there and call `go mod download` for each of those
2. or to squash all the files together to see what will happen

Number one does not look like *sexy* approach and it will complicate the Dockerfile a lot. So the second one looked like better approach.

## Squashing

Fortunately `go.mod` have script friendly structure, so could be easy to call a few unix tools to do the work. See an example from [example repository gazpacho](https://github.com/vyskocilm/gazpacho/blob/master/project1-serviceA/go.mod)

```
module github.com/vyskocilm/gazpacho/project1-serviceA

require (
	github.com/vyskocilm/gazpacho/g/cfg v0.0.0-20181109075706-a4ae50527451
	gopkg.in/yaml.v2 v2.2.1 // indirect
)

replace github.com/vyskocilm/gazpacho/g/cfg => ../g/cfg
```

What we are interested in is the part between `require (` and `)` and it turned out that `sed` is a perfect tool for.

```sh
sed -n -e '/^require (/,/^)$/p' go.mod | sed '1d;/^)$d'
        github.com/vyskocilm/gazpacho/g/cfg v0.0.0-20181109075706-a4ae50527451
        gopkg.in/yaml.v2 v2.2.1 // indirect
```

First we print everything between `require (` and `)`, see `p`rint command in first `sed` invocation. Then we strip the first line (`1d`) and line with `)`.

Then we can  combine everything to such awesome shell magic.

```sh
find $(git rev-parse --show-toplevel) -mindepth 2 -name 'go.mod' | while read GOMOD
do
    sed -n -e '/^require (/,/^)$/p' "${GOMOD}" | sed '1d;/^)$/d'
done | sort -u | grep -v 'internal.git.repo' >> go.mod
```

Which will
1. find all `go.mod` files from your monorepo (ignoring the top level one, this is `-mindepth 2` is about)
2. for **each** of those it will print the lines with dependencies
3. then all the lines will be sorted, all duplicates removed
4. additionally all dependencies points to internal repo were removed.-As [monorepo with many go modules](/blog/go-111-modules-monorepo-and-shared-code) article discuss, we do use `replace` directive. We do not care tah much about downloading internal dependencies. They are in as a part of git checkout anyway. Additionally one would need to pass `gitlab` credentials into Docker build somehow. Better to not do it.

## What about go.sum

```sh
find $(git rev-parse --show-toplevel) -mindepth 2 -name 'go.sum' | xargs sort -u | grep -v 'internal.git.repo' > go.sum
```

There is nothing to show, format is line oriented, so friendly to unix tools without any hacking

Logo by samthor@Flickr: [https://www.flickr.com/photos/samthor/5994939587]
