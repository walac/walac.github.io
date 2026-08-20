---
title: "What every kernel programmer should know about Jump Labels"
comments: true
categories: [kernel]
---

* TOC
{:toc}

This tutorial explains Linux kernel “jump labels” — also called “static
keys” — from first principles, using the actual source in this tree
(upstream v7.2) as the reference. **x86_64** is the architecture covered
throughout.

It is meant to be readable even if you have never touched kernel text
patching before. The early sections lay down the CPU and
instruction-encoding background that later sections build on — skim them
if that part is already familiar.

Core files referenced:

| File | Role |
|----|----|
| [`include/linux/jump_label.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h) | Public API, macros, [`struct static_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L86) |
| [`kernel/jump_label.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c) | Architecture-independent core logic |
| [`arch/x86/include/asm/jump_label.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h) | x86 inline asm that emits the branch |
| [`arch/x86/kernel/jump_label.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c) | x86 code-patching backend |
| [`arch/x86/include/asm/text-patching.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h) | JMP/INT3/CALL/RET encodings, [`text_gen_insn()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h#L123) |
| [`arch/x86/include/asm/nops.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/nops.h) | [`x86_nops[]`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L91) table, `BYTES_NOPx` raw byte defs |
| [`arch/x86/kernel/alternative.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c) | [`text_poke()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2668), `smp_text_poke_*()` — the SMP-safe patcher |
| [`include/linux/jump_label_ratelimit.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label_ratelimit.h) | [`struct static_key_deferred`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label_ratelimit.h#L9) |
| [`include/linux/tracepoint.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/tracepoint.h) | A real, heavily-used consumer |
| [`Documentation/staging/static-keys.rst`](https://elixir.bootlin.com/linux/v7.2-rc7/source/Documentation/staging/static-keys.rst) | Upstream overview (somewhat outdated on x86 sizes) |
| [`tools/objtool/`](https://elixir.bootlin.com/linux/v7.2-rc7/source/tools/objtool) | Compile-time rewrite of `jmp`→`nop` under [`HAVE_JUMP_LABEL_HACK`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L1399) |

Naming: **“jump label”** is the low-level mechanism — a patchable site
in `.text` plus an entry in
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/scripts/module.lds.S#L31).
**“Static key”** is the higher-level API programmers use
(`DEFINE_STATIC_KEY_*`, `static_branch_*`). People often use the two
names interchangeably, but this tutorial keeps them distinct: “static
key” for the API, “jump label” for the patching machinery underneath it.

------------------------------------------------------------------------

## 1 Hardware background (why this is hard) {#hardware-background-why-this-is-hard}

Jump labels work by rewriting machine code while the kernel is running -
self-modifying code, executed by CPUs that were never designed to expect
it. Making sense of that requires a bit of hardware background first:
how a CPU actually treats instructions and memory loads, the exact
byte-level encodings involved, and why a live, multi-core kernel can’t
just overwrite them with an ordinary write.

### 1.1 What a CPU actually does with instructions {#what-a-cpu-actually-does-with-instructions}

A modern x86_64 core does not “read one instruction, execute it,
repeat”. Roughly:

1.  **Fetch** bytes from the instruction cache (I-cache / L1i).
2.  **Decode** those bytes into microcodes (variable-length on x86 — an
    instruction can be 1–15 bytes).
3.  **Execute** out of order, with a **branch predictor** guessing which
    way conditional branches go so the pipeline stays full.
4.  Commit results in order.

Two consequences follow from that pipeline, and together they are the
entire performance case for jump labels. The first is about what a naive
`if (feature_enabled)` check actually costs. Out-of-order execution and
branch prediction make the *branch itself* close to free: predict
correctly often enough and there is no pipeline flush to pay for. But
prediction only hides the cost of guessing which way a branch goes - it
does nothing about the `feature_enabled` **load** that feeds the guess.
That load still has to happen on every single hit of the path, whether
or not the predictor gets the branch right, and it still claims a real
data-cache access and an execution port each time. Under cache pressure,
or if some other CPU ever writes that flag and bounces its cache line
out from under you, the “cheap” branch stops being cheap at all.

The second consequence is the whole point of jump labels: an
unconditional `nop` or an unconditional `jmp` sidesteps that entire
problem, because there is no flag to load and no condition to evaluate.
The “off” path can be literally empty work for the frontend - decode a
`nop`, move on - with no cache line to bounce and nothing for the branch
predictor to even weigh in on.

### 1.2 x86 instruction encoding: JMP and NOP {#x86-instruction-encoding-jmp-and-nop}

[§1.1](#what-a-cpu-actually-does-with-instructions)’s cost difference
between a load and a `nop`/`jmp` has to actually be encoded in real
bytes for a patch to swap between them, and matching sizes will matter
as soon as [§1.3](#why-you-cannot-just-memcpy-over-live-code-on-smp)
gets to atomicity - so here is the exact shape of both. x86 is a
variable-length ISA; the encodings jump labels care about:

| Instruction | Opcode bytes | Total size | Reach |
|----|----|----|----|
| `INT3` (breakpoint) | `CC` | 1 byte | n/a |
| `JMP rel8` (short) | `EB xx` | **2 bytes** | -128..+127 bytes from the end of the insn |
| `JMP rel32` (near) | `E9 xx xx xx xx` | **5 bytes** | ±2 GiB |
| 2-byte NOP | `66 90` | 2 bytes | — |
| 5-byte NOP | `0f 1f 44 00 00` (`nopl 0x0(%rax,%rax,1)`) | 5 bytes | — |

Constants live in
[`arch/x86/include/asm/text-patching.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h)
(`JMP8_INSN_*`, `JMP32_INSN_*`, `INT3_INSN_*`) and
[`arch/x86/include/asm/nops.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/nops.h)
([`BYTES_NOP5`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/nops.h#L60),
etc.). The relative displacement is measured from the **byte after** the
instruction:

    disp = dest - (addr + insn_size)

That is exactly what
[`text_gen_insn()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h#L123)
/
[`__text_gen_insn()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h#L93)
compute.

**Why matching size matters:** if you replace a 5-byte `nop` with a
5-byte `jmp`, surrounding addresses do not move. Return addresses on
stacks, other jump targets, exception tables, ORC unwind info — none of
them need updating. Patching is an *in-place* byte swap of equal length.

### 1.3 Why you cannot just `memcpy` over live code on SMP {#why-you-cannot-just-memcpy-over-live-code-on-smp}

Patching kernel text is harder than patching an ordinary data structure
because three things are true of it at once:

1.  It is mapped **read-only** after boot
    ([`CONFIG_STRICT_KERNEL_RWX`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L1581)),
    so a normal store to it would simply fault — whatever mechanism does
    the patching has to get around that on purpose, not by accident.
2.  It is being **fetched by other CPUs** concurrently — nothing pauses
    the rest of the machine while one CPU edits a function that every
    core can call at any moment.
3.  It may already be sitting half-decoded in the pipeline of another
    CPU, having been fetched moments ago but not yet executed.

Point 2 is the dangerous one, and it is worth walking through
concretely. Picture two CPUs, A and B, where A is patching a 5-byte
instruction that B keeps calling in a loop. The store A makes is not one
atomic operation — the CPU issues it as however many bus-width writes it
takes to cover 5 bytes, and each of those writes becomes visible to the
rest of the system separately. The fetch B makes can land in the middle
of that sequence, seeing some bytes from before the write and some from
after:

                         time --->

     CPU A (patcher)   [ write bytes 2-4 ]   [ write bytes 0-1 ]
                                          ^
                                          |
     CPU B (fetcher)             [ fetch all 5 bytes, right here ]
                                          |
                                          v
                     byte-by-byte: 0=old  1=old  2=new  3=new  4=new
                     = torn mix: 2 old bytes + 3 new bytes
                     = neither the old instruction
                       nor the new one — garbage

If CPU A writes five bytes while CPU B is mid-fetch of that same
instruction, B can observe this **torn** mix of old and new bytes — not
a valid instruction, and not something the decoder in B can safely
execute. x86 does *not* guarantee that a multi-byte store to a
concurrently executing instruction is atomic from the point of view of
instruction fetch.

A single-byte store, by contrast, *is* atomic for instruction fetch — no
CPU can ever see it half-written, because there is no “half” of one
byte. Both the Intel SDM and the approach the kernel takes build on
exactly that fact, turning one unsafe multi-byte write into three safe
single-step moves:

- First make the site a single-byte `INT3` (`0xCC`). That one-byte store
  is atomic, so every CPU either still sees the old instruction or
  already sees the trap — never a torn mix of the two.
- Then rewrite the remaining bytes underneath, while the `INT3` is still
  sitting on top of them. Nothing fetches through those bytes yet,
  because byte 0 is still the trap.
- Then replace the `INT3` with the first byte of the final instruction,
  again as one atomic single-byte store — this is the moment the new
  instruction becomes live.
- Between each of those three steps, **synchronize every CPU** (an IPI
  that makes every core run a serializing instruction) so that no core
  is still executing, or has stale bytes queued up in its pipeline or
  I-cache, from before that step.

That protocol lives in
[`smp_text_poke_batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
([§10](#x86-text-patching-the-gory-details)), and jump labels are only
one of several clients that share it — ftrace, static calls, kprobes,
and the alternatives-patching machinery all reuse this same three-step
dance.

### 1.4 Writing read-only kernel text: [`text_poke()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2668) {#writing-read-only-kernel-text-text_poke}

Modern kernels no longer patch text by clearing the WP bit in `%cr0`,
writing, and setting it back. That old trick worked, but it was a blunt
instrument: between the clear and the restore, every write from every
CPU could land on write-protected memory, not just the one instruction
being patched. Anything else that happened to run during that window
could corrupt memory it was never supposed to touch.
[`__text_poke()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2546)
replaces it with a narrower idea. Instead of unlocking the *existing*
mapping of `.text`, it builds a **second, private virtual mapping of the
exact same physical page** and writes through that instead.

The intuition is worth stating before the mechanism. Physical RAM does
not know or care how it is mapped; the same page of memory can be
reached through more than one virtual address at once, each with its own
permissions. `.text` normally has exactly one mapping, visible to every
CPU, always read-only and executable. That single mapping is what lets
any core fetch and run it at any moment.

[`__text_poke()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2546)
temporarily adds a *second* mapping to that same physical page:
writable, not executable, and visible only to the CPU doing the
patching. That mapping is torn down within a handful of instructions. It
is a second door into the same room. The contents of the room — the
instruction bytes — are the same no matter which door you walk through,
but only one of the two doors is ever locked.

Reaching that second mapping takes several steps, and each one closes
off a different way this could otherwise go wrong. All five below are
pieces of one function,
[`__text_poke()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2546),
working with these locals:

``` c
static void *__text_poke(text_poke_f func, void *addr, const void *src, size_t len)
{
        bool cross_page_boundary = offset_in_page(addr) + len > PAGE_SIZE;
        struct page *pages[2] = {NULL};
        struct mm_struct *prev_mm;
        unsigned long flags;
        pte_t pte, *ptep;
        spinlock_t *ptl;
        pgprot_t pgprot;
        ...
```

`func` is the actual copy routine: `memcpy`-like for a real
`text_poke()` call, `memset`-like for the `_set()` variant.
`addr`/`src`/`len` are simply the target and payload the caller passed
in.

First, the function has to identify the physical page or pages backing
the address being patched — usually one page, two if the write straddles
a page boundary. It gets there via
[`virt_to_page()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/page.h#L62)
for core kernel text, or
[`vmalloc_to_page()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/mm/vmalloc.c#L799)
for text living in a module:

``` c
if (!core_kernel_text((unsigned long)addr)) {
        pages[0] = vmalloc_to_page(addr);
        if (cross_page_boundary)
                pages[1] = vmalloc_to_page(addr + PAGE_SIZE);
} else {
        pages[0] = virt_to_page(addr);
        if (cross_page_boundary)
                pages[1] = virt_to_page(addr + PAGE_SIZE);
}
```

Second, it points a pre-allocated page-table entry at that physical
page, inside a dedicated, otherwise-empty address space called
[`text_poke_mm`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2515)
(allocated once, at boot, by
[`poking_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/init.c#L819)).
That entry is marked writable, and deliberately *not* global: it carries
no
[`_PAGE_GLOBAL`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/pgtable_types.h#L59)
bit.

That one detail is what keeps the whole scheme cheap. A non-global
mapping is only ever cached in the TLB of the current CPU, so tearing it
down later is a plain, local
[`flush_tlb_mm_range()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/tlb.c#L1428)
— no IPI to other CPUs, because no other CPU ever loaded
[`text_poke_mm`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2515)
in the first place:

``` c
pgprot = __pgprot(pgprot_val(PAGE_KERNEL) & ~_PAGE_GLOBAL);
ptep = get_locked_pte(text_poke_mm, text_poke_mm_addr, &ptl);

local_irq_save(flags);

pte = mk_pte(pages[0], pgprot);
set_pte_at(text_poke_mm, text_poke_mm_addr, ptep, pte);
if (cross_page_boundary) {
        pte = mk_pte(pages[1], pgprot);
        set_pte_at(text_poke_mm, text_poke_mm_addr + PAGE_SIZE, ptep + 1, pte);
}
```

Third, the current CPU actually switches onto that private address space
via
[`use_temporary_mm(text_poke_mm)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/tlb.c#L992),
saving whatever `mm` it had loaded so it can restore it afterward.
Writing `%cr3` changes what every virtual address on this CPU means. But
the pipeline is deep and speculative: instructions ahead of that write
may already have been fetched, decoded, or executed under the *old*
mapping. Left alone, execution could keep running past the switch on
stale translations, resolving a load or fetch as if the old address
space were still current.

The x86 architecture closes that off by defining writes to control
registers (`%cr0`/`%cr3`/`%cr4`/`%cr8`) as **serializing**: the CPU must
retire everything prior, discard any speculative work in flight, and
drop non-global TLB entries before it begins executing under the new
value. That guarantee holds on every implementation — it is part of the
ISA, not a performance accident. Loading `%cr3` here gets it for free:
the CPU is certain to see the page-table entry from the previous step
before it can fetch anything through it, with no separate
synchronization needed:

``` c
prev_mm = use_temporary_mm(text_poke_mm);
```

The switch also has to deal with a second, unrelated hazard: hardware
watchpoints. The debug registers that hold watchpoint addresses
(`%dr0`–`%dr3`) are global CPU state, not scoped to whichever address
space happens to be loaded, so a watchpoint stays armed straight through
a page-table switch regardless of serialization.

[`text_poke_mm_addr`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2516)
is deliberately placed in the low, user-range half of the address space
(the hardware sidebar below explains why), which happens to be exactly
where the watchpoints of a debugger live. If a userspace watchpoint
aliased onto that address while this CPU was mid-write through it, the
CPU would fire a debug exception in the middle of the very code-patching
machinery the kernel itself relies on. That would misdeliver a signal
that has nothing to do with whatever the debugger was actually watching
for — or worse, interrupt the sensitive write itself.

So
[`use_temporary_mm()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/tlb.c#L992)
explicitly calls
[`hw_breakpoint_disable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/debugreg.h#L109)
right after the switch, and its counterpart,
[`hw_breakpoint_restore()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/hw_breakpoint.c#L484),
restores them afterward. Breakpoints are disabled wholesale rather than
only for the specific colliding address, so this even suppresses
unrelated kernel breakpoints (e.g. ones set by perf) for that brief
window. That’s accepted as a reasonable trade-off, since the window is
so short.

Fourth, with the writable alias finally in place, the actual copy
happens by calling `func`, at the address
`text_poke_mm_addr + offset_in_page(addr)`:

``` c
func((u8 *)text_poke_mm_addr + offset_in_page(addr), src, len);
```

For a real
[`text_poke()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2668)
call, `func` is
[`text_poke_memcpy()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2528)
— the small wrapper the caller handed in as the `func` argument (the
`_set()` variant passes the memset-flavored
[`text_poke_memset()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2535)
instead):

``` c
static void text_poke_memcpy(void *dst, const void *src, size_t len)
{
        lass_stac();
        __inline_memcpy(dst, src, len);
        lass_clac();
}
```

That sequence raises two questions: why the write needs `STAC`/`CLAC`
around it at all, and why the copy inside them has to be inline rather
than a real call to
[`memcpy()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/lib/memcpy_64.S).
The hardware sidebar below answers both together.

Fifth, the temporary mapping is dismantled in the reverse order it was
built:
[`pte_clear()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/pgtable.h#L88)
removes the page-table entry,
[`unuse_temporary_mm()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/tlb.c#L1027)
switches `%cr3` back to the saved `mm` (serializing again, for the same
reason as the switch in step three), and
[`flush_tlb_mm_range()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/tlb.c#L1428)
drops the now-stale local TLB entry.

Finally, for a real
[`text_poke()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2668)
call (though not for the
[`_set`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2744)/[`_copy`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2727)
variants, which skip this) the function reads back what it just wrote
and `memcmp`s it against what was intended. Any mismatch is a
[`BUG()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/bug.h#L114):
a mechanism whose entire premise is “the bytes we write are exactly the
bytes we meant to write” cannot be allowed to fail silently — a loud
crash is the only honest response:

``` c
pte_clear(text_poke_mm, text_poke_mm_addr, ptep);
if (cross_page_boundary)
        pte_clear(text_poke_mm, text_poke_mm_addr + PAGE_SIZE, ptep + 1);

unuse_temporary_mm(prev_mm);
flush_tlb_mm_range(text_poke_mm, text_poke_mm_addr, text_poke_mm_addr +
                   (cross_page_boundary ? 2 : 1) * PAGE_SIZE, PAGE_SHIFT, false);

if (func == text_poke_memcpy)
        BUG_ON(memcmp(addr, src, len));

local_irq_restore(flags);
```

Put together, this is one physical page reached through two different
virtual addresses with two different permissions:

                              physical page (the actual RAM
                              holding the instruction bytes)
                                        ^        ^
                                        |        |
                  normal kernel         |        |   text_poke_mm_addr
                  mapping (all CPUs,    |        |   (this CPU only,
                  always present)       |        |    exists briefly)
                         |               \      /            |
                         v                \    /             v
              .text  [ RO, executable ]    \  /    [ RW, not executable ]
              (every CPU's %cr3 maps        \/     (only this CPU's %cr3
               this address, forever)                maps this address,
                                                      only while patching)

Every CPU, all the time, can execute the left-hand mapping — that one
never changes permission or address. Only the *current* CPU, and only
*during*
[`__text_poke()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2546),
can additionally reach the exact same bytes through the right-hand
mapping, and only to write them. Once the teardown step above clears
that PTE and flushes the local TLB, the right-hand mapping is gone
again; the left-hand one is all that is left, now showing the new bytes.

So the *permanent* kernel mapping of `.text` stays read-only everywhere,
all the time; only a throwaway, single-CPU-visible alias is ever
writable, and it exists for only a handful of instructions. All of this
is serialized by
[`text_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/extable.c#L27),
so two concurrent patchers never race to build competing temporary
mappings.

> **Hardware sidebar — why `STAC`/`CLAC`, and why the copy must be
> inline.** Some CPU features exist specifically to catch the kernel
> touching low, user-range addresses by *accident*. The classic bug —
> and attack surface — is a corrupted or attacker-influenced pointer
> that the kernel ends up dereferencing as if it pointed to trusted
> kernel memory. `STAC`/`CLAC` are how the kernel tells the CPU, in
> effect, “the access I’m about to make into that range is deliberate,
> stand down for a moment.” Two independent features watch for exactly
> this, and they don’t watch for the same thing:
>
> - **SMAP** (“Supervisor Mode Access Prevention”) faults if kernel code
>   accesses a page whose page-table entry has
>   [`_PAGE_USER`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/pgtable_types.h#L53)
>   set — that is, a page actually mapped user-accessible. It looks only
>   at that one permission bit, never at the numeric address.
> - **LASS** (“Linear Address Space Separation”), newer than SMAP,
>   faults on *any* kernel access to an address below the
>   canonical-address midpoint. That means anywhere numerically in the
>   user range, regardless of whether
>   [`_PAGE_USER`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/pgtable_types.h#L53)
>   is set on that particular page.
>
> [`text_poke_mm_addr`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2516)
> falls right in the gap between those two rules. To be clear, it is
> **not** actually a userspace mapping:
> [`poking_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/init.c#L819)
> sets it to
> [`TASK_UNMAPPED_BASE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/processor.h#L684),
> plus a KASLR-style random offset — the same range where calls to
> [`mmap()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/sys_x86_64.c#L82)
> made by an ordinary process would land. That’s simply because
> [`text_poke_mm`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2515)
> is built with
> [`mm_alloc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/fork.c#L1166),
> the ordinary allocator for a process address space, and a freshly
> allocated `mm` just happens to have empty space to carve one throwaway
> page out of down there. The page-table entry actually built at that
> address is an ordinary kernel-only mapping, with
> [`_PAGE_USER`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/pgtable_types.h#L53)
> left clear, so nothing about it is reachable from user mode.
>
> That distinction is exactly what splits the two checks apart. SMAP
> only ever looks at the `_PAGE_USER` bit, and that bit is clear here,
> so SMAP has nothing to object to. LASS blocks by address alone,
> regardless of the bit — and this address, purely by where it
> numerically sits, is exactly what LASS would fault on.
>
> That is why the actual code calls
> [`lass_stac()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/smap.h#L70)/[`lass_clac()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/smap.h#L65)
> rather than plain `STAC`/`CLAC`. They emit the same underlying
> instructions, just gated on
> [`X86_FEATURE_LASS`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/cpufeatures.h#L317)
> instead of
> [`X86_FEATURE_SMAP`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/cpufeatures.h#L253)
> — so the call compiles down to a no-op on CPUs without LASS, and to a
> real access-check override on CPUs that have it. Either way it works,
> whether the CPU has SMAP, LASS, both, or neither. Conceptually this is
> the same `AC`-bit mechanism the kernel already uses whenever it
> deliberately touches real userspace memory
> ([`copy_from_user()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/uaccess.h#L218)
> and friends); text poking just happens to need it too, for a
> kernel-internal mapping that only *looks* like a userspace address.
>
> Opening that window has one more consequence.
> [`objtool`](https://elixir.bootlin.com/linux/v7.2-rc7/source/tools/objtool)
> ([§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes) covers it in
> depth) enforces a build-time rule that no `call` instruction may
> appear between a `STAC` and the next `CLAC`. The reason is concrete:
> `AC` is ordinary CPU state, but unlike registers, it is not saved and
> restored across a context switch. A call is a black box — it might
> transitively reach
> [`schedule()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/sched/core.c#L7316)
> and put the task to sleep, and if that happens while `AC` is set, the
> override can leak into whichever task runs next, or fail to be
> restored when this one resumes.
>
> That rule is why the copy can never be a real call to
> [`memcpy()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/lib/memcpy_64.S).
> On x86_64, `memcpy()` is hand-written assembly living in a separate
> object, reachable only through a genuine `call` instruction.
> [`__inline_memcpy()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/string.h#L11)/[`__inline_memset()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/string.h#L21)
> sidestep that problem: forced inline, they compile to the same
> `rep movsb`/`rep stosb` sequence the real functions would use, but
> leave no `call` for objtool to flag, because there is no separate
> function left to call.

------------------------------------------------------------------------

## 2 The problem jump labels solve {#the-problem-jump-labels-solve}

Kernel code is full of rarely-taken checks that guard optional
functionality: “is tracing enabled for this tracepoint?”, “is this
security module active?”, “is this debug feature on?”. A naive
implementation:

``` c
if (some_feature_enabled)
        do_something();
```

Even when `some_feature_enabled` is almost always false, the CPU still
must:

1.  Load `some_feature_enabled` from memory (a cache line).
2.  Compare it against zero.
3.  Predict/branch on the result.

As [§1.1](#what-a-cpu-actually-does-with-instructions) worked out in
detail, modern CPUs predict step 3 well, but “well” is not “free”:
prediction hides the misprediction penalty, not the **guaranteed memory
load** in step 1. When the check sits in a hot path that runs millions
of times a second (scheduler, networking, every `trace_*()` site), that
unavoidable load adds up.

**Jump labels remove the load and the compare for the common case** by
rewriting the machine code at runtime. When the feature is off, the hot
path literally has no branch to the rare code — it is a `nop` (or an
unconditional `jmp` over an out-of-line block, depending on polarity).
When someone turns the feature on, the kernel walks every call site for
that key and overwrites `nop`↔`jmp` in place.

Tradeoff in one sentence: **toggling is expensive** (machine-wide sync,
text poke); **running the hot path is nearly free**.

------------------------------------------------------------------------

## 3 The mental model, in one diagram {#the-mental-model-in-one-diagram}

[§2](#the-problem-jump-labels-solve) described the tradeoff in words;
here is the same idea as the literal code the compiler produces for one
`if`, side by side in both of its two possible forms:

       SOURCE CODE                     FEATURE OFF (common)         FEATURE ON
       ------------                    --------------------         ----------
       if (static_branch_unlikely      nop  (2 or 5 bytes)          jmp .Lout_of_line
           (&my_key)) {                ...normal path...            ...normal path...
               rare_code();            .Lout_of_line:               .Lout_of_line:
       }                                   rare_code();                 rare_code();
                                           jmp back                     jmp back
                                       (unreachable without a jmp)

Both columns are compiled from the exact same source line — nothing
about the C code changes. What changes is which of the two
already-compiled forms happens to be sitting in memory at any given
moment, decided entirely by whether the key is currently enabled.
“FEATURE OFF” has no branch at all: `rare_code()` still exists in the
binary, but with no `jmp` pointing at it, normal execution can never
reach it — that is what “(unreachable without a jmp)” is calling out.
The `nop` is exactly as wide as the `jmp` it might become (**2 or 5
bytes** on x86_64 — see
[§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes)), so turning
one column into the other is an atomic in-place replacement of equal
length, not a resize.

That `jmp back` at the end of the out-of-line block is not part of the
jump-label machinery at all. `rare_code()` here is a real C label
([§6](#what-the-compiler-emits-x86_64) covers how `asm goto` reaches
it), and in the C source, execution simply falls through from that label
into whatever statement follows the `if`. Since the compiler physically
moved the labeled block elsewhere in the function, it has to end that
block with an ordinary unconditional `jmp` back to wherever it placed
the following statement — the same technique a compiler uses to lay out
any unlikely branch, nothing specific to jump labels. That returning
jump is fixed at compile time and never patched by anything in this
tutorial; only the `nop`/`jmp` at the top of the site ever changes.

| Key state | Instruction in the hot path | Cost when not taking the rare path | Cost when taking it |
|----|----|----|----|
| disabled (for an `unlikely` site) | `nop` | ~0 (no load, trivial decode) | N/A — nothing patched in points at `rare_code()`, so this column cannot happen |
| enabled | `jmp <out-of-line>` | one unconditional jump | jump + rare code + jump back |

Compare this to the naive version from
[§2](#the-problem-jump-labels-solve), which **always** pays the *load +
compare* price.

------------------------------------------------------------------------

## 4 How to use static keys (the cookbook) {#how-to-use-static-keys-the-cookbook}

One fact underlies almost every rule in this section: toggling a key is
not a cheap flag flip. It is real cross-CPU code patching — the
IPI-synchronized protocol walked through in
[§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish) — so every
API choice below is really a choice about how often that machinery gets
invoked and who is allowed to trigger it.

### 4.1 Minimal example {#minimal-example}

Before this section works through individual API choices - polarity,
refcounting, RO-after-init, rate limiting - it helps to see the entire
lifecycle of one key end to end: how it is declared, how the hot path
guards on it, and how something elsewhere flips it. This is a synthetic
example, not lifted from a real kernel file the way
[§4.10](#a-real-consumer-tracepoints) later is, but every line in it
corresponds to a real mechanism this tutorial covers elsewhere:

``` c
#include <linux/jump_label.h>

DEFINE_STATIC_KEY_FALSE(foo_key);

void hot_path(void)
{
        /* Fast path: compiled as NOP while the key is false. */
        if (static_branch_unlikely(&foo_key))
                do_rare_thing();

        do_common_work();
}

void foo_enable(void)
{
        static_branch_enable(&foo_key);   /* slow path: patches text */
}

void foo_disable(void)
{
        static_branch_disable(&foo_key);  /* slow path: patches text */
}
```

Every piece of that example maps onto a term from earlier or later
sections. `foo_key` compiles down to the
[`struct static_key_false`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L355)
from [§7](#core-data-structures) - a plain
[`atomic_t`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/types.h#L186)
wrapped in a type the compiler can tell apart from
[`static_key_true`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L351)
at compile time.

The `if` in `hot_path()` is the patchable site itself:
[`static_branch_unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L486)
picks the nop-by-default helper here because a `_FALSE` key paired with
`unlikely` is exactly the case where the hint agrees with the initial
value of the key, which [§5](#two-polarities-key-default-branch-hint)
and [§6](#what-the-compiler-emits-x86_64) work out in full - that
agreement is why the comment can already say “compiled as NOP” before
the key is ever toggled.

`foo_enable()`/`foo_disable()` are not part of the jump-label API
itself; wrapping
[`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)/[`static_branch_disable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L523)
in a subsystem-named function like this is a real pattern, not an
invented one -
[`__sched_core_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/sched/core.c#L473)
in the scheduler does exactly this around its own
[`DEFINE_STATIC_KEY_FALSE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L371).
Underneath that one-line call is the entire boot-vs-runtime patching
machinery covered in [§9](#life-of-a-static-key-boot-enable-disable) and
[§10](#x86-text-patching-the-gory-details).

The two comments - “Fast path” and “slow path” - are the entire tutorial
in miniature: everything before this example explains why the fast path
can be free, and everything after it explains what the slow path
actually has to do to make that true.

### 4.2 Choosing TRUE vs FALSE and likely vs unlikely {#choosing-true-vs-false-and-likely-vs-unlikely}

Getting this pair wrong is not a correctness bug — the code still runs —
it is a silent performance foot-gun. Pick the polarity that disagrees
with your steady state, and you get a real `jmp` sitting on the hot path
where a `nop` belonged. That mistake is invisible in any correctness
test, and it isn’t something you can fix at runtime: which of
[`arch_static_branch`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35)
or
[`arch_static_branch_jump`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L45)
gets compiled in ([§5](#two-polarities-key-default-branch-hint)) is
baked in at build time, not decided later by toggling the key. So it’s
worth asking two questions up front, rather than guessing:

1.  **What is the default at boot?** Most optional features start off →
    [`DEFINE_STATIC_KEY_FALSE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L371).
    Features that are on unless turned off →
    [`DEFINE_STATIC_KEY_TRUE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L362).
2.  **Which way does *this* `if` lean in steady state?** Use
    [`static_branch_unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L486)
    when the body is the rare path;
    [`static_branch_likely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L474)
    when the body is the common path.

You can mix them. A `_FALSE` key works with both `likely` and
`unlikely`; a `_TRUE` key does too. The kernel picks a compiled-in `nop`
or `jmp` so the **default** case is the cheap one
([§5](#two-polarities-key-default-branch-hint)).

Rule of thumb for most new code:

``` c
DEFINE_STATIC_KEY_FALSE(feature_key);

if (static_branch_unlikely(&feature_key))
        rare_enabled_path();
```

### 4.3 Boolean enable vs refcounted enable {#boolean-enable-vs-refcounted-enable}

This distinction exists because of a failure mode that shows up the
moment a key has more than one owner. Say two independent subsystems
both call
[`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)
on the same key, each wanting the feature on for its own reasons.
Whichever one finishes first and calls
[`static_branch_disable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L523)
turns the feature off for both — out from under the other subsystem,
which has no way to know its dependency just vanished. Refcounting
exists precisely to make “enabled” mean “at least one owner still wants
this,” instead of “the opinion of the last caller wins”:

| API | Semantics | When to use |
|----|----|----|
| [`static_branch_enable`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522) / [`disable`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L523) | Force enabled count to 1 or 0 | Single owner; simple on/off |
| [`static_branch_inc`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L513) / [`dec`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L514) | Refcount; patch only on 0↔1 | Multiple independent users |
| [`static_branch_slow_dec_deferred`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label_ratelimit.h#L29) | Dec, but delay the 1→0 patch | Userspace-driven toggles |

`inc`/`dec` treat the key as “enabled iff count ≠ 0”. The first `inc`
(0→1) patches code on; the last `dec` (1→0) patches off. Intermediate
increments are cheap atomics with **no** text poke.

Do **not** mix `enable`/`disable` with `inc`/`dec` on the same key. Both
APIs act on the same underlying
[`key->enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
counter, but each assumes a different range of values is valid: the
boolean API assumes it only ever sees 0 or 1, while `inc`/`dec` is happy
to let it climb to any count of current owners.

If some other
[`inc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L513)
caller has already pushed the count to 2 or higher,
[`static_key_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L220)
and
[`static_key_disable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L245)
won’t corrupt that count — but they won’t do what the caller expects,
either.
[`enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L220)
treats “already above zero” as “already on” and returns immediately;
[`disable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L245)
finds the count isn’t the 1 it expects for a clean shutdown and also
returns without patching anything off. Both paths hit a
[`WARN_ON_ONCE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/bug.h#L118)
and silently no-op instead, leaving the feature exactly as it was — with
only a kernel warning to show that anything went wrong.

### 4.4 Reading the state without taking the branch {#reading-the-state-without-taking-the-branch}

Every example so far, including the `hot_path()` from
[§3.1](#minimal-example), uses
[`static_branch_likely`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L474)/[`unlikely`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L486)
as the condition of an `if` - the whole point of those macros is to
*become* the patched branch. Sometimes what a caller actually wants is
the current boolean value of the key as an ordinary expression, not a
branch to take.
[`static_key_enabled()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L407)
is the separate, deliberately unoptimized API for exactly that:

``` c
if (static_key_enabled(&foo_key))
        /* plain atomic read of the count — NOT the patched fast path */
```

Two real patterns from the tree show why this exists as a separate API,
rather than just “the slow way to write an `if`”.

The first is reporting state, not branching on it.
[`arch/x86/kernel/cpu/bugs.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/cpu/bugs.c#L1893)
logs which Spectre/IBPB mitigation got selected with
`pr_info(..., static_key_enabled(&switch_mm_always_ibpb) ? "always-on" : "conditional")`
— there is no hot-path branch here at all, just a boolean being
formatted into a string once, at boot.

The second is a control-plane guard.
[`drivers/md/dm-stats.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/drivers/md/dm-stats.c#L419)
does
`if (!static_key_enabled(&stats_enabled.key)) static_branch_enable(&stats_enabled);`
before calling the actual (expensive, IPI-synchronized,
[§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish)) enable
path, specifically to avoid re-triggering a full patch round when the
feature is already on.

Both cases want the current boolean value as an ordinary expression — to
print, compose, or make a one-off decision with — which is something
[`static_branch_likely`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L474)/[`unlikely`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L486)
aren’t really built for: their asm-goto trick expects to *be* the
condition of an `if`, not to hand back a `bool` you can store or reuse.

On an actual hot path, though, still prefer the branch macros so you get
the patched instruction. Using
[`static_key_enabled()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L407)
there instead means paying, on every single call, exactly the cache-line
load this whole tutorial opened by trying to eliminate
([§1.1](#what-a-cpu-actually-does-with-instructions)).

### 4.5 Keys must be global / static storage {#keys-must-be-global-static-storage}

A static key **cannot** live on the stack or be
[`kmalloc`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/slab.h#L1051)’d. The
compiler embeds its address into
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L209)
as a link-time constant (a relative offset, [§7.3](#relationship)) —
fixed once, at link time, for the life of the kernel image.

A stack-allocated key would work right up until its function returned:
the “distance to my key” offset of the patchable site would then point
at whatever now occupies that stack slot. A
[`kmalloc`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/slab.h#L1051)’d
key has the same problem the moment it is freed. Either way, the
corruption is silent until something happens to patch or read that site
again. Typical patterns:

``` c
DEFINE_STATIC_KEY_FALSE(global_key);           /* .data */
static DEFINE_STATIC_KEY_FALSE(file_local);    /* file scope */

/* header */
DECLARE_STATIC_KEY_FALSE(global_key);
```

Arrays:

``` c
DEFINE_STATIC_KEY_ARRAY_FALSE(keys, 4);
if (static_branch_unlikely(&keys[i]))
        ...
```

Conditional on Kconfig:

``` c
DEFINE_STATIC_KEY_MAYBE(CONFIG_FOO, foo_key);
/* expands to TRUE if CONFIG_FOO=y, else FALSE */

if (static_branch_maybe(CONFIG_FOO, &foo_key))
        ...
```

### 4.6 Read-only-after-init keys {#read-only-after-init-keys}

[§3.5](#keys-must-be-global-static-storage) covered keys that live for
the life of the running kernel and can be toggled at any point in it.
Some keys never need that: a mitigation decided once at boot and never
revisited benefits from a stronger guarantee than “nothing happens to
toggle it” - a guarantee that nothing *can*, even a bug.
`DEFINE_STATIC_KEY_FALSE_RO` (and its `_TRUE_RO` counterpart) provide
exactly that:

``` c
DEFINE_STATIC_KEY_FALSE_RO(configured_once_at_boot);
```

Placed in
[`__ro_after_init`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/cache.h#L60).
You may still
[`static_branch_enable`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)/[`disable`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L523)
**during `__init`** (before
[`mark_rodata_ro()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/init_64.c#L1405)).
After that:

- The key struct — including
  [`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
  — is mapped read-only, so further enable/disable/inc/dec would fault
  on the atomic write.
- [`jump_label_init_ro()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L573)
  has **sealed** the key ([§11](#modules-the-trickiest-part)): cleared
  its `entries` pointer and set
  [`JUMP_TYPE_LINKED`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L196),
  so even if something could write
  [`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87),
  [`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
  would find no sites to patch.

Use `_RO` for “decide once at boot, then freeze” features (many security
/ mitigation toggles). If you need to flip the key at runtime for the
life of the system, use plain `DEFINE_STATIC_KEY_*`, not `_RO`.

That freeze is hardware-enforced. After
[`mark_rodata_ro()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/init_64.c#L1405)
runs, an attacker who has already won an arbitrary-write primitive
elsewhere still cannot flip the key of a hardened mitigation, because
[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
sits in genuinely read-only memory — the write faults at the hardware
level, the same way any other write to `.rodata` would.

### 4.7 Rate-limited disable (userspace-facing knobs) {#rate-limited-disable-userspace-facing-knobs}

If userspace can flip a feature rapidly, naively patching on every
toggle thrashes text and tanks performance — recall from
[§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish) that a
single toggle is a full three-phase IPI round to *every online CPU*, not
a local write. A sysctl or socket option a user flips and unflips in a
loop would otherwise turn into a machine-wide synchronization storm, one
round per flip.

The API below is deliberately **asymmetric**, and that asymmetry is the
whole design.
[`static_branch_deferred_inc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label_ratelimit.h#L97)
is nothing more than the ordinary, immediate
[`static_branch_inc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L513)
from [§4.3](#boolean-enable-vs-refcounted-enable): turning a feature
*on* is never delayed. If this key is guarding something like counting
or tracing, a delayed enable would mean silently missing whatever
happened during the delay. Turning it *off* is the side that can safely
wait, since a feature staying active a little longer than strictly
necessary is harmless.

[`static_branch_slow_dec_deferred()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label_ratelimit.h#L29)
reflects that asymmetry with two distinct branches, and knowing which
one runs is the key to the whole mechanism:

- **Not the last reference** (the count is still above 1 after
  decrementing): this is a plain, cheap atomic decrement, no different
  from an ordinary `dec` in [§4.3](#boolean-enable-vs-refcounted-enable)
  — no timer, no patch, nothing deferred at all.
- **This decrement would be the last reference** (the count is at 1,
  about to hit 0): it does **not** decrement yet.
  [`static_key_dec_not_one()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L253)
  detects this case up front and deliberately leaves the count untouched
  at 1, then hands off to
  [`schedule_delayed_work()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/workqueue.h#L853),
  which arms a timer for `timeout` jiffies. Only once that timer
  actually fires does
  [`jump_label_update_timeout()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L325)
  run the real decrement, the one that can actually reach 0 and trigger
  the patch-off.

So for the entire `timeout` window, nothing about the feature has
changed: the count is still 1, still fully enabled, still fully patched.
That is what makes the coalescing work. If a fresh
[`inc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L513)
arrives during that window, the count moves from 1 to 2 with an ordinary
atomic increment (the fast path from
[§4.3](#boolean-enable-vs-refcounted-enable)) — *before* the timer has
fired. When the timer does eventually fire, it performs one ordinary
decrement exactly as if the earlier `dec()` call had never been the
final one. By now, it genuinely isn’t: the count drops from 2 to 1, not
from 1 to 0, so the “is this the transition to zero” check inside the
real dec path (the same `cmpxchg` pattern from
[§4.3](#boolean-enable-vs-refcounted-enable)/[§9.2](#enabling-static_key_enable-static_branch_enable))
never trips, and no patch happens. A rapid enable → disable → enable
sequence that lands entirely inside one `timeout` window therefore costs
zero IPI rounds, not one per toggle.

``` c
#include <linux/jump_label_ratelimit.h>

DEFINE_STATIC_KEY_DEFERRED_FALSE(sockopt_key, HZ);

/* enable immediately */
static_branch_deferred_inc(&sockopt_key);

/* disable — may wait up to `timeout` before actually patching off */
static_branch_slow_dec_deferred(&sockopt_key);

/* force pending delayed work to finish (e.g. module exit) */
static_key_deferred_flush(&sockopt_key);
```

That last call is not just tidiness.
[`static_key_deferred_flush()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label_ratelimit.h#L32)
blocks until any pending deferred disable has actually run. The
enclosing struct,
[`struct static_key_false_deferred`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label_ratelimit.h#L21),
bundles the key itself, the `timeout`, and the `delayed_work` together.
It usually lives in memory that is about to go away, e.g. a module being
unloaded. Free that memory with the timer still armed, and when it
eventually fires,
[`jump_label_update_timeout()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L325)
will run
[`container_of()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/container_of.h#L19)
on a `delayed_work` that no longer exists — a use-after-free, not merely
a stale toggle. Concretely:

``` c
struct static_key_false_deferred {
        struct static_key_false key;
        unsigned long timeout;
        struct delayed_work work;
};
```

### 4.8 CPU hotplug / deadlock rule {#cpu-hotplug-deadlock-rule}

[`static_branch_enable`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)`/`[`disable`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L523)`/inc/dec`
take
[`cpus_read_lock()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/cpu.c#L488)
(and
[`jump_label_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L23))
so that no CPU can come online mid-patch
([§9.2](#enabling-static_key_enable-static_branch_enable)). That lock is
not reentrant: a CPU hotplug notifier already runs with the write side
of that same lock held, so calling one of these functions from inside a
notifier would have the kernel try to take a lock it already holds — a
straightforward self-deadlock, not a rare race.

If you are already inside a hotplug callback that holds the hotplug
lock, use the `*_cpuslocked` variants, which skip re-acquiring it:

``` c
static_branch_enable_cpuslocked(&key);
static_branch_disable_cpuslocked(&key);
static_branch_inc_cpuslocked(&key);
static_branch_dec_cpuslocked(&key);
```

These are **not** general-purpose — only for that context.

### 4.9 What *not* to do {#what-not-to-do}

- Do not use the deprecated
  [`struct static_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L86) +
  [`static_key_true()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L204)/[`static_key_false()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L199)
  API in new code — use
  [`struct static_key_true`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L351)/[`struct static_key_false`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L355)
  wrappers and `static_branch_*()`.
- Do not toggle keys in tight loops unless you truly understand the cost
  (and usually: don’t) — “the cost” means the full IPI-synchronized
  patching protocol of
  [§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish) on every
  single toggle, run once per key change, not amortized in any way.
- Do not toggle keys from hardirq (or any other atomic) context at all.
  This one isn’t a cost tradeoff to weigh — it cannot work. Every
  enable/disable/inc/dec path takes
  [`cpus_read_lock()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/cpu.c#L488),
  and to actually patch,
  [`jump_label_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L23);
  both are sleeping locks. Calling any of them from a context that
  cannot sleep hits the same `might_sleep()` check that catches any
  other blocking call made from hardirq context, regardless of how
  rarely it happens to run.
- Do not expect `static_branch_*` to be a substitute for locks. It is a
  branch optimization, not a synchronization primitive for data.
- Do not take the branch before
  [`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
  has run (very early boot);
  [`STATIC_KEY_CHECK_USE()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L82)
  warns on enable/inc/dec too early.

### 4.10 A real consumer: tracepoints {#a-real-consumer-tracepoints}

The example in [§3.1](#minimal-example) was synthetic, built to show the
API shape end to end. Tracepoints are the real thing: a single
`DEFINE_TRACE()` can back dozens or hundreds of `trace_foo()` call sites
scattered across vmlinux and modules alike - the exact scale
[§9.6](#end-to-end-timeline-for-one-enable) worked through when
discussing IPI-round math - which makes the polarity choice from
[§3.2](#choosing-true-vs-false-and-likely-vs-unlikely) matter in
practice, not just in theory:

``` c
/* include/linux/tracepoint.h — __DECLARE_TRACE */
static inline void trace_##name(proto)
{
        if (static_branch_unlikely(&__tracepoint_##name.key))
                __do_trace_##name(args);
        ...
}
```

Each
[`DEFINE_TRACE()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/tracepoint.h#L428)
embeds a
[`struct static_key_false`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L355)
in `struct tracepoint`, initially false. Combined with the initial false
value of the key, `_unlikely` makes every dormant `trace_foo()` site a
`nop` in the hot path. Registering a probe enables the key and
live-patches every site (vmlinux + modules) to `jmp`.

------------------------------------------------------------------------

## 5 Two polarities: key default × branch hint {#two-polarities-key-default-branch-hint}

At the moment the compiler emits a patchable site,
[`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
has not run yet — there is no live
[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
count to read, only whatever initial value the key was declared with. So
the compiler has to pick *some* concrete instruction to put there, and
it does so by pretending the future has already happened. It computes
the exact same “does the branch hint agree with the value of the key?”
comparison that the runtime patcher
([§9](#life-of-a-static-key-boot-enable-disable)) will keep recomputing
every time the key actually toggles — just using the declared initial
value in place of the live one. That is why this section and the runtime
patching logic in [§9](#life-of-a-static-key-boot-enable-disable) end up
sharing one formula instead of two: the compile-time default isn’t a
special case, it is simply what the general rule produces on day one,
before anything has ever been enabled or disabled. Two independent bits
feed that formula:

1.  **Key initial value** —
    [`DEFINE_STATIC_KEY_TRUE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L362)
    vs `_FALSE`.
2.  **Call-site hint** —
    [`static_branch_likely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L474)
    vs
    [`_unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L486).

Get the two out of sync — the mistake
[§4.2](#choosing-true-vs-false-and-likely-vs-unlikely) warns about — and
the *default* compiled-in instruction is the expensive `jmp`, not the
free `nop`, for as long as the key stays at its initial value. The
kernel arranges that the default/common case uses `nop` (fall through),
and the uncommon case uses an out-of-line `jmp`. From the comment at
[`include/linux/jump_label.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h):

                  likely()                   unlikely()
                ------------                ------------
     key=true   ...                         ...
                NOP                         JMP L
                <br-stmts>               1: ...
            L:  ...
                                         L: <br-stmts>
                                            jmp 1b
     ------------------------------------------------------------
     key=false  ...                         ...
                JMP L                       NOP
                <br-stmts>               1: ...
            L:  ...
                                         L: <br-stmts>
                                            jmp 1b

| static key / branch macro | [`static_branch_likely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L474) | [`static_branch_unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L486) |
|----|----|----|
| [`DEFINE_STATIC_KEY_TRUE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L362) | `nop` → fall into body | `jmp` out to body |
| [`DEFINE_STATIC_KEY_FALSE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L371) | `jmp` out to body | `nop` → skip body |

Memorize: **when the hint matches the initial truth of the key, you get
`nop`; when they disagree, you get `jmp`.** That “agree vs. disagree”
phrasing is exactly what XOR computes on two booleans — 0 when its
inputs match, 1 when they don’t. That’s why the eight-row table below
collapses into one line of arithmetic, twice over: once for the live,
run-time-toggling case, and once for the frozen, compile-time snapshot
described above:

       enabled  type  branch    instruction
      --------------------------------------
         0       0      0     |   NOP
         0       0      1     |   JMP
         0       1      0     |   NOP
         0       1      1     |   JMP
         1       0      0     |   JMP
         1       0      1     |   NOP
         1       1      0     |   JMP
         1       1      1     |   NOP

        dynamic (at runtime): instruction = enabled ^ branch
        static  (at compile): instruction = type    ^ branch

(`type` = compile-time initial value;
[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
= live state; `branch` = 1 for `likely`, 0 for `unlikely`.) Implemented
as
[`jump_label_type()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L455)
/
[`jump_label_init_type()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L603)
in
[`kernel/jump_label.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c).
Here is the dynamic half of the formula, verbatim —
`jump_label_init_type()` has the same shape, just reading the
compile-time `type` bit of `key` instead of
[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87):

``` c
static enum jump_label_type jump_label_type(struct jump_entry *entry)
{
        struct static_key *key = jump_entry_key(entry);
        bool enabled = static_key_enabled(key);
        bool branch = jump_entry_is_branch(entry);

        return enabled ^ branch;
}
```

### 5.1 How the macros pick the asm {#how-the-macros-pick-the-asm}

[`static_branch_likely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L474)
and
[`static_branch_unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L486)
— the two public macros from [§4](#how-to-use-static-keys-the-cookbook)
that every call site actually uses — never touch a patchable site
directly. Underneath them sit two lower-level, arch-specific helper
*functions*:

- [`arch_static_branch`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35)
  is the **nop-by-default** helper: the site it patches starts out as a
  plain `nop` (or, thanks to the compile-time hack in
  [§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes), a `jmp` that
  `objtool` has already rewritten into a same-sized `nop` before the
  kernel even boots).
- [`arch_static_branch_jump`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L45)
  is the opposite: the **jmp-by-default** helper, which leaves a real
  `jmp` at the site from the moment the object file is built.

Both helpers report their result the same unusual way: not by loading
the state of the key from memory and computing a boolean, but by
reporting which of the two hard-wired paths execution actually took
*through the patched instruction itself*. Concretely: if the byte or
bytes currently sitting at the patchable site are a `nop`, the CPU just
falls through to the next instruction, and the call reports `false`; if
they are a `jmp`, the CPU diverts to the out-of-line block containing
the body of the branch, and the call reports `true`.
([§6.1](#the-two-asm-helpers) shows the actual mechanism behind this —
an `asm goto` that lands on a label called `l_yes` when the jump is
taken.)

So the return value tells you which physical instruction is at the site
*right now*, for either helper. It does not yet tell you whether that
corresponds to “the if-body runs” for the macro that called it — that
translation is what the `!` fix-up in the next paragraph is for.

There are four (key type, branch hint) combinations — TRUE/FALSE key
crossed with likely/unlikely — but, as the list above shows, only two
distinct patchable-site *shapes* to choose from. So the entire job of
[`static_branch_likely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L474)/
[`_unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L486)
is collapsing those four cases onto the two helpers without losing any
information. That collapse happens in two independent steps. **Which
helper to call** is decided by the TRUE/FALSE type of the key, matching
each site to the shape it actually needs. **Whether to negate the
result** is decided by likely vs. unlikely, for a reason the paragraph
right after the code below walks through in detail:

``` c
/* CONFIG_JUMP_LABEL path in jump_label.h */
static_branch_likely(x):
  TRUE  key → !arch_static_branch(&(x)->key, true)       /* nop-default site */
  FALSE key → !arch_static_branch_jump(&(x)->key, true)  /* jmp-default site */

static_branch_unlikely(x):
  TRUE  key →  arch_static_branch_jump(&(x)->key, false) /* jmp-default site */
  FALSE key →  arch_static_branch(&(x)->key, false)      /* nop-default site */
```

That table is exactly the XOR formula from
[§5](#two-polarities-key-default-branch-hint) written out one case at a
time:
[`arch_static_branch`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35)
(nop-default) is the one actually called whenever `type ^ branch == 0` —
the branch hint agrees with the compiled-in value of the key, so the
cheap, fall-through case is the one that needs no jump at all.
[`arch_static_branch_jump`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L45)
(jmp-default) is called whenever `type ^ branch == 1` — the hint
disagrees with the compiled-in value of the key, so reaching the if-body
requires an actual jump even before the key is ever toggled.

The lone `!` in front of the two `likely` calls is not a mistake; it is
a polarity fix-up. Its job is to make the *meaning* of the returned
boolean — “does the if-body run?” — consistent regardless of which of
the two helpers above happened to be chosen, since the two helpers
report their own fall-through-vs-jump outcome, not “is the key enabled.”

Walking through one case makes this clear. For a TRUE key read with
`likely`, the code table above calls the nop-default helper,
[`arch_static_branch`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35)
— and here, the compiled-in default really is a `nop`. Per the calling
convention above (`nop` → falls through → `false`),
[`arch_static_branch`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35)
returns `false` in this common case, even though the if-body *does* run.
The `false` is only reporting “no jump was taken,” not “the condition is
false.” Negating that mismatch, `!false == true`, turns “no jump was
taken” back into the answer the macro actually needs: “yes, the if-body
runs” — correct, even though zero jumps were taken to get there.

The `true`/`false` argument passed alongside `&(x)->key` does more than
feed that fix-up math, though. The call site doesn’t just *use* the
branch hint once, at compile time, to help pick `nop` vs. `jmp` — it
also writes that same hint permanently into the small piece of metadata
the compiler emits for this patchable site (called a *jump table entry*;
[§6.2](#the-jump-table-entry-sidecar-metadata) shows what it looks like
in the raw asm, and [§7.2](#struct-jump_entry-relative-form) covers its
layout in detail). That matters because the `type ^ branch` formula from
[§5](#two-polarities-key-default-branch-hint) is not a one-time,
compile-time calculation: the runtime patcher recomputes it every time
the live state of the key changes
([§9](#life-of-a-static-key-boot-enable-disable)), and it needs to read
`branch` back from somewhere at that point — this stored hint is where
it comes from.

[`__builtin_types_compatible_p`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html)
distinguishes
[`struct static_key_true`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L351)
vs
[`struct static_key_false`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L355)
at compile time; anything else calls
[`____wrong_branch_error()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L405)
(unresolved symbol → link error).

------------------------------------------------------------------------

## 6 What the compiler emits (x86_64) {#what-the-compiler-emits-x86_64}

### 6.1 The two asm helpers {#the-two-asm-helpers}

[§5.1](#how-the-macros-pick-the-asm) treated
[`arch_static_branch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35)/[`arch_static_branch_jump()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L45)
as black boxes that return `false` or `true`; this is what is actually
inside them, from
[`arch/x86/include/asm/jump_label.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h):

``` c
static __always_inline bool arch_static_branch(struct static_key * const key,
                                               const bool branch)
{
        asm goto(ARCH_STATIC_BRANCH_ASM("%c0 + %c1", "%l[l_yes]")
                : :  "i" (key), "i" (branch) : : l_yes);

        return false;
l_yes:
        return true;
}

static __always_inline bool arch_static_branch_jump(struct static_key * const key,
                                                    const bool branch)
{
        asm goto("1:"
                "jmp %l[l_yes]\n\t"
                JUMP_TABLE_ENTRY("%c0 + %c1", "%l[l_yes]")
                : :  "i" (key), "i" (branch) : : l_yes);

        return false;
l_yes:
        return true;
}
```

Notice these are two separate C functions, rather than one shared
function that takes an extra argument telling it whether to build a
`nop`-shaped site or a `jmp`-shaped one — something like an imagined
`arch_static_branch(key, branch, use_jmp)`. That design is not just
unused, it is impossible here: the assembly text inside each function is
fixed, literal text, embedded straight into the compiled function body
at build time. A runtime argument (a value only known while the kernel
is running) cannot make a single function body sometimes contain one
instruction and sometimes another — the bytes are baked in for good the
moment this file is compiled.

So instead, the two possible shapes get two separate functions, each
with its own hard-coded assembly. The choice between them is made where
the macros in [§5.1](#how-the-macros-pick-the-asm) decide *which one to
call* — a decision the compiler makes once, from the TRUE/FALSE type of
the key, not something either function decides for itself while running.

The body of both functions is almost entirely raw GNU assembler (GAS)
text, handed to the compiler through the `asm goto` extension GCC
provides, instead of being written as ordinary C. If that syntax isn’t
already familiar, here is every piece of notation used below and in
[§6.2](#the-jump-table-entry-sidecar-metadata), explained once:

- **`asm goto(template : : inputs : : goto-labels)`** is a GCC extension
  to inline assembly. Plain `asm(...)` just runs some assembly, and
  control falls back into the next C statement afterward. `asm goto`
  adds one more option: the assembly can jump straight to a named C
  label, listed after the fourth colon. (There are five colon-separated
  sections in the full syntax — template, outputs, inputs, clobbers,
  goto-labels — and the two functions below leave the output section
  empty.) Falling off the end of the assembly behaves like an ordinary
  fall-through, landing on `return false;` right after the
  `asm goto(...)` statement. Jumping to the listed label instead runs
  whatever C code sits under it — here, `l_yes: return true;`. From the
  point of view of the compiler this genuinely is a two-outcome branch,
  just like an `if`, only implemented with literal machine instructions
  instead of C conditionals.
- **`%0`, `%1`, …** inside the assembly template text are placeholders
  the compiler substitutes with whatever it decided to do with each
  operand listed after the colons: a register name, a memory address, or
  (as here) a literal number. `%c0`/`%c1` are a stricter request: the
  `c` forces the compiler to print operand 0 (`key`) or operand 1
  (`branch`) as a bare constant, without any of the punctuation (such as
  the `$` prefix x86 uses on immediates) that would normally decorate a
  value. That is only possible because both operands are declared `"i"`
  below; the closing paragraph of this section explains why that
  matters.
- **`%l[l_yes]`** is the goto-specific counterpart of the same
  substitution mechanism: it prints the actual assembler label the
  compiler generated for the C label `l_yes` — a jump target, not a
  number or register.
- **Numeric local labels**, like the bare `1:` seen below, are
  disposable, reusable label names that GAS provides. `1:` simply means
  “label number 1, right here”; elsewhere in the same file, `1f` means
  “the next `1:` label going forward” and `1b` means “the nearest `1:`
  label going backward.” They exist so that a macro expanded many times
  over (as
  [`JUMP_TABLE_ENTRY`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L15)
  is, once per call site in the entire kernel) can define a label each
  time without needing a globally unique name for every expansion.
- **`.` (a lone dot)** is the *current location counter* of GAS: the
  address of whatever byte the assembler is about to emit right at that
  point in the file. An expression like `1b - .` therefore means “the
  address of that local label, minus the address of this exact spot” — a
  distance, not an absolute address.
  [§6.2](#the-jump-table-entry-sidecar-metadata) explains why the
  jump-table entry is written entirely in terms of such distances.
- Lines beginning with `.` that are not labels — `.long`, `.byte`,
  `.quad`, `.pushsection`, `.popsection`, `.balign` — are **assembler
  directives**, not CPU instructions. They tell the assembler to do
  something (emit N bytes of literal data, switch which section
  subsequent bytes are placed into, pad to an alignment boundary) rather
  than encoding an operation for the CPU to execute later. A `#` starts
  an end-of-line comment, the GAS equivalent of `//` in C.

With that vocabulary available,
[`arch_static_branch_jump()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L45)
is the more direct of the two functions to read, since its body is
written out plainly rather than hidden behind another macro. Ignoring
the surrounding C-string quoting, the assembly it hands to `asm goto`
is:

    1:
    jmp %l[l_yes]
    <jump table entry for this site, via JUMP_TABLE_ENTRY>

- `1:` defines the local label that the jump-table entry for this site
  will later point back to (the `code` field of that entry, per
  [§6.2](#the-jump-table-entry-sidecar-metadata)).
- `jmp %l[l_yes]` is a plain, unconditional x86 jump — no condition
  codes or flags are checked, the jump is always taken — to wherever the
  compiler placed the C code that sits under the `l_yes` label. This is
  the “jmp-by-default” instruction
  [§5](#two-polarities-key-default-branch-hint) keeps referring to: it
  exists, unconditionally, in the object file the moment this
  translation unit is assembled, before anything has been patched.
- `JUMP_TABLE_ENTRY("%c0 + %c1", "%l[l_yes]")` expands in place right
  there, splicing a jump-table entry describing *this exact* `1:`/`jmp`
  pair into the assembly stream.
  [§6.2](#the-jump-table-entry-sidecar-metadata) covers what that entry
  stores and why; the mechanical detail worth flagging here is the pair
  `.pushsection __jump_table, "aw"` / `.popsection` inside it, which
  temporarily redirects everything the assembler emits into a different
  ELF section
  ([`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/scripts/module.lds.S#L31),
  instead of `.text`) for just the handful of directives in between,
  then restores the previous section afterward. That is how the inline
  assembly of one function ends up contributing bytes to two entirely
  separate sections of the final binary, without needing two separate
  source locations.

[`arch_static_branch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35)
is built the same way, but its `1:`-and-instruction line is not written
out directly inside the function — it is delegated to the
[`ARCH_STATIC_BRANCH_ASM`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L26)
macro:

``` c
#ifdef CONFIG_HAVE_JUMP_LABEL_HACK
#define ARCH_STATIC_BRANCH_ASM(key, label)      \
    "1: jmp " label " # `objtool` NOPs this \n\t"   \
    JUMP_TABLE_ENTRY(key " + 2", label)
#else /* !CONFIG_HAVE_JUMP_LABEL_HACK */
#define ARCH_STATIC_BRANCH_ASM(key, label)      \
    "1: .byte " __stringify(BYTES_NOP5) "\n\t"  \
    JUMP_TABLE_ENTRY(key, label)
#endif /* CONFIG_HAVE_JUMP_LABEL_HACK */
```

That `#ifdef`/`#else` is resolved once, by the plain C preprocessor, at
build time — every kernel build takes exactly one of these two branches,
never both, depending on
[`HAVE_JUMP_LABEL_HACK`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L1399)
([§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes) explains what
decides which). The two branches emit genuinely different bytes at `1:`:

- **With the hack** (the normal case on x86_64): line for line, this is
  identical to
  [`arch_static_branch_jump()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L45)
  above (a real `1: jmp label`), except for the trailing
  `#`objtool`NOPs this` comment — purely a note to a human reading the
  disassembly, ignored by the assembler itself — and the `" + 2"`
  appended to the `key` text passed into
  [`JUMP_TABLE_ENTRY`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L15).
  That `+ 2` is ordinary integer addition, performed by the assembler
  while it computes the numeric value of the `key` field, not by the CPU
  at runtime. It sets bit 1 of that stored value as a signal for
  `objtool` to find and act on before the kernel ever boots
  ([§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes) and
  [§6.4](#assembly-level-picture) explain that signal in full).
- **Without the hack**: `1: .byte 0x0f,0x1f,0x44,0x00,0x00`. `.byte`
  tells the assembler “place these literal byte values here, unchanged.”
  There is no instruction mnemonic to assemble at all — the bytes are
  simply copied in as data. `__stringify(BYTES_NOP5)` is a C
  preprocessor trick that turns the macro name
  [`BYTES_NOP5`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/nops.h#L60)
  into that literal comma-separated text;
  [`BYTES_NOP5`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/nops.h#L60)
  itself expands to `0x0f,0x1f,0x44,0x00,0x00` on x86_64, the same five
  bytes already shown as the “5-byte NOP” in
  [§6.4](#assembly-level-picture). Those particular bytes happen to
  decode as a real, valid instruction (`nopl 0x0(%rax,%rax,1)`, a no-op
  wrapped in an unused addressing mode purely to pad it out to 5 bytes).
  But nothing about writing raw bytes with `.byte` *required* that — the
  kernel could have hard-coded any other sequence just as easily.

The last line shared by both `asm goto` statements —
`: :  "i" (key), "i" (branch) : : l_yes);` — is what makes `%c0`, `%c1`,
and `%l[l_yes]` meaningful in the first place. Reading it through the
(empty) output section, the input section lists two operands:
`"i" (key)` becomes operand `%0`, and `"i" (branch)` becomes operand
`%1`. `"i"` is a *constraint*: it tells the compiler “this operand must
end up as an immediate, compile-time-constant value” — as opposed to,
say, `"r"` (put it in a register) or `"m"` (leave it addressable in
memory).

That constraint is not a style choice: the jump-table entry from
[§6.2](#the-jump-table-entry-sidecar-metadata) needs the actual numeric
values of `key` and `branch` baked directly into `.long`/`.quad` data at
assembly time, and an immediate is the only kind of operand still
guaranteed to have a fixed, known value after this `__always_inline`
function has been inlined away and any register it might otherwise have
used no longer exists. The empty section right after the inputs is the
clobber list (nothing here touches any register or memory the compiler
isn’t already tracking), and the final `l_yes` is the goto-label list
mentioned above — the one part of this whole construct that has no
equivalent in ordinary `asm(...)`.

### 6.2 The jump table entry (sidecar metadata) {#the-jump-table-entry-sidecar-metadata}

``` c
#define JUMP_TABLE_ENTRY(key, label)                   \
        ".pushsection __jump_table,  \"aw\" \n\t"      \
        _ASM_ALIGN "\n\t"                              \
        ANNOTATE_DATA_SPECIAL "\n"                     \
        ".long 1b - . \n\t"                            \
        ".long " label " - . \n\t"                     \
        _ASM_PTR " " key " - . \n\t"                   \
        ".popsection \n\t"
```

This emits **no executable code**. It appends one
[`struct jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111)
into the
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/scripts/module.lds.S#L31)
ELF section:

| Field | Asm | Meaning |
|----|----|----|
| `code` | `.long 1b - .` | relative offset to the patchable insn |
| `target` | `.long label - .` | relative offset to the `l_yes` target |
| `key` | `_ASM_PTR key - .` | relative offset to the `static_key`, low bits = flags |

Every line in the macro is an assembler *directive*, in the sense
[§6.1](#the-two-asm-helpers) introduced — none of it is a CPU
instruction, and none of it ever executes. Reading it top to bottom with
the vocabulary of that section in hand:

- `.pushsection __jump_table, "aw"` switches which section the assembler
  is currently writing bytes into: the same directive
  [§6.1](#the-two-asm-helpers) mentioned, now with its second argument
  explained. That argument, `"aw"`, is a string of one-letter section
  *flags*. `a` means “allocatable”: these bytes occupy real memory once
  the kernel image is loaded, just like ordinary code or data, as
  opposed to bookkeeping that exists only during the build and never
  reaches the running kernel. `w` means “writable”: software is allowed
  to modify these bytes after the kernel has booted (contrast this with
  `.text`, which is flagged `"ax"` — allocatable *and* executable, but
  never writable). This is not just documentation. The page tables of
  the CPU actually enforce whether a page of memory can be written to,
  so getting this flag right is what makes it *possible* for anything to
  later modify the contents of this table.
- `_ASM_ALIGN` expands to a plain `.balign 8` on x86_64 (`.balign 4` on
  32-bit). `.balign N` pads forward — with filler bytes the CPU will
  never execute — until the current location is a multiple of `N`. It is
  needed here because the `key` field below is a native-width (8-byte,
  on x86_64) value, and leaving it unaligned, while not illegal on x86,
  is worth avoiding.
- [`ANNOTATE_DATA_SPECIAL`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/annotate.h#L112)
  is itself a small assembler macro (its own expansion is a level of
  detail this tutorial won’t unpack) that drops a short note into a
  separate, build-time-only section read exclusively by objtool. Its
  only job is telling `objtool` “what comes next is data, not
  instructions — do not try to disassemble it.” Without it, `objtool`
  would walk into this section exactly the way it walks through `.text`,
  and fail trying to decode raw offset numbers as if they were machine
  code.
- `.long 1b - .` is the first real data field. `.long`, one of the
  directives [§6.1](#the-two-asm-helpers) named, emits exactly 4 bytes
  holding the value of whatever expression follows it — here, `1b - .`.
  Both symbols are the ones [§6.1](#the-two-asm-helpers) introduced:
  `1b` is the address of the nearest preceding `1:` label (the patchable
  `nop`/`jmp` site itself), and `.` is the address of this very `.long`
  directive. GAS can evaluate `-` between two addresses like this at
  assembly time. The result is not an address at all — it is a plain,
  small, signed distance in bytes from one spot in the file to the
  other.
- `.long " label " - ."` repeats the exact same pattern for `label`,
  which by this point has been substituted with `%l[l_yes]` — the
  assembler label the compiler generated for the `l_yes` C label. Same
  directive, same “distance from here” arithmetic, just a different
  destination address.
- `_ASM_PTR " " key " - ."` is the third field, with one difference:
  `_ASM_PTR` expands to `.quad` on x86_64, a directive that emits 8
  bytes rather than the fixed 4 that `.long` emits. `key` here is the
  text `"%c0 + %c1"` (or `"%c0 + %c1 + 2"`,
  [§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes) explains the
  `+ 2`). Once the compiler substitutes real values for `%c0` and `%c1`,
  this directive ends up computing something like
  `.quad (address of the static_key) + 1 - .`: an address, plus a small
  integer, minus the current location. All of that is arithmetic GAS can
  carry out on symbols at assembly time, and it still collapses to one
  8-byte distance value in the end. That `+ %c1` is what **ORs the
  branch hint into bit 0** of the stored value — the same bit
  [`jump_entry_is_branch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L153)
  reads back later.
- `.popsection` undoes the `.pushsection` from the first line, restoring
  whichever section — almost always `.text` — was active before this
  macro was expanded, exactly as in [§6.1](#the-two-asm-helpers).

Stepping back from the line-by-line reading: `code`/`target` use `.long`
(a fixed 32 bits) while `key` uses `_ASM_PTR` (native pointer width — 64
bits on x86_64). That is not an inconsistency: the patchable instruction
and its own `l_yes` label are always emitted right next to each other,
in the same function, so a 32-bit displacement can’t help but reach. The
`static_key` itself carries no such guarantee. As the comment on
[`struct jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111)
itself puts it, the key “may be far away from the core kernel under
KASLR” — or it may simply live in a module loaded who-knows-where
relative to this call site. That one field needs the full address range
a 32-bit offset couldn’t promise.

Each of those three fields is a **self-relative** offset. The natural
first guess for what that means is “distance from the start of the
`struct jump_entry`” — i.e., you’d find the instruction by taking *the
starting address of the entry itself* and adding `code` to it. That is
not what actually happens. Instead, each field stores its distance to
whatever it points at, measured from **the address of that one field
itself** — not the address of the struct, not the address of the array,
but the address of that specific 4-or-8-byte field. It sounds like a
small difference, but it is the reason all three fields, despite sitting
at different offsets inside the struct, can be read back by the exact
same recipe. Working through the first field concretely:

       __jump_table[i].code is a field, and like any variable, it lives
       somewhere in memory. Call that address F.

       The patchable nop/jmp instruction this entry describes lives at
       some other address, C, out in .text.

       What .pushsection/.long actually wrote into the .code field, back
       at assembly time (§5.2), is the *distance* between those two
       addresses:

           value stored in entry->code  =  C - F

       To go the other way — given only the entry, find C — you need F
       again. But F is just "the address of this field", which C code can
       always get with the & operator:

           C  =  &entry->code  +  entry->code
                  ^^^^^^^^^^^^    ^^^^^^^^^^^^
                  this field's     the distance
                  own address      stored in it

       That expression is, verbatim, the body of jump_entry_code():
       "my own address, plus whatever I'm holding."

`.target` (distance to `l_yes`) and `.key` (distance to the
`static_key`, with 2 low bits reserved as flags — see
[§7.2](#struct-jump_entry-relative-form)) are computed the same way,
each from *its own* field address rather than that of the struct:
[`jump_entry_target()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L122)
is `&entry->target + entry->target`, and
[`jump_entry_key()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L127)
is the same shape with the flag bits masked off first. Because each
field carries its *own* address into the calculation, none of these
three functions needs to know or care where inside the struct its field
happens to sit. “Struct-start-relative” offsets would have needed a
different fixed adjustment hard-coded per field to account for that.
“Field-address-relative” offsets don’t, by construction.

This is also precisely what makes the table survive KASLR
([§7.2](#struct-jump_entry-relative-form)) with zero patching at boot.
Say the whole kernel image ends up loaded 0x1000 bytes higher than the
linker originally assumed. Every address in it — including both `F` and
`C` above — shifts by that same +0x1000, because the entire image moves
as one block. The *difference* `C - F`, which is the only thing ever
actually stored, does not change at all. There is nothing here for
[`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
to go fix up on this front. The encoding is already correct no matter
where the kernel ends up in memory.

### 6.3 [`HAVE_JUMP_LABEL_HACK`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L1399): why sites are 2 *or* 5 bytes {#have_jump_label_hack-why-sites-are-2-or-5-bytes}

[§6.1](#the-two-asm-helpers) showed the nop-default site of
[`arch_static_branch`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35)
written as a hand-coded `.byte BYTES_NOP5` — a fixed 5-byte NOP — but
that is not the size that actually lands in a compiled kernel. Depending
on the call site, the NOP that ends up in memory can be either a compact
2-byte instruction or the full 5-byte one, and which of the two you get
is settled long before the kernel ever boots, by a build-time trick this
section walks through in full: get the compiler to emit a real `jmp` (so
it can pick whichever encoding is actually shortest for that call site),
then have a separate build step convert that `jmp` into a NOP of the
*same* size, after compilation but before the kernel image is final.

That trick is gated by a single config option,
[`HAVE_JUMP_LABEL_HACK`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L1399).
[`arch/x86/Kconfig`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/Kconfig)
turns it on for any build with objtool available
(`select HAVE_JUMP_LABEL_HACK if HAVE_OBJTOOL`) — which, in practice,
means every modern x86_64 build. Recall
[`ARCH_STATIC_BRANCH_ASM`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L26)
from [§6.1](#the-two-asm-helpers), which branches on this exact config
option:

``` c
#ifdef CONFIG_HAVE_JUMP_LABEL_HACK
#define ARCH_STATIC_BRANCH_ASM(key, label)             \
        "1: jmp " label " # `objtool` NOPs this \n\t"    \
        JUMP_TABLE_ENTRY(key " + 2", label)
#else
#define ARCH_STATIC_BRANCH_ASM(key, label)             \
        "1: .byte " __stringify(BYTES_NOP5) "\n\t"     \
        JUMP_TABLE_ENTRY(key, label)
#endif
```

The hack exists to solve a problem the compiler cannot: a hand-written
`.byte BYTES_NOP5` always produces a 5-byte NOP, even for sites that
will only ever need the shorter 2-byte form, wasting I-cache space on
every nop-default site in the kernel. The fix is to have the compiler
emit a real, correctly-sized `jmp` — the compiler already knows how to
pick the shortest encoding that reaches its target — and then convert
that `jmp` into a NOP of the *same* size after the fact, once, at build
time. Concretely, with the hack enabled (the normal case on x86_64):

1.  The compiler emits a **real `jmp`** to `l_yes`, exactly as it would
    for an ordinary conditional branch. x86 has two different encodings
    for an unconditional jump, and the assembler is free to pick
    whichever one actually fits. `JMP rel8` is a 1-byte opcode (`0xEB`)
    followed by a single signed byte saying “how many bytes forward or
    backward from the *next* instruction”: 2 bytes total, usable only if
    the target is between 128 bytes behind and 127 bytes ahead of that
    next instruction (the range of a signed byte, -128 to +127).
    `JMP rel32` is a different 1-byte opcode (`0xE9`) followed by a
    4-byte signed displacement using the same next-instruction-relative
    scheme: 5 bytes total, but able to reach anywhere in a 64-bit kernel
    image.[^1] (“`rel8`”/“`rel32`” here just names the size, in bits, of
    that displacement number — the same self-relative idea
    [§6.1](#the-two-asm-helpers) and
    [§6.2](#the-jump-table-entry-sidecar-metadata) used for jump-table
    fields, just encoded directly in an instruction instead of stored as
    separate metadata.) The assembler already knows the real distance to
    `l_yes` when it assembles this function, so it picks the shorter
    `rel8` form whenever that distance allows it, and only falls back to
    `rel32` when the target is too far. The size, in other words, is
    already whatever is optimal for that specific call site.
2.  The jump-table key expression becomes `"%c0 + %c1 + 2"`, which
    simply sets bit 1 of the stored key value. That bit is not consumed
    by anything at runtime; it exists purely as a signal for the next
    step.
3.  **objtool**
    ([`tools/objtool/check.c:handle_jump_alt`](https://elixir.bootlin.com/linux/v7.2-rc7/source/tools/objtool/check.c#L1872)),
    which runs once over the compiled object files as part of the build,
    sees `key_addend & 2` set, and rewrites that `jmp` in place into a
    **same-sized NOP**, clearing the relocation that would otherwise
    have pointed at it. This happens entirely at build time — by the
    time the kernel boots, the bytes are already NOPs, and `objtool`
    itself is long gone. Bit 1 having done its job as an objtool-only
    signal,
    [`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
    later repurposes that same bit position for something unrelated —
    the “`__init` text” flag, via
    [`jump_entry_set_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L163)
    — since nothing at runtime ever needs the original meaning again.

By the time the kernel image is built — `objtool` has already run, long
before the kernel ever boots — every nop-default site (the ones written
with
[`arch_static_branch`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35))
contains a real NOP instruction, and it is already the smallest one that
fits: 2 bytes where the `jmp` it replaced used the `rel8` encoding, 5
bytes where it used `rel32`. Nothing shrinks or grows it later; boot
time just inherits whatever `objtool` left behind.

The catch is that
[`struct jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111)
([§6.2](#the-jump-table-entry-sidecar-metadata)) never records which of
the two sizes a given site ended up with. It could not: the size is only
decided when `objtool` runs, which is after the on-disk layout of the
struct has already been fixed by the compiler.

So later, whenever the kernel actually needs to patch a site — flipping
a NOP to a `jmp` or back, in response to
[`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)/[`static_branch_disable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L523)
— it cannot just trust a stored number. It has to look at the real bytes
sitting in memory and work out for itself whether they encode a 2-byte
or a 5-byte instruction, then build a same-sized replacement so the
layout of the surrounding code does not shift. That decode-then-patch
machinery
([`arch_jump_entry_size()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L20),
[`insn_decode_kernel()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/insn.h#L172),
[`__jump_label_patch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L36))
is exactly what [§8](#size-of-the-patchable-site-on-x86-runtime) walks
through in full; the point to take away here is only that whatever size
`objtool` committed to at build time is faithfully rediscovered and
reproduced at every later patch, so a site never ends up needing more
room than the code around it left for it.

Builds without the hack
([`CONFIG_HAVE_JUMP_LABEL_HACK`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L1399)
unset — older toolchains without `objtool` support) never go through any
of this. They use the other
[`ARCH_STATIC_BRANCH_ASM`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L26)
branch shown at the top of this section: a hand-written
`.byte BYTES_NOP5` that always emits a fixed 5-byte NOP, no matter how
close the target label actually is, and patching such a site later
always installs a 5-byte `JMP32` to match. The result is still correct —
the branch still works exactly the same way — it just never gets the
chance to use the shorter 2-byte encoding, so every nop-default site
costs 3 extra bytes of I-cache footprint compared to a hack-enabled
build.

That whole `objtool` rewrite (the numbered steps above) only applies to
[`arch_static_branch`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35),
the nop-default helper from [§6.1](#the-two-asm-helpers).
[`arch_static_branch_jump`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L45),
the jmp-default helper, builds its `asm goto` directly (also shown in
[§6.1](#the-two-asm-helpers)) instead of going through
[`ARCH_STATIC_BRANCH_ASM`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L26),
and its
[`JUMP_TABLE_ENTRY`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L15)
key expression is the plain `"%c0 + %c1"` — no `+ 2` added.

With no such tag on the entry, `objtool` has nothing telling it to touch
that `jmp`, so it is left alone and reaches boot as a real `jmp`,
exactly as a jmp-default site is supposed to (it stays a jump until
something calls
[`static_branch_disable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L523)
on it). That `jmp` is still free to come out as either the 2-byte `rel8`
form or the 5-byte `rel32` form, by the same distance-based assembler
choice described in step 1 above — the hack changes whether `objtool`
converts the instruction afterward, not which of the two `jmp` encodings
the assembler reaches for in the first place.

Bit 1 of that key field — the same low-order bits
[`jump_entry_key()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L127)
masks off before turning it into a
[`struct static_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L86)
pointer ([§6.2](#the-jump-table-entry-sidecar-metadata)) — does double
duty. During the build it is a private signal for `objtool` to convert a
`jmp` into a NOP (step 2 above). From boot onward, that same bit is
repurposed as what
[`jump_entry_is_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L158)
reads back as the `__init`-text flag ([§9.1](#boot-jump_label_init)).
Conflating the two is an easy way to misread this code cold:

      build time  --------------------------------->  boot time  ---> forever after

      bit 1 = "objtool: NOP this jmp"        jump_label_init() unconditionally
      (only meaningful to objtool;           OVERWRITES it to mean:
       consumed and discarded before                bit 1 = "jump_entry_is_init"
       the kernel ever boots)                (site is in __init text, unpatchable
                                              once init memory is freed — §8.1)

That boot-time flag is set by
[`jump_entry_set_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L163)
inside
[`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
([§9.1](#boot-jump_label_init)), and read back by
[`jump_entry_is_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L158)
— the same bit, doing unrelated jobs on either side of the build/boot
line shown above.

### 6.4 Assembly-level picture {#assembly-level-picture}

[§6.1](#the-two-asm-helpers) gave the two C helpers,
[§6.2](#the-jump-table-entry-sidecar-metadata) gave the jump-table entry
they emit, and [§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes)
explained why the bytes at the patchable site itself aren’t fixed. Put
together, for a **nop-default** site on a
[`HAVE_JUMP_LABEL_HACK`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L1399)
build (the normal case on x86_64), this is everything that lands in the
object file:

    .text:
            1:  0f 1f 44 00 00  ; 5-byte NOP if l_yes was far, OR 66 90 (2-byte) if
                ...             ; close (objtool rewrote a real `jmp` into this,
                                ; at build time — §5.3)

    __jump_table:                   ; non-executable metadata, in __jump_table (§5.2)
            .long   1b - .          ; code:   self-relative offset to the NOP above
            .long   L - .           ; target: self-relative offset to l_yes
            .quad   key+branch+2 - .; key: static_key address, branch bit set (§5.2),
                                    ;      plus the objtool-only "+2" signal (§5.3)

The two possible byte sequences shown for the `1:` label in `.text` are
genuinely different instructions, not interchangeable padding:
`0f 1f 44 00 00` is the same 5-byte NOP named in
[§6.1](#the-two-asm-helpers) (`nopl 0x0(%rax,%rax,1)` — a multi-byte
no-op instruction, dressed up with an unused addressing mode purely to
reach 5 bytes). `66 90` is the 2-byte alternative. `0x66` is the
*operand-size override prefix* of x86, normally used to shrink the
operand of a following instruction from 32 to 16 bits, stacked in front
of `0x90`, the classic single-byte `NOP` (historically the opcode for
`XCHG AX, AX`). Combining them doesn’t change what the CPU actually
does; the prefix is there purely to pad the encoding out to exactly 2
bytes. Either sequence is a genuine no-op either way: the CPU decodes
it, spends a cycle or so, and falls straight through to whatever comes
next, leaving every register and flag untouched.

Two things in that second block are easy to misread if you haven’t just
finished
[§6.2](#the-jump-table-entry-sidecar-metadata)–[§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes),
so it’s worth naming them explicitly:

- The width of the NOP (2 or 5 bytes) is **not** a compile-time toss-up
  written directly by the compiler — without the hack,
  [`ARCH_STATIC_BRANCH_ASM`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L26)
  always emits a hard-coded 5-byte NOP and nothing else ever touches it
  at build time. *With* the hack, what the compiler actually writes here
  is a real `jmp`, sized 2 or 5 bytes by the assembler based on the real
  distance to `l_yes`; `objtool` only turns that already-correctly-sized
  `jmp` into a same-sized NOP afterward. Either way, by the time this
  object file is linked into a kernel, the bytes shown above are final —
  nothing patches them again until this key is actually toggled at
  runtime.
- `+2` is not a fourth polarity option alongside `key`/`branch` — it is
  purely the build-time-only “objtool, please NOP this” signal from
  [§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes), living in
  bit 1 of the same field that carries the branch hint in bit 0. It
  means nothing by the time the kernel is running;
  [`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
  overwrites that same bit with an unrelated meaning
  ([§9.1](#boot-jump_label_init)) once boot begins.

One more x86-specific config bit is worth flagging before moving past
“what the compiler emits” to “what the kernel does with it”:
[`HAVE_JUMP_LABEL_BATCH`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L5)
is also defined on x86, which is what makes the batched, IPI-amortized
patch path in
[§9](#life-of-a-static-key-boot-enable-disable)–[§10](#x86-text-patching-the-gory-details)
available at all — without it, every jump-table entry would have to be
patched (and synchronized across every CPU) one at a time.

------------------------------------------------------------------------

## 7 Core data structures {#core-data-structures}

### 7.1 [`struct static_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L86) {#struct-static_key}

Every key you `DEFINE_STATIC_KEY_{TRUE,FALSE}` in C boils down, at
runtime, to this same underlying struct — regardless of which macro you
used (the `TRUE`/`FALSE` wrapper types that keep them distinct at
compile time show up later in this section):

``` c
struct static_key {
        atomic_t enabled;
#ifdef CONFIG_JUMP_LABEL
        union {
                unsigned long type;
                struct jump_entry *entries;
                struct static_key_mod *next;
        };
#endif
};
```

**[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)**
is the live state [§4](#how-to-use-static-keys-the-cookbook) and
[§5](#two-polarities-key-default-branch-hint) keep referring to: an
atomic refcount, `0` meaning off and any positive value meaning on (this
is what lets
[`static_branch_inc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L513)/[`_dec()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L514)
in [§4.3](#boolean-enable-vs-refcounted-enable) stack multiple owners on
one key). It can also transiently hold **`-1`**, which means “the first
[`static_key_slow_inc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L186)
on this key is in progress right now, patching the instruction stream”
([§9.2](#enabling-static_key_enable-static_branch_enable) covers that
window in detail). While that is happening, any other reader calling
[`static_key_count()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L104)
still needs to see “enabled” rather than a confusing negative number, so
that function maps `-1` back to `1` before returning it.

The second field is where this struct gets unusual: it is one word that
means two entirely different things, chosen by a tag bit hidden inside
it. That is only possible because pointers to
[`struct jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111)
and
[`struct static_key_mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L613)
are both at least 4-byte aligned in practice, which means their real
value always has its low 2 bits set to `0` — those 2 bits are free for
the taking, so the union borrows them to store extra information instead
of leaving them as always-zero padding:

| Bit | Macro | Meaning |
|----|----|----|
| 0 | [`JUMP_TYPE_TRUE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L195) | compile-time initial value of the key was `true` — this is the same `type` bit the `type ^ branch` formula from [§5](#two-polarities-key-default-branch-hint) uses |
| 1 | [`JUMP_TYPE_LINKED`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L196) | `1`: the rest of the word is a `next` pointer (a linked list); `0`: it is an `entries` pointer (a flat array) |

(Do not confuse this bit-packed word with the low 2 bits of
`jump_entry::key` itself from
[§6.2](#the-jump-table-entry-sidecar-metadata)/[§7.2](#struct-jump_entry-relative-form)
— same trick, applied twice, to two unrelated pointers in two unrelated
structs. One tags the type of *this* key itself and which pointer kind
it holds; the other tags the branch hint of a call site and its
init-section status.)

Bit 1 exists because a single, contiguous array is not always enough to
describe every call site for a key. For a key only ever used inside
`vmlinux` itself, the linker sees every call site at link time and can
sort them all into one contiguous run inside
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L209)
([§7.3](#relationship)) — `entries` just points at the start of that
run. But a key can also be used from inside a *module*, loaded long
after boot, with its own private
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/vmlinux.lds.h#L436)
section that was never linked against the main kernel image at all
([§7.4](#linker-section)). There is no way to splice the entries of a
module into the already-built vmlinux array after the fact, and modules
can be loaded and unloaded repeatedly over the lifetime of the kernel,
so the set of “all call sites for this key” can grow and shrink at
runtime. When that happens, the union switches meaning: `next` becomes
the head of a linked list of

``` c
struct static_key_mod {
        struct static_key_mod *next;
        struct jump_entry *entries;
        struct module *mod;
};
```

nodes — one node per module currently contributing call sites for this
key — rather than a direct pointer into one flat array.
[`JUMP_TYPE_LINKED`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L196)
records, for a given key, which of these two representations is
currently in effect.

None of the code outside
[`kernel/jump_label.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c)
has to know any of this: accessors like
[`static_key_entries()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L409),
[`static_key_type()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L415),
[`static_key_linked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L420),
and
[`static_key_set_entries()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L444)
mask these two bits off before handing the pointer to anyone else, so
the rest of the kernel just sees “the entries for this key,” never the
tag bits.

Finally, the two type wrappers from
[§5.1](#how-the-macros-pick-the-asm)’s
[`__builtin_types_compatible_p`](https://gcc.gnu.org/onlinedocs/gcc/Other-Builtins.html)
check are trivial by design — each is nothing but a
[`struct static_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L86)
in a differently-named box:

``` c
struct static_key_true  { struct static_key key; };
struct static_key_false { struct static_key key; };
```

Their entire purpose is to exist as two *distinct C types* the compiler
can tell apart at compile time, even though they carry identical data —
exactly what [§5.1](#how-the-macros-pick-the-asm)’s
[`static_branch_likely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L474)/[`_unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L486)
macros rely on to pick the right arch helper.

### 7.2 [`struct jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111) (relative form) {#struct-jump_entry-relative-form}

This is the struct [§6.2](#the-jump-table-entry-sidecar-metadata) has
already been building up piece by piece — the one
[`JUMP_TABLE_ENTRY`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L15)
writes one instance of, per call site, into
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/vmlinux.lds.h#L436).
x86 opts into a specific *variant* of it by selecting
[`CONFIG_HAVE_ARCH_JUMP_LABEL_RELATIVE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L509)
in its Kconfig, which is what makes the struct look like this:

``` c
struct jump_entry {
        s32 code;
        s32 target;
        long key;       /* full width: module↔vmlinux may be far under KASLR */
};
```

`code` and `target` hold the self-relative distances
[§6.2](#the-jump-table-entry-sidecar-metadata) walked through in detail
(`&entry->code + entry->code` recovers the real address of the patchable
instruction, and likewise for `target`/`l_yes`); `key`, with its low 2
bits masked off, recovers the address of the owning `static_key` the
same way. Architectures that do *not* select
[`HAVE_ARCH_JUMP_LABEL_RELATIVE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L509)
use a plainer struct instead, where all three fields simply *are*
absolute addresses — which is visible directly in the accessors of that
fallback itself (in
[`jump_entry_code()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L117)
there is just `return entry->code;`, no arithmetic at all). x86 pays the
small extra cost of the “address of self plus stored offset” computation
on every read in exchange for two things:

1.  **Size.** A kernel can easily contain tens of thousands of
    jump-table entries — one per call site. `s32` (4 bytes) for
    `code`/`target` instead of a full pointer-width field (8 bytes on
    x86_64) roughly halves the size of two-thirds of every entry,
    multiplied across the whole table.
2.  **KASLR immunity**, which
    [§6.2](#the-jump-table-entry-sidecar-metadata) already derived in
    full: since `code` and `target` are always within a couple of bytes
    of the single function they belong to, a 32-bit signed distance can
    always reach; recovering the address costs one addition instead of a
    boot-time fixup. `key` is kept at full pointer width (`long`, not
    `s32`) because it points at a `static_key`, which can live anywhere
    — including in a *different module* from the one containing this
    call site, or in `vmlinux` while the call site itself is in a module
    loaded who-knows-where in the address space
    ([§7.1](#struct-static_key)). The distance between two
    independently-placed pieces of memory like that can exceed what a
    signed 32-bit number can express, so this one field cannot be shrunk
    the way `code`/`target` were.

The low 2 bits stolen from `key` — the same trick
[§7.1](#struct-static_key) used on the pointer field of `static_key`
itself, applied here to a different pointer for a different purpose —
carry two more pieces of information:

| Bit | Meaning |
|----|----|
| 0 | [`jump_entry_is_branch`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L153): the `branch` hint from [§5](#two-polarities-key-default-branch-hint)/[§5.1](#how-the-macros-pick-the-asm) and the `"%c0 + %c1"` from [§6.2](#the-jump-table-entry-sidecar-metadata) — `1` if the call site used `likely()`, `0` for `unlikely()` |
| 1 | [`jump_entry_is_init`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L158): `1` if the code for this site lives in `__init` text — freed after boot, and therefore unpatchable from that point on |

Both accessors are exactly as small as a single bit-check suggests:

``` c
static inline bool jump_entry_is_branch(const struct jump_entry *entry)
{
        return (unsigned long)entry->key & 1UL;
}

static inline bool jump_entry_is_init(const struct jump_entry *entry)
{
        return (unsigned long)entry->key & 2UL;
}
```

Bit 1 is the same bit
[§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes) followed
through its “two lifetimes” story: at build time it briefly means
“objtool, turn this `jmp` into a `nop`,” and only *after*
[`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
has consumed that meaning and moved on does it settle into this second,
permanent meaning for the rest of the uptime of the kernel.

### 7.3 Relationship {#relationship}

[§7.1](#struct-static_key) and [§7.2](#struct-jump_entry-relative-form)
described the two structs in isolation; this is how they fit together in
memory, for a `static_key` holding a plain `entries` pointer (the
common, non-module-linked case from [§7.1](#struct-static_key)):

            struct static_key
            +------------------------+
            | enabled (atomic)       |
            | type / entries / next  |   tagged union
            +-----------+------------+
                        |
                        | first jump_entry for this key
                        v
    __jump_table[]  (sorted by key, then by code address)

      [ entries for key A ... ][ entries for key B ... ] ...
            |                           |
            +--> code / target / key <--+

Notice `static_key` does not store *how many* call sites reference it,
just a pointer to the *first* one. That is only enough information
because of how the table is sorted:
[`jump_label_sort_entries()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L80)
orders the whole
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L209)
array not by the raw bits stored in `entry->key`, but by what
[`jump_entry_key()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L127)
decodes those bits into — the actual, absolute `static_key` address of
the entry.

That distinction matters because, like `code` and `target`
([§6.2](#the-jump-table-entry-sidecar-metadata)), `key` is
self-relative: it stores a *distance to the key, measured from the
address of this entry itself in the table*, not the address of the key
directly. That’s exactly what the function does. It masks off the two
flag bits ([§7.2](#struct-jump_entry-relative-form)) to recover the
offset, then adds its own field address back in:

``` c
static inline struct static_key *jump_entry_key(const struct jump_entry *entry)
{
    long offset = entry->key & ~3L;

    return (struct static_key *)((unsigned long)&entry->key + offset);
}
```

Two entries that both belong to the same `static_key` but sit at
different slots in
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L209)
measure that distance from two different starting points (`&entry->key`
differs per slot), so their raw `key` bits will generally differ even
though they mean “the same key”:

                  addr of      raw key    decode: addr + raw key
                  entry->key   field      (jump_entry_key())
                  ----------   --------   -----------------------
    slot 0 (A):   0x1000       +0x4000    0x1000 + 0x4000 = 0x5000  ─┐
       ...                                                           ├─ same
    slot 5 (B):   0x2000       +0x3000    0x2000 + 0x3000 = 0x5000  ─┘  static_key!

                                               static_key @ 0x5000

`0x4000` and `0x3000` look unrelated as raw bit patterns — a sort on
those values would happily place A and B far apart. Only after factoring
in the address of each entry itself do both resolve to the same
`0x5000`, which is what
[`jump_entry_key()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L127)
— and therefore the sort — actually compares.

With the array sorted that way, every entry for the same key ends up
contiguous, so “find every call site for this key” is just “start at
[`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
and keep reading forward until the decoded key changes” — a plain linear
scan, no index or count required.

Within that same-key run, entries get a secondary sort by
[`jump_entry_code()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L117)
— the decoded address of the patchable instruction itself — purely so
that addresses come out in ascending order. The batched patching
machinery from [§10](#x86-text-patching-the-gory-details) needs that
ordering to do its work efficiently; without it, the entries for one key
could point at instructions scattered arbitrarily across memory in no
particular sequence.

**Why sorting a relative-offset table needs special care.** An ordinary
in-place sort works by swapping raw bytes between two array slots — but
here, swapping the bytes of two entries verbatim would silently corrupt
both of them.

The fields of each entry are self-relative
([§6.2](#the-jump-table-entry-sidecar-metadata)): they encode “distance
from *my own* address to the target.” Move the bytes of an entry to a
different slot, and its own address changes, but a naive byte-for-byte
copy would carry over distances computed for the *old* address, now
pointing at the wrong place entirely.
[`jump_label_swap()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L63)
fixes this by computing `delta`, the fixed byte distance between the two
slots being swapped, and adjusting every field by that same `delta`
(adding it in one direction, subtracting it in the other) as part of the
swap — so each field, now living at its new address, still decodes to
exactly the same absolute target it did before:

``` c
static void jump_label_swap(void *a, void *b, int size)
{
        long delta = (unsigned long)a - (unsigned long)b;
        struct jump_entry *jea = a;
        struct jump_entry *jeb = b;
        struct jump_entry tmp = *jea;

        jea->code   = jeb->code - delta;
        jea->target = jeb->target - delta;
        jea->key    = jeb->key - delta;

        jeb->code   = tmp.code + delta;
        jeb->target = tmp.target + delta;
        jeb->key    = tmp.key + delta;
}
```

Architectures using the plain, absolute-address form of
[`jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111)
([§7.2](#struct-jump_entry-relative-form)) need none of this: an
absolute address doesn’t care which slot it happens to sit in, so a
plain byte swap already works.

### 7.4 Linker section {#linker-section}

Every translation unit that expands
[`JUMP_TABLE_ENTRY`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L15)
([§6.2](#the-jump-table-entry-sidecar-metadata)) contributes its own
little `.pushsection __jump_table` … `.popsection` block, scattered
across dozens of separately-compiled `.o` files. Something still has to
gather all of those into the one contiguous `__jump_table[]` array
[§7.3](#relationship) assumes exists. That something is the main linker
script of the kernel, via
[`include/asm-generic/vmlinux.lds.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/vmlinux.lds.h#L436):

``` c
BOUNDED_SECTION_BY(__jump_table, ___jump_table)
```

This macro expands to three linker-script directives:

    __start___jump_table = .;
    KEEP(*(__jump_table))
    __stop___jump_table = .;

The middle line is the one doing the actual work: `*(__jump_table)`
tells the linker “collect the
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/vmlinux.lds.h#L436)
input section from *every* object file being linked, in whatever order
they’re linked, and place them one after another, right here” — which is
exactly how the individually-emitted entries of each translation unit
end up concatenated into one array (the same “magic section” trick the
kernel also uses for initcalls).

`KEEP(...)` matters because nothing in the C code of the kernel ever
takes the address of this section or calls into it the way it would a
normal function — as far as the dead-code elimination of the linker can
tell, it looks unreferenced and safe to discard, so `KEEP` explicitly
overrides that and forces it to stay.

The two assignments surrounding it, `__start___jump_table` and
`__stop___jump_table`, are what let
[`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
find the bounds of the array at boot ([§9.1](#boot-jump_label_init)) —
they resolve to the addresses immediately before and after the
concatenated data, with no explicit entry count needed anywhere,
mirroring how the linear scan by
[`jump_entry_key()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L127)
in [§7.3](#relationship) needed no count either.

That linker-driven concatenation only covers code built directly into
`vmlinux`. A module compiled and loaded later has no way to participate
in a linker script that already finished running long before the module
even existed, so it carries its own private
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/vmlinux.lds.h#L436)
section inside its own `.ko` file instead. When the module loader maps
that module in, it reads that section itself and records its bounds in
the two fields
[`struct module`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L511)
reserves for exactly this —
[`jump_entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L511)
(a pointer to the start of the array owned by that module) and
[`num_jump_entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L512)
(how many entries it holds, since there is no `vmlinux`-wide linker
symbol to bound it by). Those are precisely the entries that end up
wrapped in a
[`struct static_key_mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L613)
node ([§7.1](#struct-static_key)) whenever a key is shared between a
module and `vmlinux`, or between two modules.

------------------------------------------------------------------------

## 8 Size of the patchable site on x86 (runtime) {#size-of-the-patchable-site-on-x86-runtime}

[§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes) showed that a
nop-default site can end up being *either* 2 or 5 bytes, decided by the
assembler at build time based on the real distance to `l_yes`. Nothing
in
[`struct jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111)
([§7.2](#struct-jump_entry-relative-form)) records which one it ended up
as — there is no size field anywhere in it. So months or years later,
when this key is actually toggled on a running system, the code doing
the patching has to *rediscover* that size from scratch, by looking at
the live bytes currently sitting in `.text`:

``` c
/* arch/x86/kernel/jump_label.c */
int arch_jump_entry_size(struct jump_entry *entry)
{
        struct insn insn = {};

        insn_decode_kernel(&insn, (void *)jump_entry_code(entry));
        BUG_ON(insn.length != 2 && insn.length != 5);
        return insn.length;
}
```

[`insn_decode_kernel()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/insn.h#L172)
is the general-purpose x86 instruction decoder of the kernel (the same
kind of code that also has to understand arbitrary instructions for
kprobes), pointed here at the address that
[`jump_entry_code(entry)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L117)
from [§6.2](#the-jump-table-entry-sidecar-metadata) already knows how to
recover. It does not just count bytes: it genuinely parses the
instruction at that address (opcode, prefixes, displacement, all of it)
and reports how many bytes it occupies, via `insn.length`. Whatever is
sitting there right now — the original compiler-emitted `nop` or `jmp`
from [§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes), or a
previously-patched replacement from an earlier toggle — this call
decodes it and returns its true size.

`BUG_ON(insn.length != 2 && insn.length != 5)` is a sanity check, not a
normal error path: if this ever decodes to any width other than the two
shapes [§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes)
established, something has gone badly wrong (the table and the text it
describes have desynchronized), and there is no safe way to keep
patching.

This decode-on-demand step is specific to x86. Other architectures —
arm64, for instance, which defines a fixed
[`JUMP_LABEL_NOP_SIZE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/arm64/include/asm/jump_label.h#L17)
— never need it, because every patchable site on those architectures is
always the same width, known in advance, with nothing to discover. x86
is the odd one out precisely because the size optimization from
[§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes) means the width
of a site is a fact about *that specific call site*, not a constant true
of the whole kernel. The architecture-independent
[`jump_entry_size()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L171)
reflects that split: it returns the fixed
[`JUMP_LABEL_NOP_SIZE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/arm64/include/asm/jump_label.h#L17)
where an architecture defines one, and only falls back to calling
[`arch_jump_entry_size()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L20)
— the decoder above — when no such constant exists:

``` c
static inline int jump_entry_size(struct jump_entry *entry)
{
#ifdef JUMP_LABEL_NOP_SIZE
    return JUMP_LABEL_NOP_SIZE;
#else
    return arch_jump_entry_size(entry);
#endif
}
```

Once the size is known,
[`__jump_label_patch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L36)
builds the two byte sequences this site could possibly need — the `jmp`
form *and* the `nop` form, regardless of which one is about to be
installed — and picks between them:

``` c
size = arch_jump_entry_size(entry);
switch (size) {
case JMP8_INSN_SIZE:   /* 2 */
        code = text_gen_insn(JMP8_INSN_OPCODE, addr, dest);
        nop  = x86_nops[size];
        break;
case JMP32_INSN_SIZE:  /* 5 */
        code = text_gen_insn(JMP32_INSN_OPCODE, addr, dest);
        nop  = x86_nops[size];
        break;
}
```

[`JMP8_INSN_OPCODE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h#L60)
(`0xEB`) and
[`JMP32_INSN_OPCODE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h#L57)
(`0xE9`) are the same two jump encodings named in the `rel8`/`rel32`
explanation from
[§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes), just given
their real opcode values here.
[`text_gen_insn()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h#L123)
builds the actual `jmp` bytes for whichever opcode it’s given, computing
the displacement itself as `dest - (addr + size)` — precisely the
“distance from the address of the *next* instruction” convention
[§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes) described for
`rel8`/`rel32` encodings, just computed here at patch time instead of by
the assembler at build time.
[`x86_nops`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L91)
is a small lookup table of ready-made no-op byte sequences, one per
length, so
[`x86_nops[size]`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L91)
hands back a correctly-sized `nop` without having to construct one on
the fly — the same 2-byte and 5-byte encodings already named in
[§6.4](#assembly-level-picture).

Building both `code` and `nop` up front, rather than only the one being
installed, is what makes the next safety check possible. Before writing
anything,
[`__jump_label_patch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L36)
decides which of the two it *expects* to find already sitting at this
address — and, importantly, that expectation is the *opposite* of what
it is about to install: if `type` says “install a `jmp`,” the live bytes
had better currently be the old `nop` (that is the only state a
nop-default site should be transitioning *from*); if `type` says
“install a `nop`,” the live bytes had better currently be the old `jmp`.

A
[`memcmp()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/lib/string.c#L655)
of the real bytes against that expectation runs before any write. If
they don’t match, the jump-table metadata and the actual instruction
stream have diverged somehow — corruption, a bug elsewhere in the
kernel, or a race this code did not anticipate — and there is no safe
way to proceed: it logs the mismatch via
[`pr_crit`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/printk.h#L543)
and calls
[`BUG()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/bug.h#L114),
deliberately crashing rather than patching blind and risking that the
CPU ends up executing whatever garbage caused the mismatch in the first
place.

------------------------------------------------------------------------

## 9 Life of a static key: boot, enable, disable {#life-of-a-static-key-boot-enable-disable}

Everything in this section is driven by one
[`atomic_t`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/types.h#L186):
[`key->enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87).
Its value tells you both “is the feature on” and “is a patch pass
currently running”. The boolean on/off cycle looks like this:

            set to -1          patch, set 1          cmpxchg(1,0), off
    +----+             +----+                +----+                     +----+
    |  0 | ----------> | -1 | -------------> |  1 | ------------------> |  0 |
    +----+             +----+                +----+                     +----+
     off               enable                  on                        off

Read left to right, the three values in that diagram mean:

- **`0`** — off. Every site for this key holds whichever instruction
  (`nop` or `jmp`) means “disabled” for its polarity
  ([§5](#two-polarities-key-default-branch-hint)).
- **`-1`** — transient: an enable is in progress,
  [`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
  is actively rewriting every site right now. This value exists so that
  no concurrent reader can ever observe `0` during that window and
  wrongly conclude the feature is still off while the patcher is
  mid-flight;
  [`static_key_count()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L104)
  deliberately reports `-1` as “1” (enabled) to close that gap.
- **`1`** — on, with the refcount sitting at exactly one net enable (the
  refcounting ladder below explains what “net” means here).

There is no symmetric `-1`-like transient for *disable*: going `1 -> 0`
reads as briefly stale “on” instead (see [§9.3](#disabling)), which is
harmless.

Above value `1`, a second, independent ladder exists purely for
refcounting —
[`static_branch_inc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L513)/[`dec()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L514)
([§4.3](#boolean-enable-vs-refcounted-enable)) climb and descend it
*without ever touching text*, because the key is already known to be
enabled:

       1 --inc--> 2 --inc--> 3 --inc--> ...        (static_key_fast_inc_not_disabled:
       1 <--dec-- 2 <--dec-- 3 <--dec-- ...          pure atomic increment/decrement,
                                                      no jump_label_update() at all)

Only a dec that would land exactly on `1 -> 0` re-enters the boolean
cycle above and triggers a real patch-off. Put differently: of all the
edges across both diagrams, only the three that make up the boolean
cycle itself (`0 -> -1`, `-1 -> 1`, and `1 -> 0`) ever touch instruction
bytes. Every step on the refcount ladder above `1` (`1<->2`, `2<->3`, …)
is a bare atomic increment or decrement, with no
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
call anywhere in it.

That single fact is the whole point of the refcounted API in
[§4.3](#boolean-enable-vs-refcounted-enable): many callers can share a
key without each one paying for a text-patch round-trip — the cost of
patching is paid exactly once, by whichever caller happens to be the
first to enable it or the last to disable it.

### 9.1 Boot: [`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525) {#boot-jump_label_init}

Before this function ever runs, the jump-label machinery is in a
half-built state. The linker has already concatenated the slice of
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L209)
from every translation unit into one array ([§7.4](#linker-section)),
but that array is simply in link order — sites for the same key can be
scattered anywhere in it, and
[`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
still holds whatever its static initializer left there, which for a
plain `struct static_key key = STATIC_KEY_INIT_FALSE;`
([§7.1](#struct-static_key)) is nothing useful. In other words: the raw
table of call sites exists, but nothing yet knows *which* sites belong
to *which* key, so
[`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)
or
[`static_key_slow_inc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L186)
([§4.3](#boolean-enable-vs-refcounted-enable),
[§9.2](#enabling-static_key_enable-static_branch_enable)) would have
nothing to walk if called this early.

[`arch/x86/xen/multicalls.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/xen/multicalls.c)
declares
[`static struct static_key mc_debug __ro_after_init;`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/xen/multicalls.c#L60)
and registers
[`xen_mc_debug`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/xen/multicalls.c#L78)
as an
[`early_param()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/init.h#L354)
whose
[handler](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/xen/multicalls.c#L71)
calls
[`static_key_slow_inc(&mc_debug)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L186)
directly, synchronously, while
[`parse_early_param()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/init/main.c#L743)
is walking the command line. If that increment ran before
[`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
had sorted the table and pointed `mc_debug` at its entries, it would
have nothing to patch and the key would silently stay un-patched despite
the user asking for it on the command line.

The kernel guards against exactly this ordering mistake with one
boolean,
[`static_key_initialized`](https://elixir.bootlin.com/linux/v7.2-rc7/source/init/main.c#L174)
— whose only job, per its own comment, is “to generate warnings if
`static_key` manipulation functions are used before
[`jump_label_init`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
is called”;
[`jump_label_init_ro()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L573)
later even enforces it with a `WARN_ON_ONCE()`. That is why
[`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
is called very early from
[`start_kernel()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/init/main.c#L972)
in
[`init/main.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/init/main.c),
strictly before
[`parse_early_param()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/init/main.c#L743)
gets a chance to run any handler like
[`xen_parse_mc_debug()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/xen/multicalls.c#L71):

``` c
void __init jump_label_init(void)
{
        ...
        jump_label_sort_entries(iter_start, iter_stop);

        for (iter = iter_start; iter < iter_stop; iter++) {
                if (jump_label_type(iter) == JUMP_LABEL_NOP)
                        arch_jump_label_transform_static(iter, JUMP_LABEL_NOP);

                in_init = init_section_contains((void *)jump_entry_code(iter), 1);
                jump_entry_set_init(iter, in_init);

                iterk = jump_entry_key(iter);
                if (iterk == key)
                        continue;
                key = iterk;
                static_key_set_entries(key, iter);
        }
        static_key_initialized = true;
}
```

Before the loop even starts,
[`jump_label_sort_entries()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L80)
sorts the whole of
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L209)
by key, then by code address — the precondition that lets the “walk all
sites for this key as a linear scan” from [§7.3](#relationship) and the
“batch must stay address-ordered” requirement from
[§10.2](#batching-api-used-by-jump-labels) both work later.

The loop that follows does three things per entry, in this order,
trusting that the table is already sorted:

1.  Optionally run
    [`arch_jump_label_transform_static`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L402)
    for NOP sites — **a no-op on x86**, since `objtool` and the compiler
    have already left the right bytes in place
    ([§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes)); other
    architectures without that build-time trick do real work here.
2.  Mark `__init` sites, so nothing later tries to patch a call site
    living in memory that will be freed once init finishes.
3.  Point each key at the first entry of its contiguous run in the
    now-sorted table, so
    [`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
    is ready to use the moment something calls
    [`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522).

None of this touches instruction bytes for sites that are already
correct: the compiled-in `nop`/`jmp` already matches the initial value
of each key ([§5](#two-polarities-key-default-branch-hint)). What this
pass builds is bookkeeping — sort order, the `__init` flag, and the
`entries` pointer — so that a *later* toggle knows exactly which sites
to patch and in what order.

[`jump_label_init_ro()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L573)
runs much later, from
[`mark_readonly()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/init/main.c#L1511).
It walks
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L209)
a second time, but this pass skips almost everything — it only acts on
keys that
[`is_kernel_ro_after_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/sections.h#L183)
recognizes as living in
[`__ro_after_init`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/cache.h#L54-L61)
storage.[^2] For each matching key it calls:

``` c
static inline bool static_key_sealed(struct static_key *key)
{
        return (key->type & JUMP_TYPE_LINKED) && !(key->type & ~JUMP_TYPE_MASK);
}

static inline void static_key_seal(struct static_key *key)
{
        unsigned long type = key->type & JUMP_TYPE_TRUE;
        key->type = JUMP_TYPE_LINKED | type;
}
```

[`static_key_seal()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L567)
keeps only the
[`JUMP_TYPE_TRUE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L195)
bit of the current
[`key->type`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L96)
and throws the rest of the word away, replacing it with
[`JUMP_TYPE_LINKED`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L196)
plus that one preserved bit. Recall from [§7.1](#struct-static_key) that
this word is normally a tagged pointer — the low 2 bits are a type tag,
and everything above them is either the `entries` pointer the loop
earlier in this section just installed, or a module-chain `next` pointer
from [§11](#modules-the-trickiest-part). Sealing collapses it down to
just the tag, with nothing left standing above bit 1:

    before sealing (a live `entries` or `next` pointer, tagged):

        bit63                                        bit1  bit0
        [ real pointer value ..................... ] [ L ] [ T ]

    after static_key_seal():

        bit63                                        bit1  bit0
        [ 0000000000000000000000000000000000000000 ] [ 1 ] [ T ]

`T` is the preserved
[`JUMP_TYPE_TRUE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L195)
bit; `L` is
[`JUMP_TYPE_LINKED`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L196).
[`static_key_sealed()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L562)
is exactly the test for the “after” picture — `JUMP_TYPE_LINKED` set and
nothing above bit 1 — so it can recognize a sealed key at a glance,
regardless of which of the two pointer kinds that word used to hold.

Several call sites can share the same key, so this loop visits the same
key more than once as it walks the table entry by entry.
[`static_key_sealed()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L562)
is checked before
[`static_key_seal()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L567)
runs, so the first visit seals the key and every later visit for that
same key becomes a no-op.

This is safe only because `__ro_after_init` is a promise that the key is
never toggled again after boot: once sealed,
[`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
is gone, so nothing could walk it even if some later code mistakenly
tried.

The payoff shows up in [§11](#modules-the-trickiest-part). When a module
loaded afterward references a sealed key,
[`jump_label_add_module()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L690)
skips allocating and chaining a
[`struct static_key_mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L613)
for it — the bookkeeping [§11](#modules-the-trickiest-part) otherwise
needs so a *future* toggle can find and patch call sites living in other
modules. Instead, it just patches the sites of that module once,
immediately, to match the already-final value of the key.

The ordering against
[`mark_rodata_ro()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/init_64.c#L1405)
follows from the same fact: sealing is the last write anything makes to
[`key->type`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L96),
and that field sits inside the very `__ro_after_init` section
`mark_rodata_ro()` is about to make actually read-only in the page
tables. Running first just means the field has already settled into its
final value before write access to it disappears.

### 9.2 Enabling: [`static_key_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L220) / [`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522) {#enabling-static_key_enable-static_branch_enable}

[`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
([§9.1](#boot-jump_label_init)) only builds bookkeeping — it never flips
a key. Every site in the kernel image boots running whatever `nop`/`jmp`
the compiler and `objtool` baked in
([§5](#two-polarities-key-default-branch-hint),
[§6](#what-the-compiler-emits-x86_64)), on or off, and stays that way
until something calls
[`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)
or
[`static_key_slow_inc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L186)
for the first time. This section is about that first flip, which can
happen years into uptime, on a system where other CPUs may already be
executing the very instructions about to be rewritten.

[`drivers/md/dm-stats.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/drivers/md/dm-stats.c#L419)
shows a concrete case of this
([§4.4](#reading-the-state-without-taking-the-branch)): the first time a
user asks the device-mapper stats ioctl to start recording per-region
I/O counters, it flips
[`stats_enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/drivers/md/dm.c#L76),
a key that has sat disabled since boot:

``` c
if (!static_key_enabled(&stats_enabled.key))
        static_branch_enable(&stats_enabled);
```

The guard matters as much as the call: it is what keeps a second, third,
or hundredth request for the same device from re-triggering a full patch
round once the key is already on
([§4.4](#reading-the-state-without-taking-the-branch)). But the first
time through, this
[`static_branch_enable(&stats_enabled)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/drivers/md/dm-stats.c#L420)
does run, and from that call onward every `stats_enabled` check in the
I/O path — on every CPU, some of which may be running that exact code
right now — has to observe the new state, and none of them may ever see
a half-patched instruction.
[`static_key_enable_cpuslocked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L197)
below is what makes that safe:

``` c
void static_key_enable_cpuslocked(struct static_key *key)
{
        ...
        jump_label_lock();
        if (atomic_read(&key->enabled) == 0) {
                atomic_set(&key->enabled, -1);      /* "enabling" */
                jump_label_update(key);             /* patch all sites */
                atomic_set_release(&key->enabled, 1);
        }
        jump_label_unlock();
}
```

Walking through what that function does, in order:

1.  It first checks whether the key is already enabled, and bails out if
    so.
2.  [`jump_label_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L23)
    serializes all jump-label patching globally, so two callers enabling
    different keys at the same time still cannot have their text pokes
    race one another.
3.  It sets `enabled = -1` before touching any code — this is the
    `0 → -1` transient from the state diagram above, and it is what
    makes
    [`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
    below safe to run concurrently with readers:
    [`static_key_count()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L104)
    and
    [`static_key_enabled()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L407)
    both treat `-1` as enabled, so no concurrent reader ever sees a
    window of “disabled” while sites are only half-patched.
4.  [`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
    does the actual work: it patches every site for this key
    ([§9.5](#jump_label_update-__jump_label_update)).
5.  Finally, it stores `1`.

[`static_key_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L220)
wraps this in
[`cpus_read_lock()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/cpu.c#L488)
so CPUs cannot come online mid-patch.

The refcounted path,
[`static_key_slow_inc_cpuslocked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L151),
is not just this same function reused for `inc`/`dec`
([§4.3](#boolean-enable-vs-refcounted-enable)). With multiple
independent owners, more than one CPU can call it at once, and only one
of them may actually be the one that flips `0 → 1`:

``` c
bool static_key_slow_inc_cpuslocked(struct static_key *key)
{
        lockdep_assert_cpus_held();

        if (static_key_fast_inc_not_disabled(key))
                return true;

        guard(mutex)(&jump_label_mutex);
        if (!atomic_cmpxchg(&key->enabled, 0, -1)) {
                jump_label_update(key);
                atomic_set_release(&key->enabled, 1);
        } else {
                if (WARN_ON_ONCE(!static_key_fast_inc_not_disabled(key)))
                        return false;
        }
        return true;
}
```

It tries a lock-free fast path first (below), and only takes
[`jump_label_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L23)
if that fails. Past that point it is the same `0 → 1` dance as
[`static_key_enable_cpuslocked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L197)
above: exactly one caller actually performs the transition and runs
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886);
anyone else who reaches the mutex finds the key already on and just
falls back to the fast path to add their own count.

[`static_key_fast_inc_not_disabled()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L127)
is what makes “further incs” in the table from
[§4.3](#boolean-enable-vs-refcounted-enable) cost nothing more than an
atomic — no lock, no text poke:

``` c
bool static_key_fast_inc_not_disabled(struct static_key *key)
{
        int v;

        STATIC_KEY_CHECK_USE(key);
        /*
         * Negative key->enabled has a special meaning: it sends
         * static_key_slow_inc/dec() down the slow path, and it is non-zero
         * so it counts as "enabled" in jump_label_update().
         *
         * The INT_MAX overflow condition is either used by the networking
         * code to reset or detected in the slow path of
         * static_key_slow_inc_cpuslocked().
         */
        v = atomic_read(&key->enabled);
        do {
                if (v <= 0 || v == INT_MAX)
                        return false;
        } while (!likely(atomic_try_cmpxchg(&key->enabled, &v, v + 1)));

        return true;
}
```

The retry loop only succeeds once it can prove, atomically, that the
count is already an ordinary positive number. `v <= 0` catches both a
genuinely disabled key (`0`) and an enable already in progress (`-1`),
sending both cases down to the slow path above instead of incrementing a
value that doesn’t mean what it looks like yet. `v == INT_MAX` guards
against overflows.

### 9.3 Disabling {#disabling}

[§9.2](#enabling-static_key_enable-static_branch_enable) walked through
what happens when a key crosses from off to on. Disabling is the same
problem in reverse: patch every site back to its original instruction,
without letting any reader ever observe a torn one. It runs on the very
same rule as enabling — not every call to `disable`/`dec` needs to touch
an instruction at all.
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
only has to run on the one transition that actually flips what is
patched into the instruction stream: `1 → 0` for the boolean API,
`N → 0` for the refcounted one. Every other call just moves a plain
integer and can be answered with a single atomic instruction — no lock,
no text poke, because nothing about the patched code needs to change.

That is exactly the same shape as the
[`static_key_fast_inc_not_disabled()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L127)
from [§9.2](#enabling-static_key_enable-static_branch_enable) on the
enable side — a bare CAS loop, no lock, no patch, for every increment
that doesn’t cross `0 → 1`. That rule (every transition that isn’t on a
patch-triggering boundary is a bare atomic op) is what both functions
below are built around, on either side of the key.

``` c
void static_key_disable_cpuslocked(struct static_key *key)
{
        ...
        if (atomic_read(&key->enabled) != 1) {
                WARN_ON_ONCE(atomic_read(&key->enabled) != 0);
                return;
        }

        jump_label_lock();
        if (atomic_cmpxchg(&key->enabled, 1, 0) == 1)
                jump_label_update(key);
        jump_label_unlock();
}
```

[`static_key_disable_cpuslocked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L228)
is the mirror, in the boolean API, of the enable path from
[§9.2](#enabling-static_key_enable-static_branch_enable), but it gets to
skip the `-1` choreography of that path entirely — for a reason worth
spelling out rather than just noting. It first checks that the key is
currently exactly `1`, bailing out (and warning if the value isn’t `0`
either, which would mean the boolean and refcounted APIs got mixed on
this key — [§4.3](#boolean-enable-vs-refcounted-enable)) before doing
anything else. The actual disable is then a single
`atomic_cmpxchg(&key->enabled, 1, 0)`: “if the value is currently `1`,
replace it with `0`, and tell me whether you succeeded.” If some other
CPU changed it first, the compare fails and this call does nothing
further. Only on success does it call
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
to patch every site for this key back to its disabled instruction.

Compare that to enabling: there,
[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
is deliberately set to `-1` *before* any text is touched, specifically
so no reader can mistake in-progress patching for “off” (point 3 of
[§9.2](#enabling-static_key_enable-static_branch_enable)). Disabling has
no matching problem to solve. During the window between the `cmpxchg`
above and
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
finishing,
[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
already reads `0` while some sites out there are still physically
holding their “enabled” instruction — the opposite kind of staleness
from enabling (stale “on” instead of stale “off”), but just as harmless.
A reader who calls
[`static_key_enabled()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L407)
during that window is simply told “off” a few instructions before the
code itself has caught up;
[§4.4](#reading-the-state-without-taking-the-branch) already covers why
these state reads only ever need to be eventually correct, never
instruction-exact.

The refcounted side runs through a different function,
[`__static_key_slow_dec_cpuslocked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L292),
which only reaches
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
on the one decrement that actually drives the count to `0`
([`atomic_dec_and_test()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/atomic/atomic-instrumented.h#L1380)
reports true exactly then, and only then). Every decrement that lands
above `1` — `3 -> 2`, `2 -> 1`, and so on — is intercepted earlier, by
[`static_key_dec_not_one()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L253),
which performs a plain atomic decrement and returns without ever taking
the jump-label lock or looking at an instruction stream. That is the
same “every transition that isn’t on a patch-triggering boundary is a
bare atomic op” rule we discussed earlier, now seen from the decrement
side:

``` c
static bool static_key_dec_not_one(struct static_key *key)
{
        int v;

        v = atomic_read(&key->enabled);
        do {
                WARN_ON_ONCE(v < 0);

                if (WARN_ON_ONCE(v == 0))
                        return true;
                if (v <= 1)
                        return false;
        } while (!likely(atomic_try_cmpxchg(&key->enabled, &v, v - 1)));

        return true;
}
```

The CAS loop is what makes this safe against concurrent decrements: if
another CPU wins the race and changes
[`key->enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
between the read of this CPU and its `cmpxchg`, `v` is refreshed and the
loop just re-checks the same two conditions against the new value rather
than clobbering it.

### 9.4 Deferred / rate-limited dec {#deferred-rate-limited-dec}

This is the internal counterpart to the rate-limited disable in
[§4.7](#rate-limited-disable-userspace-facing-knobs), seen here from the
function-by-function angle of this section rather than the call-site
angle of the cookbook.
[§4.7](#rate-limited-disable-userspace-facing-knobs) already covers
*why* a delayed decrement exists and how the coalescing works when
toggles arrive faster than the timeout; this is only the piece that was
left out there — where the state-machine logic actually lives.

[`__static_key_slow_dec_deferred()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L346)
opens with the exact same
[`static_key_dec_not_one()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L253)
check [§9.3](#disabling) just introduced: if this decrement would not
bring the count down to the `1 -> 0` boundary, it is a plain atomic
decrement and the function returns immediately, no different from the
non-deferred path.

The two paths only diverge on the one decrement that *would* disable the
key. Where the
[`__static_key_slow_dec_cpuslocked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L292)
from [§9.3](#disabling) reaches straight for
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
at that point, this function instead calls
[`schedule_delayed_work()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/workqueue.h#L853)
— a standard kernel workqueue primitive that runs a callback once a
given delay has elapsed, rather than right away — and returns without
touching a single instruction. The count is deliberately left at `1`
rather than dropped to `0`; only the *plan* to disable has been
recorded, in the timer. In full:

``` c
void __static_key_slow_dec_deferred(struct static_key *key,
                    struct delayed_work *work,
                    unsigned long timeout)
{
        if (static_key_dec_not_one(key))
                return;

        schedule_delayed_work(work, timeout);
}
```

When that timer eventually fires,
[`jump_label_update_timeout()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L325)
runs the ordinary, undeferred decrement path from [§9.3](#disabling). If
nothing else touched the key in the meantime, the count is still exactly
`1`, the decrement finally lands on the `1 -> 0` boundary, and
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
runs for real. If instead another caller incremented the key again while
the timer was pending, the count is no longer `1` by the time the timer
fires — so
[`static_key_dec_not_one()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L253)
intercepts *that* decrement too, as an ordinary atomic op, and
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
is never reached. Nothing needed patching back, because nothing was ever
patched in the first place.

### 9.5 [`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886) → [`__jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L503) {#jump_label_update-__jump_label_update}

Every path in [§9.2](#enabling-static_key_enable-static_branch_enable)
through [§9.4](#deferred-rate-limited-dec) eventually funnels into this
function — it is the one that actually walks the sites of a key and asks
the architecture layer to patch each one. Reading the outer function
first:

``` c
static void jump_label_update(struct static_key *key)
{
        ...
        if (static_key_linked(key)) {
                __jump_label_mod_update(key);   /* walk module list */
                return;
        }
        entry = static_key_entries(key);
        if (entry)
                __jump_label_update(key, entry, stop, init);
}
```

The first branch is the module case from [§7.1](#struct-static_key): if
bit 1 of
[`key->type`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L96)
is set
([`LINKED`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L196)),
the sites of this key are not one contiguous run inside
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L209),
but scattered across a linked list of per-module entry tables
([`struct static_key_mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L613)),
so a separate helper has to walk that list instead of a flat array —
[§11](#modules-the-trickiest-part) covers
[`__jump_label_mod_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L665)
in full. Otherwise,
[`static_key_entries(key)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L409)
recovers the pointer [§9.1](#boot-jump_label_init) stored during boot —
the first entry of the contiguous run for this key inside the sorted,
vmlinux-only table — and the real patching happens in
[`__jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L503).

On x86, which defines
[`HAVE_JUMP_LABEL_BATCH`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L5),
that function looks like this:

``` c
for (; entry < stop && jump_entry_key(entry) == key; entry++) {
        if (!jump_label_can_update(entry, init))
                continue;
        if (!arch_jump_label_transform_queue(entry, jump_label_type(entry))) {
                arch_jump_label_transform_apply();
                BUG_ON(!arch_jump_label_transform_queue(...));
        }
}
arch_jump_label_transform_apply();
```

The loop condition — advance while `entry < stop` *and*
[`jump_entry_key(entry)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L127)
`== key` — only works because of the sort from
[§9.1](#boot-jump_label_init): every site belonging to this key is
guaranteed to sit in one unbroken run starting at `entry`, so the loop
can walk forward blindly and stop the instant it reaches a site
belonging to some other key, with no need to search the rest of the
table.

For each site still in range,
[`jump_label_type(entry)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L455)
computes `enabled ^ branch` — the exact static/dynamic formula from the
table in [§5](#two-polarities-key-default-branch-hint), but now
evaluated against the *current* live state of the key rather than its
compile-time initial value — to decide whether this specific site should
end up holding a
[`JUMP_LABEL_NOP`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L186)
or a
[`JUMP_LABEL_JMP`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L187).

Before acting on that answer,
[`jump_label_can_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L465)
filters out two kinds of site that must not be touched at all: one still
living in `__init` text after boot has finished (that memory may already
have been freed, which is exactly what the
[`jump_entry_is_init`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L158)
flag from [§9.1](#boot-jump_label_init) was recorded to detect), and one
that
[`kernel_text_address()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/extable.c#L94)
does not even recognize as live kernel text — built-in code that is
`__exit`-only and therefore can never run, so patching it would be
pointless even though it is technically still present:

``` c
static bool jump_label_can_update(struct jump_entry *entry, bool init)
{
        if (!init && jump_entry_is_init(entry))
                return false;

        if (!kernel_text_address(jump_entry_code(entry))) {
                WARN_ONCE(!jump_entry_is_init(entry),
                          "can't patch jump_label at %pS",
                          (void *)jump_entry_code(entry));
                return false;
        }

        return true;
}
```

Rather than patching each surviving site immediately, one at a time, the
loop splits the work into two separate jobs:
[`arch_jump_label_transform_queue()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L123)
builds up a batch, and
[`arch_jump_label_transform_apply()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L143)
executes it. Splitting them apart is what lets every site belonging to
one key ride through a single INT3-synchronized patch round
([§10.2](#batching-api-used-by-jump-labels)) instead of paying for one
round per site.

**Queuing a site.** Each call to
[`arch_jump_label_transform_queue()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L123)
computes the replacement bytes for that one site and hands
`(address, new bytes, length)` to
[`smp_text_poke_batch_add()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L3202).
That function appends the request to a pending array; nothing is written
to memory yet. The one exception is early boot: only one CPU is running,
so there is no concurrent fetcher to synchronize against and nothing
worth batching — the function calls the non-batching
[`arch_jump_label_transform()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L117)
directly instead.

**Applying the batch.**
[`arch_jump_label_transform_apply()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L143)
executes everything queued so far. It calls
[`smp_text_poke_batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
which runs the three-step INT3 dance once for the whole batch instead of
once per site.
[`__jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L503)
calls it once, after its loop ends, to flush whatever is still pending.

------------------------------------------------------------------------

## 10 x86 text patching: the gory details {#x86-text-patching-the-gory-details}

This is where the “you cannot just `memcpy` over live code” argument
from [§1.3](#why-you-cannot-just-memcpy-over-live-code-on-smp) turns
into actual working code. Roadmap: [§10.1](#early-boot-vs-live-smp)
picks early-boot-single-CPU vs. later-multi-CPU;
[§10.2](#batching-api-used-by-jump-labels) covers the batching array
that collects many sites before paying for synchronization;
[§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish) is the INT3
protocol itself — the heart of the mechanism;
[§10.4](#what-sync-means)–[§10.5](#writing-through-ro-mappings) fill in
what “synchronize” and “write to RO memory” actually mean underneath;
[§10.6](#end-to-end-timeline-for-one-enable) strings it all into one
timeline.

### 10.1 Early boot vs live SMP {#early-boot-vs-live-smp}

Very early in boot, only the boot CPU is running and `.text` is still
writable.
[`__jump_label_transform()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L82),
the function every x86 patch eventually funnels through, checks for
exactly that window and takes the cheap route whenever it still holds:

``` c
/* arch/x86/kernel/jump_label.c: __jump_label_transform() */
if (init || system_state == SYSTEM_BOOTING) {
        text_poke_early(...); /* IRQ-disabled + `memcpy()` + sync_core() */
        return;
}
smp_text_poke_single(...);      /* or batch_add during queueing (§9.2) */
```

`system_state == SYSTEM_BOOTING` is a proxy for one fact: only the boot
CPU exists so far. That alone is reason enough to skip the whole
IPI-synchronized protocol of
[§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish) — with
nobody else around to race the write,
[`text_poke_early()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2489)
can just do a plain IRQ-disabled + `memcpy()` +
[`sync_core()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/sync_core.h#L58)
and be done. `.text` also happens to still be writable at this point, so
the alias trick from [§1.4](#writing-read-only-kernel-text-text_poke)
isn’t needed either — but that is a bonus the check gets for free, not
something it verifies directly:
[`smp_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/init/main.c#L1650)
wakes every other CPU well before
[`mark_rodata_ro()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/init/main.c#L1522)
ever runs, so there is a real stretch of boot where other CPUs are
already up while `.text` is still writable. Jump labels take the full
protocol for that entire stretch anyway, because the only thing this
check ever verifies is whether this CPU is still provably alone.

The leading `init` in that condition is easy to misread as the per-site
`__init`-text flag from [§9.1](#boot-jump_label_init)
([`jump_entry_is_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L158))
— it is not. It is a separate, whole-system flag threaded down from
`init = system_state < SYSTEM_RUNNING` inside
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
itself ([§9.5](#jump_label_update-__jump_label_update)), true a little
longer than `SYSTEM_BOOTING` alone. On the batching path of x86, though,
that value never actually reaches here:
[`arch_jump_label_transform_queue()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L123)
only calls this function through its own
`system_state == SYSTEM_BOOTING` fallback, passing a hardcoded `0` for
`init` every time it does. So on x86 this condition is, in practice,
exactly `system_state == SYSTEM_BOOTING` — the `init ||` half of it only
ever matters on architectures that call this function directly, without
going through batching at all.

### 10.2 Batching API used by jump labels {#batching-api-used-by-jump-labels}

[§10.1](#early-boot-vs-live-smp) settled *how* a single site gets
patched once the decision to patch it is made; this section is about
*when* jump labels actually pull that trigger. A busy tracepoint can
have thousands of call sites sharing one key, and paying the full
IPI-synchronized protocol
([§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish)) separately
for each one would be needless — the sites can be collected first and
the expensive part paid once for the whole group. Two arch-level hooks
make that possible: one that queues the newly computed bytes for a site
without touching hardware yet, and one that flushes everything queued so
far in a single synchronized pass:

``` c
bool arch_jump_label_transform_queue(...)
{
        if (system_state == SYSTEM_BOOTING) {
                arch_jump_label_transform(entry, type);
                return true;
        }
        mutex_lock(&text_mutex);
        jlp = __jump_label_patch(entry, type);
        smp_text_poke_batch_add(addr, jlp.code, jlp.size, NULL);
        mutex_unlock(&text_mutex);
        return true;
}

void arch_jump_label_transform_apply(void)
{
        mutex_lock(&text_mutex);
        smp_text_poke_batch_finish();
        mutex_unlock(&text_mutex);
}
```

The queue collects many sites (a busy tracepoint may have thousands);
one
[`batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
amortizes the IPI syncs. The queue must stay **address-sorted**; if a
new address would break order, or the page-sized array is full,
[`smp_text_poke_batch_add()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L3202)
flushes early
([`text_poke_addr_ordered()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L3170)
in
[`alternative.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c)).
That is why
[`jump_label_cmp`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L35)
sorts by code address within each key.

**How many fit in one batch?** The pending patches live in a single
statically-allocated page,
[`struct smp_text_poke_loc`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2781-L2792):

``` c
struct smp_text_poke_loc {
        s32 rel_addr;   /* addr := _stext + rel_addr           */
        s32 disp;       /* branch displacement, for emulation  */
        u8  len;        /* 1, 2, 5, or 6 bytes                 */
        u8  opcode;     /* first opcode byte, for emulation    */
        u8  text[5];    /* the new instruction bytes           */
        u8  old;        /* byte that used to be there, for perf/PT tracing */
};                       /* 16 bytes, naturally aligned         */

#define TEXT_POKE_ARRAY_MAX (PAGE_SIZE / sizeof(struct smp_text_poke_loc))
/* 4096 / 16 = 256 entries per flush on a 4K-page x86_64 build */
```

`rel_addr` is relative to
[`_stext`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/vmlinux.lds.S#L137)[^3],
not to the entry itself like
[`jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111)
— cheap, because every patch site is, by definition, in kernel text. A
tracepoint or jump-label key with more than 256 call sites needs more
than one
[`batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
round (i.e. more than 3 IPI rounds,
[§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish)) to fully
enable/disable.

[`__jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L503)
([§9.5](#jump_label_update-__jump_label_update)) does contain a generic
“queue full → apply → retry” branch, and it looks like the natural place
to expect this 256-site overflow to be handled.

On x86 the “queue full → apply → retry” branch never runs, though: it
only fires when
[`arch_jump_label_transform_queue()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L123)
itself returns `false`, and the implementation of that function on this
architecture never returns `false`. So the branch is dead code here.

``` c
for (; (entry < stop) && (jump_entry_key(entry) == key); entry++) {

        if (!jump_label_can_update(entry, init))
                continue;

        if (!arch_jump_label_transform_queue(entry, jump_label_type(entry))) {
                /*
                 * Queue is full: Apply the current queue and try again.
                 */
                arch_jump_label_transform_apply();
                BUG_ON(!arch_jump_label_transform_queue(entry, jump_label_type(entry)));
        }
}
arch_jump_label_transform_apply();
```

The overflow is actually caught one layer further down, entirely inside
[`smp_text_poke_batch_add()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L3202)
— the entire guard is these three lines:

``` c
void smp_text_poke_batch_add(void *addr, const void *opcode, size_t len, const void *emulate)
{
        if (text_poke_array.nr_entries == TEXT_POKE_ARRAY_MAX || !text_poke_addr_ordered(addr))
                smp_text_poke_batch_finish();
        __smp_text_poke_batch_add(addr, opcode, len, emulate);
}
```

Every iteration of that `for` loop does one thing: it calls
[`arch_jump_label_transform_queue()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L123),
which computes the new bytes for one site and appends one
[`smp_text_poke_loc`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2781-L2792)
to the array — the cheap, per-site step this section has been
describing. The `if (!arch_jump_label_transform_queue(...))` branch is
the “queue full → apply → retry” path discussed above; on x86 it never
runs, since that function never returns `false`. Nothing expensive
happens inside the loop.

[`arch_jump_label_transform_apply()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L143)
sits outside the loop, called exactly once after it exits, for every
site the loop just queued. That single call is what finally triggers
[`batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941),
the three-phase IPI-synchronized protocol from
[§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish).

So the entire set of call sites for a key — whether it has one or close
to 256 — rides through on that one shared
[`batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941),
and only a key with more than 256 sites forces a second round.

### 10.3 The INT3 SMP algorithm ([`smp_text_poke_batch_finish`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)) {#the-int3-smp-algorithm-smp_text_poke_batch_finish}

This is the payoff of everything
[§1.3](#why-you-cannot-just-memcpy-over-live-code-on-smp) through
[§10.2](#batching-api-used-by-jump-labels) built toward. The key fact
from [§1.3](#why-you-cannot-just-memcpy-over-live-code-on-smp) was that
only a *single-byte* store is atomic with respect to instruction fetch —
nothing wider is. The protocol below never trusts a multi-byte write to
be safe on its own; instead it uses one atomic single-byte store to
plant a trap on top of the site, uses that trap to absorb any CPU
unlucky enough to fetch through mid-update, and only then fills in the
rest. Three writes, three synchronizations, one site at a time across
the whole batch. Documented at the top of the function in
[`alternative.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941):

    For each site in the vector:
      (1) Write INT3 (0xCC) over the first byte
          → IPI sync all CPUs   (serialize pipelines / I-caches)

      (2) Write bytes 1..N-1 of the new instruction
          → IPI sync again      unnecessary, according to Intel,
                                but better safe than sorry 

      (3) Write byte 0 of the new instruction (replaces INT3)
          → IPI sync again

Here are the actual bytes of one site across the three phases, patching
a 5-byte NOP (`0f 1f 44 00 00`) into a 5-byte `JMP rel32` (`e9` + 4-byte
displacement, shown as `?? ?? ?? ??`):

     start (before)      0f 1f 44 00 00      any fetch: executes the NOP

     phase 1 (INT3 in)  cc 1f 44 00 00      any fetch: #BP -> handler emulates
                        ^^                  the NEW instruction (jumps to l_yes)
                        trap byte

                        -------- IPI sync --------

     phase 2 (tail in)  cc ?? ?? ?? ??      same as phase 1: byte 0 is still
                        ^^                  INT3, so any fetch still traps and
                        still traps         gets emulated — the real tail bytes
                                            underneath are now correct, but
                                            nothing reads them yet

                        -------- IPI sync --------

     phase 3 (done)     e9 ?? ?? ?? ??      any fetch: executes the real JMP
                        ^^ real opcode      directly, no trap needed anymore

                        -------- IPI sync --------

The subtle point this diagram is here to make: **the observable behavior
of the site flips the instant the sync in phase 1 completes**, not at
phase 3. From phase 1 onward, any CPU landing on this address — whether
by falling into it in a hot loop or by literally executing byte 0 — gets
the effect of the *new* instruction, because the `#BP` handler always
emulates the pending *new* instruction (it was computed and stashed in
the queue back at
[`smp_text_poke_batch_add()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L3202)
time, long before phase 1 starts). A CPU that hits the address
mid-update and one that hits it after phase 3 land on the same outcome —
the only difference is whether it got there by trapping into the handler
or by executing the finished bytes directly. Phases 2 and 3 exist to
make that direct path available, so steady-state execution stops paying
the `#BP` tax.

**The writing side.** All of the above is driven by
[`smp_text_poke_batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
(it early-returns immediately if `text_poke_array.nr_entries` is 0 —
nothing queued, nothing to do). Trimmed of the
[`cond_resched()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/sched.h#L2161)
softlockup guard, the perf/Intel-PT tracing hook, and a 6-byte-opcode
edge case, it opens by arming the refcount of every CPU — the release
side of the release/acquire pairing the “Why INT3?” sidebar below
explains — then runs the three phases in order:

``` c
for_each_possible_cpu(i)
        atomic_set_release(per_cpu_ptr(&text_poke_array_refs, i), 1);
smp_wmb();
```

**Phase 1** writes `INT3` over the first byte of every site, saving the
byte it replaces (for the perf/PT tracing hook trimmed out above), then
syncs once for the whole batch:

``` c
for (i = 0; i < text_poke_array.nr_entries; i++) {
        text_poke_array.vec[i].old = *(u8 *)text_poke_addr(&text_poke_array.vec[i]);
        text_poke(text_poke_addr(&text_poke_array.vec[i]), &int3, INT3_INSN_SIZE);
}
smp_text_poke_sync_each_cpu();
```

**Phase 2** writes bytes `1..len-1` of the new instruction for every
site — safe, since byte 0 is still `INT3` — and syncs again, but only if
some site actually has tail bytes to write (a single-byte patch has
none):

``` c
for (do_sync = 0, i = 0; i < text_poke_array.nr_entries; i++) {
        int len = text_poke_array.vec[i].len;

        if (len - INT3_INSN_SIZE > 0) {
                text_poke(text_poke_addr(&text_poke_array.vec[i]) + INT3_INSN_SIZE,
                          text_poke_array.vec[i].text + INT3_INSN_SIZE,
                          len - INT3_INSN_SIZE);
                do_sync++;
        }
}
if (do_sync)
        smp_text_poke_sync_each_cpu();
```

**Phase 3** writes byte 0, replacing the `INT3` — skipped for the corner
case (discussed next) where the *new* opcode is itself `0xCC` — and
syncs again if anything actually changed:

``` c
for (do_sync = 0, i = 0; i < text_poke_array.nr_entries; i++) {
        u8 byte = text_poke_array.vec[i].text[0];

        if (byte == INT3_INSN_OPCODE)
                continue;
        text_poke(text_poke_addr(&text_poke_array.vec[i]), &byte, INT3_INSN_SIZE);
        do_sync++;
}
if (do_sync)
        smp_text_poke_sync_each_cpu();
```

**Draining after phase 3.** After the last sync, the writer cannot yet
assume every CPU has *left* the handler — a CPU could be inside
[`smp_text_poke_int3_handler()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2838)
right up until that sync completes (it entered before the sync, is still
emulating). So
[`smp_text_poke_batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
ends with this drain and the final reset:

``` c
for_each_possible_cpu(i) {
        atomic_t *refs = per_cpu_ptr(&text_poke_array_refs, i);

        if (unlikely(!atomic_dec_and_test(refs)))
                atomic_cond_read_acquire(refs, !VAL);
}
text_poke_array.nr_entries = 0;
```

`atomic_dec_and_test()` decrements the refcount of every CPU; for any
that don’t immediately hit zero, `atomic_cond_read_acquire(refs, !VAL)`
spin-waits — i.e. for stragglers still mid-handler to finish and drop
their own reference. Only then does the function reset
`text_poke_array.nr_entries = 0`, making the buffer safe to reuse for
the *next* batch.

In the common case — jump labels, static calls, ftrace, none of which
ever replace a site with a literal `0xCC` — phase 3 already wrote every
final byte, and
[`smp_text_poke_sync_each_cpu()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2771)
already fenced out any in-flight handler. So this drain loop observes
zero immediately, and the comment in the source calls it out explicitly:
*“unless the replacement instruction is INT3, this case goes unused.”*

It exists for the corner case (other clients, not jump labels) where the
*new* opcode of a site is `0xCC` itself. Byte 0 is therefore left alone
in phase 3 (writing `0xCC` over an existing `0xCC` would be a no-op
anyway, so that per-entry sync is skipped), and the only thing standing
between “batch done” and “safe to reuse the array” is the
`atomic_cond_read_acquire()` spin-wait itself.

> **Why INT3?** The whole point of writing `INT3` first (phase 1 of the
> three-phase protocol above) is that it gives the kernel a way to
> intercept any CPU that would otherwise have executed half-written
> bytes, and make it run the *finished* instruction instead. While the
> site is mid-update, any CPU that hits it takes `#BP` (a fault, vector
> 3), and that fault always lands in
> [`smp_text_poke_int3_handler()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2838),
> which is wired up as the `#BP` handler in
> [`traps.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/traps.c).
> Its locals are just `tpl` (the matched
> [`struct smp_text_poke_loc`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2781-L2792)
> `*`), `ret`, and `ip`; here is what it actually does, one check at a
> time.
>
> **1. Bail on user mode.** This handler only ever concerns itself with
> traps hit while executing kernel text:
>
> ``` c
> if (user_mode(regs))
>         return 0;
> ```
>
> **2. Confirm a batch is actually in flight**, via a **per-CPU
> refcount**,
> [`text_poke_array_refs`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2799):
>
> ``` c
> smp_rmb();
>
> if (!try_get_text_poke_array())
>         return 0;
> ```
>
> `try_get_text_poke_array()` is just an atomic increment-if-nonzero:
>
> ``` c
> static __always_inline bool try_get_text_poke_array(void)
> {
>         atomic_t *refs = this_cpu_ptr(&text_poke_array_refs);
>         return raw_atomic_inc_not_zero(refs);   /* 0 → fails: no batch active */
> }
> ```
>
> Before touching any site,
> [`smp_text_poke_batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
> **arms** the refcount of every CPU to 1 with
> [`atomic_set_release()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/atomic/atomic-instrumented.h#L83),
> then
> [`smp_wmb()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/barrier.h#L107)s,
> and only *then* writes the INT3 bytes. The `smp_rmb()` above is the
> mirror-image acquire. That release/acquire pairing is what guarantees:
> if the `#BP` of a CPU fires (meaning it *must* have fetched the INT3
> the writer stored), that same CPU is also guaranteed to see a
> fully-populated, non-zero-refcount
> [`text_poke_array`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2797)
> — never a half-written vector.
>
> **3. Find which site trapped**, binary-searching `text_poke_array.vec`
> for this `regs->ip - 1` (skipping straight to a direct compare when
> there is exactly one entry):
>
> ``` c
> ip = (void *) regs->ip - INT3_INSN_SIZE;
>
> if (unlikely(text_poke_array.nr_entries > 1)) {
>         tpl = __inline_bsearch(ip, text_poke_array.vec, text_poke_array.nr_entries,
>                               sizeof(struct smp_text_poke_loc),
>                               patch_cmp);
>         if (!tpl)
>                 goto out_put;
> } else {
>         tpl = text_poke_array.vec;
>         if (text_poke_addr(tpl) != ip)
>                 goto out_put;
> }
>
> ip += tpl->len;
> ```
>
> **4. Emulate** the *new* instruction by editing `regs` and returning —
> the return address of the `#BP` handler becomes the effect of the
> emulated instruction, so `iret` resumes execution as if the new bytes
> had actually executed:
>
> ``` c
> switch (tpl->opcode) {
> case INT3_INSN_OPCODE:
>         goto out_put;             /* someone else's deliberate INT3 */
>
> case RET_INSN_OPCODE:
>         int3_emulate_ret(regs);
>         break;
>
> case CALL_INSN_OPCODE:
>         int3_emulate_call(regs, (long)ip + tpl->disp);
>         break;
>
> case JMP32_INSN_OPCODE:
> case JMP8_INSN_OPCODE:
>         int3_emulate_jmp(regs, (long)ip + tpl->disp);
>         break;
>
> case 0x70 ... 0x7f: /* Jcc */
>         int3_emulate_jcc(regs, tpl->opcode & 0xf, (long)ip, tpl->disp);
>         break;
>
> default:
>         BUG();
> }
>
> ret = 1;
> ```
>
> - `JMP rel8`/`rel32` (also a pending **NOP**, encoded as `JMP` with
>   `disp == 0`, i.e. “jump to the next instruction”):
>   [`int3_emulate_jmp()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h#L136)
>   just overwrites `regs->ip`.
> - `CALL`/`RET`/`Jcc`: same idea (push a fake return address / pop one
>   / conditionally add the displacement) — jump labels never generate
>   these, but static calls and ftrace share this exact engine and do.
> - If the *replacement* opcode is itself `0xCC` (some other text-poke
>   client is intentionally installing a breakpoint, not passing through
>   this emulator): the handler does **not** consume the trap — it falls
>   through so that debugging infrastructure (kgdb, kprobes) gets its
>   `#BP`.
>
> Jump labels only ever exercise the `JMP32`/`JMP8` case — the
> `RET`/`CALL`/`Jcc` arms exist because static calls and ftrace share
> this exact handler.
>
> **5. Release the refcount** and report the trap as handled:
>
> ``` c
> out_put:
>         put_text_poke_array();
>         return ret;
> ```
>
> So no CPU ever executes a torn multi-byte instruction: control either
> sees the old bytes (before the sync in phase 1 completes everywhere),
> takes the INT3+emulation path (the entire window from phase 1 to phase
> 3), or sees the finished new bytes (after the sync in phase 3).

### 10.4 What “sync” means {#what-sync-means}

The three-phase protocol from
[§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish) names “IPI
sync” as a step three times over, without ever saying what happens when
that IPI lands. This section answers that — not what the *patching* CPU
does ([§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish)
already covered that), but what every *other* CPU is forced to do in
response.

[`smp_text_poke_sync_each_cpu()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2771)
is the function invoked at each of those three points, and its job is
deceptively narrow: send an IPI to every CPU other than the one doing
the patching, have each of them run
[`sync_core()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/sync_core.h#L58),
and block until every last one has reported back. That blocking is not
incidental — the whole structure of
[§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish) depends on
each phase being globally visible before the next one begins. If the
patching CPU raced ahead and wrote the tail bytes of phase 2 while some
other core was still mid-fetch on the phase-1 view of the site, the
entire point of planting the INT3 trap first would be defeated.

What actually happens on the receiving end is where the CPU model from
[§1.1](#what-a-cpu-actually-does-with-instructions) finally pays for
itself. A modern core does not execute an instruction the moment it sees
its bytes: it fetches ahead of where it is currently retiring, decodes
into microcodes, and may be holding several instructions of that
pipeline in flight at once. A CPU that fetched the old bytes moments
before the patch landed can still be sitting on a stale decode of them —
and no ordinary memory write, however carefully sequenced, undoes that.
Something has to reach into the pipeline itself and discard the stale
work. That is exactly what a **serializing instruction** is
architecturally defined to do: retire everything already in flight, drop
any speculative or partially-decoded work, and guarantee that the very
next fetch goes out fresh.

Two ways to get that guarantee are available, and which one runs depends
on the CPU generation.
[`X86_FEATURE_SERIALIZE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/cpufeatures.h#L431),
present on most CPUs manufactured since around 2020, provides a
dedicated instruction whose only job is this flush — cheap and direct:

``` c
static __always_inline void serialize(void)
{
        /* Instruction opcode for SERIALIZE; supported in binutils >= 2.35. */
        asm volatile(".byte 0xf, 0x1, 0xe8" ::: "memory");
}
```

That comment is the real reason this is written as raw `.byte` values
rather than a mnemonic the assembler recognizes: `SERIALIZE` was only
added to binutils in 2.35, so a kernel built with an older assembler
still needs to be able to emit the instruction’s three raw opcode bytes
(`0F 01 E8`) by hand. The `"memory"` clobber tells the compiler this
call is a full optimization barrier — it must not reorder ordinary
memory accesses across it.

On older hardware, the kernel falls back to
[`iret_to_self()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/sync_core.h#L22).
An **interrupt frame** is just the handful of words — return `SS`,
`RSP`, `RFLAGS`, `CS`, and `RIP` — that the CPU itself pushes onto the
stack whenever a real interrupt or exception fires, and which `iret`
later pops to hand control back to whatever was running. Normally
software never builds one by hand; the CPU builds it automatically at
the moment of a real trap, the same way it did for the `#BP` frame the
INT3 handler above edited before its own `iret`.
[`iret_to_self()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/sync_core.h#L22)
does the CPU’s half of that job itself: it pushes those same five words
by hand, with no real interrupt behind them, sets the saved `RIP` field
to the very next instruction after the `iret`, and then executes `iret`
against that fabricated frame. As far as the CPU can tell, this is a
completely ordinary return from an interrupt, so it does everything
architecture requires of one — including the serializing flush this
function exists to get. But because the fabricated return address is
simply “keep going from here,” nothing about the actual control flow
changes: execution lands right back where it would have anyway, one
instruction later.

Here is the actual body, from
[`arch/x86/include/asm/sync_core.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/sync_core.h#L22)
(the 64-bit variant — a 32-bit build takes a shorter path that skips the
`SS`/`RSP` pushes, since a same-privilege 32-bit `iret` doesn’t pop
them):

``` c
static __always_inline void iret_to_self(void)
{
        unsigned int tmp;

        asm volatile (
                "mov %%ss, %0\n\t"      /* SS is a segment register; it can't be   */
                                         /* pushed directly in this form, so copy   */
                                         /* it into a GPR first                     */
                "pushq %q0\n\t"          /* frame field 1 (bottom): return SS       */
                "pushq %%rsp\n\t"        /* frame field 2: return RSP — but this    */
                                         /* captures RSP *after* the SS push above  */
                                         /* already moved it down by 8              */
                "addq $8, (%%rsp)\n\t"   /* ...so correct the just-pushed copy back */
                                         /* up by 8, to the RSP value from before   */
                                         /* this function started pushing anything  */
                "pushfq\n\t"             /* frame field 3: return RFLAGS            */
                "mov %%cs, %0\n\t"       /* same GPR trick as SS, for CS this time  */
                "pushq %q0\n\t"          /* frame field 4: return CS                */
                "pushq $1f\n\t"          /* frame field 5 (top): return RIP — the   */
                                         /* address of local label "1:" below, i.e. */
                                         /* the instruction right after this one    */
                "iretq\n\t"              /* pop all five fields and "return" — the  */
                                         /* CPU treats this exactly like returning  */
                                         /* from a genuine interrupt                */
                "1:"                     /* execution resumes here, indistinguishable */
                                         /* from simply falling through to this point */
                : "=&r" (tmp), ASM_CALL_CONSTRAINT : : "cc", "memory");
}
```

The `iret` is architecturally required to be a serializing event on
every CPU, which is precisely the guarantee this fallback needs — it
works identically at any privilege level (so it survives under
paravirtualization) and never exits to a hypervisor, both properties
this code cannot give up. The price is that it measures a bit more than
twice as slow as the dedicated instruction, and it carries one side
effect worth knowing about even though it doesn’t matter for this call
site: it unconditionally unmasks NMIs, something the fast-path
`SERIALIZE` instruction simply does not do.

One more candidate is conspicuously missing. `CPUID` also serializes,
and on paper looks like the most portable option of all. The kernel
avoids it here for a practical reason, not a correctness one: under
virtualization, `CPUID` commonly traps out to the hypervisor, and this
is exactly the kind of hot, latency-sensitive path — run on every online
CPU, on every single key toggle — that cannot tolerate an unpredictable
VM exit in the middle of it.

Put together, the choice between the two real options is not a fallback
chain padded with special cases — it is a single feature check, decided
once per call:

``` c
static __always_inline void sync_core(void)
{
        if (static_cpu_has(X86_FEATURE_SERIALIZE)) {
                serialize();
                return;
        }
        iret_to_self();
}
```

### 10.5 Writing through RO mappings {#writing-through-ro-mappings}

Every store [§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish)
walked through — the INT3, the tail bytes, the final first byte — is not
a raw write to `.text`. Each one is a full call to
[`text_poke()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2668),
meaning each one pays the entire
[§1.4](#writing-read-only-kernel-text-text_poke) dance in full: build
the temporary alias in
[`text_poke_mm`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2515),
switch `%cr3` onto it, copy through `STAC`/`CLAC`, switch back, tear the
alias down. Nothing about these writes being unusually small — as small
as the single INT3 byte in phase 1 — or unusually frequent lets any of
them skip a step.

All of that happens under
[`text_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/extable.c#L27),
and this is the piece [§1.4](#writing-read-only-kernel-text-text_poke)
leaned on without yet saying where it comes from:
[`text_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/extable.c#L27)
is what stops two unrelated patchers — jump labels, static calls,
ftrace, kprobes, the alternatives machinery — from ever building two
competing temporary mappings onto the same
[`text_poke_mm`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2515)
address space at once. Jump labels layer a second lock,
[`jump_label_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L23),
on top of that, but the two are not protecting the same thing:
[`text_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/extable.c#L27)
serializes individual pokes at the hardware level, while
[`jump_label_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L23)
serializes the higher-level operation of enabling or disabling one whole
key — the
[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
counter update and the walk over the
[`jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111)
run for that key, not just the bytes it eventually writes.

The two are also held for deliberately different spans. On the queueing
path ([§10.2](#batching-api-used-by-jump-labels)),
[`text_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/extable.c#L27)
is acquired and released once per call to
[`arch_jump_label_transform_queue()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L123)
— bracketing only the
[`__jump_label_patch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L36)
computation for that one site and its single
[`smp_text_poke_batch_add()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L3202)
append. It is free again in between sites, so some unrelated
[`text_poke()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2668)
caller elsewhere in the kernel is free to interleave its own single-site
poke while jump labels are still accumulating theirs for this key. Only
once the whole batch is ready does
[`arch_jump_label_transform_apply()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L143)
take
[`text_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/extable.c#L27)
back and hold it continuously across the entire three-phase
[`smp_text_poke_batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
from [§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish) — that
phase genuinely cannot tolerate a second patcher walking in mid-batch,
since the correctness argument of the INT3 protocol assumes the batch
array it iterates is exactly the one it built.
[`jump_label_mutex`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L23),
by contrast, stays held across all of that from the first line of
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
onward — there is no benefit to releasing it early, since doing so would
only let a second, unrelated
[`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)
call start interleaving its own bookkeeping with that of this one, not
let this one finish any faster.

### 10.6 End-to-end timeline for one enable {#end-to-end-timeline-for-one-enable}

[§10.1](#early-boot-vs-live-smp) through
[§10.5](#writing-through-ro-mappings) examined the machinery one piece
at a time — which patch path early boot takes, how sites get batched,
the three INT3 phases, what “synchronize” actually does on the wire, and
how each of those writes reaches memory that is nominally read-only.
Laid end to end, a single
[`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)
call looks like this:

    static_branch_enable(&key)
      cpus_read_lock()
      jump_label_mutex
      enabled = -1
      jump_label_update(key)
        for each jump_entry of key:
          __jump_label_patch()           # compute nop↔jmp bytes, sanity memcmp
          smp_text_poke_batch_add()      # append to vector (may flush if full)
        smp_text_poke_batch_finish()
          arm text_poke_array_refs = 1 on every CPU (smp_wmb)
          text_poke INT3 × N
          IPI sync                          # step 1
          text_poke tails (bytes 1..N-1) × N
          IPI sync                          # step 2 ("paranoid")
          text_poke first bytes × N
          IPI sync                          # step 3
          drain: wait for text_poke_array_refs == 0 on every CPU
          text_poke_array.nr_entries = 0    # buffer free for next batch
      enabled = 1 (release)
      unlock…

After this, every previously-`nop` site for that key is a `jmp` (or vice
versa), and the hot path behavior has flipped — without any flag load.

Notice what does, and does not, scale with the number of call sites. A
key can have one
[`jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111)
or a few hundred, but the number of IPI rounds is fixed at three,[^4]
because
[`smp_text_poke_batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
runs its three phases once over the *entire* batched vector, not once
per site ([§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish)).
The only thing that grows that fixed cost is the
[§10.2](#batching-api-used-by-jump-labels) overflow case: past 256
queued sites, a second full
[`batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
round is required, so a tracepoint with, say, 300 call sites pays six
IPI rounds total, not `300 x 3`.

------------------------------------------------------------------------

## 11 Modules: the trickiest part {#modules-the-trickiest-part}

Static keys often live in `vmlinux` (or module *A*) while call sites
live in module *B* — a tracepoint defined in the core kernel, say, with
`trace_*()` call sites scattered across several drivers loaded as
modules. The picture from [§7.1](#struct-static_key) of
[`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
as one pointer into one contiguous run of
[`jump_entry`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L111)
records only holds while every call site for a key lives in a single
object.

Modules break that assumption in the least convenient way possible: they
load and unload independently of `vmlinux` and of each other, in an
order nothing can predict ahead of time, so the representation has to be
able to grow and shrink at runtime instead of being settled once at boot
the way [§9.1](#boot-jump_label_init) settles it for the `vmlinux`-only
case. That is what makes this section “the trickiest part” — every step
has to stay correct no matter how many objects currently contribute to a
key, or in what order they arrived. Over its lifetime, that one
word/pointer union from [§7.1](#struct-static_key) can end up in exactly
three states:

- **One home only:** every call site for the key lives in the same
  single object (`vmlinux`, or one module), so
  [`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
  points directly at the contiguous run of that object. This is the fast
  path, and the common case.
- **Linked:** a second object has registered call sites for the same
  key, so one pointer is no longer enough.
  [`key->next`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L98)
  instead heads a list of
  [`struct static_key_mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L613)
  nodes, one per contributing object, each pointing at the run that
  object owns:

``` c
/*
               next                  next                  next
 key (LINKED) -----> static_key_mod -----> static_key_mod -----> NULL
                     (mod A / vmlinux)     (mod B)
                         |                    |
                      entries              entries
                         |                    |
                         v                    v
                    [jump_entry…]        [jump_entry…]
*/
struct static_key_mod {
        struct static_key_mod *next;
        struct jump_entry *entries;
        struct module *mod;
};
```

- **Sealed:** a key living in
  [`__ro_after_init`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/cache.h#L60)
  storage has its entries pointer permanently forgotten once
  [`jump_label_init_ro()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L573)
  runs ([§9.1](#boot-jump_label_init) covers the mechanics and the bit
  layout in full). A sealed key never needs linking into anything again,
  by any module, ever.

Concretely: say the
[`static_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L86)
of a tracepoint is declared in `vmlinux`, but nothing in the core kernel
itself calls it — only the driver module *A* does, loaded first. Module
*A* becomes the sole contributor for the key, so
[`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
points straight at the own run of module *A*, no list involved (the
one-home-only case above, even though the *struct* of the key lives in
`vmlinux` while its only *call site* lives in a module — those are two
independent facts, and only the second one matters here). Module *B* now
loads and also calls the same tracepoint. A second object just started
contributing, so the key flips into linked mode: one
[`static_key_mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L613)
node is built to wrap what
[`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
already pointed at (the run of module *A*), a second node is built for
module *B*, and
[`key->next`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L98)
now heads that two-node list — exactly the diagram above.

Exactly two functions drive every transition between those three states:
[`jump_label_add_module()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L690)
when a module loads, and
[`jump_label_del_module()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L762)
when one unloads. Neither is called directly — both run from a module
notifier[^5]
([`jump_label_module_notify()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L817),
registered with
[`.priority = 1`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L845))
that hooks
[`MODULE_STATE_COMING`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L308)/
[`MODULE_STATE_GOING`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L309)
— the two transitions the module loader fires while a module is being
mapped in and while it is being torn down.

The priority value is not an arbitrary tie-breaker — it enforces a
strict order. Notifier chains run higher-priority callbacks first, and
while jump labels register at
[`.priority = 1`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L845),
tracepoints register their own, separate module notifier at
[`.priority = 0`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/tracepoint.c#L688)
(lower) — so the jump-label notifier always runs first. That ordering
matters because tracepoints are themselves built on static keys
([§4.10](#a-real-consumer-tracepoints)). When a new module loads
([`MODULE_STATE_COMING`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L308)),
its notifier wants to start touching the static keys behind its own
tracepoints — but those keys are only ready to be touched once
[`jump_label_add_module()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L690)
has registered (and patched where needed), the jump entries of that
module. Running the jump-label notifier first guarantees that the setup
is already done by the time the tracepoint notifier runs.

### 11.1 Loading: [`jump_label_add_module()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L690) {#loading-jump_label_add_module}

This is the function that actually produces every transition described
above, once per key contributed by the newly-loaded module. Trimmed of
its early-return-if-empty check, the `__init`-flag bookkeeping
[§9.1](#boot-jump_label_init) already covered, and its `-ENOMEM` error
paths, here it is, one piece at a time.

The signature, its local state, and the first thing it does:

``` c
static int jump_label_add_module(struct module *mod)
{
        struct jump_entry *iter_start = mod->jump_entries;
        struct jump_entry *iter_stop = iter_start + mod->num_jump_entries;
        struct jump_entry *iter;
        struct static_key *key = NULL;
        struct static_key_mod *jlm, *jlm2;

        jump_label_sort_entries(iter_start, iter_stop);
```

[`jump_label_sort_entries()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L80)
sorts the jump table of this module — the same routine
[§9.1](#boot-jump_label_init) used on the `vmlinux`-wide table, for the
same reason. After sorting, all entries that reference the same
[`static_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L86)
sit next to each other in the array.

That adjacency matters because the work this function does — allocating
[`static_key_mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L613)
nodes, wiring them into the list, deciding whether to patch — is
per-key, not per-call-site. A module that calls `trace_sched_switch()`
at ten different places still needs just one list node for that
tracepoint key, not ten. With entries sorted by key, the loop can handle
each distinct key exactly once: run the per-key body when the first
entry for a key appears, then skip all subsequent entries for the same
key with a single pointer comparison.

That skip pattern is what the top of the loop implements:

``` c
        for (iter = iter_start; iter < iter_stop; iter++) {
                struct static_key *iterk = jump_entry_key(iter);

                if (iterk == key)
                        continue;
                key = iterk;
```

The `for` loop advances `iter` through every entry in the table, one at
a time. On each iteration,
[`jump_entry_key()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L127)
reads which
[`static_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L86)
that entry belongs to. If it is the same key the loop just finished
processing, `continue` skips the entry — all per-key work was already
done when the first entry of that group was reached. Only when `iterk`
differs from `key` does execution fall through into the per-key body
below.

Two consequences follow from that design. First, the per-key body runs
*once per distinct key in this module*, not once per call site, and each
of the three states from the introduction of
[§11](#modules-the-trickiest-part) corresponds to exactly one path
through it. Second, `iter` at the point where the body runs always
points at the first entry of a key group in the sorted table — the start
of a contiguous run. That matters at the end of the function, where
`iter` is passed as a range start to
[`__jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L503),
which walks forward from there to patch every call site for that key in
this module.

Once inside the per-key body, the first check asks whether this module
even needs to consider the linked-list machinery at all — the
**one-home-only** case:

``` c
                if (within_module((unsigned long)key, mod)) {
                        static_key_set_entries(key, iter);   /* one home only */
                        continue;
                }
```

[`within_module()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L657)
asks whether the
[`static_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L86)
*struct itself* — not a call site, not a jump entry, but the struct that
holds
[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
and
[`entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
— lives inside memory owned by `mod`.

If it does, then `mod` is, by construction, the very first and only
object that has ever contributed call sites for this key. The reason is
physical: before this module loaded, the memory backing that
[`static_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L86)
was not even mapped. No other module or `vmlinux` could have built a
jump entry referencing an address that did not yet exist. The
one-home-only direct-pointer form is therefore not just adequate here,
it is the only form this key has ever needed — which is why this branch
`continue`s past all the linked-list machinery that follows.

A key that fails that check has call sites outside this module, so
before building any list node it is worth asking whether one is even
possible — the **sealed** case:

``` c
                if (static_key_sealed(key))
                        goto do_poke;   /* sealed: patch once, keep no link */
```

A sealed key has already forgotten its `entries`/`next` union for good
([§9.1](#boot-jump_label_init)), so there is nothing left to link this
module into. Instead the `goto` skips straight past all the
list-building below, to a single comparison shared with the ordinary
case — reached down in the text.

An ordinary, unsealed key that reaches this point is the **linked**
case. This is where the list from the introduction of
[§11](#modules-the-trickiest-part) actually gets built or grown, in two
steps.

The first step handles a one-time transition. Up to this point the key
might still be in direct-pointer form — one pointer, one contributing
object, no list. Before anything can be prepended to it, that existing
pointer has to be wrapped in a list node so there is a `next` field to
link through:

``` c
                jlm = kzalloc(sizeof(struct static_key_mod), GFP_KERNEL);
                if (!static_key_linked(key)) {
                        jlm2 = kzalloc(sizeof(struct static_key_mod), GFP_KERNEL);
                        scoped_guard(rcu)
                                jlm2->mod = __module_address((unsigned long)key);
                        jlm2->entries = static_key_entries(key);
                        static_key_set_mod(key, jlm2);
                        static_key_set_linked(key);
                }
```

[`static_key_linked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L420)
returns false only when the key is still in direct-pointer form, meaning
exactly one object has contributed to it so far. The code allocates
`jlm2` to retroactively wrap whatever
[`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
already pointed at — the run belonging to that first object — discovers
which module owns that object via
[`__module_address()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/module/main.c#L3887),
and installs `jlm2` as the head of a new one-node list. Once
[`static_key_set_linked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L428)
flips the linked bit, this wrapping step never runs again for this key —
every later module finds
[`static_key_linked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L420)
already true and skips straight into the second step.

The second step prepends a node for the newly-arriving module onto the
now-guaranteed-to-exist list:

``` c
                jlm->mod = mod;
                jlm->entries = iter;
                jlm->next = static_key_mod(key);
                static_key_set_mod(key, jlm);
                static_key_set_linked(key);
```

`jlm->entries` is set to `iter` — the first entry of this key group in
the sorted table, the same pointer the one-home-only case would have
stored directly into
[`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97).
[`static_key_set_mod()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L436)
makes `jlm` the new head of the list, with the previous head linked
behind it via `jlm->next`.

Both the sealed-key
[`goto`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L727)
and the ordinary path above fall into the same final check, once per
key:

``` c
do_poke:
                if (jump_label_type(iter) != jump_label_init_type(iter))
                        __jump_label_update(key, iter, iter_stop, true);
        }
        return 0;
}
```

Every call site a module ships with is compiled around one fixed
assumption: the default value the key had at compile time — the same
`type ^ branch` formula from
[§5](#two-polarities-key-default-branch-hint), exposed here as
[`jump_label_init_type()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L603)
— which decided whether the assembler macro emitted this site as a `nop`
or a `jmp` in the first place ([§5.1](#how-the-macros-pick-the-asm)),
frozen into the module from then on. But the *live* jump label type
value can have moved away from that compiled-in default already, if
something else called
[`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)/`disable()`
on this key before the module ever loaded.
[`do_poke`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L754)
catches exactly that mismatch and patches the new sites immediately,
before the code of the module has a chance to run and observe them in
the wrong state — whether it arrived there via the sealed-key
[`goto`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L727)
above, or by falling through normally after linking the key into the
list.

### 11.2 Toggling an already-linked key {#toggling-an-already-linked-key}

When someone calls
[`static_branch_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)/[`disable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L523)/[`inc()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L513)/[`dec()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L514)
on a key whose call sites span more than one object, the flat array scan
from [§9.5](#jump_label_update-__jump_label_update) is not enough — the
sites are scattered across separate per-object tables, each with its own
bounds, and the patcher has to visit every one of them.

[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
detects that case: if
[`static_key_linked()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L420)
returns true, it hands off to
[`__jump_label_mod_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L665)
instead of doing the walk itself. That function walks the linked list,
calling
[`__jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L503)
once per node:

``` c
static void __jump_label_mod_update(struct static_key *key)
{
        struct static_key_mod *mod;

        for (mod = static_key_mod(key); mod; mod = mod->next) {
                struct jump_entry *stop;
                struct module *m;

                if (!mod->entries)
                        continue;

                m = mod->mod;
                if (!m)
                        stop = __stop___jump_table;
                else
                        stop = m->jump_entries + m->num_jump_entries;
                __jump_label_update(key, mod->entries, stop,
                                    m && m->state == MODULE_STATE_COMING);
        }
}
```

The loop visits each
[`static_key_mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L613)
node in the list and passes that `entries` pointer and matching `stop`
bound to
[`__jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L503).
Three details in that loop deserve explanation.

First, `stop` cannot be a single kernel-wide constant the way it is in
the flat walk from [§9.5](#jump_label_update-__jump_label_update). Each
module has its own private
[`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/scripts/module.lds.S#L31)
section ([§7.4](#linker-section)), bounded by the
[`jump_entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L511)/[`num_jump_entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L512)
fields of that module. Without the correct per-node bound,
[`__jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L503)
would walk forward past the end of a module table into unrelated memory.
The loop has to look up that bound for each node individually.

Second, `m` — the
[`mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L616)
field of the list node, not the function parameter — can be `NULL`. That
is not an error: it is the node that represents `vmlinux` itself. The
`NULL` originates in [§11.1](#loading-jump_label_add_module): the
first-time linking step calls
[`__module_address()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/module/main.c#L3887)
to discover which module owns the key, and
[`__module_address()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/module/main.c#L3887)
returns `NULL` for addresses inside the core kernel. For that node the
correct bound is the global
[`__stop___jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L209),
which marks the end of the vmlinux-wide table.

Third,
[`mod->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L615)
can genuinely be `NULL`. That happens when an object defines a key but
contains no call sites for it — the linking code from
[§11.1](#loading-jump_label_add_module) still creates a list node to
represent that object (it *is* a contributor), but there is nothing
there to patch. Concretely: a module exports a
[`DEFINE_STATIC_KEY_FALSE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L371),
and call sites in other modules are the only consumers. Skipping the
node is correct.

The final argument, `m && m->state == MODULE_STATE_COMING`, tells
[`__jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L503)
whether to also patch entries living in the `__init` section of the
module. A module still in
[`MODULE_STATE_COMING`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L308)
has not finished running its init function yet, so its init section is
still mapped and its call sites there are reachable. A module already in
[`MODULE_STATE_LIVE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L307)
has discarded that section — patching into freed memory would be a
use-after-free, not a harmless no-op.

### 11.3 Unloading: [`jump_label_del_module()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L762) {#unloading-jump_label_del_module}

[`jump_label_del_module()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L762)
is the mirror image of
[`jump_label_add_module()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L690),
run on
[`MODULE_STATE_GOING`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L309):
for each distinct key this module contributes to, it finds and removes
the
[`static_key_mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L613)
node that [§11.1](#loading-jump_label_add_module) created. The
text-patching machinery does not need to undo anything here — the
`.text` section of the module is about to be unmapped entirely, so the
patchable sites in it simply cease to exist. What *does* need cleaning
up is the linked-list bookkeeping that still references them.

The function uses the same sorted-table, skip-by-key loop structure as
loading. Three of the four skip cases mirror
[§11.1](#loading-jump_label_add_module) directly:

``` c
        for (iter = iter_start; iter < iter_stop; iter++) {
                if (jump_entry_key(iter) == key)
                        continue;

                key = jump_entry_key(iter);

                if (within_module((unsigned long)key, mod))
                        continue;

                /* No @jlm allocated because key was sealed at init. */
                if (static_key_sealed(key))
                        continue;

                /* No memory during module load */
                if (WARN_ON(!static_key_linked(key)))
                        continue;
```

The first three are familiar from
[§11.1](#loading-jump_label_add_module): skip duplicate entries for the
same key (the sorted-table grouping from the sort step), skip keys whose
struct lives inside this module (the struct vanishes with the module, so
there is no list to update), and skip sealed keys (no node was ever
created for them). The fourth is a defensive check: if the key is not in
the linked state at this point, something went wrong during loading —
likely an allocation failure that
[`jump_label_add_module()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L690)
could not recover from. The
[`WARN_ON`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/bug.h#L118)
flags the inconsistency without crashing[^6], and the `continue` skips
the key rather than dereferencing a pointer that was never set up.

For every key that passes all four checks, the function walks the linked
list to find and splice out the node belonging to this module:

``` c
                prev = &key->next;
                jlm = static_key_mod(key);

                while (jlm && jlm->mod != mod) {
                        prev = &jlm->next;
                        jlm = jlm->next;
                }

                /* No memory during module load */
                if (WARN_ON(!jlm))
                        continue;

                if (prev == &key->next)
                        static_key_set_mod(key, jlm->next);
                else
                        *prev = jlm->next;

                kfree(jlm);
```

The `while` loop advances through the list until it finds the node whose
[`mod`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L616)
field matches the departing module, keeping `prev` pointed at the `next`
pointer of the preceding node so the splice has something to patch. If
no matching node is found — again a sign that something went wrong
during loading — a second
[`WARN_ON`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/bug.h#L118)
fires and the key is skipped. Otherwise, the standard singly-linked-list
splice removes the node: if it was the head of the list
(`prev == &key->next`), the `next` pointer of the key itself is updated
via
[`static_key_set_mod()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L436);
if it was somewhere in the middle, the `next` pointer of the predecessor
is patched directly.

After the splice, one more step checks whether the list can be
eliminated entirely:

``` c
                jlm = static_key_mod(key);
                /* if only one etry is left, fold it back into the static_key */
                if (jlm->next == NULL) {
                        static_key_set_entries(key, jlm->entries);
                        static_key_clear_linked(key);
                        kfree(jlm);
                }
```

If exactly one node remains, the key no longer needs the list form —
only one object still contributes to it. The code folds that last node’s
`entries` pointer back into
[`key->entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
directly, clears the
[`LINKED`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L196)
bit, and frees the node. This is the linked-state transition from
[§11.1](#loading-jump_label_add_module) running in reverse: a key that
needed the list form only because two objects happened to overlap
returns to the one-home-only direct-pointer form the moment that overlap
ends.

------------------------------------------------------------------------

## 12 Fallback: [`CONFIG_JUMP_LABEL=n`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L130) {#fallback-config_jump_label-n}

[`CONFIG_JUMP_LABEL`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L130)
is optional. Most distro kernels end up with it on — arm64 selects it
outright, and on x86
[`PREEMPT_DYNAMIC`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/Kconfig.preempt#L130)
pulls it in — but a minimal config can legitimately leave it off.
Everything from [§5](#two-polarities-key-default-branch-hint) onward
assumed the option was enabled. This section covers what happens when it
is not.

Without `CONFIG_JUMP_LABEL`, the struct shrinks to a bare counter. The
`entries`/`next`/`type` union from [§7](#core-data-structures) is
compiled out entirely:

``` c
struct static_key {
        atomic_t enabled;
};
```

[`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L251)
sets
[`static_key_initialized`](https://elixir.bootlin.com/linux/v7.2-rc7/source/init/main.c#L174)
to `true` and returns. There is no jump table to sort, no entries to
pre-patch.

The call-site macros turn into ordinary branch-hinted conditionals.
[`static_branch_likely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L500)
and
[`static_branch_unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L501)
reduce to:

``` c
#define static_branch_likely(x)   likely_notrace(static_key_enabled(&(x)->key))
#define static_branch_unlikely(x) unlikely_notrace(static_key_enabled(&(x)->key))
```

No `asm goto`, no jump table, no patching — just a
[`likely_notrace()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/compiler.h#L78)/[`unlikely_notrace()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/compiler.h#L79)
hint around a read of `enabled`.

[`static_key_count()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L246)
is a plain
[`raw_atomic_read()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/atomic/atomic-arch-fallback.h#L455):

``` c
static __always_inline int static_key_count(struct static_key *key)
{
        return raw_atomic_read(&key->enabled);
}
```

Compare this with the `CONFIG_JUMP_LABEL=y` version in
[`kernel/jump_label.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L95),
which clamps negative values (`n >= 0 ? n : 1`). That clamp exists
because
[`static_key_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L220)
temporarily sets `enabled` to `-1` while the patching pass runs (see
[§9.2](#enabling-static_key_enable-static_branch_enable)). Without
patching, `enabled` never goes negative, so the clamp is unnecessary.

[`static_key_enable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L307)
and
[`static_key_disable()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L318)
are simpler for the same reason. Each one checks whether `enabled`
already holds the target value and returns early if so. If it holds
something unexpected (neither 0 nor 1), a
[`WARN_ON_ONCE`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/bug.h#L118)
fires. Otherwise, a plain `atomic_set()` writes the new value. No
`cmpxchg`, no intermediate `-1`, no
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
call.

[`jump_label_lock()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L304)/[`jump_label_unlock()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L305)
are empty stubs — there is no patch pass to serialize.

The net effect on every hot path is exactly the cost
[§2](#the-problem-jump-labels-solve) opened with: a memory load of
[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87),
a compare, and a conditional branch. The
[`likely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/compiler.h#L76)/[`unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/compiler.h#L77)
hint steers the branch predictor the same way a compiled-in NOP or JMP
would, but it cannot eliminate the branch itself — that is the
optimization that `CONFIG_JUMP_LABEL=y` adds.

Jump labels are an **optimization**, not a correctness feature: behavior
matches; only the mechanism changes.

------------------------------------------------------------------------

## 13 Worked micro-example (bytes on the wire) {#worked-micro-example-bytes-on-the-wire}

Every mechanism described so far — the asm helpers, the jump-table
entry, the objtool hack, the size-discovery check, the
[`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
state machine, and the INT3 protocol — touches one concrete call site at
some point in its life. This section traces a single, minimal site
through that entire life, from compile time through one enable and one
disable, close enough to see the actual instruction bytes change rather
than just the names of the steps that change them. Suppose:

``` c
DEFINE_STATIC_KEY_FALSE(k);

void f(void)
{
        if (static_branch_unlikely(&k))
                printk("on\n");
        something();
}
```

Two small assumptions turn this from a symbolic description into an
actual trace, and neither changes anything about *how* the mechanism
works, only which specific numbers show up:

1.  `f()` is called after boot, on a live, multi-CPU system — the
    interesting [§10](#x86-text-patching-the-gory-details) INT3 path,
    not the single-CPU [§10.1](#early-boot-vs-live-smp) boot shortcut (a
    boot-time toggle of this same key would use
    [`text_poke_early()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2489)
    instead, with no INT3 involved at all).
2.  The compiler places `l_yes` — the out-of-line block containing the
    `printk()` call — 80 bytes past the end of the patch site. 80 fits
    in a signed byte, so the build-time trick from
    [§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes) picks the
    2-byte `JMP rel8` encoding rather than the 5-byte `rel32` form. A
    farther `l_yes` would just mean 5 bytes instead of 2 everywhere
    below ([§1.2](#x86-instruction-encoding-jmp-and-nop)); nothing else
    about the trace would change.

### 13.1 Compile / link / `objtool` ([`HAVE_JUMP_LABEL_HACK`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/Kconfig#L1399)) {#compile-link-objtool-have_jump_label_hack}

From the source code above, the compiler, linker, and `objtool` produce
the bytes that sit in `vmlinux` at the patch site. For orientation, here
is where the three pieces end up — the patch site in `.text`, the
out-of-line target, and the sidecar entry in `__jump_table`:

     .text (function f)                  __jump_table (one entry)
     ──────────────────────────         ──────────────────────────
           ...                           code:   delta to 1:
      1:   [  2 bytes  ]  patch site     target: delta to l_yes
           ...                           key:    delta to &k.key + 2
           call something                         bit 0 = 0 (branch)
           ret                                    bit 1 = 1 (objtool)
           ...
      l_yes:     80 bytes past 1:+2
           call printk
           jmp back ------> (after 1:+2)

The six steps below trace how those bytes arrive at their final state:

1.  `k` is a `FALSE` key read with
    [`static_branch_unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L486).
    Per the table in [§5.1](#how-the-macros-pick-the-asm), that
    combination calls the nop-default helper
    [`arch_static_branch(&k.key, false)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35),
    not the jmp-default
    [`arch_static_branch_jump()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L45).
    That is the `type ^ branch` formula from
    [§5](#two-polarities-key-default-branch-hint) at work: `type = 0`
    (`FALSE`), `branch = 0` (`unlikely`), so `type ^ branch = 0` — the
    hint agrees with the default, and the nop-default path gets chosen.

2.  The `asm goto` inside
    [`arch_static_branch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L35)
    emits `1: jmp l_yes` at the patch site, plus one raw row in
    [`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/vmlinux.lds.h#L436)
    via
    [`JUMP_TABLE_ENTRY()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/jump_label.h#L15)
    ([§6.2](#the-jump-table-entry-sidecar-metadata)): `code` is the
    self-relative distance to `1:`, `target` is the self-relative
    distance to `l_yes`, and `key` is the self-relative distance to
    `&k.key + 0 + 2`.

    That `+ 2` puts a `1` in bit 1 of the stored `key` address. Bit 0 is
    `branch` — here `0`, matching the
    [`unlikely()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/compiler.h#L77)
    hint. Bit 1 is the build-time signal to `objtool`: “NOP this jmp”
    ([§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes)); after
    boot,
    [`jump_entry_set_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L163)
    repurposes this same bit as the `__init`-text flag. At runtime,
    [`jump_entry_key()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L127)
    masks both bits off to recover the real address of `k`.

3.  The assembler picks the encoding by the real distance to `l_yes` —
    here, 80 bytes forward (assumption 2 above), well inside the
    -128..+127 reach of a signed byte. It emits the 2-byte `JMP rel8`
    form: opcode `EB`, followed by
    `disp = dest - (addr + insn_size) = 80 = 0x50` (the disp formula
    from [§1.2](#x86-instruction-encoding-jmp-and-nop)). The two bytes
    actually sitting at `1:` right after assembly, before `objtool` ever
    runs, are `EB 50`.

4.  During the build, `objtool` calls
    [`handle_jump_alt()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/tools/objtool/check.c#L1872)
    ([§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes) step 3),
    which sees bit 1 set in the stored `key` operand and rewrites those
    exact two bytes, in place, from the `jmp` (`EB 50`) to the 2-byte
    NOP (`66 90`, the NOP encoding from
    [§1.2](#x86-instruction-encoding-jmp-and-nop)) — same size, so
    nothing around the site shifts. By the time `vmlinux` is linked, the
    live bytes at `1:` are already `66 90`, and `objtool` itself is long
    gone.

5.  At boot,
    [`jump_label_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L525)
    sorts
    [`__jump_table`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/asm-generic/vmlinux.lds.h#L436)
    by key, then loops over every entry ([§9.1](#boot-jump_label_init)).
    For the one entry belonging to `k`, the loop body does the
    following:

         jump_label_init()
         ├── jump_label_sort_entries()                sort __jump_table by key
         └── for each entry:                          (k has exactly one)
             ├── jump_label_type() = NOP              enabled(0) ^ branch(0)
             │   └── arch_jump_label_transform_static()  no-op on x86
             ├── jump_entry_set_init(entry, false)    code not in __init
             └── static_key_set_entries(&k, entry)    wires k.entries → entry

    [`jump_label_type()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L455)
    computes `enabled(0) ^ branch(0) = NOP` — the expected state matches
    the live bytes, which are already `66 90`. So
    [`arch_jump_label_transform_static()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L402)
    is a genuine no-op: x86 never overrides the generic fallback, whose
    entire body is one comment,
    `/* nothing to do on most architectures */`. No instruction bytes
    get rewritten at this site.

6.  The same loop iteration does two pieces of bookkeeping that wire `k`
    into the runtime data structures. First,
    [`jump_entry_set_init()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L163)
    checks whether the code at `1:` lives in `__init` text — it does
    not, so bit 1 of the stored `key` address (the same bit `objtool`
    used in step 2) gets cleared to `0`. Second,
    [`static_key_set_entries()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L444)
    points
    [`k.entries`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L97)
    at this entry — the pointer that
    [`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
    will follow when
    [`static_branch_enable(&k)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)
    runs later. With that in place, the hot path in `f()` from the very
    first time it runs is: decode `66 90` (falls through, no load of
    `k`, [§1.1](#what-a-cpu-actually-does-with-instructions)), then
    `call something`.

In summary, the same two bytes at `1:` passed through three stages
before the kernel ever ran a line of `f()`:

     Stage                Bytes at 1:    Why
     ─────────────────    ───────────    ──────────────────────────────────
     After assembly       EB 50 (JMP)   assembler picks rel8 for +80 distance
     After objtool        66 90 (NOP)   handle_jump_alt() sees bit 1 in key
     At boot              66 90 (NOP)   jump_label_init(): NOP expected, NOP found

### 13.2 Runtime [`static_branch_enable(&k)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522) {#runtime-static_branch_enable-k}

[§10.6](#end-to-end-timeline-for-one-enable) already lays out the full
call chain a batch of sites goes through on enable; `k` has exactly one
entry, so this is that same chain with the actual bytes for every step
filled in. Both directions of the public API are one-line macros in
[`include/linux/jump_label.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h):

``` c
#define static_branch_enable(x) static_key_enable(&(x)->key)
#define static_branch_disable(x)    static_key_disable(&(x)->key)
```

The full call chain for this one enable, with the concrete byte values
for `k` filled in at each level:

     static_branch_enable(&k)
     └─ static_key_enable(&k.key)
        ├── cpus_read_lock / jump_label_lock
        ├── enabled: 0 --> -1              callers already see "on"
        ├── jump_label_update(key)
        │   └─ __jump_label_update()
        │      ├── type = enabled(true) ^ branch(0) = JMP
        │      ├── arch_jump_label_transform_queue()
        │      │   └─ __jump_label_patch(entry, JMP)
        │      │      ├── size  = 2        (live-decode of 66 90)
        │      │      ├── code  = EB 50    (text_gen_insn)
        │      │      ├── nop   = 66 90    (x86_nops[2])
        │      │      ├── memcmp(addr, nop) -- pre-flight OK
        │      │      └── smp_text_poke_batch_add(addr, EB 50, 2)
        │      └── arch_jump_label_transform_apply()
        │          └─ smp_text_poke_batch_finish()
        │             └── INT3 three-phase: 66 90 --> EB 50
        ├── enabled: -1 --> 1              atomic_set_release
        └── jump_label_unlock / cpus_read_unlock

The numbered steps below walk through this chain in detail:

1.  [`static_branch_enable(&k)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L522)
    expands to
    [`static_key_enable(&k.key)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L220),
    which acquires
    [`jump_label_lock()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L25)
    and walks the `0 → -1 → (patch) → 1` state machine from
    [§9.2](#enabling-static_key_enable-static_branch_enable).
    [`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
    starts at `0`, gets set to `-1` (“enabling in progress”), and
    [`static_key_count()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L104)
    already reports `-1` as “on” — so no caller sees a false “off”
    window while patching runs. Then
    [`jump_label_update(&k.key)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
    does the actual patching, and only after it returns does
    [`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
    get published as `1` with release ordering.

2.  [`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
    finds the one entry of `k` and computes
    [`jump_label_type(entry)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L455)
    = `enabled ^ branch`.
    [`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
    is the transient `-1` from step 1, which
    [`static_key_enabled()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L407)
    reports as `true`; `branch` is the stored hint bit, `0`.
    `true ^ false = JMP` — the live nop becomes a jmp.

3.  [`arch_jump_label_transform_queue()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L123)
    calls
    [`__jump_label_patch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L36),
    which re-derives the size by decoding the live bytes at `1:` —
    [`arch_jump_entry_size()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L20)
    returns 2, matching what `objtool` left behind
    ([§8](#size-of-the-patchable-site-on-x86-runtime)). The function
    then builds both sequences: `nop = 66 90` (from `x86_nops[2]`) and
    `code = EB 50` (from
    [`text_gen_insn()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h#L123)
    — the same bytes computed in
    [§13.1](#compile-link-objtool-have_jump_label_hack) step 3, since
    `addr` and `dest` have not moved). Because this is a nop→jmp
    transition, it calls
    [`memcmp()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/lib/string.c#L655)
    to verify the live bytes are currently `66 90`; a mismatch would be
    a
    [`BUG()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/bug.h#L114).
    They match, so the patch `66 90` → `EB 50` gets queued via
    [`smp_text_poke_batch_add()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L3202)
    ([§10.2](#batching-api-used-by-jump-labels)).

4.  The loop of
    [`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
    over the entries of `k` ends here — there is only the one — so
    [`arch_jump_label_transform_apply()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L143)
    immediately calls
    [`smp_text_poke_batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941),
    which runs the three-phase protocol from
    [§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish) on this
    one queued site, now with the real two bytes instead of a
    placeholder:

        start (before)     66 90              any fetch: executes the 2-byte NOP

        phase 1 (INT3 in)  cc 90              any fetch: #BP -> handler emulates
                           ^^                 the pending JMP (jumps to l_yes)
                           trap byte

                           -------- IPI sync --------

        phase 2 (tail in)  cc 50              same as phase 1: byte 0 still
                           ^^                 traps and gets emulated — byte 1
                           still traps        is now its final value underneath

                           -------- IPI sync --------

        phase 3 (done)     eb 50              any fetch: executes the real
                           ^^ real opcode     JMP rel8 directly, no trap needed

                           -------- IPI sync --------

5.  From the instant the sync in phase 1 completes, any CPU landing on
    this address already gets the effect of the jump via emulation
    ([§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish));
    phases 2-3 only retire the trap-and-emulate path in favor of the
    real bytes. Once phase 3 lands, every subsequent call to `f()`
    decodes `EB 50`, jumps 80 bytes forward into `l_yes`, runs
    `printk("on\n")`, then hits the compiler-emitted `jmp back`
    ([§3](#the-mental-model-in-one-diagram)) and falls into
    `something()`.

### 13.3 [`static_branch_disable(&k)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L523): same protocol, asymmetric math {#static_branch_disable-k-same-protocol-asymmetric-math}

Disabling runs the mirror call,
[`static_branch_disable(&k)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L523)
→
[`static_key_disable(&k.key)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L245)
— a single
[`atomic_cmpxchg(&key->enabled, 1, 0)`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L239)
instead of the enable state machine, because disable has no in-progress
state to protect ([§9.3](#disabling): a reader who sees stale “on” for a
few more instructions is exactly the harmless case, unlike stale “off”).
If that `cmpxchg` succeeds,
[`jump_label_update()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L886)
runs again, this time computing
[`jump_label_type()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/jump_label.c#L455)
`= false ^ false = NOP`
([`enabled`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L87)
now `0`, `branch` still `0`).

[`__jump_label_patch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L36)
rebuilds the *same* two sequences as before — `nop = 66 90`,
`code = EB 50`, both unchanged, since `addr`, `dest`, and `size` haven’t
moved — but this time expects the live bytes to be the jmp and installs
the nop. That is the one genuine asymmetry in this whole worked example:
enabling always needs a fresh, target-specific displacement computed by
[`text_gen_insn()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/include/asm/text-patching.h#L123);
disabling never does, because the encoding of a NOP does not depend on
where the branch would have gone — the exact same fixed bytes go back
every time a site of this size is disabled, no matter what it was
jumping to.

The same three-phase protocol
([§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish)) runs
again, in the opposite byte direction:

     start (before)     eb 50              executes the JMP rel8

     phase 1 (INT3 in)  cc 50              #BP -> handler now emulates the
                        ^^                 pending NOP (a JMP with disp == 0
                        trap byte          — not a special case)

                        -------- IPI sync --------

     phase 2 (tail in)  cc 90              byte 1 now its final value; byte 0
                        ^^                 still traps
                        still traps

                        -------- IPI sync --------

     phase 3 (done)     66 90              executes the real NOP directly
                        ^^ real opcode

                        -------- IPI sync --------

Put together, the build-time `66 90` of this one site, the `EB 50` of
the first enable, and the `66 90` of this disable again are the entire
lifecycle that
[§1](#hardware-background-why-this-is-hard)-[§10](#x86-text-patching-the-gory-details)
spent this whole tutorial describing in the abstract — the same two
bytes, chosen and re-derived by a different mechanism at each stage, but
never touched by anything other than the three sanctioned writers: the
assembler once at compile time, `objtool` once at build time, and
[`__jump_label_patch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L36)
through the protocol from
[§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish) every time
after that.

| Stage | Live bytes at `1:` | Who wrote them |
|----|----|----|
| After assembly, before `objtool` | `EB 50` | Compiler/assembler ([§6.1](#the-two-asm-helpers), [§1.2](#x86-instruction-encoding-jmp-and-nop)) |
| After `objtool`, at boot | `66 90` | [`handle_jump_alt()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/tools/objtool/check.c#L1872) ([§6.3](#have_jump_label_hack-why-sites-are-2-or-5-bytes)) |
| After `static_branch_enable(&k)` | `EB 50` | [`__jump_label_patch()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/jump_label.c#L36) via INT3 protocol ([§10.3](#the-int3-smp-algorithm-smp_text_poke_batch_finish)) |
| After `static_branch_disable(&k)` | `66 90` | Same, reverse direction |

------------------------------------------------------------------------

## 14 Further reading in-tree {#further-reading-in-tree}

- [`Documentation/staging/static-keys.rst`](https://elixir.bootlin.com/linux/v7.2-rc7/source/Documentation/staging/static-keys.rst)
  — historical overview and old `getppid` instruction dump (still
  useful; ignore fixed “always 5 bytes” claims for modern x86).
- [`arch/x86/kernel/alternative.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
  —
  [`smp_text_poke_batch_finish`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941)
  comment block (canonical INT3 algorithm write-up).
- [`tools/objtool/check.c`](https://elixir.bootlin.com/linux/v7.2-rc7/source/tools/objtool/check.c#L1872)
  —
  [`handle_jump_alt()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/tools/objtool/check.c#L1872)
  for the jmp→nop hack.
- [`include/linux/jump_label.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/jump_label.h#L1)
  — the the top comment in the header: deprecated vs. current API,
  behavioral model, the “absolute slow paths” warning,
  deferred-decrement rationale.
- [`include/linux/tracepoint-defs.h`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/tracepoint-defs.h#L39)
  —
  [`struct tracepoint`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/tracepoint-defs.h#L39)
  embeds both a
  [`static_key_false`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/tracepoint-defs.h#L41)
  and a
  [`static_call_key`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/tracepoint-defs.h#L42)
  — the largest real-world consumer of static keys, and the intersection
  of both sister mechanisms.
- Sister mechanisms with the same poke engine: [**static
  calls**](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/static_call.h),
  [**ftrace**](https://elixir.bootlin.com/linux/v7.2-rc7/source/Documentation/trace/ftrace-design.rst),
  [**kprobes**](https://elixir.bootlin.com/linux/v7.2-rc7/source/Documentation/trace/kprobes.rst)
  — same
  [`text_poke`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2668)
  / INT3 infrastructure, different metadata sections.

------------------------------------------------------------------------

[^1]: A 4-byte signed field covers ±2 GiB, and x86_64 kernels are built
    with `-mcmodel=kernel`, which packs the entire kernel image into the
    top 2 GiB of the address space specifically so that a 32-bit
    relative displacement is always enough — no two locations inside
    `vmlinux` can ever be farther apart than a `rel32` can reach.

[^2]: `__ro_after_init` is a section attribute
    (`include/linux/cache.h:60`, `__section(".data..ro_after_init")`)
    for data that is written during boot but never again afterward —
    unlike `const`, which the compiler must be able to enforce at
    compile time, this is a promise the *author* makes about runtime
    behavior. The kernel makes that promise real at
    [`mark_rodata_ro()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/mm/init_64.c#L1405)
    time, when the whole `.data..ro_after_init` section is remapped
    read-only in the page tables, so any later write attempt — a bug, or
    an author breaking their own promise — faults instead of silently
    corrupting state.

[^3]: `_stext` is a linker-defined symbol, not a C variable — it marks
    the address where the `.text` section of the kernel begins, set
    directly in
    [`arch/x86/kernel/vmlinux.lds.S`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/vmlinux.lds.S#L137).
    Every function in core kernel text lives at some fixed offset from
    it, which is what lets `smp_text_poke_loc.rel_addr` (a plain `s32`)
    address any patch site with 4 bytes instead of the full 8-byte
    pointer `jump_entry.key` needs
    ([§6.2](#the-jump-table-entry-sidecar-metadata)) for its
    potentially-far-away `static_key`.

[^4]: Strictly, phases 2 and 3 each sync only if at least one site in
    the batch actually needed a write in that phase, tracked by a
    `do_sync` counter inside
    [`smp_text_poke_batch_finish()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/arch/x86/kernel/alternative.c#L2941).
    The phase 3 sync, for instance, is skipped if the final first byte
    of every site already happens to equal `INT3`. An ordinary
    nop-to-jmp toggle always writes something in both phases, so three
    rounds is what actually happens in practice; the fixed count just
    isn’t unconditional at the code level.

[^5]: A module notifier is a callback registered against the kernel’s
    module-load notifier chain
    ([`module_notify_list`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/module/main.c#L164))
    via
    [`register_module_notifier()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/module/main.c#L166).
    The module loader walks that chain with
    [`blocking_notifier_call_chain()`](https://elixir.bootlin.com/linux/v7.2-rc7/source/kernel/notifier.c#L368)
    at each state transition
    ([`MODULE_STATE_COMING`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L308),
    [`MODULE_STATE_GOING`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/module.h#L309),
    etc.), invoking every registered
    [`struct notifier_block`](https://elixir.bootlin.com/linux/v7.2-rc7/source/include/linux/notifier.h#L54)
    in priority order — this is the generic mechanism subsystems use to
    react to modules loading/unloading, not something jump labels
    invented.

[^6]: Unless [`panic_on_warn = 1`](https://lwn.net/Articles/969923/)
