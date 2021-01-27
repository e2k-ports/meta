basically, we base on the [official Linux instructions](https://chromium.googlesource.com/chromium/src/+/master/docs/linux/build_instructions.md)
with some changes specific for e2k.

# depot_tools

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
 
 
