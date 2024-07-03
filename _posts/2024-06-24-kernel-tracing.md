---
title: 'Debugging the Linux Kernel through tracing'
category: kernel
comments: true
tags: [kernel, trace]
---

Disclaimer: this is a mental note in the form of a blog
post for future references. Most of it is a summary
of a debug session to solve
[this issue](https://lore.kernel.org/all/171983202713.2215.17043038912457274824.tip-bot2@tip-bot2/).

Kernel is one of hardest piece of software to debug. Although it has some
limited support for a
[classical debug session](https://www.kernel.org/doc/html/latest/dev-tools/gdb-kernel-debugging.html),
it is often impractical. Not surprisingly, over the years, kernel hackers
developed a lot of tools to aid the art of debugging the Linux kernel.
Tracing tool are among the most useful techniques.

Tracing is a technique used to monitor the live kernel in real-time, involving 
a logging mechanism to record kernel activity. In this post, we will provide 
an overview of two kernel features that enable tracing: [tracepoints](https://
docs.kernel.org/trace/tracepoints.html) and [kprobes](https://docs.kernel.org/
trace/kprobes.html). We'll interface with these through the [trace-cmd](https://
www.trace-cmd.org/) command, though alternatives like [bpftrace](http://walac.github.io/
kernel-tty/) can also be used.

Tracepoints
-----------

A tracepoint is a predefined static hook to call a function (probe) that
allow tracing tools to log system activity. Tracepoints are listed inside
the category *events* in the tracing filesystem, under `sys/kernel/tracing/events/`.
You can also list them using the `trace-cmd list` command:

```sh
$ trace-cmd list -e | head
qrtr:qrtr_ns_service_announce_new
qrtr:qrtr_ns_service_announce_del
qrtr:qrtr_ns_server_add
qrtr:qrtr_ns_message
sunrpc:rpc_xdr_sendto
sunrpc:rpc_xdr_recvfrom
sunrpc:rpc_xdr_reply_pages
sunrpc:rpc_clnt_free
sunrpc:rpc_clnt_killall
sunrpc:rpc_clnt_shutdown
```

Let's use `trace-cmd` to trace `timer:hrtimer_expire_entry`, which triggers every time
an [hrtimer](https://docs.kernel.org/timers/hrtimers.html) callback executes:

```
trace-cmd record -e timer:hrtimer_expire_entry sleep 0.01
CPU0 data recorded at offset=0x2a5000
    197 bytes in size (4096 uncompressed)
CPU1 data recorded at offset=0x2a6000
    212 bytes in size (4096 uncompressed)
CPU2 data recorded at offset=0x2a7000
    132 bytes in size (4096 uncompressed)
CPU3 data recorded at offset=0x2a8000
    163 bytes in size (4096 uncompressed)
CPU4 data recorded at offset=0x2a9000
    211 bytes in size (4096 uncompressed)
CPU5 data recorded at offset=0x2aa000
    209 bytes in size (4096 uncompressed)
CPU6 data recorded at offset=0x2ab000
    189 bytes in size (4096 uncompressed)
CPU7 data recorded at offset=0x2ac000
    189 bytes in size (4096 uncompressed)
```

`trace-cmd record` captures events while the given command runs. In this
example, we ran sleep 0.01 to capture events for just 10ms, as hrtimer
generates many entries.

The logs are saved in a binary file named *trace.dat*. To convert these logs
to a text format, use `trace-cmd report`:

```sh
trace-cmd report | head 
cpus=8
          <idle>-0     [002] 21403.059823: hrtimer_expire_entry: hrtimer=0xffffacffe3163cf8 now=21402099965347 function=hrtimer_wakeup/0x0
          <idle>-0     [007] 21403.059855: hrtimer_expire_entry: hrtimer=0xffffacffe0fcfb20 now=21402099998095 function=hrtimer_wakeup/0x0
           Timer-34807 [002] 21403.059858: hrtimer_expire_entry: hrtimer=0xffff90d2cc326158 now=21402100001643 function=tick_nohz_handler/0x0
 IPDL Background-4695  [003] 21403.059859: hrtimer_expire_entry: hrtimer=0xffff90d2cc3a6158 now=21402100001860 function=tick_nohz_handler/0x0
           sleep-45035 [001] 21403.059859: hrtimer_expire_entry: hrtimer=0xffff90d2cc2a6158 now=21402100001874 function=tick_nohz_handler/0x0
 Isolated Web Co-29032 [004] 21403.059859: hrtimer_expire_entry: hrtimer=0xffff90d2cc426158 now=21402100002003 function=tick_nohz_handler/0x0
          <idle>-0     [006] 21403.059861: hrtimer_expire_entry: hrtimer=0xffff90d2cc526158 now=21402100004124 function=tick_nohz_handler/0x0
          <idle>-0     [005] 21403.059863: hrtimer_expire_entry: hrtimer=0xffff90d2cc4a6158 now=21402100006647 function=tick_nohz_handler/0x0
    dav1d-worker-44563 [000] 21403.059864: hrtimer_expire_entry: hrtimer=0xffff90d2cc226158 now=21402100006686 function=tick_nohz_handler/0x0
```

The logs are dominated by the `tick_nohz_handler` function,
but we can filter it out using the `-f` option:

```sh
$ grep -w tick_nohz_handler /proc/kallsyms | cut -f1 -d' '
ffffffff86232650
$ trace-cmd record -e timer:hrtimer_expire_entry -f 'function != 0xffffffff86232650' sleep 0.01
```

The function parameter is a machine-sized integer containing the address of the
callback function. We must translate the function name to its address
using the */proc/kallsyms* file.

```sh
$ trace-cmd report
cpus=8
          <idle>-0     [006] 21953.174222: hrtimer_expire_entry: hrtimer=0xffffacffc6b43d10 now=21952218640837 function=hrtimer_wakeup/0x0
          <idle>-0     [001] 21953.174606: hrtimer_expire_entry: hrtimer=0xffffacffc1e17c18 now=21952219021472 function=hrtimer_wakeup/0x0
        Renderer-4729  [001] 21953.175599: hrtimer_expire_entry: hrtimer=0xffff90cf80307c98 now=21952220003877 function=intel_uncore_fw_release_timer/0x0
        Renderer-4729  [001] 21953.175602: hrtimer_expire_entry: hrtimer=0xffff90cf8191d418 now=21952220003877 function=intel_uncore_fw_release_timer/0x0
          <idle>-0     [007] 21953.177506: hrtimer_expire_entry: hrtimer=0xffffacffe3163a58 now=21952221925981 function=hrtimer_wakeup/0x0
        Renderer-4729  [001] 21953.179598: hrtimer_expire_entry: hrtimer=0xffff90cf80307c98 now=21952224002950 function=intel_uncore_fw_release_timer/0x0
    dav1d-worker-45360 [002] 21953.182593: hrtimer_expire_entry: hrtimer=0xffffacffe0fcfb88 now=21952227002482 function=hrtimer_wakeup/0x0
          <idle>-0     [001] 21953.183062: hrtimer_expire_entry: hrtimer=0xffffacffcd667c58 now=21952227481587 function=hrtimer_wakeup/0x0
    dav1d-worker-45358 [007] 21953.184810: hrtimer_expire_entry: hrtimer=0xffffacffe062fc10 now=21952229231336 function=hrtimer_wakeup/0x0
           sleep-45667 [000] 21953.184852: hrtimer_expire_entry: hrtimer=0xffffacffc2ae7b10 now=21952229272526 function=hrtimer_wakeup/0x0
           sleep-45667 [000] 21953.184944: hrtimer_expire_entry: hrtimer=0xffffacffc143f810 now=21952229364853 function=hrtimer_wakeup/0x0
```

To view the fields available for filtering and their types, print the event
format:

```sh
$ trace-cmd list -e timer:hrtimer_expire_entry -F
system: timer
name: hrtimer_expire_entry
ID: 388
format:
        field:unsigned short common_type;       offset:0;       size:2; signed:0;
        field:unsigned char common_flags;       offset:2;       size:1; signed:0;
        field:unsigned char common_preempt_count;       offset:3;       size:1; signed:0;
        field:int common_pid;   offset:4;       size:4; signed:1;

        field:void * hrtimer;   offset:8;       size:8; signed:0;
        field:s64 now;  offset:16;      size:8; signed:1;
        field:void * function;  offset:24;      size:8; signed:0;
```

The fields prefixed with `common_` are common to all events from the same system.
Say, you want to trace all timer events but only for a specific PID:

```sh
$ trace-cmd record -e 'timer:*' -f 'common_pid == 1' sleep 1
```

To get the stack trace for each event, use `-T`:

```sh
$ trace-cmd record -e timer:hrtimer_expire_entry -f 'function != 0xffffffff86232650' -T sleep 0.01
$ trace-cmd report
cpus=8
          <idle>-0     [002] 24703.375192: hrtimer_expire_entry: hrtimer=0xffffacffc616fdc0 now=24702442127437 function=hrtimer_wakeup/0x0
          <idle>-0     [002] 24703.375200: kernel_stack:         <stack trace >
=> trace_event_raw_event_hrtimer_expire_entry (ffffffff86218921)
=> __hrtimer_run_queues (ffffffff8621c491)
=> hrtimer_interrupt (ffffffff8621d1aa)
=> __sysvec_apic_timer_interrupt (ffffffff8608db25)
=> sysvec_apic_timer_interrupt (ffffffff8715a20c)
=> asm_sysvec_apic_timer_interrupt (ffffffff8720160a)
=> cpuidle_enter_state (ffffffff8715c2f9)
=> cpuidle_enter (ffffffff86de023d)
=> do_idle (ffffffff8619ef17)
=> cpu_startup_entry (ffffffff8619f179)
=> start_secondary (ffffffff8608bacb)
=> common_startup_64 (ffffffff8603ddad)
```

Static probes are invaluable for understanding system behavior, but they
require kernel developers to make them available. What if you need to trace
a part of the kernel without tracepoints?

Kernel probes
-------------

Kernel probes, also known as kprobes, allow you to dynamically create
tracepoints to log information. These probes can be attached to almost any
kernel address, including functions and their entry/exit points.

To create a probe, you can write directly to the tracing filesystem, but
I rather prefer to use
[perf-probe](https://man7.org/linux/man-pages/man1/perf-probe.1.html)<sup>[1](#ft1)</sup>.

A few weeks ago I was debugging a
[memory leak](https://lore.kernel.org/all/171983202713.2215.17043038912457274824.tip-bot2@tip-bot2/)
reported in the RT kernel. At some point I realized there was something odd between the
[deadline scheduler](https://docs.kernel.org/scheduler/sched-deadline.html) and the
hrtimer (now you know why I previously picked the hrtimer as a tracepoint example).
The entry point to start the timer is the function
[start_dl_timer](https://elixir.bootlin.com/linux/v6.10-rc6/source/kernel/sched/deadline.c#L1044).
Unfortunately, there is no tracepoint we can use to track its call. Let's create
one:

```sh
$ perf probe start_dl_timer
Added new event:
  probe:start_dl_timer (on start_dl_timer)

$ trace-cmd record -e probe:start_dl_timer sleep 1
```

Notice that, by default, `perf-probe` put the probe inside the "probe" group.
This is all good but doesn't give much information. We would like to know the PID,
`task_struct` reference count and the timer object to relate to the hrtimer tracepoints.
`start_dl_timer` receives a pointer to
[struct sched_dl_entity](https://elixir.bootlin.com/linux/v6.10-rc6/source/include/linux/sched.h#L598),
containing the
[timer object](https://elixir.bootlin.com/linux/v6.10-rc6/source/include/linux/sched.h#L651). `perf-probe`
allows you to dereference struct members, so adding the timer object to the trace is a piece of cake.
Firstly, we need to remove the previously defined probe:

```sh
$ perf probe --del probe:start_dl_timer
Removed event: probe:start_dl_timer
```

Let's see which lines we can probe:

```sh
$ perf probe -L start_dl_timer
      0  static int start_dl_timer(struct sched_dl_entity *dl_se)
         {
      2         struct hrtimer *timer = &dl_se->dl_timer;
                struct dl_rq *dl_rq = dl_rq_of_se(dl_se);
                struct rq *rq = rq_of_dl_rq(dl_rq);
                ktime_t now, act;
                s64 delta;
         
                lockdep_assert_rq_held(rq);
         
                /*
                 * We want the timer to fire at the deadline, but considering
                 * that it is actually coming from rq->clock and not from
                 * hrtimer's time base reading.
                 */
                act = ns_to_ktime(dl_next_period(dl_se));
     16         now = hrtimer_cb_get_time(timer);
     17         delta = ktime_to_ns(now) - rq_clock(rq);
                act = ktime_add_ns(act, delta);
         
                /*
                 * If the expiry time already passed, e.g., because the value
                 * chosen as the deadline is too small, don't even try to
                 * start the timer in the past!
                 */
     25         if (ktime_us_delta(act, now) < 0)
                        return 0;
         
                /*
                 * !enqueued will guarantee another callback; even if one is already in
                 * progress. This ensures a balanced {get,put}_task_struct().
                 *
                 * The race against __run_timer() clearing the enqueued state is
                 * harmless because we're holding task_rq()->lock, therefore the timer
                 * expiring after we've done the check will wait on its task_rq_lock()
                 * and observe our state.
                 */
     37         if (!hrtimer_is_queued(timer)) {
     38                 if (!dl_server(dl_se))
     39                         get_task_struct(dl_task_of(dl_se));
     40                 hrtimer_start(timer, act, HRTIMER_MODE_ABS_HARD);
                }
         
                return 1;
```

The `-L` option lists the source code line numbers where we can install
the probe hook<sup>[2](#ft2)</sup>.

Line 16 seems a good candidate, since the line 2 defines a local timer
variable that holds the timer address. This is useful because we don't
have a syntax in `perf-probe` to take the address of a struct member.
Just to illustrate member dereference, we will also print the timer
function. Let's confirm which variables are available to use:

```sh
$ perf probe -V start_dl_timer:16
Available variables at start_dl_timer:16
        @<start_dl_timer+67>
                ktime_t act
                struct dl_rq*   dl_rq
                struct hrtimer* timer
                struct rq*      rq
                struct sched_dl_entity* dl_se
```

The syntax to specify a function and line number is `<function>:<line>`.
The `-V` option shows the available variables.

We mentioned we want to print the PID and the `task_struct` reference count.
But where are they? `dl_se` is a member of `task_struct` (called `dl` inside
the struct), so we can get the pointer to the `task_struct` using the
[container_of](https://elixir.bootlin.com/linux/v6.10-rc6/source/include/linux/container_of.h#L18)
macro. There is only one problem: we don't have `container_of`! But we can do it manually
if we know the offset of `dl_se`
[inside task_struct](https://elixir.bootlin.com/linux/v6.10-rc6/source/include/linux/sched.h#L803).
We can do it with the help of [gdb](https://www.sourceware.org/gdb/):

```sh
 gdb /usr/lib/debug/lib/modules/$(uname -r)/vmlinux
Reading symbols from /usr/lib/debug/lib/modules/5.14.0-469.el9.x86_64/vmlinux...
(gdb) p &((struct task_struct *)0)->dl
$1 = (struct sched_dl_entity *) 0x280
(gdb) p &((struct task_struct *)0)->pid
$2 = (pid_t *) 0xa98
(gdb) p &((struct task_struct *)0)->usage
$3 = (refcount_t *) 0x30
```

This little trick works by casting 0 to `struct task_struct *` and then
taking the address of the of field we are interested. This will give the
field offset. Now we know the layout of the struct:

```c
struct task_struct {
  // ...
  refcount_t usage;             // offset 0x30
  // ...
  struct sched_dl_entity dl;    // offset 0x280
  // ...
  pid_t pid;                    // offset 0xa98
};
```

From the function argument, we know the address of the field `dl`.
The `task_struct` address is obtained by subtracting 0x280 from
the `dl_se` pointer parameter. From there, to get a field address,
just add the respective offset:

```
usage   = -0x280 + 0x030 = -0x250
pid     = -0x280 + 0xa98 = +0x818
```

We can finally define our custom probe:

```sh
$ perf probe 'start_dl_timer:16 hrtimer=timer timer->function pid=+0x818(%di) usage=-0x250(%di)'
Added new event:
  probe:start_dl_timer_L16 (on start_dl_timer:16 with hrtimer=timer function=timer->function pid=+0x818(%di) usage=-0x250(%di))
```

Notice we can give a name to the probe parameters, and also we can dereference
structures to get their members. The field name becomes the probe parameter
name if we don't give one explicitly. Also notice the usage of the `rdi` register
here<sup>[2](#ft3)</sup>. According to the
[x86_64 calling convention](https://wiki.osdev.org/System_V_ABI#x86-64),
it holds the function first argument. In our case, `dl_se`. Also, we had
to specify the type `s32` for pid and usage parameters. If you don't specify
a type, the default type is `x64`. `s32` means *signded 32 bits*. Let's use
our new custom probe:

```sh
$ trace-cmd record -e probe:start_dl_timer_L16 stress-ng --cyclic 5 --timeout 10s --minimize --quiet
$ trace-cmd report
cpus=1
 stress-ng-cycli-51628 [000] 79677.069484: start_dl_timer_L16:   (ffffffff84b6c843) hrtimer=0xffff9f9dcaa649d8 function=0xffffffff84b72150 pid=51628 usage=1
 stress-ng-cycli-51628 [000] 79677.069708: start_dl_timer_L16:   (ffffffff84b6c843) hrtimer=0xffff9f9dcaa649d8 function=0xffffffff84b72150 pid=51628 usage=1
```

We can again consult the `/proc/kallsyms` to check what function is the hrtimer callback:

```sh
$ grep ffffffff84b72150 /proc/kallsyms 
ffffffff84b72150 t dl_task_timer
```

Summary
-------

In this post, we explored the fundamentals of tracing the Linux kernel using
static tracepoints and dynamic probes. These tools offer a powerful and
flexible means of diagnosing and understanding kernel behavior. If you have
any questions or comments, feel free to leave them below.

Going deeper
------------

This post only scratches the surface. If you're eager to delve deeper,
here are some valuable resources:
* [Event Tracing](https://www.kernel.org/doc/html/v4.19/trace/events.html)
* [Kprobe-base Event Tracing](https://www.kernel.org/doc/html/v4.19/trace/kprobetrace.html)
* [perf-probe man page](https://man7.org/linux/man-pages/man1/perf-probe.1.html)
* [trace-cmd website](https://www.trace-cmd.org/)

----------------------------------------------------------------------------------------------

<a name="ft1">1</a>: perf probes are not exactly the same as kernel probes,
but in practice the differences don't matter.

<a name="ft2">1</a>: you need the kernel debug symbols installed. For
RHEL base systems, install it using `dnf install -y kerne-debuginfo`.

<a name="ft3">2</a>: the probe syntaxs does not make a difference between
`rdi` and `edi`. You just specify `di`. The same thing applies to the
other registers.
