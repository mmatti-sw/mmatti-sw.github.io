---
layout: post
title: "Enabling C23 _BitInt on Power Architecture"
subtitle: "ABI design for three variants, two cross-platform middle-end bugs, and what it cost in generated code"
description: "How C23 _BitInt was enabled in GCC for ppc64le, ppc64be and ppc32: one ABI rule for three variants, two big-endian middle-end bugs, and measured codegen cost."
author: "Manjunath S Matti"
date: 2026-10-07
tags: [gcc, powerpc, c23, bitint, abi, ppc64le, ppc64be, ppc32]
---

*Manjunath S Matti · IBM Linux Technology Center*

C23 introduced `_BitInt(N)`, an integer type with exactly the width you ask for. GCC has carried the front-end and middle-end machinery for it since GCC 14 — but the rs6000 back end never implemented the one target hook that switches it on, so `_BitInt` was rejected outright on every Power target.

This is an account of closing that gap: designing a binary interface that covers all three Power ELF variants with a single rule, discovering that supported-on-paper big-endian code in the middle end had never been checked — one path untested, another masked — and measuring what the chosen design cost in generated code.

Three pieces of work came out of it. The target support is [PR target/117584](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=117584) ([patch](https://gcc.gnu.org/pipermail/gcc-patches/2026-August/728079.html)). Two target-independent miscompilations found along the way went upstream separately: [PR middle-end/126939](https://gcc.gnu.org/pipermail/gcc-patches/2026-August/728125.html), and [PR middle-end/127378](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=127378), [committed in September 2026](https://gcc.gnu.org/cgit/gcc/commit/?id=d9d078c216150bbe0397ded72f035c35b476e88e). Neither was a PowerPC bug. Both sat on big-endian paths that no target had put under real load — one simply untested on the only big-endian target that could reach it, the other masked there by that target's padding semantics.

---

## 1. Background: what `_BitInt` is, and why it needs an ABI

C23 defines `_BitInt(N)` as an integer exactly `N` bits wide — `N` bits in total, the sign bit included for signed types, so `_BitInt(7)` carries one sign bit and six value bits. The width is not rounded up to the next power of two, and the type is not subject to the usual integer promotions:

```c
_BitInt(7)             small;   /* 7 bits, signed, range -64..63 */
unsigned _BitInt(200)  wide;    /* 200 bits, unsigned */
_BitInt(1024)          huge;    /* 1024 bits */
```

`N` runs from 1 (unsigned) or 2 (signed) up to `BITINT_MAXWIDTH`, which GCC sets to 65535. Unsigned `_BitInt(N)` arithmetic is modulo 2^`N`; signed `_BitInt(N)` follows C's ordinary signed-integer rules, so overflow is undefined rather than wrapping. `sizeof` reports the storage the target chooses, which may be wider than `N` bits — and choosing that storage is precisely what this work is about. The motivating use cases are cryptography and hardware-description code, where the difference between 128 and 129 bits is semantically meaningful rather than an implementation detail.

GCC implements this generically. The C front end parses the type and the `wb` literal suffix; a dedicated GIMPLE pass, `gimple-lower-bitint.cc`, decomposes operations on wide values into sequences over fixed-size **limbs** — machine-word-sized chunks the native ALU can handle — falling back to libgcc routines where inlining is impractical.

All of that is target-independent, and all of it is inert until the back end answers one question: *what size is a limb, and how are limbs laid out?* That answer is a target hook:

```c
#define TARGET_C_BITINT_TYPE_INFO rs6000_bitint_type_info
```

Without it, GCC rejects `_BitInt` on that target. PowerPC was in exactly that position through GCC 14 and 15: the front end knew the syntax, the middle end knew how to lower it, and the back end had nothing to say about layout or calling convention.

The hook is not merely a compiler switch. What it returns *is* the ABI — it determines object size, alignment, memory layout, and how values cross function boundaries. Get it wrong and you have not written a slow compiler; you have written an incompatible one.

---

## 2. Four fields that decide everything

The hook fills in a small struct, and each field is a decision with consequences reaching into codegen, the ABI, and the runtime library.

```c
static bool
rs6000_bitint_type_info (int n, struct bitint_info *info)
{
  if (n <= 8)        info->limb_mode = QImode;
  else if (n <= 16)  info->limb_mode = HImode;
  else if (n <= 32)  info->limb_mode = SImode;
  else               info->limb_mode = TARGET_64BIT ? DImode : SImode;

  info->abi_limb_mode = info->limb_mode;
  info->big_endian    = BYTES_BIG_ENDIAN;
  info->extended      = bitint_ext_undef;

  return true;
}
```

**`limb_mode`** — the machine mode the lowering pass uses for each element when decomposing a wide value. Pick DImode and a `_BitInt(200)` becomes four 64-bit chunks with carries propagating between them.

**`abi_limb_mode`** — the chunk size the *ABI* describes: how the value is laid out in memory and passed across calls. It need not equal `limb_mode`.

**`big_endian`** — whether limbs are ordered most-significant-first in memory.

**`extended`** — what the bits above position `N-1` in the most significant limb contain: garbage, or a proper sign/zero extension.

Getting any of these wrong produces no warning. It produces wrong code, or an ICE deep inside a GIMPLE pass, and the failure mode gives very little hint about which field caused it.

---

## 3. Designing the ABI: one rule for three variants

Before writing compiler code, the interface has to be fixed — and Power makes that harder than most targets, because it has three live ELF variants: little-endian 64-bit (ppc64le, ELFv2), big-endian 64-bit (ppc64be), and big-endian 32-bit (ppc32). Any design that special-cases one of them yields an endianness-conditional binary interface: hard to document, harder to audit, and unattractive to ratify.

### The central question: register-width limbs, or 128-bit chunks?

Two designs were genuinely on the table.

AArch64's AAPCS64 uses 128-bit chunks for values wider than 128 bits — `_BitInt(65..128)` is one `__int128`, and wider values are `__int128[M]` arrays. There is a real argument for it: a `_BitInt(96)` becomes a single quantity rather than two chunks with a carry chain, and on a target where that maps onto a register pair with hardware support, the code is tighter.

x86-64 and s390x go the other way, specifying 64-bit limbs uniformly — *despite both having a native `__int128`*. That detail matters, and it undercuts the most natural argument for the AArch64 approach.

The register-width design won, for three reasons in descending order of weight:

**It generalises across all three variants.** The 32-bit Power ABI has no `__int128` at all, so a 128-bit-chunk rule cannot apply there and would require a separate 32-bit rule with a different unit. "Use the widest integer that fits in one general-purpose register" covers all three variants in one sentence.

**It matches the majority of ratified ABIs.** Power's rule now aligns structurally with x86-64 and s390x, differing from AAPCS64 only in the chunk unit — a defensible, documented divergence rather than an idiosyncrasy.

**Big-endian support depends on it.** This is the constraint that actually shaped the design, and it deserves its own section.

### Why `abi_limb_mode == limb_mode` unlocks big-endian

From Jakub Jelinek's comments on PR117584, describing the middle end as of GCC 16.1:

> Starting with GCC 16.1 there is big-endian support, there is DPD support and there is support for 3 padding bits schemes […] The only thing not supported is the big-endian case where ABI limb mode is different from limb mode (s390x as the only big-endian supported target with `_BitInt` has DImode ABI limb mode as well as limb mode).

That eliminates an entire design space. On little-endian you may set `abi_limb_mode` wider than `limb_mode` — AArch64 uses DImode limbs internally with a TImode ABI chunk, which works because, as Jakub put it elsewhere in the same bug, differing limb sizes are fine "as long as the endianity of the limbs matches ordering of limbs inside of the arrays."

On **big-endian**, that combination is not implemented. Not deprecated, not slow — absent. A back end requesting it gets an assertion failure or silently wrong lowering.

AArch64 sidestepped this by declining big-endian entirely:

```c
/* aarch64_bitint_type_info */
if (TARGET_BIG_END)
  return false;     /* big-endian _BitInt simply unsupported */
```

s390x sidestepped it by using DImode for both fields. PowerPC could not take AArch64's route — ppc64be is a live, supported configuration with real users, and shipping `_BitInt` on ppc64le while returning `false` on ppc64be would leave the target half-done. So it takes s390x's route and generalises it.

Setting `abi_limb_mode = limb_mode` unconditionally means the unsupported path is never reached on either endianness. The payoff: **one rule describes all three configurations.** A `_BitInt` wider than one limb is an array of limbs, ordered least-to-most-significant on little-endian and most-to-least on big-endian, exactly like any other multi-word integer. There is no PowerPC-specific limb ordering, no special case for ppc64be, and no endianness `#ifdef` in the hook at all — `big_endian = BYTES_BIG_ENDIAN` simply reports what the target already is.

This is also what forced the choice in the previous section. Using TImode for 65–128 would have pushed `abi_limb_mode` to diverge from `limb_mode` at larger widths to keep the ABI sensible — and that divergence is precisely what big-endian cannot do.

### The resulting layout

| Width `N` | ppc64le and ppc64be — size / alignment | ppc32 — size / alignment |
|---|---:|---:|
| 1 – 8 | 1 B / 1 B | 1 B / 1 B |
| 9 – 16 | 2 B / 2 B | 2 B / 2 B |
| 17 – 32 | 4 B / 4 B | 4 B / 4 B |
| 33 – 64 | 8 B / 8 B | 4⌈N/32⌉ B / 4 B |
| > 64 | 8⌈N/64⌉ B / 8 B | 4⌈N/32⌉ B / 4 B |

Below the limb width the representation is just the corresponding standard type — `char`, `short`, `int`, and `long` on ppc64. Above it the value is an array of limbs: `unsigned long[⌈N/64⌉]` on ppc64, `unsigned int[⌈N/32⌉]` on ppc32. Within each limb, bytes follow the target's normal byte order, so no Power-specific byte-layout exception is needed anywhere.

Two consequences are worth stating plainly, because both are visible ABI differences rather than internal details.

`_BitInt(65..128)` on ppc64 is 16 bytes but **8-byte aligned**, not 16-byte aligned like `__int128` — so `__alignof__(_BitInt(128))` is 8, and the value does not require an even-numbered starting register when passed by value. On ppc32, `_BitInt(33..64)` is 8 bytes but **4-byte aligned**, not like `long long`. In both cases alignment follows the limb, never a wider type that happens to have the same size. Neither is wrong — the C standard mandates neither — but they are the price of having big-endian work at all.

---

## 4. Padding bits: nobody's business

GCC supports three schemes for the bits above position `N-1` in the top limb:

| Scheme | Meaning | Targets |
|---|---|---|
| `bitint_ext_undef` | Padding is unspecified | x86-64, ia32, aarch64 |
| `bitint_ext_full` | All padding is sign/zero extended | s390x, arm, RISC-V |
| *(partial)* | Extended, except with more than 63 padding bits | LoongArch |

`bitint_ext_full` looks attractive — a callee can trust the top limb and skip a masking step. It also produced the most memorable failure of the project:

```
gcc.dg/dfp/bitint-1.c: In function 'tests192':
error: statement uses released SSA name
during GIMPLE pass: bitintlower
internal compiler error: cannot update SSA form
```

Tracking that through `update_ssa` and `cleanup_tree_cfg` led back to the same PR117584 discussion: `bitint_ext_full` was the scheme that "currently blocks arm support," and its interaction with big-endian lowering was not exercised. Combined with an `abi_limb_mode` wider than `limb_mode`, it also trips an assertion in `bitint_precision_kind` that explicitly requires the target *not* be big-endian.

The patch uses `bitint_ext_undef`. It matches x86-64 and AArch64, it is the scheme the lowering pass has the most mileage on, and — importantly — the PowerPC psABI has not yet ratified a position on `_BitInt` padding. Committing to `bitint_ext_full` would be the compiler unilaterally deciding a question the ABI document has not answered. `undef` is conservative in both the engineering and the political sense.

Like limb ordering, it applies identically on both endiannesses. It also had a consequence nobody anticipated: `bitint_ext_undef` on a big-endian target is a combination no supported configuration had ever presented, and Section 7 is what came of being the first to present it.

---

## 5. Plumbing the calling convention

Three call-related functions need to know about `_BitInt`.

**Large values go by reference.** Anything over 16 bytes under ELFv2, or 8 bytes on 32-bit, is passed and returned through a hidden pointer, exactly as an aggregate of that size already is. The threshold lives in one place so the callers cannot drift apart:

```c
static bool
rs6000_bitint_large_p (HOST_WIDE_INT size)
{
  if (size < 0)
    return true;                      /* variable-length: always large */
  if (DEFAULT_ABI == ABI_ELFv2)
    return (unsigned HOST_WIDE_INT) size > 16;
  return (unsigned HOST_WIDE_INT) size > (TARGET_64BIT ? 16 : 8);
}
```

`rs6000_return_in_memory` and `rs6000_pass_by_reference` each call it and nothing else. A caller and callee disagreeing about whether a value is in registers or behind a pointer manifests as memory corruption in unrelated code, so centralising the rule is cheap insurance.

Both sites test `BITINT_TYPE_P` rather than `TREE_CODE (type) == BITINT_TYPE`. That is not stylistic: `BITINT_TYPE_P` also matches an `ENUMERAL_TYPE` whose underlying type is `_BitInt` — C2Y's bit-precise enumerations — so those get the same ABI treatment for free rather than silently falling through to the integer path.

---

## 6. The bug that only appears on return

The subtlest problem in the target patch is one no amount of reasoning about limbs would have predicted.

PowerPC, like most targets, promotes narrow integers to a full register when passing or returning them. A `_BitInt(24)` looks like a sub-word integer to the generic promotion logic, which duly widens it:

```c
  if ((INTEGRAL_TYPE_P (valtype)
       && GET_MODE_BITSIZE (mode) < (TARGET_32BIT ? 32 : 64))
      || POINTER_TYPE_P (valtype))
    mode = TARGET_32BIT ? SImode : DImode;
```

But the mode for a `_BitInt` is not the back end's to choose. It was fixed by `TARGET_C_BITINT_TYPE_INFO`, and the middle end has already computed the type layout on that basis. Promoting here yields a mode that does not match the return register the middle end expects, and the mismatch surfaces as an ICE in `emit_move_insn` — far from the code that caused it.

The fix excludes `_BitInt` from promotion in both places it happens:

```c
/* rs6000_function_value */
  if ((INTEGRAL_TYPE_P (valtype)
       && !BITINT_TYPE_P (valtype)
       && GET_MODE_BITSIZE (mode) < (TARGET_32BIT ? 32 : 64))
      || POINTER_TYPE_P (valtype))
    mode = TARGET_32BIT ? SImode : DImode;

/* rs6000_promote_function_mode */
  if (type && BITINT_TYPE_P (type))
    return mode;
```

Two conditions, four lines. This is the sort of interaction that makes back-end work slow: the hook and the promotion logic were written years apart, neither is wrong in isolation, and the incompatibility appears only for a specific combination of width and position.

---

## 7. The code nobody had checked

Staying inside the middle end's supported envelope turned out to be necessary but not sufficient — because *supported* and *exercised* are different properties.

Big-endian `_BitInt` lowering had been written. It had been reviewed. It had shipped. What it had not been was *checked* — not at scale, and not by a target whose configuration would make a mistake visible. Enabling Power was the first serious audit that code received, and it surfaced two independent wrong-code bugs in `gimple-lower-bitint.cc`, both behind `bitint_big_endian` guards. Neither had been observed before, but for different reasons — one had never been exercised, the other was actively masked — and the distinction is worth keeping straight, because only the second one follows from a choice made in this work.

### The `memmove` that moved a value

With the hook in place and the ABI plumbing done, the generic torture tests on ppc64 big-endian failed in `gcc.dg/torture/bitint-93.c` and `bitint-94.c`, and in `bitint-32.c` through `bitint-37.c`. These are not PowerPC tests. They are target-independent tests of `__builtin_add_overflow`, `__builtin_sub_overflow` and `__builtin_mul_overflow` on wide `_BitInt` operands, and they had been passing on x86-64 for two years.

The cause was in `finish_arith_overflow`. When a double-width multiplication produces a value occupying more limbs than the destination can hold, the pass emits a `memmove` to shift the result down. The destination argument was built correctly; the source was not:

```c
g = gimple_build_call (fn, 3,
                       build_fold_addr_expr (unshare_expr (obj)),   /* dest: address ✓ */
                       src,                                         /* src: a VALUE ✗ */
                       build_int_cst (size_type_node,
                                      obj_nelts * m_limb_size));
```

`src` is a `MEM_REF` — a value of array type — handed to `memmove`, whose second parameter is a pointer. In C this would be a diagnosable type error; in GIMPLE, built programmatically, nothing catches it. Debugging it under `gdb` on the failing binary showed a `SIGSEGV` inside `memmove` with raw limb data where the source address belonged.

The fix is one line:

```c
                       build_fold_addr_expr (src),
```

and it is free at runtime: `build_fold_addr_expr` of a `MEM_REF` folds straight back to the underlying pointer, so no dereference is materialised and the generated code is what the original author clearly intended.

### The loop that stopped one limb short

The second bug is subtler, and it illustrates the reflection problem from Section 3 better than any amount of prose about limb ordering.

When a narrower signed value is assigned into a wider `_BitInt` object, `lower_mergeable_stmt` computes the low limbs with straight-line code and then emits a loop — the *separate_ext* loop — to splat the sign extension across the remaining limbs. Consider assigning a 135-bit value into a 512-bit object: 135 bits is two full 64-bit limbs plus seven bits, so three limbs are computed directly and the other five must be filled with the sign.

On little-endian the emitted loop is exactly right:

```
# _34 = PHI <3(2), _35(3)>              ← start at limb 3
VIEW_CONVERT_EXPR<unsigned long[8]>(a)[_34] = _33;
_35 = _34 + 1;                          ← ascending
if (_35 != 8) goto <bb 3>;              ← stop after limb 7
```

Five iterations, limbs 3 through 7. On big-endian the limb array is most-significant-first, so the loop must run the other way — and here is what the pass actually emitted:

```
# _35 = PHI <4(2), _36(3)>              ← start at limb 4
VIEW_CONVERT_EXPR<unsigned long[8]>(a)[_35] = _34;
_36 = _35 + 18446744073709551615;       ← descending (that is -1)
if (_36 != 0) goto <bb 3>;              ← stop when idx_next hits 0
```

Four iterations: limbs 4, 3, 2 and 1. Limb 0 is never written — and on big-endian limb 0 is the *most significant* one. The top 64 bits of a 512-bit result keep whatever happened to be in the object beforehand.

The error is in what the termination test compares. Both versions test `idx_next`, the already-incremented index. On little-endian that is correct by construction. On big-endian the increment is −1, so testing `idx_next != 0` stops one iteration early relative to the mirrored bound. Stated symmetrically: little-endian visits offsets `bo_idx + start + i + I`, so big-endian should visit `bo_idx + total - 1 - start - i - I` over the same range of `I` — the correction is to add `total - 1` and subtract where little-endian adds. There was already a special case for `bitint_big_endian && rem != 0`, which is presumably how the bug survived review: it covered one shape of the problem and not the general one.

The committed fix applies the mirrored bound properly, with one pragmatic exception:

```c
  /* For big-endian, if bo_idx + total - 1 - end is all ones, then
     compare idx (which is equal to idx_next + 1) against 0 instead.  */
  if (bitint_big_endian && bo_idx + total - end == 0)
    g = gimple_build_cond (NE_EXPR, idx, size_zero_node, NULL_TREE, NULL_TREE);
  else
    g = gimple_build_cond (NE_EXPR, idx_next,
                           size_int (bo_idx + (bitint_big_endian
                                               ? total - 1 - end
                                               : end)),
                           NULL_TREE, NULL_TREE);
```

The exception is the common case. Whenever there is no bit-field store and no remainder, the mirrored bound evaluates to all-ones, and comparing `idx != 0` is both faster and avoids reasoning about how an all-ones `unsigned HOST_WIDE_INT` maps onto the target's `size_t`.

Isolating this one took an A/B run of every `dg-do run` test in `gcc.dg/bitint-*.c` and `gcc.dg/torture/bitint-*.c`, at `-O0` and `-O2`, for `-m32` and `-m64` — 472 executables each way — to establish that the candidate fix moved exactly one test from fail to pass and regressed nothing. That evidence mattered more than the diagnosis: an early version of the fix keyed only on `rem`, which repaired the reported case but broke `gcc.dg/torture/bitint-81.c`, whose destinations are wide bit-fields where a trailing bit-field store already covers the limb. The version that went upstream generalises past both.

### Why nobody had hit either

The two bugs went unnoticed for different reasons, and the second reason is the more interesting one.

The `memmove` bug is structurally reachable on s390x — the only other big-endian target with `_BitInt` — and simply had not been tested there. That is an ordinary coverage gap.

The separate_ext bug is not *observable* on s390x, which is a different thing, and the reason traces directly back to Section 4. The descending loop is only emitted for big-endian limb layout, so no little-endian target reaches it at all; x86-64 and AArch64 are clean by construction. s390x does reach it, and the loop runs one iteration short there exactly as it did on Power — but s390x sets `extended = bitint_ext_full`, every padding bit guaranteed sign- or zero-extended, so other code writes the most significant limb unconditionally and the missing iteration leaves no trace. Power sets `bitint_ext_undef`, and nothing covers for the short loop.

The bug therefore required `big_endian = true` **and** `extended = bitint_ext_undef` together, and no supported target had ever presented that combination. The padding-bit decision in Section 4 was made on grounds of caution and psABI neutrality; its unplanned side effect was to make Power the first target that could observe this class of defect at all.

### Two bugs, one shape

Both are wrong-code bugs. Both live in `gimple-lower-bitint.cc`. Both are guarded by `bitint_big_endian`. Both were written by people who understood exactly what big-endian limb ordering requires — the `-1` increment and the descending loop are not accidents, they are deliberate and correct — and both got the *boundary* wrong in a way only execution can reveal.

That is the characteristic failure mode of mirrored code. Reflecting a loop is easy to do approximately and hard to do exactly, because the direction change is visible in the source while the off-by-one hides in the termination test. Code review catches the former. Only running it catches the latter.

There is a general lesson in the shape of this. Being the *second* implementation of anything is a distinct role from being the first. The first defines the interface; the second discovers which parts of it were only ever true on paper. Big-endian `_BitInt` support had been written into the middle end, and it was real work — but a code path whose output nobody has checked is a hypothesis, not a feature, whether it never ran or ran under conditions that hid the result.

---

## 8. libgcc, and PowerPC's two 128-bit floats

Everything above concerns the compiler. The runtime library is a separate problem, and on PowerPC a genuinely unusual one.

Conversions between `_BitInt` and floating point are too complex to inline, so they become libgcc calls — `__floatbitintdf`, `__fixtfbitint`, and so on. Most targets get these nearly free: `libgcc/soft-fp/` has generic implementations and the `t-softfp` makefile fragment wires them up. PowerPC has two complications.

### Two formats named TF

Almost every target has one 128-bit floating-point type. PowerPC has two:

- **TFmode** — historically IBM double-double: a pair of `double`s summed, roughly 106 bits of precision when the two halves are adjacent, no extra exponent range.
- **KFmode** — true IEEE 754 binary128, 113-bit mantissa.

The generic `libgcc/soft-fp/floatbitinttf.c` is written for IEEE quad. On AArch64 it compiles directly, because there TFmode *is* IEEE quad. On PowerPC it cannot: compiled against IBM double-double it would produce silently wrong results.

The existing build machinery already solves the general version of this with `sed`. `t-float128` generates KF sources from the generic TF ones by textual substitution, and `float128-sed` maps the symbol names:

```
s/__floatbitinttf/__floatbitintkf/g
s/__fixtfbitint/__fixkfbitint/g
```

So the IEEE-128 half is nearly free — adding `floatbitintkf` and `fixkfbitint` to `fp128_softfp_funcs` suffices. (The substitutions must go into *both* `float128-sed` and `float128-sed-hw`; the hardware variant is a separate file, and omitting it there breaks POWER9+ builds specifically.)

The IBM double-double half needs real source files. They take a shortcut through IEEE-128:

```c
/* floatbitinttf-ibm128.c, compiled with -mabi=ibmlongdouble */
extern __float128 __floatbitintkf (const UBILtype *, SItype);

long double
__floatbitinttf (const UBILtype *i, SItype iprec)
{
  return (long double) __floatbitintkf (i, iprec);
}
```

Convert to IEEE-128 first, then narrow; the reverse widens IBM→IEEE and defers to `__fixkfbitint`. Both compile with `-mabi=ibmlongdouble` so `long double` means the format they claim to handle — the same pattern the existing decimal conversions use.

The shortcut is exact for ordinary values but not for every value, and the reason is a common misreading of double-double precision. "About 106 bits" describes a double-double whose low half sits directly below its high half. Nothing in the format requires that: the low `double` can sit arbitrarily far down. `2^100 − 2^−20` is a valid double-double, but it spans 121 bits, more than IEEE-128's 113. Widening it rounds to `2^100`, so truncating to an integer yields `2^100` where the correct answer is `2^100 − 1`. The routing is exact only when the two halves span 113 bits or fewer. Converting the two halves separately, with the carry handled explicitly, closes that gap, and it is a follow-up item.

### The fragment PowerPC never included

The second complication was less expected. PowerPC Linux does not have `t-softfp` in its `tmake_file`. It never needed it — the target has its own float128 machinery — but the consequence is that the generic mechanism giving every other target its `_BitInt` conversions had no effect here at all.

Missing as a result: the plain binary conversions (`__floatbitintsf`, `__fixsfbitint`, `__floatbitintdf`, `__fixdfbitint`), which `softfp_bitint_func_list` provides unconditionally to every `t-softfp` target — and, when decimal floating point is enabled, the entire DPD set.

That last one explains a failure that looked like a compiler bug. PowerPC's decimal floating point uses DPD (Densely Packed Decimal); x86 uses BID. From Jakub, on the same PR:

> Currently the dfp ↔ `_BitInt` conversion functions are written only for BID, not for DPD, although the DPD ones could be fairly similar, just with encoding/decoding the format differently.

DPD support arrived in GCC 16.1 — but arriving in `soft-fp/` is not the same as being *built*, and without `t-softfp` nothing compiled them. So `gcc.dg/dfp/bitint-1.c` failed for two independent reasons at once: the `bitint_ext_full` lowering bug, and a set of runtime functions that simply did not exist in the library.

The fix wires both sets in directly, sourced from `soft-fp/` and added to `LIB2ADD_ST` (static library only, matching `t-softfp`):

```makefile
rs6000_bitint_bin_funcs = fixsfbitint floatbitintsf \
                          fixdfbitint floatbitintdf

LIB2ADD_ST += $(rs6000_bitint_bin_src)

ifeq ($(decimal_float),yes)
rs6000_bitint_dec_funcs = bitintpow10 \
                          fixsdbitint floatbitintsd \
                          fixsdti fixunssdti floattisd floatuntisd \
                          ...
LIB2ADD_ST += $(rs6000_bitint_dec_src)
endif
```

Note `bitintpow10` and the TImode↔DFP conversions in that list: the decimal `_BitInt` routines depend on them, and they were absent for the same reason. All eight new symbols are exported under a fresh `GCC_17.0.0` version node.

---

## 9. Testing what the hook actually claims

Eleven execution tests accompany the patch. A few are worth calling out for what they catch rather than what they cover.

**`bitint-endian-powerpc.c`** matters most for this design, and the obvious version of it is useless. Checking the byte order of a single 32-bit value tests the target's endianness, which was never in doubt. What needs testing is *limb ordering within a multi-limb object*:

```c
_BitInt(96) val = 0x010203040506070809101112wb;
unsigned char bytes[12];
memcpy (bytes, &val, 12);

#if __BYTE_ORDER__ == __ORDER_BIG_ENDIAN__
  if (bytes[0] != 0x01 || bytes[11] != 0x12)  __builtin_abort ();
#else
  if (bytes[0] != 0x12 || bytes[11] != 0x01)  __builtin_abort ();
#endif
```

Without this, flipping `info->big_endian` in the hook would not fail a single test.

**`bitint-abi-call.c`** deliberately straddles the by-reference threshold, testing `_BitInt(128)` (in registers) against `_BitInt(129)` (by reference) with identical call shapes, plus mixed integer/`_BitInt` signatures and enough arguments to spill to the stack.

**Most tests compile at `-O0`.** Not laziness: at `-O2` the compiler constant-folds nearly every one of these expressions and the test passes without executing a single instruction of the lowered code. `-O0`, or `noinline` on the operations under test, is what forces real code generation.

The size tests exercise non-power-of-two widths specifically — 79, 113, 153, 257, 620 — because `ceil(n/64)` is exactly the kind of formula that is off by one at boundaries and correct everywhere else.

One thing these eleven tests did *not* catch is worth recording. Neither middle-end bug was found by anything written for this patch — both came out of `gcc.dg/`, and that is the correct division of labour. Target tests should assert what the target claims: these sizes, these alignments, this byte order, this calling convention. The generic suite asserts that the language works. When enabling a feature on a new target, the first useful signal is not your own tests passing but the existing target-independent suite either passing, or failing in ways nobody has seen before. The separate_ext fix accordingly shipped with a new generic test, `gcc.dg/bitint-145.c`, rather than a PowerPC one — which is now the tripwire the next big-endian target inherits for free.

---

## 10. What the design cost in generated code

The register-width choice has a measurable code-quality cost, and it is worth quantifying rather than hand-waving.

Under the 128-bit-chunk alternative, a `_BitInt(65..128)` lives in one TImode register pair: the sign bit sits at a fixed position and can be extracted with a single instruction. Under register-width limbs the same value spans two DImode registers, and every operation touching the sign bit must reconstruct it explicitly across both.

To measure it, a comparison script compiled the same multi-width test suite with both compiler builds, split the assembly per function, and reported per-function instruction-count deltas.

| Target | Opt | Δ insns | Δ% | Where |
|---|---|---|---|---|
| ppc64le (m64) | O0 | +7,639 | +13.1% | 65–128 bit band only |
| ppc64le (m64) | O2 | +402 | +2.1% | 65–128 bit band only |
| ppc64be (m64) | O0 | +7,337 | +16.4% | 65–128 bit band only |
| ppc64be (m64) | O2 | +587 | +4.0% | 65–128 bit band only |
| ppc32 (m32) | O0 | −9 | −0.0% | neutral |
| ppc32 (m32) | O2 | −14 | −0.1% | neutral |

These are suite-wide static instruction counts from that comparison, not application benchmarks: they say how much code the two designs emit for the same sources, not how fast it runs. Nothing here has been measured on a workload, and the two are not interchangeable — a sign-reconstruction sequence of three dependent instructions costs differently from three independent ones. Read the table as an upper bound on the disturbance, not as a performance claim.

Three things stand out.

**The cost is strictly confined to 65–128 bits.** Functions on ≤64-bit values are byte-for-byte identical between builds. Functions above 128 bits are equal or *smaller*. The band that regresses is exactly the one that changed from "a single machine-mode scalar, not lowered at all" to "a two-limb object going through the same lowering path as wider values."

**`-O2` costs far less than `-O0`.** The optimiser folds many instances of the sign-reconstruction idiom, bringing the suite-wide impact to +2–4%. Individual worst cases — bitwise NOT, AND-NOT, variable right shift in that band — reach +8 to +13 instructions per function.

**ppc32 is neutral, and that is informative.** On 32-bit the limb selection is identical under both designs (SImode throughout), so the only variable is the padding-semantics choice. The result — under 0.1%, in the *better* direction — is therefore a clean isolated measurement showing that `bitint_ext_undef` versus `bitint_ext_full` costs essentially nothing there.

### A big-endian-specific effect, still under investigation

On ppc64be at `-O2`, four functions — signed variable right shift on 129/132/160/192-bit values — grow by 38–44 instructions each, a regression entirely absent on little-endian, where the same functions shrink slightly.

The natural first hypothesis was that the limb-size change caused it. It did not: at those widths the limb configuration is *identical* between the two builds, since the older scheme also used DImode limbs above 128 bits on big-endian. The only thing that differs there is the padding semantics. That, together with the fact that the unsigned counterparts grow only slightly, points to the cost of recomputing shifted-in sign bits from bit `N-1` rather than reading them from guaranteed-extended padding.

That attribution is preliminary, and a confirming experiment is outstanding. It is recorded here as an open question rather than a settled finding — the effect is small in absolute terms and does not bear on correctness.

### The path to recovery

Most of the 65–128 cost is a well-defined peephole target: a short, recognisable sign-extension idiom, applied in some cases to a value that is already in the required form. Folding it in the rs6000 back end would recover much of the `-O2` regression while keeping the generalisation properties of the uniform rule. This is quality-of-implementation work, not an ABI property — it can land later without any compatibility consequence.

---

## 11. Regression status

With the target patch and the `finish_arith_overflow` fix applied, before the separate_ext fix landed:

- **ppc64le-linux** — 903 `_BitInt`-related passes, zero failures.
- **ppc64be-linux (`-m64` and `-m32`)** — 1,620 passes, 3 failures: one generic torture test at `-m32`, and one byte-layout test on both multilibs whose expected values were still being corrected.

That remaining `-m32` torture failure is worth following, because it was initially filed under "pre-existing, unrelated to layout" — the category every porter uses for noise. It was not noise. It was the separate_ext bug of Section 7, and the fix for it closed that entry. A failure attributed to the environment and a failure that is genuinely yours look identical from the summary line; the only way to tell them apart is to open each one.

For context, the interim design that represented `_BitInt(65..128)` as a single 128-bit scalar produced the *identical* big-endian pass/fail set. The difference between the two designs is code quality in that band, not correctness.

---

## 12. Where Power fits among `_BitInt` ABIs

| Architecture | Limb / chunk | Padding | Endianness |
|---|---|---|---|
| x86-64 psABI | 64-bit, uniform | Unspecified | LE only |
| s390x ELF supplement | 64-bit, uniform | Always extended | BE only |
| AAPCS64 | 128-bit for N > 128 | Unspecified | Spec covers both; GCC implements LE only |
| **This work (Power)** | **Native register width** | **Unspecified** | **Both, identically** |

Power now matches x86-64 and s390x structurally, and is the only one of the four stated as a single rule that covers a 64-bit little-endian, a 64-bit big-endian, and a 32-bit big-endian variant without special-casing any of them.

On the LLVM side, Clang declares `_BitInt` available on PowerPC but has never been given PowerPC-specific layout rules; its generic default — a 64-bit alignment cap, 64-bit chunking, indirect passing above 128 bits — already agrees with this specification. Adopting it should therefore need no `BitIntMaxAlign` override for PowerPC, unlike AArch64 which sets 128 to match AAPCS64. Confirming with LLVM PowerPC maintainers that the current default is *intended*, rather than unconfigured-by-omission, is an open item.

---

## 13. The specification still has to be ratified

The compiler work settles what GCC does. It does not settle what the Power ABI *says*, and those are different things — a target hook is an implementation choice until a standards body writes it down, after which it is a contract.

A draft specification covering all three variants now exists, written against the layout in Section 3 and the padding rule in Section 4. Getting it ratified runs into two practical obstacles worth recording, because neither is a technical problem.

The 64-bit Power ELF ABI is maintained by the OpenPOWER Foundation, and its published specification explicitly covers both endiannesses — "this document establishes both big-endian and little-endian application binary interfaces" — so it is the right single venue for ppc64le and ppc64be together. The current published version is 2.1.5, last updated in December 2020, and contains no C23 or `_BitInt` content at all. Changes go to the `syssw-elfv2abi` community mailing list rather than as pull requests against the source repository, with a Developer Certificate of Origin sign-off in the kernel and GCC style. The repository shows an open-issue backlog and no recent commits, so list responsiveness is unverified; a short scoping question is the sensible first move, ahead of a full proposal in the Foundation's document format.

The second obstacle is sharper. **The governed specification is 64-bit only.** Its title and scope are the 64-Bit ELF ABI; 32-bit Power runs under the older SVR4 supplement, which predates OpenPOWER Foundation governance and appears to have no current maintainer. So the ppc32 `_BitInt` ABI — fully implemented, tested on both multilibs, and specified in the same single sentence as the other two — may have no standards body able to ratify it. The likely outcome is that ppc32's rule ends up documented in GCC and LLVM and nowhere else: a de facto ABI by implementation agreement rather than a ratified one. That is worth asking about explicitly rather than assuming, because the answer changes how the 32-bit rule should be described.

There is a related loose end on the LLVM side. Clang already declares `_BitInt` available on PowerPC without having been given PowerPC-specific layout rules, and its generic default — 64-bit alignment cap, 64-bit chunking, indirect passing above 128 bits — happens to agree with this specification exactly. Unlike AArch64, which sets `BitIntMaxAlign` to 128 to match AAPCS64, Power should need no override. But "already agrees" and "deliberately agrees" are different claims, and only the second is worth putting in a specification. Confirming with LLVM's PowerPC maintainers that the current behaviour is intended rather than unconfigured-by-omission is an open item.

---

## 14. Lessons

**Lead ABI design by principle, not compiler convenience.** The initial temptation was to match GCC's internal representation and use 128-bit chunks on ppc64le. That would have been correct for little-endian and produced an endianness-conditional interface no working group would ratify. Stepping back to ask "what rule applies uniformly?" produced a better answer — and, as it turned out, the only one big-endian could implement.

**Wrong results are not always wrong ABI choices.** The big-endian overflow failures initially looked like a padding-semantics problem and pushed the investigation toward requiring extended padding. Tracing them to a miscompilation instead took reading GIMPLE dumps, comparing against little-endian output, and eventually a debugger session showing limb data where a pointer belonged. Before concluding an ABI choice is wrong, exhaust the possibility that the compiler is.

**Measure the trade-offs explicitly.** "There is a code-quality cost in the 65–128 range" is a much weaker claim than the same statement with instruction-count deltas across a multi-width suite, broken down by width and operation, with assembly side by side for the worst cases. The measurement also caught a big-endian-specific effect that no amount of reasoning had predicted.

**A conservative choice can still be a novel one.** `bitint_ext_undef` was picked to avoid committing the ABI to something the psABI had not ratified — the least adventurous option available. It nonetheless produced a configuration no supported target had ever run, and that is what exposed the separate_ext bug. "Conservative" means low risk of being *wrong*; it does not mean low risk of being *first*. The two get conflated, and the second one is where the surprises live.

**Expect to fix things that are not yours.** Enabling a feature on a new target does not merely consume the middle end's support for it — it audits that support. A meaningful share of the debugging here went into code outside `config/rs6000/` entirely, in a file no PowerPC maintainer owns, and the results benefit every big-endian target that comes after. Budget for it — and budget for the collaboration, too: the second fix was written by the middle end's own maintainer once the bug was isolated, reported and A/B-validated, which is faster and more durable than patching around it downstream. If you are the second implementer of anything, some fraction of your work is retroactively finishing the first.

---

## References

- GCC Bugzilla: [PR target/117584](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=117584) — rs6000 `_BitInt` support
- rs6000 ABI patch: [gcc-patches 728079](https://gcc.gnu.org/pipermail/gcc-patches/2026-August/728079.html)
- `gimple-lower-bitint.cc` fix (overflow `memmove`): [PR middle-end/126939](https://gcc.gnu.org/pipermail/gcc-patches/2026-August/728125.html) · [gcc-patches 728125](https://gcc.gnu.org/pipermail/gcc-patches/2026-August/728125.html)
- `gimple-lower-bitint.cc` fix (big-endian separate_ext loop): [PR middle-end/127378](https://gcc.gnu.org/bugzilla/show_bug.cgi?id=127378) · [commit d9d078c](https://gcc.gnu.org/cgit/gcc/commit/?id=d9d078c216150bbe0397ded72f035c35b476e88e)
- ABI specification draft — to be submitted to `syssw-elfv2abi@mailinglist.openpowerfoundation.org`
- C23: ISO/IEC 9899:2024, §6.2.6.3 (bit-precise integer types)
- x86-64 psABI: System V ABI, AMD64 Architecture Processor Supplement
- AAPCS64: Procedure Call Standard for the Arm 64-bit Architecture
- s390x ELF ABI Supplement

