# LLVM backend for MIPS-X: work plan

Status: draft, 2026-10-27. Clean-side document. Expands §3.1 of
[`open-source-stack.md`](open-source-stack.md) into work packages that
each fit one working session of about 100k tokens.

## 1. Where things are

| What | Where |
|---|---|
| LLVM fork | `~/src/openmipsx/llvm-project`, branch `mipsx`, based on tag `llvmorg-23.1.4`. Blobless clone of upstream; `origin` is `llvm/llvm-project` |
| SDK repository | `~/src/openmipsx/mipsx-sdk`, created in session S1 (§4) |
| Instruction set | [`processor-es3890.md`](https://github.com/openmipsx/devices/blob/main/common/processor-es3890.md), with the public manual `mipsx.isetman.pdf` (local copy in `~/src/openmipsx/reference/`, not redistributed) for pipeline timing (manual §3) |
| Image format | [`firmware-image-format.md`](https://github.com/openmipsx/devices/blob/main/common/firmware-image-format.md) |
| Independent decoder | `~/src/openmipsx/ghidra-mipsx`, used only on our own binaries |

### Clean-room rules for every session

The toolchain is clean-side ([`clean-room.md`](https://github.com/openmipsx/devices/blob/main/clean-room.md)).
Sessions read only the specs in openmipsx/devices, the plans in openmipsx/.github,
`~/src/openmipsx/reference/`, the LLVM tree and other public sources. They never open `~/src/openmipsx/dirty-side/`: the vendor downloads, the
unpacked or decompiled material, or Ghidra projects of vendor firmware.
Tests use hand-written assembly and C, never vendor bytes. The ABI is
ours to choose (§7 of the processor spec only describes the stock
firmware's convention, much of it Hypothesis).

## 2. Facts that shape the backend

From the processor spec and manual §3:

- **Word-addressed code, byte-addressed data.** The PC, branch
  displacements, `jspci` offsets and return addresses are word
  addresses. LLVM's MC layer and lld assume byte addresses, so fixups,
  relocations and symbol values need a clear rule for which unit each
  one is in. AVR has the same problem (program memory is word-addressed)
  and is the model for this: see `AVRAsmBackend.cpp`, its `pm()`
  and `gs()` modifiers and `lld/ELF/Arch/AVR.cpp`.
- **Two delay slots after every branch and jump.** Branches can squash
  their slots when not taken; jumps never squash.
- **No hardware interlocks** (manual §3.2, §3.3). A loaded value cannot be
  used by the next instruction (one load delay slot), and a load must not
  overwrite its own address register. Whether the ES3890 added interlocks
  is unknown, so the compiler assumes it did not.
- **No load-upper-immediate.** Immediates are 17-bit signed. A 32-bit
  constant takes `addi`, a shift and `addi` (three instructions), or a
  load from a literal pool through a base register.
- **No multiply or divide.** `mstep`, `mstart` and `dstep` work one bit at
  a time through the MD special register; C needs helper routines.
- **Only six branch conditions**, comparing two registers: eq, ne, signed
  lt and ge, unsigned lo and hs. There is no set-on-compare, so `setcc`
  becomes a branch or a bit trick.
- **Shifts.** `sh` is a funnel shift with an immediate amount; `asr` is
  arithmetic right by an immediate. There is no variable shift in the
  manual (the spec says later firmware uses one, but it is not described),
  so variable shifts need a helper or an unrolled sequence.
- **ELF.** `EM_MIPS_X` = 51 is already in LLVM 23's `ELF.h`, so the fork
  does not need to add it. The triple `mipsx` does not collide inside
  LLVM: `Triple.cpp` matches MIPS names exactly, and the one `starts_with
  ("mips")` test only looks for an `r6` suffix. Other tools (autoconf's
  `config.sub`, picolibc's Meson files) may still match `mips*`; check in
  S15.

## 3. Build setup

The machine has 12 cores and 62 GB of RAM, with `cmake` 3.31 and `ninja`.
Install `clang`, `lld` and `ccache` from Debian first: building LLVM with
host Clang and lld, through ccache, makes rebuilds much faster.

```
cmake -G Ninja -S llvm -B build \
  -DCMAKE_BUILD_TYPE=Release -DLLVM_ENABLE_ASSERTIONS=ON \
  -DCMAKE_C_COMPILER=clang -DCMAKE_CXX_COMPILER=clang++ \
  -DLLVM_USE_LINKER=lld -DLLVM_CCACHE_BUILD=ON \
  -DLLVM_TARGETS_TO_BUILD="" -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=MIPSX \
  -DLLVM_ENABLE_PROJECTS="clang;lld"
```

`clang` and `lld` can be left out of `LLVM_ENABLE_PROJECTS` until S5 and
S13 need them. A first build takes tens of minutes; run it in the
background. Test with `ninja -C build check-llvm-mc-mipsx` style targets,
or `build/bin/llvm-lit -v llvm/test/MC/MIPSX` directly.

## 4. Sessions

Each session ends with: tests passing, commits on `mipsx` (or in the SDK
repository), and its row in §5 updated. Commits are a clean series: one
topic per commit, prefixed `[MIPSX]`, so the series can later move to a
new LLVM release (§6).

The order is fixed up to S7. After that, S8 to S12 are sequential, and
S13 to S18 can be reordered once the compiler can run tests.

### Phase A: assembler, linker and simulator

**S1. ABI document and SDK repository.** Create `mipsx-sdk` (licence
chosen from P5, MIT or Apache 2.0). Write `docs/abi.md`, deciding:

- register roles: zero, stack pointer, return address, arguments, return
  values, callee-saved, temporaries, code base registers, a literal pool
  or small data base, and any register reserved for the assembler;
- how a function pointer is represented in data (recommended: a byte
  address, so C pointer arithmetic and comparisons behave normally; an
  indirect call then shifts right by 2 first);
- how direct calls reach their target, given `jspci`'s ±64K-word reach
  from a base register (one base per 256 KB of code, or a long-call
  sequence), and how far branches are handled;
- how 32-bit constants and addresses are built;
- stack layout, alignment, varargs, struct passing and return;
- assembler syntax: operand order (the manual's `addi src1,#imm,dest`),
  register names, pseudo-instructions (`nop`, `mov`, `li`, `la`, `b`,
  `call`, `ret`), relocation operators, and whether the assembler fills
  delay slots (recommended: never; slots are explicit in assembly);
- ELF: `e_flags` bits for the ES3890 extensions, and the full list of
  relocation types with their units and formulas.

Done when the document answers every question S2 to S10 will ask. No
code. About half a session; use the rest to start S2.

**S2. Target skeleton and instruction descriptions.** Add the `MIPSX`
arch to `Triple`, the experimental target to CMake, `TargetInfo`, and
TableGen files for registers (r0 to r31, PSW, MD, the PC specials),
instruction formats (§2 of the spec) and every instruction in §3 to §6,
including the ES3890 byte and halfword forms. Add `MCTargetDesc` with
`MCInstPrinter`, `MCCodeEmitter` and `MCAsmInfo`. Reference: Lanai, whose
32-bit fixed-width encoding is closest. Done when `llvm-mc
-triple=mipsx -show-encoding` prints the right word for each instruction,
checked by a lit test under `llvm/test/MC/MIPSX/` that covers every
encoding in the spec (including the `sh` shift-amount encoding of §5).

**S3. Assembler parser and disassembler.** `AsmParser` (operands,
registers, immediates with range checks, the S1 pseudo-instructions) and
`Disassembler` (generated decoder tables). Done when every instruction
round-trips through `llvm-mc` assembly and `-disassemble`, out-of-range
immediates give errors, and unknown words decode as `.word` rather than
crashing.

**S4. Object files and relocations.** `MCAsmBackend` fixups for branch
displacements, `jspci` offsets and 32-bit data words, with
`ELFRelocs/MIPSX.def`, the ELF object writer, and support in
`llvm-readobj`, `llvm-objdump` and `RelocationResolver`. Get the word and
byte units right here: a branch fixup resolves to (target − branch) / 4 −
2. Add the `e_flags` from S1. Done when `llvm-mc -filetype=obj` produces
objects that `llvm-objdump -dr` disassembles with the right relocations,
local branches resolve without relocations, and lit tests cover each
relocation type and the out-of-range errors.

**S5. lld.** `lld/ELF/Arch/MIPSX.cpp` (references: MSP430, 89 lines, and
AVR for word-addressed code), relocation range checks, and a default
linker script defining the code base symbols from S1. Done when a
multi-file assembly program links, `llvm-objdump` shows the right
targets, and lld lit tests cover each relocation. Cross-check: import the
linked ELF into Ghidra with ghidra-mipsx and compare the disassembly (our
own binary only).

**S6. Instruction-set simulator** (SDK repository, in C for speed). Loads
an ELF; runs the instruction set from the spec; models both delay slots,
squashing, MD and the step instructions; **detects and reports** load-use
hazards and loads that overwrite their address register, rather than
silently giving the wrong answer. A simulator-only I/O page gives
`putchar`, `exit(code)` and a cycle counter. Done when hand-written tests
for every instruction pass, and a lit configuration in the LLVM tree can
run a linked program on the simulator and check its output (this is how
S8 onwards test generated code).

**S7. Runtime start and image packer** (SDK repository). `crt0.S` (stack
pointer, base registers, `.bss` clearing, `.data` copy, call `main`), the
linker scripts for simulator and device, and the packer: ELF in, upgrade
image out, written from the format spec alone, containing only a type 2
section (stack plan §3.5). Done when an assembly program runs on the
simulator through `crt0`, and the packer's output passes every check the
format spec lists (verify with a test that re-reads the image using only
the spec's rules). This completes stack plan step 1 for assembly; the LED
blink of step 2 waits for the GPIO spec.

### Phase B: C code generation

**S8. Code generation skeleton.** `TargetMachine`, `Subtarget`,
`RegisterInfo`, the calling convention from S1 in `CallingConv.td`,
`ISelLowering` for formal arguments, returns and simple direct calls,
basic frame lowering, and a pass that fills every delay slot with `nop`
and pads load-use hazards with `nop` (correct but slow; S11 makes it
fast). Patterns for 32-bit add, sub, and, or, xor, not, loads and stores
of every width with sign and zero extension, and small immediates. Done
when `llc -mtriple=mipsx` compiles simple functions (lit tests in
`llvm/test/CodeGen/MIPSX/`) and a few of them run correctly on the
simulator.

**S9. Constants, addresses and control flow.** 32-bit constant and
global address materialisation per S1 (with a literal pool if chosen),
conditional branches with operand swapping for the missing conditions,
`setcc` and `select` lowering, `br_cc`, jump tables, shifts by constants
through `sh` and `asr`, and variable shifts through a helper call. Done
when lit tests cover each lowering and a test program with loops,
switches and globals runs on the simulator.

**S10. Calls and frames.** Indirect calls (byte-to-word conversion if S1
chose byte function pointers), callee-saved spills and restores, large
stack frames beyond the 17-bit offset, frame pointer, dynamic `alloca`,
varargs, struct arguments and returns, and `i64` arithmetic (expanded by
LLVM's type legaliser, with carry handled by compare and branch or the
`subnc` trick). Done when a calling-convention test (many argument types
and counts, called through pointers, varargs) runs on the simulator.

**S11. Delay slot filler and hazard scheduling.** Replace S8's `nop`
padding with a filler that moves useful instructions into both slots,
uses squashing branches when it can only fill from the taken path, and
respects the load delay. References: `SparcDelaySlotFiller.cpp` and
`LanaiDelaySlotFiller.cpp` (one slot each; extend to two). Add a hazard
recogniser or scheduler model so the scheduler separates loads from
their uses. Done when every earlier simulator test still passes with no
hazard reports, and the share of `nop` in slots on a benchmark set is
measured and recorded.

**S12. Multiply, divide and other helpers.** Write `__mulsi3`,
`__divsi3`, `__udivsi3`, `__modsi3`, `__umodsi3`, the 64-bit versions and
the variable shifts, in assembly on `mstep`, `mstart` and `dstep`, for
compiler-rt's builtins (or the SDK, if compiler-rt is too awkward to
build at this stage). Teach the backend to call them, and to expand
multiplication by constants into shifts and adds. Done when exhaustive
edge-case tests (zero, ±1, INT_MIN, powers of two, random values) pass on
the simulator against a host reference.

### Phase C: Clang, libraries and hardening

**S13. Clang.** `clang/lib/Basic/Targets/MIPSX`, the ABI in
`clang/lib/CodeGen/Targets/MIPSX.cpp` (matching S1), and a bare-metal
driver toolchain that finds `crt0`, the linker script and a sysroot and
links with lld (reference: the existing `BareMetal` toolchain and Lanai).
Inline assembly constraints and register names. Done when `clang
--target=mipsx -O2 hello.c` produces an ELF that prints on the simulator,
and Clang's preprocessor and ABI lit tests pass.

**S14 and later, repeated: execution test triage.** Run a C test suite on
the simulator: compiler-rt's builtin tests first, then the GCC C torture
execute tests (from the LLVM test-suite's `SingleSource`) at `-O0` and
`-O2`. Each session fixes the largest group of failures and records the
pass rate in §5. Plan on several of these sessions.

**S15. picolibc.** Build picolibc for MIPS-X with the SDK's crt0 and
the simulator's I/O page for stdio; a Meson cross file and a CMake
toolchain file in the SDK. Check that `mipsx` is not caught by `mips*`
patterns. Done when picolibc's own test suite runs on the simulator.

**S16. Size and real code.** Compile lwIP, littlefs and BearSSL (stack
plan §2) and compare code size against the flash budget in stack plan
§3.4. Fix what blocks compilation; record sizes; add size-oriented
improvements (literal pool sharing, `-Os` tuning) only where measured.

**S17. Debug information.** DWARF register numbers, CFI for the frame
layout, and line tables that account for word-addressed code, so
`llvm-dwarfdump` and, later, a debugger stub make sense of our binaries.
Optional for the first device image.

**S18. CI and releases.** GitHub Actions for the fork (build with only
MIPS-X, run lit) and the SDK (simulator tests, packer tests), release
archives for Linux x86-64 and arm64 and macOS, and a container image
(stack plan §3.1, Publishing).

## 5. Status

| Session | Status | Notes |
|---|---|---|
| S1 ABI and SDK | Done | `mipsx-sdk` created (Apache 2.0); `docs/abi.md` v1 fixes registers, byte function pointers, linker-selected code-base calls, calling convention, syntax, `e_flags` and 13 relocations |
| S2 Skeleton and instructions | Done | Commits 9dddf2671390 (Triple), 43d8fa6eeff5 (target + MC). 9 lit tests in `test/MC/MIPSX` cover every encoding; a basic AsmParser exists. Left for S3: `%hi/%lo/%gprel/%call/%w` operators, pseudos `li la call jmp lsl lsr` and `bgt ble bhi bls` (+sq), `lsl`/`lsr` printing, `.word`/`.half` directives, the disassembler. Left for S4: fixup resolution, relocations, `e_flags` |
| S3 Parser and disassembler | Done | Commits 311585e33e79 (operators, pseudos, `.word`/`.half`), 806b819243b4 (disassembler). 15 lit tests in `test/MC/MIPSX` and `test/MC/Disassembler/MIPSX`; every encoding round-trips. Agrees word for word with the S6 tools on 402 words. Left for S4: resolve fixups, relocations (`%w` in data is `FK_Data_4` + specifier), `e_flags`, `EM_MIPS_X` to `mipsx` in `ELFObjectFile` (objdump needs `--triple=mipsx` until then) |
| S4 Object files | Done | Commits 19a03d9176a1 (ELF relocs, `e_flags`, Object/readobj/RelocationResolver), dba79ffa8159 (fixups and relocations), 28f56e3b2138 (`bsq`). 20 MIPSX lit tests: every relocation type, local resolution, range and alignment errors, `e_flags`, DWARF. `%call`/`%gprel` are always left to the linker; `.reloc` takes `R_MIPSX_*` names |
| S5 lld | Done | `lld/ELF/Arch/MIPSX.cpp`: commits 49da398736e0, 70f41f0b233d (R_MIPSX_NONE fix), b1cddbae5fe9 (4 lit tests: every relocation in a two-file link, CALL17 base selection and errors, range/alignment errors, emulation and `e_flags`). lld defines `__mipsx_cb0/1` and `_gp` when the script does not. crt0 links with `sim.ld` and `device.ld` unchanged; layout matches `link_layout()`; runs, packs and verifies (mipsx-sdk 1b97a5e, `make check-llvm`) |
| S6 Simulator | Done (lit config left for S8) | `mipsx-sdk/sim`: C simulator with both delay slots, squashing, MD steps, PSW/traps, I/O page; reports abi §9 hazards (exit 125); 69 tests from a spec-written test assembler (`make check`); `sim/mipsx-run` (`--expect-exit`, `--print-exit`) for lit |
| S7 crt0 and packer | Done | `mipsx-sdk/rt`: `crt0.S` (runs on the simulator via mxasm's sim.ld-like link mode), `sim.ld`/`device.ld`; `mipsx-sdk/tools`: Python packer (one type 2 section, uncompressed, position-independent loader stub, runs on the simulator from two flash placements) and independent verifier; top-level `make check`. Link-tested with lld in S5 (`make check-llvm`, mipsx-sdk 1b97a5e): no script changes needed. Before flashing: see `tools/README.md` risks |
| S8 CodeGen skeleton | Done | Commits 70c1b86df36a (libcalls), 0785f32e67ab (codegen), d21a99e79bd1 (simulator tests). 6 llc tests in `test/CodeGen/MIPSX`, 3 execution tests in `test/CodeGen/MIPSX/sim` (need `MIPSX_SDK`; pass at -O2 and -O0 with no hazards). Beyond the plan: globals (%hi/%lo), constant shifts, large frames via r1, `br`. Not yet: conditional branches, setcc/select, variable shifts (S9); indirect calls, frame pointer, varargs, byval (S10); slot filling (S11) |
| S9 Constants and control flow | Done | Commits f88991da954f, 4dec51aa3dda. br_cc (10 conditions, swaps), analyzeBranch, select/setcc diamonds, jump tables (`%w` entries, no shift), indirectbr (byte address, lsr 2), 2-instruction shifted constants, variable shifts via `__ashlsi3`/`__lshrsi3`/`__ashrsi3`, branch relaxation via `jspci r0,#%call(L),r0`. Calls to the SDK's 32-bit shift/mul/div helpers already use clobber-list register masks (part of S12). 15 llc tests; `sim/control.ll` (loops, switch, globals, helpers) passes at -O2 and -O0 |
| S10 Calls and frames | Done | Commits 7bbe94fafea5, 639b59fbe1cf. Indirect calls (lsr 2 + jspci), `-mattr=+long-calls`, frame pointer (fp = CFA, ra/fp at CFA-4/-8, body from fp, epilogue resets sp from fp), dynamic alloca (unreserved call frames), varargs (custom va_arg: 4-byte words, no 8-alignment), i64 via legaliser + 64-bit helper masks. `sim/callconv.ll` (generated, 38 checks: 0-12 mixed args, function-pointer table, varargs, struct returns, sret, alloca, 80 KB frame, fp, recursion) passes at -O2/-O0. Not done: byval (Clang won't use it), tail calls, stack realignment > 8, CFI (S17), `__builtin_return_address` |
| S11 Delay slots and hazards | Done | Commits a1e6481c12e4, 213ab0f6a725, b71f8528bcd6. Filler (two slots from above; squashing fill from the target for loops), padding only where needed, sched model (load latency 2, pre- and post-RA). Benchmark set = the 9 `sim/*.ll` programs (incl. new bubble, crc32, sieve, matmul), measured by `sim/Inputs/measure.py` (dynamic, `mipsx-sim --stats`, includes SDK helpers): delay-slot nops 94.2% → 25.5% (51637 → 13989 of 54838), cycles 186433 → 140856 (−24%); per program after: bubble 37%, callconv 8%, crc32 6%, matmul 12%, sieve 41%. All sim tests pass with no hazards at -O2 and -O0. CodeGen tests now autogenerated (`update_llc_test_checks.py` knows mipsx) |
| S12 Helpers | Done | SDK: `mipsx-sdk/rt/builtins` (commit e4849b6): the 16 helpers on mstart/mstep/dstep, division by zero gives q = 0, r = n, about 196,000 rows checked, clobber lists verified, cycles in its README. Backend: register masks for all 16 helpers from those clobber lists (added with S9/S10); commit 642aa072f929: mul by constant → shifts/adds of its non-adjacent form (≤ 6 instructions, else `__mulsi3`), udiv + urem → one `__udivsi3` call (remainder in r3); div by 2^k was already shifts, other constant divisors stay calls. Commit 78c098543a45: `sim/muldiv.ll` (generated, 22 groups of 32/64-bit edge values) passes at -O2 and -O0. L4 answered for now: helpers live in the SDK |
| S13 Clang | Done | Commits 16f1011c4d08 (`Basic/Targets/MIPSX`: abi §3 types, `__mipsx__`/`__MIPSX__`/`__mipsx_es3890__`/`__mipsx_byte_access__`/`__mipsx_long_calls__`, no inline atomics, asm names and `I`/`J`/`K`), 9819e685db0d (`CodeGen/Targets/MIPSX.cpp`: aggregates ≤ 16 bytes as `[N x i32]`, larger by pointer to an 8-aligned copy, returns ≤ 8 bytes in r2/r3 else sret, `_BitInt` > 64 and complex as aggregates, va_arg in 4-byte words), 67d0924e3e53 (driver: sysroot = `--sysroot` or `<bin>/../lib/clang-runtimes/mipsx`, links `crt0.o`, `-T <board>.ld`, `-lbuiltins -l<board>` and `-lc` if present with ld.lld; `-mboard=sim\|device`, `-mcpu`, `-mlong-calls`), 08d0431b31ca (backend inline asm: `r`, `m` as `off[rN]`, `I`/`J`/`K`), c71b5152f6e1 (filler bug: lost CFG edge when a block has two branches to one target). SDK 3e3f847: `make sysroot MIPSX_LLVM_BIN=...` builds `build/sysroot`. Tests: `clang/test/{Preprocessor/init-mipsx.c,Sema/inline-asm-validate-mipsx.c,CodeGen/MIPSX/{abi,varargs}.c,Driver/mipsx-toolchain.c}`; `clang/test/CodeGen/MIPSX/sim` (needs `MIPSX_SDK` and the sysroot): `hello.c` prints at -O2/-O0, `callconv.c` checks Clang against hand-written assembly per abi §7 (args, split i64, varargs, struct/sret returns, an asm caller). Not done: soft-float (`__adddf3`, `__nedf2`, … are missing: any FP arithmetic fails to link), atomics/`__sync` runtime calls, libc (S15), `-verify-machineinstrs` fails after the S11 filler (liveness inside delay-slot bundles; pre-existing) |
| S14 Test triage | Done | Runtime (mipsx-sdk d748a56): `libbuiltins.a` gains compiler-rt's generic builtins (soft-float, complex, overflow, bit counting; built by `make sysroot` from the LLVM checkout) and `rt/atomic.c` (libatomic-style `__atomic_*`, PSW I bit masked, abi §13.6). Driver (mipsx-sdk ca08f1e): `tests/run_tests.py builtins\|torture` on mipsx-sim, with the test-suite's skip lists and flags, picolibc or a small test libc (`--libc mini`); llvm-test-suite cloned to `~/src/openmipsx/llvm-test-suite`. **Before**: no soft-float or atomics, so nothing using `float`/`double` linked (85 torture tests at -O2 call soft-float helpers directly, and any printf does): builtins 0/98, torture not measurable. **With the runtime** (and the atomics fix): builtins 95/99 at -O0, 91/99 at -O2; torture 1387/1411 (98.3%) at -O0, 1392/1411 (98.7%) at -O2. **Fixes** (llvm-project, one commit each, lit test each): 0bbdd772ff4b atomics → AtomicExpand + `__atomic_*` calls, fence = MEMBARRIER (was "Cannot select AtomicSwap"); 0936d8dd9516 stack realignment for alignment > 8 (fatal error before: 10 torture tests at -O0, 3 at -O2, compiler-rt atomic_test); 2c4c687a6177 vector `getSetCCResultType` (assertion: 6 at -O0, 6 at -O2); b3984f1e9938 `__builtin_return_address`/`frame_address` (2 + 2); 8530b3a411c7 spill reloads without the load's early-clobber, new `LDfi` (live intervals broken: greedy crashes, e.g. divsf3_test, picolibc strptime, and silent misallocation: 20021120-1 -O2, addsf3new/muldf3new -O2, muldc3 in libbuiltins, printf in the test libc; same bug as S15's `docs/compiler-bugs/reload-earlyclobber.md`); c63c0d9520da `__attribute__((aligned))` = 8 (alloca-1 -O2); 3f189e4f0725 call frame reserved unless dynamic allocas (pr70903 -O0: sp moved around calls in a realigned frame); ac2286603aaf driver links `libc.a` from a relative `--sysroot`; b56191ec36b5 delay-slot bundle liveness (internal reads, kill flags): **`-verify-machineinstrs` now passes** on the whole torture suite at -O0 and -O2. Remaining torture failures before the end were test-library gaps (heap, `%hh`/`%#o`/`%g`) and one missed `-fwrapv`, fixed in the driver. **After**: builtins 101/101 (picolibc) and 98/98 (test libc) at -O0 and -O2; torture 1414/1414 (picolibc) and 1411/1411 (test libc) at -O0 and -O2, also with `-mllvm -verify-machineinstrs`; 100 torture tests unsupported (suite skip lists, `__builtin_setjmp`, `__int128`, file system, mmap). Not done: `__builtin_setjmp`/`longjmp` (pr64242, built-in-setjmp), realignment with dynamic allocas (error), `-O1`/`-Os`/`-O3` runs, the ieee/ and builtins/ torture subdirectories |
| S14+ Test triage (further) | Not started | Candidates: the torture `ieee/` and `builtins/` directories, -Os and -O3, picolibc's tests (S15), lwIP etc. (S16) |
| S15 picolibc | Done | picolibc 1.8.12 cloned to `picolibc`, branch `mipsx`: 36c42f55a3 (`libc/machine/mipsx/setjmp.S`, `_JBLEN` 13, `__IEEE_BIG_ENDIAN`: Clang has no `__FLOAT_WORD_ORDER__`), 7a5ea2b83e (`libos/semihost/machine/mipsx`: stdin/stdout/stderr, read/write on the I/O page; `_exit` stays in libsim.a), 7ff65dffbb (Meson property `test_runtime_link_args`: tests link with the driver's crt0/script/libs). `mipsx` is not caught by `mips*`: Meson matches `cpu_family` exactly (alias table maps only `mips64`), headers test `__mips__`, CMake uses `CMAKE_SYSTEM_PROCESSOR` exactly. Config: `-Os`, no picocrt, no TLS (errno global), no atomic ungetc. SDK: 90fe14e (`.fini_array` incl. picolibc's `.fini_array_onexit`, `__heap_start/__heap_end`), ca6742d (abi §13.7: jmp_buf is 13 words, not 12), 81733f3 (`meson/mipsx.txt`, `cmake/mipsx.cmake`; ctest/meson test run on the simulator), eb37619 (`make libc` installs headers, libc.a, libm.a, libsemihost.a; libsim.a gains the I/O-page stdio so printf needs no flags; `make check-libc`; `docs/picolibc.md`). **picolibc tests on the simulator (llvm-project 3f189e4f0725): 277 built, 257 pass, 2 expected failures, 17 skipped (files, dirs, rlimits, other targets), 1 fails (`test-argv`: no argv, crt0 calls main(0, 0)); 259/260 of those that run.** Bug found: `docs/compiler-bugs/reload-earlyclobber.md` (early-clobber reloads broke live intervals: crash on strptime, 5 wrong-result/hang tests), fixed by S14 in 8530b3a411c7. Not done: device stdio (no UART driver; printf with `-mboard=device` fails to link), argv, file I/O, `--args` in mipsx-run |
| S16 Size | Done | Upstream lwIP 2.2.1, littlefs 2.11.3, BearSSL 0.6 in `~/src/openmipsx/third_party`, unmodified; configs, self-tests and size report in `mipsx-sdk/third_party-builds` (mipsx-sdk b32ce7d, 02e9aa2; `make`, `make check`, `make check-bearssl`, `make sizes`). lwIP: `NO_SYS`, IPv4, Ethernet/ARP, TCP, UDP, ICMP, DHCP, DNS, frag., SNTP, TFTP, httpd; BearSSL client: TLS 1.2, ECDHE_RSA/ECDSA, AES-128-GCM/ChaCha20, P-256, `i15`/`m15`, `x509_minimal` (and `init_full`). **Nothing blocked compilation**: all three built at -Os and -O2 with the S15 compiler, and their tests passed. **Self-tests on the simulator**: lwIP TCP echo 20000 bytes and UDP 1–3000 bytes (fragmented) over loopback plus DHCP/httpd/SNTP/TFTP on a dummy Ethernet interface; littlefs on its RAM block device (dirs, files 0–20000 bytes, 400 rewrites with compaction, seek/truncate/rename/remove, remount); a TLS ClientHello: all pass at -Os/-O2 (and -Oz), with `-verify-machineinstrs` clean. BearSSL `test_crypto` (one simulator run per test, names on stdin, 2×10¹⁰-cycle limit), -Os and -O2: 34 pass, 9 n/a (x86/POWER-only code), no failures. 14 stop at the cycle limit (DES Monte Carlo, constant-time HMAC, RSA, C25519, EC_prime, ECDSA: big numbers with no multiplier). DES, HMAC, C25519 and ECDSA pass vectors before the limit; RSA and EC_prime do not finish their first vector (they start with RSA private keys up to 4096 bits and ten rounds of P-256/384/521 mul/muladd). **RSA/EC correctness checked**: `bearssl/pk_kat.c` (`make check-pk-kat`, first step of `check-bearssl`), BearSSL's own vectors for what a TLS client runs, passes all 34 checks at -Os and -O2 on the simulator, as on the host: no compiler bug. Cycles at -Os (-O2 within 2%; s at the guessed 81 MHz, no cache misses modelled): RSA-2048 public/PKCS#1 verify `i15` 47.4 M (0.58 s), `i31` 40.4 M; P-256 `p256_m15` mul 56.2 M (0.69 s), mulgen 37.8 M, ECDSA verify 104 M (1.28 s); `prime_i15` P-256 mul 119 M, ECDSA verify 238 M; P-384 ECDSA verify 701 M (8.6 s, too slow); X25519 `c25519_m15` 28.4 M (0.35 s). A handshake's public-key work with a 2-certificate RSA-2048 chain: ~200 M cycles (2.5 s) with X25519, 236 M with P-256; an ECDSA P-256 chain ~410 M (5 s); a P-384 intermediate is too slow, so provisioning servers should use RSA (or P-256-only) certificates. 64–71% of EC time is `__mulsi3` (~25 cycles a call); `-DBR_SLOW_MUL15=1` gains 4–8% (passes); `BR_CT_MUL15` is optimised away by Clang; `__mulsi3`'s early exits are not constant-time (ECDHE key, `ghash_ctmul32`). The simulator runs ~50 M cycles/s, so it is not the bottleneck (`pk_kat` takes about 100 s). littlefs's own runner and lwIP's unit tests need argv, files, fork and libcheck, so not run. **Sizes** (flash = text + rodata + data linked from each library, -Os without assertions; S15 compiler → S16): lwIP 88,232 → 82,345 (−Oz 80,361; with assertions 103,247; −O2 99,329; .bss 46 KB); littlefs 45,292 → 44,192 (−Oz 42,524); BearSSL minimal client 88,859 → 88,227 (−Oz 85,739; −O2 106,327); `init_full` 125,691 → 124,831; plus ~19 KB shared run-time (printf, helpers). **Against stack plan §3.4**: lwIP is at the top of its 50–80 KB line (80 KB at -Oz, 82 KB at -Os, before a driver); BearSSL fits its 100–150 KB line with room (86–88 KB minimal, 121–125 KB full); littlefs (42–44 KB) has no line of its own and would take most of the 20–40 KB kernel-and-drivers line, so the simple key/value log alternative is worth weighing. Unlinked, MIPS-X is 9–24% larger than MIPS32, 1.1–1.4× RV32I and 1.9–2.3× Thumb-2 (no LUI, no multiplier: 1,323 `__mulsi3` calls in BearSSL, two delay slots, load delay). **Size fixes** (llvm-project, one commit each, with tests): a0c91cfdfa55 `%lo` folded into load/store offsets (lwIP −1,260 bytes); 718de6d83e00 small data: ≤ 8-byte variables in `.sdata`/`.sbss` through gp, abi §6.3 (`-mipsx-ssection-threshold`, `-mipsx-extern-sdata`; addresses outside the variable keep %hi/%lo; lwIP −2,587); 6fd9ed531416 delay-slot filler moves target instructions instead of copying when the branch is the only way in, and fills from a single-predecessor fall-through (lwIP −2,040, littlefs −1,008, BearSSL −624). lwip_test also runs 10.5% fewer cycles. Sysroot rebuilt with the new compiler (all objects must agree on small data; a mismatch is a GPREL17 link error). Torture: 1414/1414 at -O0, -O2, -Os and -Oz (picolibc). picolibc's own tests with the rebuilt sysroot: 257 pass, 2 expected fail, 17 skipped, 1 fail (test-argv, as at S15). **Not done, measured**: a gp-relative literal pool for the remaining %hi/%lo (369 left in lwip_test, 322 in lfs_test; ~1–2 KB each); save/restore millicode for prologues at -Os (est. 3–4% on lwIP); remaining nops are 16% of lwIP code (4.5% load delay, 11% branch slots); Clang's `-G` is not wired to the threshold. **Multiply** (follow-up): llvm-project d63693be8e1f multiplies inline (`MULSTEPS`: multiplier at the top of MD, `mstart` + n−1 `mstep`, nop for P4) when known bits show an operand fits in ≤ 17 bits (signed), except at -Oz; a nop also separates a call's return from a following `movtos md` (`__mulsi3` ends with msteps in its return slots; abi §9.2). mipsx-sdk 5d6bcee makes `__mulsi3` constant-time (always 32 steps, 35 cycles; was 14–48 by operand, which leaked the X25519 scalar: 28.4 M vs 27.8 M by input). `pk_kat` passes at -Os and -O2; cycles at -Os (mipsx-sdk 30415a5): RSA-2048 `i15` 47.4 → 37.4 M, `i31` 40.3 M; `prime_i15` P-256 ECDSA 238 → 179 M, P-384 701 → 519 M; `p256_m15` mul 56.2 → 70.6 M, ECDSA verify 104 → 123 M (1.52 s), X25519 28.4 → 36.2 M (constant-time cost: m15 limbs come from memory, so known bits cannot show them narrow). Handshake (RSA-2048 chain + X25519) ~185 M cycles. Sizes: minimal TLS client +1,184 bytes at -Os (1.3%), +1,232 at -O2, lwIP +36–48, littlefs/`init_full` and -Oz unchanged; `__mulsi3` −224 bytes. Torture 1414/1414 at -O0/-O2/-Os/-Oz. **Not done**: BearSSL `MUL15` masking (`(x & 0x7FFF) * y` gives P-256 ECDSA 75 M but fails `c25519_m15`, so per use); `__muldi3` is not constant-time (`m31`/`i31`, `ghash_ctmul`) |
| S17 Debug info | Done | llvm-project 403c2827c4d6 (CIE: CFA = r29 + 0, return address column 31 = `DW_CFA_val_expression` r31 × 4, so unwinders get byte addresses; DWARF numbers r0–r31 = 0–31, psw 32, md 33 were already in the TableGen), 6af7660f6cb2 (prologue CFI from a late pass, `MIPSXFrameCFI`, after the post-RA scheduler; the filler looks through CFI: `.text` is identical with and without `-g` at -O0/1/2/s/z; def_cfa_offset after the sp adjustment, offsets after each callee-saved store, `def_cfa r30, 0` once fp is set (fp frames, dynamic alloca, realignment); large frames need nothing extra; no epilogue CFI), 33005e231ca6 (`getFrameIndexReference`: `DW_OP_fbreg` in fp frames was off by the frame size), a69fcb6f755b (lld test; no lld change needed). mipsx-sdk b234d52: abi §2.4, §4 (every DWARF address is a byte address), new §8.6 (CFI rules), D15. Tests: `MC/MIPSX/cfi-regs.s`, `CodeGen/MIPSX/cfi.ll` (each frame shape), `DebugInfo/MIPSX/{frames,cfi-same-code}.ll`, `lld/test/ELF/mipsx-debug.s` (`.debug_*`/`.eh_frame` relocations, line table, `--eh-frame-hdr`, symbolizer, objdump `--line-numbers`); 14 llc tests regenerated (functions without `nounwind` now show CFI). Linked `clang -g` programs at -O0/-O2: `llvm-dwarfdump --verify` clean, symbolizer/objdump `--source` right; a hand unwinder replaying `mipsx-sim --trace-file` with the `--debug-frame` rows recovers leaf → mid → main → `_start` exactly at -O0 (fp chain) and -O2. Torture with `-g -mllvm -verify-machineinstrs`: 1414/1414 at -O0 and -O2. Not exact (abi §8.6): inside epilogues, and up to two instructions where the filler moved a prologue instruction into a later branch's slots. Not done: CFI for the SDK's hand-written assembly (crt0, helpers: no FDE, so unwinding stops there), a debugger stub |
| S18 CI | Done (local only: nothing pushed or published) | llvm-project 6558f757aca1: `.github/workflows/mipsx.yml` (only MIPSX, clang;lld, ccache; checks out mipsx-sdk and picolibc, `make check`, `make libc`, `make check-llvm`, MIPSX lit tests in llvm/clang/lld with `MIPSX_SDK`, fails if the sim tests are unsupported; manual run adds check-llvm/clang/lld and the builtins/torture suites) and `.github/MIPSX.md` (upstream workflows stay in the tree; most are guarded by `github.repository`, the rest are turned off per repository with `gh workflow disable`). mipsx-sdk 04bbe72: `.github/workflows/ci.yml` (`make check` on Linux x86-64/arm64 and macOS; with the latest toolchain release: smoke test, `make check-llvm`, sysroot + picolibc from the commit under test, compiler-rt builtins tests), `release.yml` (LLVM at `release/refs` on ubuntu-22.04, ubuntu-22.04-arm, macos-14; package, smoke test, artifact; on a `v*` tag a GitHub release with SHA256SUMS and `ghcr.io/<owner>/mipsx-toolchain` for amd64/arm64), `release/package.py` (cmake --install components + the SDK and picolibc built in a work copy into `lib/clang-runtimes/mipsx`, simulator, packer wrappers, CMake/Meson files, docs, licences, `BUILDINFO`), `release/smoke-test.sh`, `release/Containerfile`. Tested locally on x86-64 against `llvm-project/build`: archive 102 MB (`.tar.xz`, 394 MB unpacked; packaging takes under 2 minutes with ccache-warm objects); from a clean directory, `clang --target=mipsx` hello.c with picolibc printf runs on the simulator at -O2/-O0, a `-mboard=device` program packs and verifies, a CMake project builds and passes ctest; the container image builds and does the same. actionlint 1.7.7 passes all three workflows. Not tested: anything on GitHub (runners, caches, releases, ghcr), the arm64 and macOS builds. To publish: see mipsx-sdk `release/README.md` |

## 6. Keeping the fork current

Once a year (stack plan §3.1), rebase `mipsx` onto the newest release
tag: `git rebase --onto llvmorg-<new> llvmorg-23.1.4 mipsx`, fix the
build and tests, and record the new base in §1. Keeping commits to one
topic each makes the conflicts small. The blobless clone fetches old
file versions on demand, so the rebase needs network access.

## 7. Open questions

| # | Question | Decided in |
|---|---|---|
| L1 | Does the ES3890 interlock on load-use, unlike the manual? If it does, the hazard padding can go. Needs a test on hardware | After first device image |
| L2 | Does the ES3890 have a variable shift instruction, and how is it encoded? The spec mentions one in later firmware | Dirty side, then S9 |
| L3 | Function pointer representation and long-call strategy | S1: decided. Function pointers are byte addresses (indirect call shifts right by 2). Direct calls are one `jspci` through r0, r23 or r24, chosen by the linker (R_MIPSX_CALL17), reaching at least 1 MB of code; `-mlong-calls` for more. `mipsx-sdk/docs/abi.md` §5, D2, D3 |
| L4 | compiler-rt builtins or SDK for the helper routines | S12: decided. In the SDK for now (`mipsx-sdk/rt/builtins`, hand-written assembly), with libgcc/compiler-rt names and the standard convention, so they can move to compiler-rt's `lib/builtins/mipsx/` once Clang and lld can build it (S13–S15) |
