---
title: "My blog runs on Kubernetes and I like it"
date: 2023-12-01T21:37:31+01:00
draft: false
---

As I approach that famous middle age now, I realised that most of my
online presence is controlled by big US companies. Pretty ironic for someone
who has been using Linux and open source for decades, isn't?

So I went through a bit of a transformation process starting with deleting my Twitter account. Right after
THE takeover. And joined the Fediverse at
[@vyskocilm@witter.cz](https://witter.cz/@vyskocilm). After Bram died I joined https://opencollective.com/neovim as a financial contributor.

The next step was to remove myself from github. The glorious AI-powered
proprietary future is not something I enjoy and do not want to be a part of.
When I moved to Codeberg I realised that the service provided by the community
does not offer CI. But that's something I can fix. Can't I?

So I finally decided to do what I never did - rent a VM and host myself.

## Prologue

Despite being a long time Linux user, I am a very _lame_ admin myself. I have
never enjoyed configuring, deploying or managing servers and server software,
configuring networks, firewalls and all that stuff.

So, while one may be tempted to just `bash`

![Is that all you need?](de115a5356391189.png)
[@nixCraft](https://mastodon.social/@nixCraft/111397002672304875)

Anyway! Setting up a VM is never easy. You need to

 1. choose an operating system
 2. make sure it's updated regularly
 3. install additional software
 4. learn how to configure system packages and network and firewall
 5. learn how to configure additional packages
 6. repeat it for all new server

So the truth is that _every_ single approach involves a learning curve in how
to do it. You can choose to do everything manually. And provision the server
via ssh+tmux+vim and a `bash` and make your own nice pet server with your own
scripts. Or you can use advanced configuration software like Ansible or Salt to
deploy the server and build a cattle.

My advantage was that I was inexperienced and hadn't invested deeply in any of the
technology. So I was free to choose.

## Episode

It turns out that I am actually not that _inexperieced_. The platform I am most
familiar with is Kubernetes. This is how things are being deployed almost
everywhere I have worked last few years. Additionally I use [openSUSE
Aeon](https://en.opensuse.org/Portal:Aeon), the immutable desktop distribution
based on [openSUSE MicroOS](https://en.opensuse.org/Portal:MicroOS). 

So my preferred solution will include MicroOS as a base. I don't want to play
with the system itself. I expect it to take care of itself via automatic atomic
updates and recovery in case of problems. This base speaks in a favor of a
container based deployment. As usual with a technology, it is both a solution and
a curse at the same time.

> As a developer I want to have everything in a container.
> So I need an additional software to take a care of my containers.

Kubernetes may not be perfect, but it's the solution I am most familiar with.
Which invalidates my previous claim about being inexperienced, doesn't it?
Biases are _everywhere_.

The problem is that I had no ides how to automatically provision a VM and put
the Kubernetes there.

## Welcome Kube Hetzner

Luckily, there is a solution.
[github.com/kube-hetzner/terraform-hcloud-kube-hetzner](https://github.com/kube-hetzner/terraform-hcloud-kube-hetzner)

> A highly optimized, easy-to-use, auto-upgradable, HA-default & Load-Balanced,
> Kubernetes cluster powered by k3s-on-MicroOS and deployed for peanuts on
> Hetzner Cloud 🤑 🚀 

 ✅ MicroOS Tumbleweed based
 ✅ Kubernetes provided by [k3s](https://k3s.io) distribution
 ✅ Ready installation instructions and Terraform templates
 ✅ Can scale down to a single node cluster
 ✅ Compatible with Arm nodes, dropping the price lower

## The good

The installation instruction are heavily documented as well as the provided
Terraform file. So creating a Hetzner Cloud account, installing all needed
tools and then following the documentation was good enough.

This a snippet of ssh configuration. Pretty easy and nothing surprising for anyone who
knows a bit about ssh and ssh-agent.

The installation instructions are well documented, as is the supplied Terraform
file. So setting up a Hetzner Cloud account, installing all the necessary tools
and then following the documentation was good enough. The initial MicroOS
snapshot creation costs around 0.01€.

This is a snippet of the ssh configuration. Quite simple and nothing surprising
for anyone who knows a bit about ssh and ssh-agent.


```terraform
  # Customize the SSH port (by default 22)
  ssh_port = 1234

  # * Your ssh public key
  ssh_public_key = file("~/.ssh/vme_id_ed25519.pub")
  # * Your private key must be "ssh_private_key = null"
  # when you want to use ssh-agent for a Yubikey-like
  # device authentication or an SSH key-pair with a
  # passphrase.
  # For more details on SSH see
  # https://github.com/kube-hetzner/kube-hetzner/...
  ssh_private_key = null
```

## The bad

Documentation is written by experts for experts. Sometimes it is clear, sometimes less so.

### TLS setup

The TLS setup was particularly rough and I spent few hours reading the documentation.

> Create your issuers as described here https://cert-manager.io/docs/configuration/acme/.
> 
> Then in your Ingress definition, just mentioning the issuer as an annotation
> and giving a secret name will take care of instructing Cert-Manager to
> generate a certificate for it! You just have to configure your issuer(s)
> first with the method of your choice.

Everyone knows that being technically correct is the best way to be correct.
Mandatory reference to Futurama needed. I really struggled to understand the
instructions. The [Traefik
documentation](https://traefik.io/blog/secure-web-applications-with-traefik-proxy-cert-manager-and-lets-encrypt/)
is easier to follow. Documentation now points to this blog as well.

### Configuring Kubernetes

> That said, you can also use pure Terraform and import the kube-hetzner module
> as part of a larger project, and then use things like the Terraform helm
> provider to add additional stuff, all up to you!


The documentation is again _helpful_ in its special way as it refuse to
describe _how_ to achieve the goal. And I realized I don't know how to setup it
to talk with kubernetes via ssh. As I am a bit paranoid I disabled the access
to `kube-api` via https.

```terraform
  # For maximum security, it's best to disable it completely
  # by setting it to null. However, in that case, to get access
  # to the kube api, you would have to connect to any control
  # plane node via SSH, as you can run kubectl from within these.
  firewall_kube_api_source = null
```

Fortunately [Kustomize](https://kustomize.io) and especially
[kustomization_user_deploy
example](https://github.com/kube-hetzner/terraform-hcloud-kube-hetzner/tree/master/examples/kustomization_user_deploy)
were good enough to solve this problem.

### MicroOS

I found a packaging bug in the MicroOS `cloud-init` package. Which was easy to find
and propose [the fix](https://build.opensuse.org/request/show/1130364). And
that's an additional advantage of using openSUSE for me. I know how to fix
things there. Or at least I can try.

```
(50/54) Installing: cloud-init-23.3-1.1.aarch64 [.......
/var/tmp/rpm-tmp.MzH9hQ: line 4: /usr/lib/rpm/fdupes_wrapper: No such file or directory
warning: %post(cloud-init-23.3-1.1.aarch64) scriptlet failed, exit status 127
.done]
```

Due the nature of an atomic update nothing is installed and the system is not affected.


## The ugly

The learning curve - while I understand the Kubernetes, the truth is vanilla
k8s can't do a much. There is a whole ecosystem to deal with missing functionality.
Which in my eyes is more complex than the k8s itself.

 * Kubernetes - while the _regular_ stuff like Service/Deployment/Pods/Container
   is well-known to me, things like initContainers, managing volumes and others
   not so much
 * CertManager for Lets Encrypt certificates
 * Traefik Ingress Controller enabling the pods to be called from outside
 * Terraform
 * Helm
 * Kustomize.io
 * Traefik and Ingress
 * Proper logging and a monitoring
 * Proper security setup

## The conclusion

There is an elephant in the room. Do I need rented VM with Kubernetes and
Terraform and Helm and Packer and Kustomize to run a single nginx for my
static blog?

> Not at all!

However it was _a lot of fun_ and actually not _that_ much work. Thanks to the
wonderful [kube-hetzner](https://github.com/kube-hetzner) and
[openSUSE](https://opensuse.org) communities and their efforts. Additionally
having all infrastructure setup defined in Teraform and Kustomize and stored in
`git` with no additional effort is a nice plus.

And I do plan to use the VM as a CI runner for the project I just moved
to [Codeberg](https://codeberg.org/gonix/gio). Stay tuned!

