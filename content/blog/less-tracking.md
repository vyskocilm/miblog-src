---
title: "Less tracking of readers"
date: 2024-02-26T14:31:21+01:00
image: "blog/less-tracking/UBlock_Origin.png"
draft: false
---

When I moved my blog to [self hosted](/blog/vme-kubernetes/) virtual machine, I
later realised that the content itself was still calling external services for
web fonts and css and a javascript.

It makes no sense to move from Microsoft owned Github Pages to self-hosted and
force users to visit Google, unpkg.org and cdnjs.cloudflare.com every time they
visit my humble blog.

# Self-hosted

Starting from today the blog server hosts

 * `FiraSans` and `FiraCode` fonts instead of relying on Google Fonts
 * `pure-min.css` and `grids-responsive-min.css` so that unpkg.com is no longer called
 * use native [syntax highlighting](https://gohugo.io/content-management/syntax-highlighting/) of Hugo
   and no longer donwload `highlight.js` from a CloudFlare CDN

# New look

While I was doing changes to fork of hestia-pure, I used a
[willem.dev](https://www.willem.dev/articles/url-path-parameters-in-routes/) as
an inspiration and changed the look of the site. It uses colors from my
favorite color theme
[gruvbox](https://github.com/morhetz/gruvbox?tab=readme-ov-file#light-mode-1).

