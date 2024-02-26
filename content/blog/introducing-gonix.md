---
title: "Introducing Gonix: unix userland written in Go"
date: 2024-02-13T09:51:58+01:00
image: "content/blog/introducing-gonix/cover.png"
draft: true
---

> It has been said that unix shell and the scripting is the worst form of scripting except all
> those other forms that have been tried from time to time.

Unix utilities, unix pipe and unix shell are not the best known programming
language. There were are and will be attempts to improve the situation. Like
venerable awk, programming languages like Perl, Python or Ruby, [Go
scripting](https://bitfieldconsulting.com/golang/scripting) or a PowerShell.

Yet despite all the flaws the unix scripts are everywhere. One can find them in
Makefile, embedded in yaml or as a content of a source code in any known
language. Even the the OCI image building tools like Dockerfile are powered by
shell

```Dockerfile
RUN bash -c "here; \
    is the; \
    mighty script"
```

# The advantages

Most scripts starts via interactive shell, which makes easy to start, adapt and
test the code in an extremely tight feedback loop. They rely on unix userland,
which exists in one form or another on every platform. And even if developers
can't agree on a single language they like, most of them are fine with having a
few lines of shell here and there.

# The disadvantages

Despite the years of a standardization are unix utilities not compatible across
platforms. This is not a problem for Linux users as they can expect the bash
and GNU utilities being installed everywhere. However other platforms like OS X
does ship with they're own set of utilities, which do differ from what GNU
versions do.

In other words

1. unix utilities are _tightly_ coupled with a system
1. that means they can't be easily swapped
1. unix utilities must be installed on a target system

# Installed?

I am sure the last limitation might be surprising for most of the people. Of
course the `cat` must exists on a filesystem before it can be called.

```Dockerfile
# first I need to install coreutils
RUN zypper --non-interactive install coreutils
# so I can call cat later on
RUN cat /ets/passwd
```

This was not a problem for a traditional unix machines, where a mighty smart
admin carefully prepared the system, configured a shared NFS and installed a
whole bunch of a different utilities in magical paths like `/bin`,
`/usr/local/gnu`, `/opt/solaris/bin`.

# Gonix

Given Go's heritage (rob, ken, ian, rsc) it is not surprising that implementing
unix tools in there is really straightforward task.

```go
// cat is a barebone version of /bin/cat
func cat() error {
    _, err := io.Copy(os.Stdout, os.Stdin)
    return err
}
// wcl is a barebone version of /bin/wc -l
func wcl() error {
    var lines int
    s := bufio.NewScanner(os.Stdin)
    for s.Scan() {
        if s.Err() != nil {
            return s.Err()
        }
        lines++
    }
    fmt.Fprintf(os.Stdout, "%d\n", lines)
    return nil
}
```

Even cooler thing is that because of an existence of `io.Pipe` and goroutines
it is easy to avoid calling an operating system and implement the unix pipe
entirely in Go. First one need to implement a proper abstraction - this one is from [codeberg.org/gonix/gio/unix](https://codeberg.org/gonix/gio/src/commit/28cf35822a8fce08afb4821c864d4471303d85d1/unix/unix.go#L27).

```go
type StandardIO interface {
	Stdin() io.Reader
	Stdout() io.Writer
	Stderr() io.Writer
}

type Filter interface {
	Run(context.Context, StandardIO) error
}
```

And the tools should implement a proper dependency injection instead of relying
on a global values from `os` package.

```go
import "codeberg.org/gonix/gio/unix"

type cat struct{}
func (cat) Run(_ context.Context, stdio unix.StandardIO) error {
    _, err := io.Copy(stdio.Stdout(), stdio.Stdin())
    return err
}
```

# Native pipe

While the real pipe implementation handles a lot and there's a
[codeberg.org/gonix/gio](https://codeberg.org/gonix/gio/) implementing
everything, including a support for having a pipe of different type than a
stream of bytes. The idea is as simple as this

```go
// cat | wc -l in a pure Go
func catwcl(ctx context.Context, stdio unix.Stdio) error {
    catStdin := stdio.Stdin()
    wcStdin, catStdout := io.Pipe()
    wcStdout := stdio.Stdout()

    var wg sync.WaitGroup()
    wg.Add(1)
    go func() {
        defer wg.Done()
        stdio := unix.NewStdio(catStdin, catStdout, os.Stderr)
        cat{}.Run(ctx, stdio)
    }()
    wg.Add(2)
    go func() {
        defer wg.Done()
        stdio := unix.NewStdio(wcStdin, wcStdout, os.Stderr)
        wcl{}.Run(ctx, stdio)
    }()
    wg.Wait()
    return nil
}
```

# Native shell

Having the native unix tools and a pipe works to an extend. However unix and a shell are synonymous, so having a real native shell interpreter in Go would be nice. Fortunately there is an amazing project `mvdan.cc/sh`, which implements a shell parser and an _interpreter_ in Go. The design is an extraordinary flexible allowing me to integrate gonix/gio with a just a few lines of Go code.

Here is the script demonstrating the capabilities. It

1. shows that the `PATH` is empty and variables must be assigned explicitly
2. that you can run `gonix/cat` command - the name is arbitrary and I use
   `gonix/` prefix to distinguish the various unit utilities implementations and to show this is
   a Go and not an ordinary shell
3. The pipe `cat | wc -l` works too
4. But running `grep` does not as $PATH is empty and there is no native command to be called

```go
const src = `
echo "########## Showcasing the gonix ##########"
echo "## 1.    Shows that the PATH is empty"
echo PATH=${PATH}

echo "## 2.    Shows that the gonix/cat does work as a cat"
gonix/cat --number ${GLOBAL}

echo "## 3.    Shows that the piping works too"
gonix/cat ${GLOBAL} | gonix/wc -l

echo "## 4.    Shows that script can't run a grep"
grep
`
```

And a code itself is a thin wrapper on top of the `mvdan.cc/sh/interp`

```go
func main() {

	commands := map[string]func([]string) (unix.Filter, error){
		"gonix/cat": func(args []string) (unix.Filter, error) {
            return cat.New().FromArgs(args)
        },
		"gonix/wc":  func(args []string) (unix.Filter, error) {
            return wc.New().FromArgs(args)
        },
	}

    // execGonix is an integration between goinx and mvdan.cc/sh
	execGonix := func(next interp.ExecHandlerFunc) interp.ExecHandlerFunc {
		return func(ctx context.Context, args []string) error {
			fromArgs, ok := commands[args[0]]
			if !ok {
				return next(ctx, args)
			}

			hc := interp.HandlerCtx(ctx)
			cmd, err := fromArgs(args[1:])
			if err != nil {
				return err
			}
			stdio := unix.NewStdio(hc.Stdin, hc.Stdout, hc.Stderr)
			return cmd.Run(ctx, stdio)
		}
	}

	file, _ := syntax.NewParser().Parse(strings.NewReader(src), "")
	runner, _ := interp.New(
		interp.Env(expand.ListEnviron("GLOBAL=/etc/passwd")),
		interp.StdIO(nil, os.Stdout, os.Stdout),
		interp.ExecHandlers(execGonix),
	)
	runner.Run(context.TODO(), file)
}
```

And with an output proving it really works.

```sh
michal@gonix:~/projects/gonix/gsh> go run main.go
########## Showcasing the gonix ##########
## 1.    Shows that the PATH is empty
PATH=
## 2.    Shows that the gonix/cat does work as a cat
root:x:0:0:root:/root:/bin/bash
messagebus:x:499:499:User for D-Bus:/run/dbus:/usr/sbin/nologin
lp:x:498:497:Printing daemon:/var/spool/lpd:/usr/sbin/nologin
michal::1000:1000:michal:/home/michal:/usr/bin/bash
## 3.    Shows that the piping works too
6
## 4.    Shows that script can't run a grep
"grep": executable file not found in $PATH
```

# Wasi

Initially I was thinking that writing a unix utilities in Go would be a great
and funny side project. It turned out it is not. I gave up after implementing a
few. Fortunately other devs are rentlesly working on a Web Assembly and
specifically WASI Web Assembly System Interface, which preview1 is an exact
match for a project like mine.

The initial experiments looks promising, it is easy to build the utilities from
[sbase](https://core.suckless.org/sbase/) project and wrap them into a
compatible interface providing more complete unix experience.

There are a plenty of TODOs and code lives inside my [private
repo](https://codeberg.org/vyskocilm/wasi-experiments), but the approach looks
doable. The end goal is to have a secure - thanks to web assembly and sane
defaults - yet pretty extensible shell scripting engine, which won't need a
single native file installed on a target system.

# Credits

The most interesting thing on this project is that how little I actually did
myself and how much I was able to "steal", adapt and integrate.

 1. Unix creators for the idea of unix tools and a pipe and scripting
 1. Go creators for an easy, but powerful language
 1. The authors for all the unix utilities, which I can compile to wasm and use in this context
 1. The author of `mvdan.cc/sh`
 1. The authors of `https://wazero.io` - they promptly fixes a blocker bug after 1.0 release

## Cover

Cover image created from

 * https://tremaineeto.medium.com/who-first-created-and-drew-the-golang-gopher-bc25149acdc2
 * https://www.deviantart.com/dollarakshay/art/Unix-Terminal-Logo-735161243
