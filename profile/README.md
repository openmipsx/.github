<p align="center">
  <img src="https://raw.githubusercontent.com/openmipsx/.github/main/profile/openmipsx.svg" width="120" alt="openmipsx logo: a chip marked with an X">
</p>

# openmipsx

Open tools for **MIPS-X**, the 1980s Stanford RISC design that ESS
Technology built into its ES38xx media chips. The main target is the
**ES3890F**, the processor in the Sipura and Linksys SPA and PAP2T
analogue telephone adapters.

MIPS-X is not the commercial MIPS architecture, and no mainstream
compiler or disassembler supports it. This organisation builds the
compiler, libraries, simulator and analysis tools that it lacks, with
open replacement firmware for those adapters as the goal.

## Repositories

| Repository | What |
|---|---|
| [mipsx-sdk](https://github.com/openmipsx/mipsx-sdk) | ABI, instruction-set simulator, start-up code, firmware image packer and the toolchain release builds |
| [llvm-project](https://github.com/openmipsx/llvm-project/tree/mipsx) (branch `mipsx`) | LLVM with an experimental `MIPSX` target: Clang, lld and the LLVM tools |
| [picolibc](https://github.com/openmipsx/picolibc/tree/mipsx) (branch `mipsx`) | The C library, ported to MIPS-X |
| [ghidra-mipsx](https://github.com/openmipsx/ghidra-mipsx) | Ghidra processor module for MIPS-X and a loader for SPA firmware images |

## Status

Experimental. Clang compiles C for MIPS-X, and the output is tested on
the simulator. Nothing built with these tools has run on real hardware
yet, and the first toolchain release is in preparation.

## Clean room

Everything here is written from public documentation, chiefly Paul
Chow's *MIPS-X Instruction Set and Programmer's Manual* (Stanford
CSL-86-289,
[PDF](https://www.eecg.utoronto.ca/~pc/research/publications/mipsx.isetman.pdf)),
and from specifications that record facts only. No vendor code is
included.

Linksys, Cisco and Sipura are trademarks of their owners. This project
is not affiliated with or endorsed by them.
