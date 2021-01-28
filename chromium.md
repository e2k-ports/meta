basically, we base on the [official Linux instructions](https://chromium.googlesource.com/chromium/src/+/master/docs/linux/build_instructions.md)
with some changes specific for e2k.

# [depot_tools](https://commondatastorage.googleapis.com/chrome-infra-docs/flat/depot_tools/docs/html/depot_tools.html)

```
git clone https://chromium.googlesource.com/chromium/tools/depot_tools.git
```

```
export PATH=$PATH:$HOME/depot_tools
```

## [vpython](https://chromium.googlesource.com/infra/infra/+/master/doc/users/vpython.md)

depot_tools are trying to run **vpython** binaries (x86_64 ELF binaries), it could be bypassed by:
```
export VPYTHON_BYPASS="manually managed python not supported by chrome operations"
```

## [cipd](https://chromium.googlesource.com/chromium/src/+/master/docs/cipd.md)

as CIPD is written in **GoLang**, it cannot be easily compiled on e2k.

it should be enough to run x86_64 ELF binary via [RTC](https://www.altlinux.org/%D0%AD%D0%BB%D1%8C%D0%B1%D1%80%D1%83%D1%81/rtc) instead.

first off, create the following script named **rtcrun**:

```
#!/usr/bin/env bash

echo "RTC args: $@"
ARGS="$@"
/opt/mcst/rtc-4.0/bin/rtc_opt_rel_p1_x64_ob --path_prefix /opt/mcst/os_elbrus.4.0-rc4.x86_64/ -b $HOME -b /etc/passwd -b /etc/group -b /tmp -- /bin/bash -lc "$ARGS"
```

then, add it to the PATH:

```
export PATH=$PATH:$HOME/rtc
```

don't forget to chmod:

```
chmod +x $HOME/rtc/rtcrun
```

apply the following simple patch to the depot_tools:

```
diff --git a/cipd b/cipd
index 06fdbf29..13534c5c 100755
--- a/cipd
+++ b/cipd
@@ -58,6 +58,9 @@ if [ -z $ARCH ]; then
     *86)
       ARCH=386
       ;;
+    e2k)
+      ARCH=amd64
+      ;;
     mips*)
       # detect mips64le vs mips64.
       ARCH="${UNAME}"
@@ -202,7 +205,7 @@ function clean_bootstrap() {
 #
 # It is more efficient that redownloading the binary all the time.
 function self_update() {
-  "${CLIENT}" selfupdate -version-file "${VERSION_FILE}" -service-url "${CIPD_BACKEND}"
+  rtcrun "${CLIENT}" selfupdate -version-file "${VERSION_FILE}" -service-url "${CIPD_BACKEND}"
 }
 ```

## [gn](https://gn.googlesource.com/gn/)
 
it's required to use latest gn with [e2k support](https://gn-review.googlesource.com/c/gn/+/10980/2).

clone the repo:
```
git clone https://gn.googlesource.com/gn
```

build:
```
python build/gen.py
ninja -C out
```

add to the PATH (before the depot_tools!):
```
export PATH=$PATH:$HOME/gn/out
```

== [clang](https://chromium.googlesource.com/chromium/src/+/master/docs/clang.md) ==

Chromium ships with bundled clang. As there is no e2k clang yet, there is a trick to make Chromium build with MCST LCC instead.

first off, add the missing e2k toolchain definition:
```
diff --git a/build/toolchain/linux/BUILD.gn b/build/toolchain/linux/BUILD.gn
index bbd973225bdf..ce524a77d113 100644
--- a/build/toolchain/linux/BUILD.gn
+++ b/build/toolchain/linux/BUILD.gn
@@ -316,3 +316,27 @@ gcc_toolchain("mips64") {
     is_clang = false
   }
 }
+
+clang_toolchain("clang_e2k") {
+  toolchain_args = {
+    current_cpu = "e2k"
+    current_os = "linux"
+    is_clang = true
+  }
+}
```

then, go to the `third_party/llvm-build/Release+Asserts/bin` and change executables to use the wrapper instead:
```
mv clang clang.bak
mv clang++ clang++.bak
mv llvm-ar llvm-ar.bak
ln -s wrapper clang
ln -s wrapper clang++
ln -s /usr/bin/ar llvm-ar
```
and use the simple wrapper:
```
#!/usr/bin/env python

from __future__ import print_function
import fnmatch
import subprocess
import sys


def compare(lst1, lst2):
    if len(lst1) != len(lst2):
        return False
    for i in range(len(lst1)):
        if not fnmatch.fnmatch(lst1[i], lst2[i]):
            return False
    return True

def remove(args, elements):
    if isinstance(elements, str):
        elements = [elements]
    n = len(elements)
    c = len(args)
    for i in reversed(range(c - n + 1)):
        if compare(args[i:i + n], elements):
            del args[i:i + n]

args = sys.argv

exe = args[0]
args.pop(0)
is_cxx = 'clang++' in exe
exe = '/opt/mcst/bin/l++' if is_cxx else '/opt/mcst/bin/lcc'

sys.stdout.flush()


to_remove = [
    ['-Xclang', 'find-bad-constructs'],
    ['-Xclang', 'check-ipc'],
    "-fcolor-diagnostics",
    "-fcrash-diagnostics-dir=*",
    "-mllvm",
    "-instcombine-lower-dbg-declare=0",
    "-fcomplete-member-pointers",
    ['-Xclang', '-fdebug-compilation-dir', '-Xclang', '.'],
    ['-Xclang', '-add-plugin', '-Xclang', '-plugin-arg-find-bad-constructs'],
    "-no-canonical-prefixes",
    ['-Xclang', '-debug-info-kind=constructor'],
    "-gsplit-dwarf",
    "-ggnu-pubnames",
    "-ftrivial-auto-var-init=pattern",
    "-fno-trigraphs",
    "-Wthread-safety",
    "-Wextra-semi",
    "-Wheader-hygiene",
    "-Wstring-conversion",
    "-Wtautological-overlap-compare",
    "-Wbool-conversion",
    "-Wconstant-conversion",
    "-Wenum-conversion",
    "-Wliteral-conversion",
    "-Wnon-literal-null-conversion",
    "-Wobjc-literal-conversion",
    "-Wbitfield-enum-conversion",
    "-Wexit-time-destructors",
    "-Wglobal-constructors",
    "-Wconditional-uninitialized",
    "-Wextra-semi-stmt",
    "-Winconsistent-missing-destructor-override",
    "-Wnewline-eof",
    "-Wredundant-parens",
    "-Wreturn-std-move-in-c++11",
    "-Wshadow-field",
    "-Wtautological-type-limit-compare",
    "-Wundefined-reinterpret-cast",
    "-Wunneeded-internal-declaration",
    "-Wweak-template-vtables",
    "-Wrange-loop-analysis",
    "-Wshorten-64-to-32",
    "-Wdeprecated-copy",
    "-Wsuggest-destructor-override",
    "-D__DATE__=",
    "-D__TIME__=",
    "-D__TIMESTAMP__=",
    '-nostdinc++',
    '-isystem*/libc++/trunk/include',
    '-isystem*/libc++abi/trunk/include',
    "-Werror",
    "-Wl,--color-diagnostics",
    "-Wl,--no-call-graph-profile-sort",
    "-Wl,--gdb-index",
]

for i in to_remove:
    remove(args, i)

if is_cxx:
    remove(args, "-std=c11")

sys.stdout.flush()

args = [exe] + args

status = subprocess.call(args)
sys.exit(status)
```
MCST LCC doesn't support a lot of flags, so wrapper filters all them out.
