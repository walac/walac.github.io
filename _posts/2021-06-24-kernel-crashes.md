---
title: 'A crash course on debugging kernel crashes using the crash utility'
category: kernel
comments: true
tags: [linux, kernel, crash]
---

Another day I picked a bug reporting that in of the course
[rcutorture](https://www.kernel.org/doc/html/latest/RCU/torture.html) test
[kernel-rt-debug](https://access.redhat.com/documentation/en-us/red_hat_enterprise_linux_for_real_time/8/html-single/installation_guide/index)
reported some warnings and then the kernel hang:

```
rcu_torture_writer: Testing asynchronous GPs. 
rcu_torture_writer: Testing normal GPs. 
------------[ cut here ]------------ 
DEBUG_LOCKS_WARN_ON(this_cpu_read(softirq_ctrl.cnt)) 
WARNING: CPU: 2 PID: 7996 at kernel/softirq.c:169 __local_bh_disable_ip+0x4ae/0x800 
Modules linked in: rcutorture torture rpcsec_gss_krb5 auth_rpcgss nfsv4 ....
CPU: 2 PID: 7996 Comm: rcu_torture_rea Kdump: loaded Not tainted 4.18.0-240.10.rt14.5.el8.x86_64+debug #1 
Hardware name: Hewlett-Packard ProLiant DL388eGen8, BIOS P73 06/01/2012 
RIP: 0010:__local_bh_disable_ip+0x4ae/0x800 
Code: 08 84 d2 0f 85 c7 02 00 00 8b 15 bd f7 73 07 85 d2 0f 85 37 fc ff ff 48 c7 c6 80 fd e7 8e 48 c7 ...
RSP: 0018:ffff8883d243fab0 EFLAGS: 00010286 
RAX: 0000000000000000 RBX: 0000000000000200 RCX: 0000000000000000 
RDX: 0000000000000004 RSI: 0000000000000004 RDI: 0000000000000246 
RBP: ffff8883d243fae8 R08: fffffbfff1f08f59 R09: fffffbfff1f08f58 
R10: 0000000000000000 R11: ffffffff8f847ac3 R12: ffff888104d88000 
R13: 0000000000000010 R14: ffff8883d243fb98 R15: 0000000000000064 
FS:  0000000000000000(0000) GS:ffff8883d3800000(0000) knlGS:0000000000000000 
CS:  0010 DS: 0000 ES: 0000 CR0: 0000000080050033 
CR2: 000056094f904f44 CR3: 00000002c8228005 CR4: 00000000000606e0 
Call Trace: 
 ? rcutorture_one_extend+0x5af/0xa30 [rcutorture] 
 rcutorture_one_extend+0x5c0/0xa30 [rcutorture] 
 rcu_torture_one_read+0x562/0x900 [rcutorture] 
 ? rcutorture_one_extend+0xa30/0xa30 [rcutorture] 
 ? lock_timer_base+0x112/0x260 
 ? lock_downgrade+0x6f0/0x6f0 
 rcu_torture_reader+0x240/0x53d [rcutorture] 
 ? rcu_torture_timer+0x1c0/0x1c0 [rcutorture] 
 ? lock_downgrade+0x6f0/0x6f0 
 ? lock_contended+0xde0/0xde0 
 ? lock_acquire+0x134/0x4b0 
 ? rcu_torture_one_read+0x900/0x900 [rcutorture] 
 ? trace_hardirqs_on_caller+0x309/0x4d0 
 ? _raw_spin_unlock_irqrestore+0x45/0xf0 
 ? __kthread_parkme+0xb6/0x180 
 ? rcu_torture_timer+0x1c0/0x1c0 [rcutorture] 
 kthread+0x30c/0x3d0 
 ? kthread_create_on_node+0xc0/0xc0 
 ret_from_fork+0x3a/0x50 
irq event stamp: 10175619 
hardirqs last  enabled at (10175619): [<ffffffff8e8c8c74>] _raw_spin_unlock_irq+0x24/0xd0 
hardirqs last disabled at (10175618): [<ffffffff8e8baabc>] __schedule+0x1bc/0x22f0 
softirqs last  enabled at (10175616): [<ffffffff8cbefc15>] __local_bh_enable_ip+0xb5/0x1a0 
softirqs last disabled at (10175613): [<ffffffffc18d6aef>] rcutorture_one_extend+0x5af/0xa30 [rcutorture] 
---[ end trace 0000000000000002 ]--- 
```

The log is bigger than that but for this post only this part is worthy.
The warning reported is from
[this code fragment](https://git.kernel.org/pub/scm/linux/kernel/git/rt/linux-rt-devel.git/tree/kernel/softirq.c?h=linux-5.12.y-rt-rebase#n173)
<sup>[1](#ft1)</sup>:

```c
void __local_bh_disable_ip(unsigned long ip, unsigned int cnt)
{
  unsigned long flags;
  int newcnt;

  WARN_ON_ONCE(in_hardirq());

  /* First entry of a task into a BH disabled section? */
  if (!current->softirq_disable_cnt) {
    if (preemptible()) {
      local_lock(&softirq_ctrl.lock);
      /* Required to meet the RCU bottomhalf requirements. */
      rcu_read_lock();
    } else {
*     DEBUG_LOCKS_WARN_ON(this_cpu_read(softirq_ctrl.cnt));
    }
  }

  ...
```

As it shows up in the log, `__local_bh_disable_ip` is called by
[rcutorture_one_extend](https://git.kernel.org/pub/scm/linux/kernel/git/rt/linux-rt-devel.git/tree/kernel/rcu/rcutorture.c?h=linux-5.12.y-rt-rebase#n1410):

```c
static void rcutorture_one_extend(int *readstate, int newstate,
          struct torture_random_state *trsp,
          struct rt_read_seg *rtrsp)
{
  unsigned long flags;
  int idxnew = -1;
  int idxold = *readstate;
  int statesnew = ~*readstate & newstate;
  int statesold = *readstate & ~newstate;

  WARN_ON_ONCE(idxold < 0);
  WARN_ON_ONCE((idxold >> RCUTORTURE_RDR_SHIFT) > 1);
  rtrsp->rt_readstate = newstate;

  /*
   * First, put new protection in place to avoid critical-section gap.
   * Disable preemption around the ATOM disables to ensure that
   * in_atomic() is true.
   */
  if (statesnew & RCUTORTURE_RDR_BH)
*   local_bh_disable();
  if (statesnew & RCUTORTURE_RDR_RBH)
    rcu_read_lock_bh();
  if (statesnew & RCUTORTURE_RDR_IRQ)
    local_irq_disable();
  if (statesnew & RCUTORTURE_RDR_PREEMPT)
    preempt_disable();
  if (statesnew & RCUTORTURE_RDR_SCHED)
    rcu_read_lock_sched();
  preempt_disable();
  if (statesnew & RCUTORTURE_RDR_ATOM_BH)
*   local_bh_disable();
  if (statesnew & RCUTORTURE_RDR_ATOM_RBH)
    rcu_read_lock_bh();
  preempt_enable();
  if (statesnew & RCUTORTURE_RDR_RCU)
    idxnew = cur_ops->readlock() << RCUTORTURE_RDR_SHIFT;

  /*
   * Next, remove old protection, in decreasing order of strength
   * to avoid unlock paths that aren't safe in the stronger
   * context.  Disable preemption around the ATOM enables in
   * case the context was only atomic due to IRQ disabling.
   */
  preempt_disable();
  if (statesold & RCUTORTURE_RDR_IRQ)
    local_irq_enable();
  if (statesold & RCUTORTURE_RDR_ATOM_BH)
    local_bh_enable();
  if (statesold & RCUTORTURE_RDR_ATOM_RBH)
    rcu_read_unlock_bh();
  preempt_enable();
  if (statesold & RCUTORTURE_RDR_PREEMPT)
    preempt_enable();
  if (statesold & RCUTORTURE_RDR_SCHED)
    rcu_read_unlock_sched();
  if (statesold & RCUTORTURE_RDR_BH)
    local_bh_enable();
  if (statesold & RCUTORTURE_RDR_RBH)
    rcu_read_unlock_bh();

  if (statesold & RCUTORTURE_RDR_RCU) {
    bool lockit = !statesnew && !(torture_random(trsp) & 0xffff);

    if (lockit)
      raw_spin_lock_irqsave(&current->pi_lock, flags);
    cur_ops->readunlock(idxold >> RCUTORTURE_RDR_SHIFT);
    if (lockit)
      raw_spin_unlock_irqrestore(&current->pi_lock, flags);
  }

  /* Delay if neither beginning nor end and there was a change. */
  if ((statesnew || statesold) && *readstate && newstate)
    cur_ops->read_delay(trsp, rtrsp);

  /* Update the reader state. */
  if (idxnew == -1)
    idxnew = idxold & ~RCUTORTURE_RDR_MASK;
  WARN_ON_ONCE(idxnew < 0);
  WARN_ON_ONCE((idxnew >> RCUTORTURE_RDR_SHIFT) > 1);
  *readstate = idxnew | newstate;
  WARN_ON_ONCE((*readstate >> RCUTORTURE_RDR_SHIFT) < 0);
  WARN_ON_ONCE((*readstate >> RCUTORTURE_RDR_SHIFT) > 1);
}
```

As shown, `rcutorture_one_extend` calls `__local_bh_disable_ip` in more than
one location. As the functions offsets also appear in the log, finding
the source code line is straightforward, I will show you how to do it later.
But what I am really interested is on the values of the variables `statesnew` and
`statesold` as they control the several conditional calls this function makes.
Knowing the value of these variables will help us on
getting more informed guesses about the context in which `__local_bh_disable_ip` was called
when it triggered the warning. So, in the rest of this post, I am going to use the
[crash utility](https://github.com/crash-utility/crash) to reveal the value of
these variables. `crash` extends
[gdb](https://www.gnu.org/software/gdb/) so we can utilize it to debug kernel and
[kdump](https://en.wikipedia.org/wiki/Kdump_(Linux)) crash files.

I am not going to tell you the procedures to set up `kdump` and generate the `vmcore`
<sup>[2](#ft2)</sup> file, there are plenty of docs around explaining how to do that,
my focus here is `crash`. You need to use a debug kernel to show the warning and you
must run `sysctl panic_on_warn=1` to generate a crash dump on warning.

Assuming you have a `vmlinux` image and a `vmcore` dump, you start `crash` like so:

```
# crash /usr/lib/debug/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/vmlinux vmore
```

The `dmesg` command shows what's the content of the kernel messages buffer. Let's use the
[bt](https://crash-utility.github.io/help_pages/bt.html) command to see the call stack trace:

```
crash> bt -s
PID: 31010  TASK: ffff8882024a3500  CPU: 0   COMMAND: "rcu_torture_rea"
 #0 [ffff8881d2067620] machine_kexec+888 at ffffffff8a920d28
 #1 [ffff8881d2067740] __crash_kexec+171 at ffffffff8ac938bb
 #2 [ffff8881d2067858] panic+476 at ffffffff8aa01605
 #3 [ffff8881d2067918] __warn.cold.6+27 at ffffffff8aa018aa
 #4 [ffff8881d2067958] report_bug+412 at ffffffff8c788c8c
 #5 [ffff8881d2067988] do_error_trap+306 at ffffffff8a854542
 #6 [ffff8881d20679d0] do_invalid_op+54 at ffffffff8a854f86
 #7 [ffff8881d20679f0] invalid_op+20 at ffffffff8ca00e04
    [exception RIP: __local_bh_disable_ip+1198]
    RIP: ffffffff8aa1b37e  RSP: ffff8881d2067aa0  RFLAGS: 00010282
    RAX: 0000000000000000  RBX: 0000000000000200  RCX: ffffffff8adbc90e
    RDX: 0000000000000000  RSI: 0000000000000004  RDI: 0000000000000246
    RBP: ffff8881d2067ad8   R8: fffffbfff1d271e2   R9: 0000000000000000
    R10: ffffffff8e938f0b  R11: fffffbfff1d271e1  R12: ffff8882024a3500
    R13: 0000000000000010  R14: ffff8881d2067b98  R15: 0000000000000044
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
 #8 [ffff8881d2067ae0] rcutorture_one_extend+1483 at ffffffffc19f745b [rcutorture]
 #9 [ffff8881d2067b40] rcu_torture_one_read+1763 at ffffffffc19f8033 [rcutorture]
#10 [ffff8881d2067dc8] rcu_torture_reader+578 at ffffffffc19f8762 [rcutorture]
#11 [ffff8881d2067f10] kthread+976 at ffffffff8aa80ed0
#12 [ffff8881d2067f50] ret_from_fork+58 at ffffffff8ca0026a
crash> 
```

In my case, I am using the linux image provided by the `kernel-rt-debug-debuginfo`
<sup>[3](#ft3)</sup> file. The `-s` option displays the offsets of the functions. By the way,
typing [help](https://crash-utility.github.io/help.html) shows a summary of the
available commands, and typing `help <cmd>` shows the documentation for the command
`cmd`. `crash` forwards any command not listed there
to `gdb`. You can also force to route to `gdb` by prefixing the command with
[gdb](https://crash-utility.github.io/help_pages/gdb.html). Therefore, to display the variables
we are interested we just set the stack frame to the `rcutorture_one_extend` function
and print the variables:

```
crash> frame 8
crash: prohibited gdb command: frame
crash>  gdb frame 8
crash: prohibited gdb command: frame
```

Err, you didn't think it would be that easy, did you? Sadly, we will need to do
it manually. Before anything else, let me instruct `crash` to load the debug symbols
for the loadable modules with the
[mod](https://crash-utility.github.io/help_pages/mod.html) command:

```
crash> mod -S                                                                                                                                                                                                                                                     
     MODULE       NAME                    SIZE  OBJECT FILE                                                                                                                                                                                                       
ffffffffc04678c0  dm_mod                397312  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/drivers/md/dm-mod.ko.debug
ffffffffc047ee00  dm_log                 36864  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/drivers/md/dm-log.ko.debug
ffffffffc0497a80  dm_region_hash         36864  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/drivers/md/dm-region-hash.ko.debug
ffffffffc04ae300  dm_mirror              65536  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/drivers/md/dm-mirror.ko.debug
ffffffffc04c6f80  ip_tables              69632  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/net/ipv4/netfilter/ip_tables.ko.debug
ffffffffc0545900  sysfillrect            28672  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/drivers/video/fbdev/core/sysfillrect.ko.debug
ffffffffc054c000  syscopyarea            24576  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/drivers/video/fbdev/core/syscopyarea.ko.debug
ffffffffc055b340  crc32c_intel           24576  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/arch/x86/crypto/crc32c-intel.ko.debug
ffffffffc056ee40  mgag200                69632  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/drivers/gpu/drm/mgag200/mgag200.ko.debug
ffffffffc057adc0  ata_generic            20480  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/drivers/ata/ata_generic.ko.debug
ffffffffc0583680  t10_pi                 20480  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/block/t10-pi.ko.debug
ffffffffc0595200  mdio                   28672  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/drivers/net/mdio.ko.debug
ffffffffc05a55c0  ata_piix               61440  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/drivers/ata/ata_piix.ko.debug
ffffffffc05aa180  libcrc32c              16384  /usr/lib/debug/usr/lib/modules/4.18.0-311.rt7.92.el8.test.x86_64+debug/kernel/lib/libcrc32c.ko.debug
...

```

Now we can use the [dis](https://crash-utility.github.io/help_pages/dis.html)
command to find out the source line where we call `__local_bh_disable_ip`:

```
crash> dis -l rcutorture_one_extend
/usr/src/debug/kernel-rt-4.18.0-311.rt7.92.el8.test/linux-4.18.0-311.rt7.92.el8.test.x86_64/kernel/rcu/rcutorture.c: 1235
0xffffffffc19f6e90 <rcutorture_one_extend>:     data32 data32 data32 xchg %ax,%ax [FTRACE NOP]
/usr/src/debug/kernel-rt-4.18.0-311.rt7.92.el8.test/linux-4.18.0-311.rt7.92.el8.test.x86_64/kernel/rcu/rcutorture.c: 1238
0xffffffffc19f6e95 <rcutorture_one_extend+5>:   movabs $0xdffffc0000000000,%rax
0xffffffffc19f6e9f <rcutorture_one_extend+15>:  push   %rbp
...
...
...
/usr/src/debug/kernel-rt-4.18.0-311.rt7.92.el8.test/linux-4.18.0-311.rt7.92.el8.test.x86_64/./include/linux/bottom_half.h: 19
0xffffffffc19f7434 <rcutorture_one_extend+1444>:        mov    $0x200,%esi
0xffffffffc19f7439 <rcutorture_one_extend+1449>:        mov    $0xffffffffc19f7434,%rdi
0xffffffffc19f7440 <rcutorture_one_extend+1456>:        callq  0xffffffff8aa1aed0 <__local_bh_disable_ip>
0xffffffffc19f7445 <rcutorture_one_extend+1461>:        jmpq   0xffffffffc19f6f44 <rcutorture_one_extend+180>
0xffffffffc19f744a <rcutorture_one_extend+1466>:        mov    $0x200,%esi
0xffffffffc19f744f <rcutorture_one_extend+1471>:        mov    $0xffffffffc19f744a,%rdi
0xffffffffc19f7456 <rcutorture_one_extend+1478>:        callq  0xffffffff8aa1aed0 <__local_bh_disable_ip>
/usr/src/debug/kernel-rt-4.18.0-311.rt7.92.el8.test/linux-4.18.0-311.rt7.92.el8.test.x86_64/kernel/rcu/rcutorture.c: 1264
0xffffffffc19f745b <rcutorture_one_extend+1483>:        testb  $0x80,-0x30(%rbp)
...
...
...
```

The `-l` option provides the source code line number information. We see that the `1483`
offset is at `rcutorture.c:1264` and the instruction above is the call we are
looking for<sup>[4](#ft4)</sup>. Now we can look at the source code to see the
corresponding line:

```c
crash> dis -s rcutorture_one_extend
FILE: kernel/rcu/rcutorture.c
LINE: 1235

  1230   * change, do a ->read_delay().
  1231   */
  1232  static void rcutorture_one_extend(int *readstate, int newstate,
  1233                                    struct torture_random_state *trsp,
  1234                                    struct rt_read_seg *rtrsp)
  1235  {
  1236          unsigned long flags;
  1237          int idxnew = -1;
  1238          int idxold = *readstate;
  1239          int statesnew = ~*readstate & newstate;
  1240          int statesold = *readstate & ~newstate;
  1241  
  1242          WARN_ON_ONCE(idxold < 0);
  1243          WARN_ON_ONCE((idxold >> RCUTORTURE_RDR_SHIFT) > 1);
  1244          rtrsp->rt_readstate = newstate;
  1245  
  1246          /*
  1247           * First, put new protection in place to avoid critical-section gap.
  1248           * Disable preemption around the ATOM disables to ensure that
  1249           * in_atomic() is true.
  1250           */
  1251          if (statesnew & RCUTORTURE_RDR_BH)
  1252                  local_bh_disable();
  1253          if (statesnew & RCUTORTURE_RDR_RBH)
  1254                  rcu_read_lock_bh();
  1255          if (statesnew & RCUTORTURE_RDR_IRQ)
  1256                  local_irq_disable();
  1257          if (statesnew & RCUTORTURE_RDR_PREEMPT)
  1258                  preempt_disable();
  1259          if (statesnew & RCUTORTURE_RDR_SCHED)
  1260                  rcu_read_lock_sched();
  1261          preempt_disable();
  1262          if (statesnew & RCUTORTURE_RDR_ATOM_BH)
  1263 *                local_bh_disable();
  1264          if (statesnew & RCUTORTURE_RDR_ATOM_RBH)
  1265                  rcu_read_lock_bh();
...
```

The line `1263` is where  we make the call.

As I promised at the beggining, we will now recover the values of the local variables
`statesnew` and `statesold`. Let's start by looking at the beginning of the
`rcutorture_one_extend` function assembly code:

```
crash> dis -l rcutorture_one_extend
/usr/src/debug/kernel-rt-4.18.0-311.rt7.92.el8.test/linux-4.18.0-311.rt7.92.el8.test.x86_64/kernel/rcu/rcutorture.c: 1235
0xffffffffc19f6e90 <rcutorture_one_extend>:     data32 data32 data32 xchg %ax,%ax [FTRACE NOP]
/usr/src/debug/kernel-rt-4.18.0-311.rt7.92.el8.test/linux-4.18.0-311.rt7.92.el8.test.x86_64/kernel/rcu/rcutorture.c: 1238
0xffffffffc19f6e95 <rcutorture_one_extend+5>:   movabs $0xdffffc0000000000,%rax
0xffffffffc19f6e9f <rcutorture_one_extend+15>:  push   %rbp
0xffffffffc19f6ea0 <rcutorture_one_extend+16>:  mov    %rsp,%rbp
0xffffffffc19f6ea3 <rcutorture_one_extend+19>:  push   %r15
0xffffffffc19f6ea5 <rcutorture_one_extend+21>:  mov    %esi,%r15d
0xffffffffc19f6ea8 <rcutorture_one_extend+24>:  push   %r14
0xffffffffc19f6eaa <rcutorture_one_extend+26>:  mov    %rdi,%r14
0xffffffffc19f6ead <rcutorture_one_extend+29>:  push   %r13
0xffffffffc19f6eaf <rcutorture_one_extend+31>:  push   %r12
0xffffffffc19f6eb1 <rcutorture_one_extend+33>:  push   %rbx
0xffffffffc19f6eb2 <rcutorture_one_extend+34>:  sub    $0x28,%rsp
0xffffffffc19f6eb6 <rcutorture_one_extend+38>:  mov    %rdx,-0x48(%rbp)
0xffffffffc19f6eba <rcutorture_one_extend+42>:  mov    %rdi,%rdx
0xffffffffc19f6ebd <rcutorture_one_extend+45>:  shr    $0x3,%rdx
0xffffffffc19f6ec1 <rcutorture_one_extend+49>:  mov    %rcx,-0x38(%rbp)
0xffffffffc19f6ec5 <rcutorture_one_extend+53>:  movzbl (%rdx,%rax,1),%edx
0xffffffffc19f6ec9 <rcutorture_one_extend+57>:  mov    %rdi,%rax
0xffffffffc19f6ecc <rcutorture_one_extend+60>:  and    $0x7,%eax
0xffffffffc19f6ecf <rcutorture_one_extend+63>:  add    $0x3,%eax
0xffffffffc19f6ed2 <rcutorture_one_extend+66>:  cmp    %dl,%al
0xffffffffc19f6ed4 <rcutorture_one_extend+68>:  jl     0xffffffffc19f6ede <rcutorture_one_extend+78>
0xffffffffc19f6ed6 <rcutorture_one_extend+70>:  test   %dl,%dl
0xffffffffc19f6ed8 <rcutorture_one_extend+72>:  jne    0xffffffffc19f7856 <rcutorture_one_extend+2502>
0xffffffffc19f6ede <rcutorture_one_extend+78>:  mov    (%r14),%r13d
/usr/src/debug/kernel-rt-4.18.0-311.rt7.92.el8.test/linux-4.18.0-311.rt7.92.el8.test.x86_64/kernel/rcu/rcutorture.c: 1239
0xffffffffc19f6ee1 <rcutorture_one_extend+81>:  mov    %r15d,%r12d
0xffffffffc19f6ee4 <rcutorture_one_extend+84>:  not    %r12d
0xffffffffc19f6ee7 <rcutorture_one_extend+87>:  mov    %r13d,%eax
0xffffffffc19f6eea <rcutorture_one_extend+90>:  and    %r13d,%r12d
0xffffffffc19f6eed <rcutorture_one_extend+93>:  not    %eax
0xffffffffc19f6eef <rcutorture_one_extend+95>:  and    %r15d,%eax
0xffffffffc19f6ef2 <rcutorture_one_extend+98>:  mov    %eax,-0x30(%rbp)
```

The only parameters which care are `readstate` and `newstate`.
According to the
[x86_64 calling convention](https://aaronbloomfield.github.io/pdr/book/x86-64bit-ccc-chapter.pdf),
they are passed, respectively, in the `edi` and `esi` registers. Below you find
a commented version of the `rcutorture_one_extend` function assembly code up
to the part that set both variables:

```
movabs $0xdffffc0000000000,%rax
push   %rbp
mov    %rsp,%rbp  # setup the function stack frame
push   %r15       # save register r15
mov    %esi,%r15d # move newstate to r15d
push   %r14       # save r14
mov    %rdi,%r14  # move readstate to rdi
push   %r13       # save r13
push   %r12       # save r12
push   %rbx       # save rbx
sub    $0x28,%rsp # allocate space for the local variables

#######################################################################
# this part doesn't interest us
mov    %rdx,-0x48(%rbp)
mov    %rdi,%rdx
shr    $0x3,%rdx
mov    %rcx,-0x38(%rbp)
movzbl (%rdx,%rax,1),%edx
mov    %rdi,%rax
and    $0x7,%eax
add    $0x3,%eax
cmp    %dl,%al
jl     0xffffffffc19f6ede <rcutorture_one_extend+78>
test   %dl,%dl
jne    0xffffffffc19f7856 <rcutorture_one_extend+2502>
#######################################################################

mov    (%r14),%r13d     # move *readstate to r14d
mov    %r15d,%r12d      # move newstate to r12d
not    %r12d            # r12 = ~newstate
mov    %r13d,%eax       # move *readstate to eax
and    %r13d,%r12d      # r12d = *readstate & ~newstate = statesold
not    %eax             # eax = ~*readstate
and    %r15d,%eax       # eax = ~*readstate & newstate = statesnew
mov    %eax,-0x30(%rbp) # statesnew = (int) rbp[-6]
```

From the code above, `statesold` ends up in the register `r12d` and `statesnew`
in the position `rbp-6` (in 64 bits sized chunks) of the stack frame.
We shall start deducing the value of `statesnew`. We can take advantage
of the option `-f` of the [bt](https://crash-utility.github.io/help_pages/bt.html)
command to display the content of the stack frame:

```
crash> bt -f
...
 #8 [ffff8881d2067ae0] rcutorture_one_extend at ffffffffc19f745b [rcutorture]
    ffff8881d2067ae8: ffffffff8abc8c10 ffff8881d2067e00
    ffff8881d2067af8: 000000003a40cf78 ffff8881d2067c00
    ffff8881d2067b08: ffff888300000044 ffff8881d2067bd8
    ffff8881d2067b18: ffffffffc1a5f300 ffff8881d2067c28
    ffff8881d2067b28: ffff8881d2067b98 ffff8881d2067d40
    ffff8881d2067b38: ffff8881d2067e00 ffffffffc19f8033
...
```

The stack frame starts at `0xffff8881d2067b38` (remember the stack grows downwards).
The first position corresponds to the `rbp` push at the function preamble,
we then count 6 positions to the right upwards and we reach the address
`0xffff8881d2067b08` whose value is `0xffff888300000044`. Since `statesnew` is
of an `int` type, we only count the first 32 bits and we finally find the
value `0x44`. If we look at the
[kernel/rcu/rcutorture.c](https://git.kernel.org/pub/scm/linux/kernel/git/rt/linux-rt-devel.git/tree/kernel/rcu/rcutorture.c?h=linux-5.13.y-rt-rebase#n60)
source file, we conclude that `statesnew = RCUTORTURE_RDR_PREEMPT | RCUTORTURE_RDR_ATOM_BH`.

`statesold` is stored in the `r12d` register, but we can't rely in register value because
it probably has already been clobbered buy one of the `rcutorture_one_extend` callees,
but it must be saved in someone's stack frame. Here is the prologue of the
`__local_bh_disable_ip`, the first callee:

```
crash> dis __local_bh_disable_ip
0xffffffff8aa1aed0 <__local_bh_disable_ip>:     data32 data32 data32 xchg %ax,%ax [FTRACE NOP]
0xffffffff8aa1aed5 <__local_bh_disable_ip+5>:   push   %rbp
0xffffffff8aa1aed6 <__local_bh_disable_ip+6>:   mov    %rsp,%rbp
0xffffffff8aa1aed9 <__local_bh_disable_ip+9>:   push   %r15
0xffffffff8aa1aedb <__local_bh_disable_ip+11>:  push   %r14
0xffffffff8aa1aedd <__local_bh_disable_ip+13>:  push   %r13
0xffffffff8aa1aedf <__local_bh_disable_ip+15>:  push   %r12
...
```

Indeed `statesold` is saved in the `__local_bh_disable_ip` stack frame, more
precisely in `rbp-4`. Below I dump the function stack frame:

```
crash> bt -f
...
 #7 [ffff8881d20679f0] invalid_op at ffffffff8ca00e04
    [exception RIP: __local_bh_disable_ip+1198]
    RIP: ffffffff8aa1b37e  RSP: ffff8881d2067aa0  RFLAGS: 00010282
    RAX: 0000000000000000  RBX: 0000000000000200  RCX: ffffffff8adbc90e
    RDX: 0000000000000000  RSI: 0000000000000004  RDI: 0000000000000246
    RBP: ffff8881d2067ad8   R8: fffffbfff1d271e2   R9: 0000000000000000
    R10: ffffffff8e938f0b  R11: fffffbfff1d271e1  R12: ffff8882024a3500
    R13: 0000000000000010  R14: ffff8881d2067b98  R15: 0000000000000044
    ORIG_RAX: ffffffffffffffff  CS: 0010  SS: 0018
    ffff8881d20679f8: 0000000000000044 ffff8881d2067b98 
    ffff8881d2067a08: 0000000000000010 ffff8882024a3500 
    ffff8881d2067a18: ffff8881d2067ad8 0000000000000200 
    ffff8881d2067a28: fffffbfff1d271e1 ffffffff8e938f0b 
    ffff8881d2067a38: 0000000000000000 fffffbfff1d271e2 
    ffff8881d2067a48: 0000000000000000 ffffffff8adbc90e 
    ffff8881d2067a58: 0000000000000000 0000000000000004 
    ffff8881d2067a68: 0000000000000246 ffffffffffffffff 
    ffff8881d2067a78: ffffffff8aa1b37e 0000000000000010 
    ffff8881d2067a88: 0000000000010282 ffff8881d2067aa0 
    ffff8881d2067a98: 0000000000000018 ffff8881d2067e00 
    ffff8881d2067aa8: ffffffffc19f744a ffff8881d2067bd8 
    ffff8881d2067ab8: 0000000000000010 0000000000000010 
    ffff8881d2067ac8: ffff8881d2067b98 0000000000000044 
    ffff8881d2067ad8: ffff8881d2067b38 ffffffffc19f745b
...
```

I guess you can see that `statesold` location is `0xffff8881d2067ab8` and
its value is `0x10`. If we do the same exercise of looking for the corresponding
macro(s) definition(s) for the value, we discover that
`statesold = RCUTORTURE_RDR_SCHED`.

---------------------------------------------------------------------------------

<a name="ft1">1</a>: the watchful reader may find some inconsistencies
among the line numbers reported in the log and in the git tree, that's because
I am debugging the
[RHEL](https://www.redhat.com/en/technologies/linux-platforms/enterprise-linux)
8 kernel.

<a name="ft2">2</a>: the name of the crash file `kdump` generates.

<a name="ft3">3</a>: notice that I am using the `rt-debug` kernel to reproduce the problem.
This bug only manifests in the realtime kernel. Be sure to use the debuginfo package
corresponding to your kernel.

<a name="ft4">4</a>: bear in mind that the call stack entries indicate the return addresses.

<a name="ft5">5</a>: `local_bh_disable` is an inline function that calls `__local_bh_disable_ip`:

```c
static inline void local_bh_disable(void)
{
        __local_bh_disable_ip(_THIS_IP_, SOFTIRQ_DISABLE_OFFSET);
}
```
