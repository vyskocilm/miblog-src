---
title: "openSUSE Aeon: desktop is better when immutable"
date: 2024-01-14T20:04:14+01:00
image: "img/hintjens-logo.jpg"
draft: true
---

The [openSUSE project](https://en.opensuse.org/) is not short of offerings.
Leap, Tumbleweed, MicroOS, Leap Micro, discontinued Kubic and new projects
Aeon, Kalpa and Slowrock. Looks like openSUSE people loves inventing new names
:-)

Let me focus on an immutable desktop [openSUSE
Aeon](https://en.opensuse.org/Portal:Aeon). And why I think it ban be _the
best_ option for one's desktop.

# What is it

> openSUSE Aeon provides only a minimal base system with a GNOME Desktop
> Environment and Basic Configuration Tools ONLY. All Applications, Browsers,
> Codecs, etc are provided by flatpaks from Flathub.

[https://en.opensuse.org/Portal:Aeon](https://en.opensuse.org/Portal:Aeon)

# In a layman terms

The problem is of course how to fully describe it. On the one hand it is like a
any other Linux distribution. Just with a special features. So openSUSE Aeon is a

 1. variant of openSUSE rolling update distribution Tumbleweed (this may
    change, however is true at the time of a writing)
 1. it contains a narrower set of packages - those needed for GNOME desktop and
    a few others needed for a typical desktop usage
 1. it has immutable root filesystem, which user is _not_ supposed to modify
 1. this does include installing of other packages - this is possible, however
    _not recommended_ unless absolutely necessary
 1. as it is based on [MicroOS](https://en.opensuse.org/Portal:MicroOS)
    technology, updates are installed automatically and user "activates" them
    simply by rebooting.

# Why would one use it?

Lets get one thing straight and documentation mention it

> It is NOT for everyone

The end result is a Linux distribution with `vim`, `shell`, `podman`,
`systemd`, `glibc`, `pipewire`, `kernel`, `python`, `perl`, `openssl` and a
GNOME desktop with a few packages. There are a few differences

Like one can't install his favorite editor
```sh
michal@immutable> sudo zypper in neovim
[sudo] password for michal: 
This is a transactional-server, please use transactional-update to update or modify the system.
```

Or touch the root fs at all
```sh
michal@immutable> sudo touch /usr/bin/foo
touch: cannot touch '/usr/bin/foo': Read-only file system
```

It should be obvious that the system actively prevents its modification. Those
are not impossible, but not as convenient experience as a typical Linux
distribution (like Tumbleweed) offers. And not something recommended to do.
This make Aeon a bit unpleasant for power-users willing to change and modify an
every part of their system.

On the other hand Linux power users have a plenty of other offerings, don't they?

# The target user

> People looking for a just works experience and who do not care that much
> about a system underneath. It aims to deliver Android or ChromeOS like
> experience using a Linux distribution.

And here is me - I use Linux nearly two decades now. Started with Mandrake
Linux distribution in 200x and went through a lot of distro hopping including
Debian, Slackware, Ubuntu or Gentoo. Since I joined SUSE as an employee, I and
openSUSE user since then, despite no longer being employed.

I went through crazy phases, like building my own kernel to upgrade 2.4 to 2.6
on my Slackware, painstakingly changing my Gentoo USE flags for my poor AMD
Duron CPU and waiting for X and KDE to appear, from using a bleeding edge
packages like KDE 4.0, switched to rolling release openSUSE, switched to
systemd, Wayland or recently pipewire before it was popular.

My system grown to 3500 packages. With texlive, it'd be attacking 4500 if not
more. Here I must admire the stability of openSUSE Tumbleweed. Despite being a
rolling release I can hardly remember a problems with distribution upgrades.

Yet I become a happy user of openSUSE Aeon on my desktop I do my $dayjob from.
I become less and less interested in poking with the internals, configuring
stuff or even changing the desktops. I more appreciate the fact it works and
all it needs is a restart from time to time in order to upgrade.

Just like my phone or the Turris router or even my server do.

# How to install the neovim the right way

Fortunately it is _easy_ to get the mutable experience on an immutable
distribution. There is [distrobox](https://distrobox.it) installed and
configured, so one is a single command from his neovim, tmux, fd, ripgrep, go
or rust compilers or from a normal `zypper` (or `yum`, or `apt` or anything
else).

```sh
michal@immutable> nvim README.md
bash: nvim: command not found
michal@immutable> distrobox-enter mutable
Starting container...                   	 [ OK ]
Installing basic packages...            	 [ OK ]
Setting up read-only mounts...          	 [ OK ]
Setting up read-write mounts...         	 [ OK ]
Setting up host's sockets integration...	 [ OK ]
Integrating host's themes, icons, fonts...	 [ OK ]
Setting up package manager exceptions...	 [ OK ]
Setting up rpm exceptions...            	 [ OK ]
Setting up sudo...                      	 [ OK ]
Setting up groups...                    	 [ OK ]
Setting up users...                     	 [ OK ]
Executing init hooks...                 	 [ OK ]

Container Setup Complete!
michal@mutable> sudo zypper install neovim
michal@mutable> neovim
# the neovim window
```

Distrobox feels like a magic. It runs the mutable guest system, which can be
_any_ of many [supported
distributions](https://distrobox.it/compatibility/#containers-distros). It runs
them from OCI container image via podman and ensures that most of the host is
seamlesly integrated into running container. You have a full access to $HOME,
you can start CLI or graphical programs, you have an access to sound and
network and so on and so on.

I started with a single big "box" containing everything. After a while I have
switched to having more specialized boxes. Which I can create and destroy when
not needed. This got easier with
[assemble](https://distrobox.it/usage/distrobox-assemble/) command allowing me
to describe the image in simple ini file. Here is the one I use for
[cnf](https://github.com/openSUSE/cnf), the command not found handler written
in Rust.

```
[cnf]
image=registry.opensuse.org/opensuse/distrobox
additional_packages=fd fd-bash-completion fzf fzf-bash-completion fzf-tmux glibc-locale git neovim ripgrep ripgrep-bash-completion tmux vim-fzf ShellCheck wl-clipboard rustup
init=false
nvidia=false
pull=true
root=false
replace=true
start_now=false
init_hooks=ln -sf /usr/bin/distrobox-host-exec /usr/local/bin/flatpak
volume=~/.local/share/tmux/resurrect/cnf/:~/.local/share/tmux/resurrect/
```

Actually it was the command not found handler task which learned me to enjoy
more boxes. I ended up with three distinct boxes, which got created and destroyed as
needed.

 * `cnf` - the dev environment
 * `ubuntu-latest` - for experimenting and debugging Github actions
 * `osc` - the packaging stuff. Unfortunately the `osc` command is huge and
   don't play well with containers and need a rootful one, so I decided to
   create a specialized box just of a packaging.

Embracing boxes gave me more confidence to experiment. For instance when
changing my neovim setup from old vim script style to Lua and native plugins. I
experimented a lot in a specialized container with a private home. The same was
when I was checking the `zellij` instead of `tmux`. I created a box, played
around and removed once it was not needed.

/*      CUT         */

# What the hell is the immutable distribution

Let me introduce this part with a quote of [H.L. Mencken](https://en.wikiquote.org/wiki/H._L._Mencken)

> Explanations exist; they have existed for all time; there is always a
> well-known solution to every human problem — neat, plausible, and wrong. 

which is more commonly shortened to

> For every complex problem there is an answer that is clear, simple, and wrong.

# Clear, simple and wrong answer

So let me start with a totally wrong explanation. MicroOS (Desktop or any other
flavor) is a standard (openSUSE Tumbleweed)[https://get.opensuse.org/tumbleweed/]
distribution with three exceptions

1. `/` is read-only
2. instead of calling (https://manpages.opensuse.org/Tumbleweed/zypper/zypper.8.en.html)[zypper] (SUSE variant of apt, dnf, pacman, apk and so) to get the software, you use much longer command called [transactional-update](https://manpages.opensuse.org/Tumbleweed/transactional-update/transactional-update.8.en.html)
3. you must reboot the system _each_ time you have typed the long `transactional-update` command

While everything is _technically_ right. MicroOS is a regular Tumbleweed, it
uses the same build system, the same rpm packages, the same repositories, the
same technologies. And despite the fact the default is opinionated, one can
actually install _any_ Tumbleweed package there or to add any of the existing
third party repositories. And it's true the `zypper` does not work due
read-only nature of `/` and you need to call the longer one and a reboot.

Yet it is 100% misleading at the same time.

# A better answer - elevator pitch

It turns out it is _hard_ to explain the whole concept. For common non-nerdy
folks they simply do not care. And talking to fellow nerds is _worse_. They do
have an emotional bond to the technology and their own vision how things should
be done.

It is probably a reason MicroOS Desktop states

> It is NOT for everyone.

which is a terrible elevator pitch by the way. However understandable given the
nerdosphere around.

It turns out that there is a pretty good analogy, which can be found in a
famous video called _Tale of two brains_.

> Everything is connected with EVERYTHING.

Which is actually the best, shortest and correct explanation of what a
traditional distribution is. And more importantly what it _tries_ to be. One
big single coherent package created by tens of thousands independent pieces
developed by thousands of different development teams, each one with its own
goals, cadence and so on.

And it turns out that the opposite is the great analogy for what all immutable
systems tries to be

> Made up of a boxes

Which is funnier considering the immutable systems (at least on desktops) are
hardly usable without all the toolboxes, distroboxes and so on. Spoiler alert:
even on pure immutable system you are most likely going to have an everything
is connected with everything part anyway. Like the old Chinese Ying Yang
principle immutable systems allows you to combine the _best_ of both worlds.

The problem with this is - it is a terrible pitch too. It makes a sense only
as an opposite to traditional distribution (maybe we shall call them a legacy
now).

# The good

Let me summarize what we know at this stage

1. MicroOS is a flavor of openSUSE Tumbleweed (there are MicroOS flavors of Leap and SLES as far as I know)
2. It is 100% compatible, so everything you can install on Tumbleweed you can install on MicroOS
3. You must use a longer command and a reboot

The problem is that none of these points tells you _why_ is the MicroOS different (any why is this a good thing).

The fundamental difference is that MicroOS have a strong distinction between _base
OS_ and applications installed on top of it.

The typical Linux distribution has a concept of native packages (rpm, debs,
...) which are organized to repositories. Typically there is a blessed _main_
repository and a bunch of third party ones. But there is no isolation between
them. Any third party can replace anything from a main repo.

```
[manually installed application (Slack, ...)]
[main repository                            | [ third party repo (Chrome)           ]
|    [base system][libraries][applications] | [ third party repo (codecs)           ]
|                                           ] [ third party repo (additional stuff) ]
[kernel]
```

Is it expected that to get a sane results, all the main and third party
repositories must roll at the same pace. _Everything is connected to
everything_, remember? That means that a bug in a single repo breaks the whole
system, dependency conflict will break the system, large scale upgrade like
newer glibc breaks the system and so.

My personal argument _against_ this scheme is that every third party repo means
you give a root access to more entities. If you add Chrome repository, Google
inevitably has a root access on your machine.

This system has it's own strengths of course

 * Wanna to have a different kernel? You can replace it - either build your own, or add a third party repo
 * Wanna to have a newest gcc (mpv, vlc, codecs, ...)? Build from source or add 3rd party repo
 * Wanna to replace X11 and Wayland (Pulse/Pipewire/systemd/sysvs/whatever)? Simply install/unistall stuff on your running system
 * Wanna to have everything tailored according your personal needs? Install Gentoo and tune your USE flags

And I must admit - I've been there done that

 * Compiling mplayer on old Mandrake Linux with gcc 2.96? Done
 * Trying to install and build Linux 2.6 on an ancient Slackware with 2.4? Done
 * Using a Gentoo for a while? Done
 * Replacing sysvinit by systemd as early as possible? Done
 * Using KDE 4.0 right after the release? Done

The immutable systems _distinguishes_ between the base system and the app
layer(s) on top. It is not a specific of MicroOS, the similar distinction
exists with Fedora Silverblue and other distributions. And to some extend this
distinction exists in BSD land too. And it definitely exists in systems like
OS X, Windows. Android and ChromeOS is a perfect and well know example of it
and it adds an immutability on top of it too.

So this is how my MicroOS looks like

```
[ ~/bin/: manually installed stuff ]
[ distrobox for work: openSUSE Tumbleweed ]
[ distrobox for home: openSUSE Tumbleweed ]
[Flatpak layer
    [ Firefox ][ Chrome ] [ Slack ] [ Libreoffice ] [ Fractal ] ...
]
[MicoOS base - kernel+SELinux+firmware+glibc+podman+basic Gnome]
```

_Made up of a boxes_ now makes more sense, does not it?

The system is organized into different boxes, each updated in own pace

1. MicroOS is updated automatically and daily via transaction-update.timer (and a service)
2. Flatpacks are updated when their upstream or packager uploads a new release
3. Distroboxes are updated semi-randomly (I may add the user service to do it for me)

# But that is horrible!

The immutable distribution is a _deviation_ from a standard Linux distribution model. So it must be wrong

 * How many openssl copies do you have?
 * How many bundled dependencies are there?
 * What about my poor storage and memory?
 * DLL hell, Windows, meh
 * I hate the Software Stores
 * I hate Gnome/Wayland/systemd/btrfs/....
 * and so on

Remember the _It (MicroOS) is not for everyone_?. It's a terrible sales pitch,
but great advice if you do not like the model with boxes.

And yes, the way how the MicroOS and a similar systems are designed is
_different_. Fortunately there are plenty of analogies to other fields of a computer
science and engineering. So lets dive in

# Transactions

> In computer science, transaction processing is information processing [1]
> that is divided into individual, indivisible operations called transactions.
> Each transaction must succeed or fail as a complete unit; it can never be
> only partially complete. 

[Wikipedia Transaction processing](https://en.wikipedia.org/wiki/Transaction_processing)

An upgrade of Tumbleweed is in-place. That means during `zypper dup` operation,
each package of a system, which is one or thousands of files scattered on a
main storage. While an underlying package manager `rpm` goes lengths to ensure
this in-place update to be safe, this is not and cannot be a _safe_ operation.
Not for a single one, neither for more.

 * you cannot reliably rollback
 * if an upgrade fails in a middle, the result is an inconsistent system
 * no easy way how to rollback to previous state (especially if one don't like btrfs)
 * for most of the times replacing the shared libraries of a running programs
   is not safe and `zypper` and other tools actually do recommend a _reboot_
   after the upgrade. Patching the system is not only installing new version,
   but ensuring all components use it.

Immutable systems _never_ installs an update in-place. There are a lot of
approaches from btrfs (as used by MicroOS), through rpm-ostree (Silverblue) to
real base system images. See [overview on
lwn.net](https://lwn.net/Articles/922968/) for more details.

So the `transactional-update` is the MicroOS tool which

 * installs an upgrade (or package(s)) in a new btrfs snapshot
 * if the operation fails, the new snapshot is discarded. This provides an _atomicity_ of the operation
 * and as the most reliable way how to ensure new version is up and running, the _reboot_ will boot from new snapshot
 * however older versions of systems are there as well, providing a support for a _roolback_

In other words

> MicroOS provides atomic and revertible base system upgrades in an exchange for a _reboot_.

# Consistency

Personally speaking the consistency is what the Linux distributions are valued the most.

> One (g)libc to rule them all,
>    one (g)cc to find them,
> One repository to bring them all
>   and in the darkness bind them. 

Consistency and an almost unlimited _hackability_ of course.  Hence the
argument _How many openssl copies do you have?_ and his cousins.

So when someone like me do propose using Flatpack or other boxes. He must be
Lunatic. The heretic. God forbid he wants to use a _static linking_ or language
native package manager.

## A (consistent) slippery slope

> Thou shalt a single libssl version use as as well as libgrob-0.0.1~gitdeadbeff
> and other blessed dependencies as the wise men behind the holy repository says.

> Thou shalt not use anything than a holly packaging format given us by the
> Lord. The package must be built using a book of spells to be blessed.

> It is still considered a pure to add the honest 3rd party repository as I
> want to play non-holly media.

> Here is the personal repository of a dubious quality replacing an holly
> dependency with _newest_ one I actually must posses.

> I gave up! I want my $proprietary-stuff to be up-to-date, so I added a
> Google/Zoom/other corp repo, ok?

> D'oh - `wget https://example.net/foo-1.23.1.tar.gz; tar -c /usr/local -xf foo-1.23.1.tar.gz`

> I will go straight to the hell `curl https://example.net/foo-1.23.1/install.sh | sudo sh -`

```
#beentheredonethat
```

The problem with the consistency and especially when it is so _strong_ is that

 1. it slows down the system to a crawl. The reason our loved distributions
    works is a lot of labour intensive work dealing with a build problems and
    fixing everything. And as we have so many different distributions, the
    effort is of course duplicated
 2. maybe we nerds want a strong consistency because we like it as it
    simplifies a reasoning about a complex systems. But a real world the
    [consistency can be
    overrated](https://two-wrongs.com/data-consistency-is-overrated).

And frankly speaking, the are _zero_ guarantees of a consistency between your main, codes, vlc,
personal with updated foo and Google Chrome repository. Each one moves in it's own pace.

> MicroOS does one thing - it guarantee a strong consistency of the base
> Operating System box itself.

Consistency of a whole system depends on the respective maintainers of the
additional boxes. But they will have a hard time to screw anything than their
own box.

# More analogies

There are a lot more analogies, however I feel they'll be repeating the above

 * _shared mutable state_ - the `/` with main and dozens of external re

# Flatpak (flathub) bad

# Installation

# Usage - the distrobox

# Podman vs Docker

# The vision

The vision of MicroOS is

 * unattended upgrades
 * zero management operating system
 * Linux for my grandma
 * ...
