# Building the replacement firmware from open-source parts

Status: proposal, 2026-10-27; toolchain route (P1) decided the same day.
Clean-side document: it names public
open-source projects and refers to the specs in [openmipsx/devices](https://github.com/openmipsx/devices),
with no code or text from the vendor firmware. See
[`clean-room.md`](https://github.com/openmipsx/devices/blob/main/clean-room.md) for the process.

## 1. Summary

Most of the replacement firmware can be assembled from existing
open-source components sized for small embedded systems. The board's 2 MB
of SDRAM and 1 MB of flash ([`hardware.md`](https://github.com/openmipsx/devices/blob/main/spa3000/hardware.md)) are
enough for a SIP gateway with G.711 voice on the FXS port.

The obstacles are not missing software. They are:

1. no C compiler or assembler for MIPS-X (§3.1; plan: an LLVM backend);
2. the ES3890F's on-chip peripherals, which have no public documentation
   (§3.2);
3. the chip's second core, which may be needed for voice processing
   (§3.3).

The compiler comes first: none of the components below can be built
without it.

Open-source code, public datasheets and public standards are all allowed
on the clean side, so using these components does not affect the
clean-room separation. Their licences still have to be followed (§5).

## 2. Components

| Function | Candidates | Notes |
|---|---|---|
| Kernel | FreeRTOS; or a small scheduler of our own | A port needs a context switch, a timer tick and interrupt entry and exit, all from the processor spec and the timer and interrupt controller specs (§3.2). |
| TCP/IP, DHCP, DNS, TFTP, SNTP | lwIP | Made for systems this size. Its apps directory includes DHCP, DNS, TFTP, SNTP and a small HTTP server. |
| Ethernet driver | Written from the RTL8019AS datasheet; NE2000 drivers in lwIP ports, Linux (`ne2k`) and the BSDs (`ed`) as references | The chip is NE2000-compatible, 10 Mbit/s only. |
| Configuration storage | littlefs; or a simple key/value log | The flash uses the standard AMD/JEDEC command set and has sixteen 64 KB sectors ([`hardware.md`](https://github.com/openmipsx/devices/blob/main/spa3000/hardware.md)). |
| TLS, certificates | BearSSL (smallest); Mbed TLS | RSA handshakes will be slow on this CPU. Only needed for HTTPS provisioning and the admin page. |
| Web administration | lwIP httpd; or a small server of our own | |
| Provisioning and firmware upgrade | Our own, on lwIP and the TLS library | Behaviour taken from the device's public administration guide. Images must follow [`firmware-image-format.md`](https://github.com/openmipsx/devices/blob/main/common/firmware-image-format.md). |
| SIP user agent | libre (the library under baresip); osip2 with eXosip | The largest and most demanding part. PJSIP is probably too big for 1 MB of flash. |
| RTP, jitter buffer | libre; oRTP | |
| Voice codec: G.711 µ-law and A-law | Our own (a lookup table); spandsp | Required. See §2.1 for the full codec set. |
| Key presses: RFC 4733 telephone events | Our own; libre | Required. Sends DTMF as events rather than audio. |
| Comfort noise: RFC 3389 | Our own; spandsp's noise generator | Required for interworking with far ends that suppress silence. |
| Voice codec: G.729A (with Annex B) | bcg729 | Optional, only if the CPU can run it (P2). Patents expired in 2017. |
| Voice codec: G.722 | spandsp | Optional, for compatibility only (§2.1). |
| Echo cancellation, DTMF detection and generation, call-progress tones, caller ID (FSK and DTMF) | spandsp, taking only the modules needed | The line echo canceller is essential: without it, callers hear their own echo once network delay is added. |
| FXS (Si3210) and FXO (Si3050 and Si3019) drivers | Silicon Labs datasheets and application notes; the Asterisk DAHDI `wctdm` driver, which drives the same chip family, as a reference | Register-level control over SPI, audio over PCM. How they connect to the ES3890F is open question H3 in [`hardware.md`](https://github.com/openmipsx/devices/blob/main/spa3000/hardware.md). |
| Voice prompts | Our own recordings, or a text-to-speech engine run at build time | |

### 2.1 Codec set

Both telephone ports are narrowband. The Si3210 on the phone port samples
at 8 kHz (its public datasheet), which carries audio up to about 3.4 kHz,
and the line port connects to a narrowband landline. G.711 carries that band with
essentially no loss, so it gives the best quality this hardware can
deliver. Wideband codecs cannot improve on it here; they would only help
interworking with a far end that refuses G.711, which is rare. Bandwidth
is no longer a reason to compress: a G.711 call needs under 100 kbit/s in
each direction, including packet headers.

Every SIP phone, adapter, PBX and provider supports G.711, RFC 4733
events and comfort noise, so these alone make a usable adapter. In order
of priority:

| Priority | Feature | Reason |
|---|---|---|
| Required | G.711 µ-law and A-law, RFC 4733 telephone events, RFC 3389 comfort noise | The common ground of every SIP device |
| Optional | G.729A, with Annex B | Some provider trunks and enterprise PBXs still insist on it. Only if the CPU can run it in real time (P2). |
| Optional | G.722 | Cheap; standard on desk phones and PBXs; no quality benefit on these ports |

Not planned:

| Codec | Reason |
|---|---|
| G.726 | Rarely negotiated; G.711 covers its use |
| G.723.1 | Legacy, heavy to run, adds delay |
| iLBC, GSM, Speex | Legacy; displaced by Opus |
| Opus | Probably too heavy for this CPU without floating point, and no benefit on an 8 kHz port |
| AMR-WB | Mobile networks; licensed |
| Fax (G.711 pass-through, T.38 relay) | Not needed for the intended use. Can be added later from spandsp, which has fax and modem tone detection and a T.38 gateway. Until then, fax and modem calls are unsupported: echo cancellation and silence suppression will corrupt them. |

## 3. Obstacles

### 3.1 Toolchain

No mainstream compiler, assembler or linker supports MIPS-X
([`firmware-image-format.md`](https://github.com/openmipsx/devices/blob/main/common/firmware-image-format.md),
processor section).

**Decision (P1): an LLVM backend**, published so that others can use it.

#### What is needed

A compiler is only one part. The full set is:

| Part | Notes |
|---|---|
| ABI document | Written first; everything else follows it. Our firmware does not have to copy the stock convention: we choose register use, stack layout, how function pointers work given word-addressed code, and how 32-bit constants are built, since there is no load-upper-immediate (literal pools, or `addi` and shifts). The base register values go in our image's load directory. |
| Assembler and disassembler | Two delay slots after every branch and jump, squashed branches, and the ES3890's byte and halfword loads and stores ([`processor-es3890.md`](https://github.com/openmipsx/devices/blob/main/common/processor-es3890.md)) |
| Linker | ELF relocations for MIPS-X: at least the 16-bit word displacement of branches, the 17-bit offset of `jspci` calls from a code base register, and 32-bit byte and word addresses. Linker scripts define the code base symbols. |
| C compiler | C99 at least: lwIP, libre and Mbed TLS need it |
| Runtime | `crt0` (stack pointer and base registers), multiply and divide helpers built on `mstep` and `dstep` (there is no hardware multiply), and a C library: picolibc |
| Image packer | ELF in, upgrade image out. Written from [`firmware-image-format.md`](https://github.com/openmipsx/devices/blob/main/common/firmware-image-format.md) on the clean side, not adapted from the unpacker |
| Instruction-set simulator | Runs the compiler tests in CI and tests images before anything is flashed (§3.5). It also lets people without the hardware contribute. MAME has a MIPS-X disassembler but, as far as is known, no emulator core. |

#### Why LLVM

- One TableGen description of the instructions gives the assembler,
  disassembler and encoder (`llvm-mc`, `llvm-objdump`). lld needs only a
  small file for the relocations. Clang becomes a cross-compiler with
  `--target`, and lit and FileCheck give a test framework.
- Its licence (Apache 2.0 with the LLVM exception) fits whatever is
  chosen for P5.
- Small existing backends to follow: Lanai and SPARC (delay-slot filling,
  to be extended to two slots) and MSP430 (simple, no hardware multiply).

Rejected:

| Option | Reason |
|---|---|
| vbcc, lcc | Not open-source licences: as understood, vbcc allows non-commercial use and needs the author's permission to distribute modified versions or for commercial use, and lcc is similar (check current terms). Either blocks publishing the toolchain. lcc is also C89 only. |
| Amsterdam Compiler Kit | BSD and retargetable, but C89 only, which rules out most of §2 |
| GCC with binutils | Its annulled delay slots match MIPS-X squashing well, but it means maintaining a GCC port, a binutils port and newlib as three projects, with harder internals |

#### Publishing

- **LLVM fork** with an experimental target `MIPSX`, kept as a clean patch
  series on one LLVM release branch and moved to a new release about once
  a year. Upstream acceptance is unlikely (LLVM wants active maintainers
  for each target), but a clean series keeps it possible.
- **SDK repository** (MIT or Apache 2.0): the ABI document, `crt0`, linker
  scripts, the multiply and divide helpers, picolibc build settings, a
  CMake toolchain file, the image packer, the simulator and an LED-blink
  example.
- **Binaries**: CI builds the fork with only the MIPS-X target
  (`-DLLVM_TARGETS_TO_BUILD="" -DLLVM_EXPERIMENTAL_TARGETS_TO_BUILD=MIPSX`)
  and publishes release archives for Linux (x86-64, arm64) and macOS,
  plus a container image with the toolchain and SDK together.
- **Identifiers**: MIPS-X already has an official ELF machine number,
  `EM_MIPS_X` = 51 (in glibc's `elf.h`, binutils, Ghidra and LLVM 23's
  `ELF.h`). The ES3890's differences from the
  Stanford manual can be marked in `e_flags`. Check that a triple
  such as `mipsx-unknown-elf` is not caught by tools that match any
  `mips*` prefix; pick another name if it is.
- **Clean room**: the toolchain is clean-side. Tests use hand-written
  assembly and C test suites run on the simulator, never vendor bytes.
  Disassembly of our own test binaries can be checked against the Ghidra
  extension as an independent decoder.

#### Effort

The assembler, linker and simulator come first and should take days to a
couple of weeks. That is enough for step 2 of §4 in assembly. A C backend
good enough for lwIP is the large part, probably months; hardware
bring-up in assembly can continue alongside it. The work is broken into
sessions in a separate LLVM backend plan.

### 3.2 ES3890F peripherals

The new firmware needs register-level descriptions of:

- the interrupt controller and timers;
- GPIOs for the LEDs, the relay and any button;
- the external bus chip selects for the flash and the RTL8019AS;
- the SPI or serial control path to the Si3210 and Si3050;
- the PCM path that carries audio to and from them.

The ES3890F has no public datasheet that covers these. They have to come
from dirty-side analysis, written up as specs under
[openmipsx/devices](https://github.com/openmipsx/devices). This is the slowest part of the project, and the
order the specs arrive in sets the order in which work can proceed (§4).

### 3.3 Second core and CPU speed

The ES3890F has two cores. Only the main core's instruction set is known.
Hypothesis (from dirty-side analysis): the stock firmware hands voice
processing to the second core or to a dedicated peripheral block. If
that cannot be documented, all codec and signal-processing work runs on
the main core, and its clock speed is not yet known.

- G.711 with echo cancellation and DTMF detection should be manageable
  on the main core.
- G.729 may not be. Measure before committing to it.

### 3.4 Flash budget

The application, its configuration and voice prompts must fit in 1 MB,
alongside whatever the boot section needs. Rough sizes:

| Part | Size estimate |
|---|---|
| Kernel and drivers | 20–40 KB |
| lwIP and its apps | 50–80 KB |
| TLS (BearSSL) | 100–150 KB |
| SIP and RTP | 150–300 KB |
| Codecs and signal processing | 50–150 KB |
| Voice prompts | 50–100 KB |

That fits, but without much to spare. Compressing the image, as the
format already allows, helps.

### 3.5 Booting and recovery

At first the new image has to be loaded by the existing boot section, so
it must follow [`firmware-image-format.md`](https://github.com/openmipsx/devices/blob/main/common/firmware-image-format.md)
exactly. Before flashing anything experimental, there needs to be a way
to recover a unit that no longer boots: open question H7 in
[`hardware.md`](https://github.com/openmipsx/devices/blob/main/spa3000/hardware.md) (reading and rewriting the flash in
circuit), and task 4 in [`clean-room.md`](https://github.com/openmipsx/devices/blob/main/clean-room.md).

The stock section B helps here. It is a network recovery program that
installs only type 2 sections, never itself or the boot section
(format spec §5 and §7.1). If an experimental image is ever written so
that it fails the boot selector's checks, the unit falls back to section
B and can be reflashed over Ethernet. But an image that passes the checks
and then hangs is still started every time. Unless something can force
section B to run (format spec Q7), that case needs the in-circuit route.
The stock application's upgrader rewrites the boot and recovery sections
whenever an image contains them and they differ (format spec §7.2). So
our early images should contain only a type 2 section, which leaves the
stock section B and boot section in place.

## 4. Suggested order of work

Each step proves the one before it.

1. **Toolchain** (§3.1). ABI document, then assembler, linker, simulator
   and image packer; the C compiler follows and can overlap steps 2 and 3.
2. **First image.** Blink an LED from an image loaded by the existing
   boot section, written in assembly if the C compiler is not ready. This
   proves the image format, the entry conditions and the toolchain.
   Needs: GPIO spec, recovery path.
3. **Kernel.** Timer interrupt and task switching. Needs: interrupt
   controller and timer spec.
4. **Network.** RTL8019AS driver and lwIP; ping, then DHCP, then a web
   page. Needs: bus and interrupt wiring for the Ethernet chip.
5. **Phone port.** Si3210 bring-up: ringing, hook detection, dial tone,
   DTMF detection, using G.711 on the main core. Needs: SPI and PCM spec.
6. **SIP calls** between the phone port and the network, with G.711,
   RFC 4733 events and comfort noise (§2.1).
7. **Configuration, web administration and firmware upgrade.**
8. **Line port.** Si3050 and Si3019: FXO calls and caller ID.
9. **TLS and provisioning.**
10. **Optional codecs**: G.729A if the main core is fast enough or the
    second core can be used, and G.722.

Steps 1 to 6 make a working SIP adapter for one analogue phone. Matching
the stock firmware's full feature set depends mostly on §3.3.

## 5. Licences

The licence of each component decides the licence of the combined image:

| Component | Licence (check current versions) |
|---|---|
| LLVM (toolchain only; its compiler-rt runtime, if used, is linked into the image) | Apache 2.0 with LLVM exception |
| picolibc | BSD-style |
| FreeRTOS, lwIP, littlefs | MIT or BSD-style |
| BearSSL | MIT |
| Mbed TLS | Apache 2.0 (or GPL 2.0 or later) |
| libre | BSD |
| osip2, eXosip | LGPL; eXosip is GPL with a commercial option |
| spandsp | LGPL 2.1 (library) |
| bcg729 | GPL 3 |
| DAHDI `wctdm` | GPL 2; use as a reference for chip behaviour, or follow the GPL if code is reused |

An image linked with GPL code must be distributed with its source under
the GPL. That suits an open project, but decide early whether the
project's own code is to be GPL-compatible.

## 6. Open questions

| # | Question | Blocks |
|---|---|---|
| P1 | ~~Which compiler route (vbcc or lcc, LLVM, GCC)?~~ Decided 2026-10-27: an LLVM backend, published with an SDK (§3.1) | — |
| P2 | Main-core clock speed, and how many cycles G.711 with echo cancellation, and G.729A, take | Whether G.729A is offered |
| P3 | Can the second core be documented and programmed? | Voice processing on the main core vs. offloaded |
| P4 | Can the boot section be kept as is, and what does it require of an image? | First image |
| P5 | Licence for the project's own code | Choice of SIP stack and codecs |
