---
title: 'A tale of my first Go patch'
category: go
comments: true
tags: [go]
---

For the last year I have been using the [Go](https://golang.org/)
programming language, working on the implementation of the
[taskcluster-worker](https://github.com/taskcluster/taskcluster-worker)
project.  If you want to learn more about `taskcluster-worker`, you can
read [this post](https://walac.github.io/taskcluster-worker-macosx-engine/).

In the `taskcluster-worker` task payload, there is a field called `command`,
that specifies the command the task must run. Through a configuration file,
you can specify either the spawned command should run as the current user
or to run the command with a newly created user. The pseudo code bellow
illustrates how this works:

```go
var usr *User
if createUser {
  usr = CreateUser()
} else {
  usr = CurrentUser()
}

RunCommand(command, usr)

if createUser {
  DeleteUser(usr)
}
```

Our first production use for `taskcluster-worker` it to run Firefox automated
tests inside the OSX environment. We began configuring `taskcluster-worker` to
run each task with a new user to provide some kind of task isolation, but it
didn't work well. Some tests, as you might wonder, require we run the browser
UI inside a desktop session. To allow this, the taskcluster-worker daemon runs
as a
[Launch Agent](https://developer.apple.com/library/content/documentation/MacOSX/Conceptual/BPSystemStartup/Chapters/CreatingLaunchdJobs.html).
The problem is that the test itself runs as a different user, under the desktop
session owned by the taskcluster-worker user. As the user that runs Firefox is
different from the user owning the desktop session, the tests weren't allowed
to execute several operations, like clipboard copy/paste. Because of this that
we introduced the option to run the test as the current user.

When implementing the option to run tasks as the current user, I found
a subtle problem with Go process creation package, described next.

# The problem

The [`os/exec`](https://golang.org/pkg/os/exec/) package allows you to spawn
new processes. For example, the following code will spawn the
[`echo`](https://linux.die.net/man/1/echo) command and
print the "hello-world" string:

```go
package main

import (
  "fmt"
  "os"
  "os/exec"
)

func main() {
  cmd := exec.Command("echo", "hello-world")
  cmd.Stdout = os.Stdout
  cmd.Stderr = os.Stderr
  if err := cmd.Run(); err != nil {
    fmt.Println(err)
  }
}
```

Here we use the [`Command`](https://golang.org/pkg/os/exec/#Command) function to
create a [`Cmd`](https://golang.org/pkg/os/exec/#Cmd) structure. We then connect
the `stdout` and `stdin` channels of the child process to the parent process
and spawn the new process. It runs as expected:

```shell
$ go run echo.go
hello-world
```

So far so good. You can also set the user in which the process will run.
Let's demonstrate it by expliciting setting the current user as the user
account of the child process:

```go
package main

import (
  "fmt"
  "os"
  "os/exec"
  "os/user"
  "strconv"
  "syscall"
)

func main() {
  usr, err := user.Current()
  if err != nil {
    panic(err)
  }

  uid, err := strconv.Atoi(usr.Uid)
  if err != nil {
    panic(err)
  }

  gid, err := strconv.Atoi(usr.Gid)
  if err != nil {
    panic(err)
  }

  cmd := exec.Command("echo", "hello-world")
  cmd.Stdout = os.Stdout
  cmd.Stderr = os.Stderr
  cmd.SysProcAttr = &syscall.SysProcAttr{
    Credential: &syscall.Credential{
      Uid: uint32(uid),
      Gid: uint32(gid),
    },
  }
  if err = cmd.Run(); err != nil {
    fmt.Println(err)
  }
}
```

The `exec.Cmd` structure accepts a system specific
[`SysProcAttr`](https://golang.org/pkg/syscall/#SysProcAttr)
structure pointer, which has a
[`Credential`](https://golang.org/pkg/syscall/#Credential) field used to
set the user and group ids of the child process. This is a no-op
operation, as by default the child process runs with the same user
and group ids of its parent.  But when we run it:

```shell
$ go run echo.go
fork/exec /bin/echo: operation not permitted
```

![What?](/images/golangpatch/what.jpeg)

Wait! Can't we run a process with our own account??? This is non-sense.
Let's investigate the root cause of this weird behavior. A system
call must be returning `EPERM` and causing the whole thing to collapse.
We can use [`dtrace`](https://dtrace.org/blogs/about/) to discover
exactly what system call is failing:

```shell
$ sudo dtrace -n 'syscall:::return/execname == "echo" && arg0 < 0 && errno == EPERM/{}'
dtrace: description 'syscall:::return' matched 522 probes
CPU     ID                    FUNCTION:NAME
  0    338                 setgroups:return
```

The above dtrace script logs all system calls executing in the context of our
process that return `EPERM` error. Explaining the details of `dtrace` is beyond
the scope of this post, please refer to the [dtrace website](https://dtrace.org)
to find the documentation about it.

The output shows that the [`setgroups`](https://linux.die.net/man/2/setgroups)
system call is the guilty one.  If you look at the system call
[manual page](https://linux.die.net/man/2/setgroups) you see that it requires
admin privileges to succeed. But, why is Go runtime calling it? Below you
see the relevant piece of code inside Go
[syscall package](https://golang.org/pkg/syscall/) (as I am running a
macOS, this is the code specific for BSD kernels, Linux and Solaris
specific source codes are similar):

```go
// User and groups
if cred := sys.Credential; cred != nil {
	ngroups := uintptr(len(cred.Groups))
	groups := uintptr(0)
	if ngroups > 0 {
		groups = uintptr(unsafe.Pointer(&cred.Groups[0]))
	}
	_, _, err1 = RawSyscall(SYS_SETGROUPS, ngroups, groups, 0)
	if err1 != 0 {
		goto childerror
	}
	_, _, err1 = RawSyscall(SYS_SETGID, uintptr(cred.Gid), 0, 0)
	if err1 != 0 {
		goto childerror
	}
	_, _, err1 = RawSyscall(SYS_SETUID, uintptr(cred.Uid), 0, 0)
	if err1 != 0 {
		goto childerror
	}
}
```

The reason is that the `Credential` struct has a `Groups` field representing an
array of supplementary group ids to pass to `setgroups`, and when
`Credential != nil`, `setgroups` gets always called, even if didn't set the
`Groups` property. And that's where the problem lies,  you can't set
process user and group id without issuing a call to `setgroups`. The solution
is to find a way to say "don't bother with supplementary groups".

# Attempt #1: don't call `setgroups` if `len(Groups) == 0`

My first solution was the obvious one, don't call `setgroups` if
the length of `Groups` is zero. It works fine, except for one detail:
we now can't clear the supplementary groups of the child process, which
is, above all, a security problem.

# Attempt #2: don't call `setgroups` if `Groups == nil`

After the security concern, my idea was to distinguish a empty group
(`[]uint32{}`) from a `nil` group. If `Groups == nil`, we skip
`setgroups`, otherwise we call it. This removes the previous issue,
but introduces another: it subtly breaks backward compatibility.
Previously, if you left `Groups` intact, the child process would
always try to clear supplementary groups, now it doesn't.

# Attempt #3: call `setgroups` only if `Groups` and `getgroups` mismatch

One of the code reviewers suggested to call
[`getgroups`](https://linux.die.net/man/2/getgroups) to get the
current set of supplementary groups and only call `setgroups` if the
current and new groups set mismatch. Implementing this was trickier than
you might think. Here is a fairly straightforward implementation:

```go
func groupsMismatch(g1, g2 []uint32) bool {
	if len(g1) != len(g2) {
		return true
	}
	table := map[uint32]struct{}
	for _, g := range g1 {
		table[g] = struct{}{}
	}
	for _, g := range g2 {
		if _, ok := table[g]; !ok {
			return true
		}
	}

	return false
}

// ...

// User and groups
if cred := sys.Credential; cred != nil {
	ngroups := uintptr(len(cred.Groups))
	groups := uintptr(0)
	if ngroups > 0 {
		groups = uintptr(unsafe.Pointer(&cred.Groups[0]))
	}
	var groupsize uintptr
	groupsize, _, err1 = RawSyscall(SYS_GETGROUPS, 0, 0, 0)
	if err1 != 0 {
		goto childerror
	}
	cgroups := make([]uint32, groupsize)
	groupsize, _, err1 = RawSyscall(SYS_GETGROUPS, groupsize, uintptr(unsafe.Pointer(&cgroups[0])), 0)
	if err1 != 0 {
		goto childerror
	}
	if groupsMismatch(cgroups, cred.Groups) {
		_, _, err1 = RawSyscall(SYS_SETGROUPS, ngroups, groups, 0)
		if err1 != 0 {
			goto childerror
		}
	}
	_, _, err1 = RawSyscall(SYS_SETGID, uintptr(cred.Gid), 0, 0)
	if err1 != 0 {
		goto childerror
	}
	_, _, err1 = RawSyscall(SYS_SETUID, uintptr(cred.Uid), 0, 0)
	if err1 != 0 {
		goto childerror
	}
}
```

This is a classic linear time array mismatch algorithm implemented
in the `groupsMismatch` function. By definition, the array of groups are
unique, so we don't have to worry about duplicated values. When the current
supplementary group set is different from the new one, we call `setgroups`
to apply the new group set. When we run it:

```shell
wcosta@Wanders-MBP:~ $ work/golang/bin/go run test.go
fatal error: stack growth after fork

runtime stack:
runtime.throw(0x10d65ca, 0x17)
	/Users/wcosta/work/golang/src/runtime/panic.go:596 +0x95
runtime.newstack(0x0)
	/Users/wcosta/work/golang/src/runtime/stack.go:965 +0xcea
runtime.morestack()
	/Users/wcosta/work/golang/src/runtime/asm_amd64.s:398 +0x86

goroutine 1 [running]:
runtime.makeslice(0x10b1940, 0xe, 0xe, 0x0, 0xe, 0x0)
	/Users/wcosta/work/golang/src/runtime/slice.go:39 +0xf6 fp=0xc4200499d8 sp=0xc4200499d0
syscall.forkAndExecInChild(0xc4200103d0, 0xc42000c380, 0x3, 0x3, 0xc420082000, 0x24, 0x24, 0x0, 0x0, 0xc420049cd8, ...)
	/Users/wcosta/work/golang/src/syscall/exec_bsd.go:171 +0x707 fp=0xc420049ad0 sp=0xc4200499d8
syscall.forkExec(0xc420010360, 0x9, 0xc42000c260, 0x2, 0x2, 0xc420049cd8, 0x20, 0xc42000c360, 0xc42001c000)
	/Users/wcosta/work/golang/src/syscall/exec_unix.go:193 +0x368 fp=0xc420049bf0 sp=0xc420049ad0
syscall.StartProcess(0xc420010360, 0x9, 0xc42000c260, 0x2, 0x2, 0xc420049cd8, 0x2, 0x4, 0xc420016180, 0xc420049ca8)
	/Users/wcosta/work/golang/src/syscall/exec_unix.go:240 +0x64 fp=0xc420049c48 sp=0xc420049bf0
os.startProcess(0xc420010360, 0x9, 0xc42000c260, 0x2, 0x2, 0xc420049e80, 0xc420012480, 0x23, 0x23)
	/Users/wcosta/work/golang/src/os/exec_posix.go:45 +0x1fb fp=0xc420049d30 sp=0xc420049c48
os.StartProcess(0xc420010360, 0x9, 0xc42000c260, 0x2, 0x2, 0xc420049e80, 0x0, 0x0, 0xc42006e750)
	/Users/wcosta/work/golang/src/os/exec.go:94 +0x64 fp=0xc420049d88 sp=0xc420049d30
os/exec.(*Cmd).Start(0xc420080000, 0xc42000c201, 0xc42000c300)
	/Users/wcosta/work/golang/src/os/exec/exec.go:359 +0x502 fp=0xc420049ed8 sp=0xc420049d88
os/exec.(*Cmd).Run(0xc420080000, 0xc42000c300, 0xc420049f68)
	/Users/wcosta/work/golang/src/os/exec/exec.go:277 +0x2b fp=0xc420049f00 sp=0xc420049ed8
main.main()
	/Users/wcosta/test.go:36 +0x21d fp=0xc420049f88 sp=0xc420049f00
runtime.main()
	/Users/wcosta/work/golang/src/runtime/proc.go:185 +0x20a fp=0xc420049fe0 sp=0xc420049f88
runtime.goexit()
	/Users/wcosta/work/golang/src/runtime/asm_amd64.s:2197 +0x1 fp=0xc420049fe8 sp=0xc420049fe0

goroutine 17 [syscall, locked to thread]:
runtime.goexit()
	/Users/wcosta/work/golang/src/runtime/asm_amd64.s:2197 +0x1
exit status 2
```

![Say What?](/images/golangpatch/saywhat.jpeg)

Before diving into the error, let's talk a bit about the history of the
[`fork`](https://linux.die.net/man/2/fork) system call. When `fork` was
created, there was no concept of
[threads](https://en.wikipedia.org/wiki/Thread_\(computing\)),
forking the process was the way multiprocessing was achieved. As you probably
know, when `fork` is called, a child process is spawned which is identical to
the parent process, like memory values and files opened. But what if a
multithreaded process calls `fork`? The answer is that only the thread that
invoked the system call is copied to the child process, but the state of the global
memory remains the same, and that includes the state of mutexes and semaphores
held in another thread. That means you can execute a very restrict set of
operations in the child process. Allocating memory from the heap is dangerous,
as the heap is often protected by a mutex, and another thread could be in the
middle of a heap allocation when the process was forked. Trying to allocate
memory from the heap in this circumstance will deadlock the child process.

Another thing you need to know is that the goroutine stack is resizable, and
they are quite small when goroutines are created. If you are interested in
the details of goroutines stack management, I recommend this
[blog post](https://blog.cloudflare.com/how-stacks-are-handled-in-go/).

Ok, with this background I can now explain what happened. The stack
trace points to the line `cgroups := make([]uint32, groupsize)`,
which tries to allocate a new array in the stack, the stack limit
is reached and then the go runtime tries to allocate more memory
to the stack, and the go runtime
[detects it is a forked process](https://github.com/golang/go/blob/master/src/runtime/stack.go#L932-L934)
and panics. The conclusion is that even a function call may cause
a runtime panic.

In the end, to implement this patch I had to declare a global
`[]uint32` large enough to store the groups returned by `getgroups`
(in environments I looked at, it doesn't need to be more than 16
entries large) and implemented an \\(O(n^2)\\) but constant
space matching algorithm.

It turns out it actually didn't reach its goal, because often a new
child process inherits the parent's supplementary groups. Therefore, even
so the caller doesn't care about supplementary groups, `setgroups`
can be called anyway.

# Accepted solution: define a new member in the `Credential` struct

After all previous attempts, I admitted defeat and added a `NoSetGroups`
member to the `Credential` struct so that when it is `true`,
it skips supplementary groups management.
In a ideal world it would be called `SetGroups` with the logic inverted,
but I had to do so in order to preserve backward compatibility.

If you are curious about the final patch, just look at the
[git commit](https://github.com/golang/go/commit/79f6a5c7bd684f2e6007ee505b522440beb86bf0).

By the end of the day, it is just astonishing how a subtle issue with
a so simple patch yielded so much trouble. Finding the
perfect solution is often almost impossible, but even finding a good
one isn't that simple.
