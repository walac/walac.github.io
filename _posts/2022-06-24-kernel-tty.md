---
title: 'Improving the Kernel TTY throughput'
category: kernel
comments: true
tags: [linux, kernel, tty]
---

It is incredible how we can find room for improvement even in the eldest
and battle-tested codebases out there. This post is about one of these
cases.

I was investigating a [soft lockup](https://is.gd/AasYhu) bug report which
only happened in an `HP Proliant DL380 Gen9`. The trigger for the issue is
when we try to load the [scsi_debug](https://is.gd/pBmMXP) with a few hundred
virtual devices:

```sh
$ modprobe scsi_debug virtual_gb=1 add_host=2 num_tgts=600
Message from syslogd@storageqe-25 at Jul 15 10:29:35 ...
 kernel:watchdog: BUG: soft lockup - CPU#33 stuck for 22s! [modprobe:35056]

Message from syslogd@storageqe-25 at Jul 15 10:30:23 ...
 kernel:watchdog: BUG: soft lockup - CPU#25 stuck for 22s! [migration/25:140]

Message from syslogd@storageqe-25 at Jul 15 10:30:23 ...
 kernel:watchdog: BUG: soft lockup - CPU#18 stuck for 23s! [modprobe:35056]
```

And below, we show the `dmesg` output:

```sh
watchdog: BUG: soft lockup - CPU#33 stuck for 22s! [modprobe:35056]
rcu: 	14-...!: (1 GPs behind) idle=24a/1/0x4000000000000002 softirq=12191/12192 fqs=5030
Modules linked in:
rcu: 	33-...!: (20094 ticks this GP) idle=f9e/1/0x4000000000000000 softirq=8891/8891 fqs=5031
 scsi_debug
	(detected by 14, t=80246 jiffies, g=346621, q=270790)
 loop
NMI backtrace for cpu 14
 dm_service_time
CPU: 14 PID: 0 Comm: swapper/14 Tainted: G               X --------- ---
 dm_multipath
Hardware name: HP ProLiant DL380 Gen9/ProLiant DL380 Gen9, BIOS P89 02/17/2017
 rfkill
Call Trace:
 intel_rapl_msr
 <IRQ>
 intel_rapl_common sb_edac
 dump_stack+0x64/0x7c
...
...
CPU: 18 PID: 35056 Comm: modprobe Tainted: G             L X --------- ---
Hardware name: HP ProLiant DL380 Gen9/ProLiant DL380 Gen9, BIOS P89 02/17/2017
RIP: 0010:number+0x21b/0x340
Code: 4c 8d 44 28 01 48 39 c3 76 03 c6 00 20 48 ...
RSP: 0018:ffffb03ec3650b98 EFLAGS: 00000046
RAX: 0000000000000000 RBX: ffffb03f43650cff RCX: 0000000000000038
RDX: ffffb03ec3650baf RSI: 0000000000000001 RDI: 0000000000000000
RBP: 00000000fffffffd R08: ffffb03ec3650d06 R09: 0000000000000004
R10: ffffb03ec3650c98 R11: ffffb03ec3650d06 R12: 0000000000000020
R13: 0000000000000000 R14: 0000000000004a0e R15: 0000000000000a00
FS:  00007f519cfb1740(0000) GS:ffff9b096fa00000(0000) knlGS:0000000000000000
CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033
CR2: 000055a8b65190b8 CR3: 0000000550bec004 CR4: 00000000001706e0
Call Trace:
 <IRQ>
 vsnprintf+0x363/0x560
 sprintf+0x56/0x70
 info_print_prefix+0x7b/0xd0
 record_print_text+0x52/0x150
 console_unlock+0x144/0x330
 vprintk_emit+0x14d/0x230
 printk+0x58/0x6f
 print_modules+0x62/0xaf
 watchdog_timer_fn+0x190/0x200
 ? lockup_detector_update_enable+0x50/0x50
 __hrtimer_run_queues+0x12a/0x270
 hrtimer_interrupt+0x110/0x2c0
 __sysvec_apic_timer_interrupt+0x5f/0xd0
 sysvec_apic_timer_interrupt+0x6d/0x90
 </IRQ>
```

Notice `printk()` and (in special) `console_unlock()` in the stack trace.
This is not a coincidence. It happened every single time.

I found out that `scsi_debug` sends a lot of information to the console output,
causing the kernel to spend most of its time flushing data to the serial TTY.
`vprintk_emit()` disables preemption before calling
`console_unlock()`<sup>[1](#ft1)</sup>. Because the preemption is disabled
and the serial TTY driver holds a `spinlock` while writing to the port, the
[RCU stall detector](https://docs.kernel.org/RCU/stallwarn.html) reports a stall.
A slow serial console causing an RCU stall is well known and
[documented](https://docs.kernel.org/RCU/stallwarn.html):

```
"Booting Linux using a console connection that is too slow to keep up with the boot-time
console-message rate. For example, a 115Kbaud serial console can be way too slow to keep
up with boot-time message rates and will frequently result in RCU CPU stall warning
messages. Especially if you have added debug printk()s."
```

In the `hrtimer` interrupt handler, the kernel calls the `check_rcu_stall()` function,
which will call `printk()` to dump the stall information. At this time, the interrupt handler
becomes the console owner and keeps flushing the remaining data to the serial
console<sup>[1](#ft2)</sup>. The CPU will then get stuck in the interrupt routine, and
triggers a soft lockup.

I suspected there was something odd with the serial port. It is configured with
115200 bps, 8 bits per data. Considering one start bit and one stop bit, it gives
an expected throughput of approximately 11520 bytes per second. Let's use
[bpftrace](https://github.com/iovisor/bpftrace) while loading the `scsi_debug`
driver to verify if it is indeed the case:

```sh
$ bpftrace -e '
BEGIN {
	@ = 0;
}

kfunc:serial8250_console_putchar {
	@++;
}

interval:s:1 {
	printf("%d B/s\n", @);
	@ = 0;
}'
Attaching 3 probes...
2522 B/s
2517 B/s
2518 B/s
2507 B/s
2510 B/s
2414 B/s
...
```

`serial8250_console_putchar()` is the function that effectively writes
a character to the serial port. We count how many times we call it per
second.

Maybe the tracing machinery is causing an overhead, jeopardizing
the serial performance? Let's try another approach:

```sh
$ bpftrace -e '
BEGIN {
	@ = (uint32) 0;
}

kfunc:uart_console_write {
	@ += args->count;
}

interval:s:1 {
	printf("%u B/s\n", @);
	@ = 0;
}'
Attaching 3 probes...
1838 B/s
2460 B/s
2536 B/s
2415 B/s
2461 B/s
2247 B/s
2521 B/s
2441 B/s
2409 B/s
2378 B/s
2353 B/s
2398 B/s
2376 B/s
2414 B/s
...
```

Which has less overhead, as we don't call the probe function for each
character sent. Again the throughput is not what we would expect.

The scripts above report the overall system throughput. The next step
is determining if the suboptimal performance is due to system overhead
or the serial TTY code itself. The following `bpftrace` script measures
more precisely the throughput of the serial console driver:

```sh
$ bpftrace -e '
kfunc:uart_console_write {
  @c[cpu] = args->count;
  @t[cpu] = nsecs;
}

kretfunc:uart_console_write {
  $elapsed = nsecs - @t[cpu];
  $rate = @c[cpu] * 1000000000 / $elapsed;
  printf("%ld B/s\n", $rate);
}'
Attaching 2 probes...
2206 B/s
2502 B/s
2585 B/s
2553 B/s
2559 B/s
2537 B/s
2555 B/s
2557 B/s
2554 B/s
2557 B/s
2561 B/s
2542 B/s
2534 B/s
2255 B/s
2555 B/s
2482 B/s
...
```

No luck. To facilitate my tests, I created a
[small driver + app](https://github.com/walac/serial-console-test)
that calls the console driver the same way `printk()` does. After
loading the `serco` driver, use `sertest -n <N> <driver-path>` to
send `N` bytes to the console. Let's test it:

```sh
$ time ./sertest -n 2500 /dev/serco

real    0m0.997s
user    0m0.000s
sys     0m0.997s
```

Ok, 2500 bytes sent in approximately 1 second. That matches our previous
results. With the help of the [function tracer](https://trace-cmd.org/),
we can see what is going on in more detail:

```sh
$ trace-cmd record -p function_graph -g serial8250_console_write \
   ./sertest -n 1 /dev/serco

$ trace-cmd report

            |  serial8250_console_write() {
 0.384 us   |    _raw_spin_lock_irqsave();
 1.836 us   |    io_serial_in();
 1.667 us   |    io_serial_out();
            |    uart_console_write() {
            |      serial8250_console_putchar() {
            |        wait_for_xmitr() {
 1.870 us   |          io_serial_in();
 2.238 us   |        }
 1.737 us   |        io_serial_out();
 4.318 us   |      }
 4.675 us   |    }
            |    wait_for_xmitr() {
 1.635 us   |      io_serial_in();
            |      __const_udelay() {
 1.125 us   |        delay_tsc();
 1.429 us   |      }
...
...
...
 1.683 us   |      io_serial_in();
            |      __const_udelay() {
 1.248 us   |        delay_tsc();
 1.486 us   |      }
 1.671 us   |      io_serial_in();
 411.342 us |    }
```

So, it takes 411 us to send just one single byte.
The function [serial8250_console_write()](https://is.gd/EfkkTZ)
is the one responsible to dispatch the TTY data to the serial port (it
calls `uart_console_write()`). It writes a character and waits for the
chip controller to process the data before sending the next. Modern
serial controllers have a FIFO of at least 16 bytes. Maybe we can
exploit that and improve the situation. Well, that is what
[I did](https://is.gd/pk2Yje). Let's how things go now:

```sh
$ time ./sertest -n 2500 /dev/serco

real    0m0.112s
user    0m0.056s
sys     0m0.055s
```

Wow! Much better! Now, if we send 11500 bytes, it should take approximately one second:

```sh
$ time ./sertest -n 11500 /dev/serco

real    0m3.142s
user    0m0.000s
sys     0m3.142s
```

![confused](/images/confused.jpg)

Let's run `trace-cmd` one more time, but I will only show the piece of
the trace output that interests us:


```sh
sertest-21649 [008]  2360.845784: funcgraph_exit:         1.942 us   |        }
sertest-21649 [008]  2360.845784: funcgraph_entry:        1.710 us   |        io_serial_in();
sertest-21649 [008]  2360.845786: funcgraph_entry:                   |        __const_udelay() {
sertest-21649 [008]  2360.845786: funcgraph_entry:                   |          delay_tsc() {
sertest-21649 [008]  2360.845786: funcgraph_entry:        0.128 us   |            preempt_count_add();
sertest-21649 [008]  2360.845787: funcgraph_entry:        0.130 us   |            preempt_count_sub();
sertest-21649 [008]  2360.845787: funcgraph_entry:        0.130 us   |            preempt_count_add();
sertest-21649 [008]  2360.845787: funcgraph_entry:        0.131 us   |            preempt_count_sub();
sertest-21649 [008]  2360.845787: funcgraph_entry:        0.131 us   |            preempt_count_add();
sertest-21649 [008]  2360.845788: funcgraph_entry:        0.130 us   |            preempt_count_sub();
sertest-21649 [008]  2360.845788: funcgraph_exit:         1.666 us   |          }
sertest-21649 [008]  2360.845788: funcgraph_exit:         1.915 us   |        }
sertest-21649 [008]  2360.845788: funcgraph_entry:      + 12.787 us  |        io_serial_in();
sertest-21649 [008]  2360.845801: funcgraph_entry:                   |        __const_udelay() {
sertest-21649 [008]  2360.845801: funcgraph_entry:                   |          delay_tsc() {
sertest-21649 [008]  2360.845801: funcgraph_entry:        0.152 us   |            preempt_count_add();
sertest-21649 [008]  2360.845802: funcgraph_entry:        0.151 us   |            preempt_count_sub();
sertest-21649 [008]  2360.845802: funcgraph_entry:        0.151 us   |            preempt_count_add();
sertest-21649 [008]  2360.845802: funcgraph_entry:        0.143 us   |            preempt_count_sub();
sertest-21649 [008]  2360.845803: funcgraph_entry:        0.145 us   |            preempt_count_add();
sertest-21649 [008]  2360.845803: funcgraph_entry:        0.147 us   |            preempt_count_sub();
sertest-21649 [008]  2360.845803: funcgraph_exit:         1.909 us   |          }
sertest-21649 [008]  2360.845803: funcgraph_exit:         2.204 us   |        }
sertest-21649 [008]  2360.845803: funcgraph_entry:        1.743 us   |        io_serial_in();
sertest-21649 [008]  2360.845805: funcgraph_entry:                   |        __const_udelay() {
```

`io_serial_in()` usually executes in less than 2 us, but
sometimes something odd happens, and the controller takes more than 10
us to reply. That is the reason we can't get the expected speed of 115200 bps
for such an amount of data.

We see a 25% throughput boost when working with the original
`modprobe sci_debug` scenario. Even though we could not reach
maximum performance, we improved things considerably.


<a name="ft1">1</a>: `console_unlock()` will flush the `printk()` ring
buffer before unlocking the console.

<a name="ft2">1</a>: This behavior is vital to ensure essential
information flushes from the ring buffer in the case of fatal errors.
