# AFL-Morello

This is a port of AFL (specifically the LLVM instrumentation) for the CHERI architecture on Morello. It is likely not compatible with traditional architectures.

## Background

Fuzz testing on Morello is interesting for a few reasons. CHERI traps mean that benign or obscure memory violation bugs that may not directly cause crashes or hangs can be captured by the fuzzer. This means these illegal behaviours can be detected without the need for UBSan, ASAN etc. which would be required if testing on a non-CHERI system.

Unfortunately, at time of writing, Morello does not provide the clang instrumentation which is required for tools like LibFuzzer. This is why we port AFL, which is capable of injecting it's own instrumentation while compiling.

## How to Use

The following demonstrates how we built AFL and compiled and tested our code. This compiles AFL in hybrid mode, and the binary itself in purecap mode.

```sh
CC=clang XCC=clang XCFLAGS='' gmake AFL_NO_X86=1
cd llvm_mode/
LLVM_CONFIG=llvm-config-morello CC=cc CXX=cc XCC=cc XCFLAGS='' gmake
cd ..
AFL_CC=cc AFL_CXX=c++ ./afl-clang-fast -Xclang -morello-vararg=new -march=morello -mabi=purecap ./sample.c
./afl-fuzz -i in -o out -t 100000 ./a.out
```

This port has been tested on [FuzzGoat](https://github.com/fuzzstati0n/fuzzgoat) and compared to standard AFL running on WSL2, where it captured the same details.

## Writeup
AFL instrumentation works by injecting a plugin into clang to add instrumentation
during compilation. [AFL-CHERI](https://github.com/CTSRD-CHERI/AFL-CHERI) already provided a method to cross compile for CheriABI, this meant we could borrow the
change used to allow CheriABI instrumentation (modifying mapptr global address space to 200). The next problem was to allow the plugin
to work using the version of clang on Morello (clang-13), the AFL-CHERI repo used a legacy plugin and plugin injection which did
not seem to work on Morello (the plugins were not loaded at all). We modify the plugin code to use the modern clang plugin
structure and inject it using the -fpass-plugin option rather than passing straight to cc1 using -Xclang and -load.
Finally we make minor changes to the Makefiles to ensure libaries/objects used by clang/afl-fuzz are explicitly compiled
for hybrid and everything used by the application is compiled for purecap.