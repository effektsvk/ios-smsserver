# Building and installing libsmserver locally

This fork carries a patched `sendTyping:` (branch `typing-send`, package version `0.9.3-1`) that makes typing indicators actually transmit on iOS 16.7. The full story — the IMCore gates that were reverse-engineered from the phone's dyld cache, the install flow, the trust-cache caveat, verification — lives in the bot repo: `flight-finder-bot/docs/phone-tweak.md` and `flight-finder-bot/docs/imessage-transport.md` ("Typing and tapbacks on iOS 16"). This file is just the build recipe.

The GitHub workflow is currently broken (retired `macos-14` runner, then platform-cancelled runs; see ssut#2). Build on Linux instead — it takes under a minute once the one-time setup is done:

- theos (no toolchain inside needed), `theos/sdks` `iPhoneOS16.5.sdk` in `$THEOS/sdks/`,
- `/usr/bin/clang{,++}` symlinked as `$THEOS/toolchain/linux/iphone/bin/clang{,++}`, `ld64.lld` from the `lld` package,
- [ProcursusTeam/ldid](https://github.com/ProcursusTeam/ldid) built and installed.

```sh
export THEOS=<path to theos>
cd libsmserver
make clean && make package ARCHS=arm64 THEOS_PACKAGE_SCHEME=rootless FINALPACKAGE=1
#   the compile works; the theos link step fails on Ubuntu (its clang is patched
#   to refuse -fuse-ld). Link the object directly:
/usr/bin/ld64.lld -dylib -arch arm64 -platform_version ios 15.0 16.5 \
  -syslibroot "$THEOS/sdks/iPhoneOS16.5.sdk" \
  -L"$THEOS/vendor/lib/iphone/rootless" \
  -o .theos/obj/arm64/libsmserver.dylib .theos/obj/arm64/Tweak.xm.*.o \
  -lobjc -lc++ -framework Foundation -framework CoreFoundation -framework UIKit -lroot_oldabi \
  -rpath /var/jb/Library/Frameworks -rpath /var/jb/usr/lib \
  -rpath '@loader_path/.jbroot/Library/Frameworks' -rpath '@loader_path/.jbroot/usr/lib' \
  -install_name '@rpath/libsmserver.dylib' -undefined dynamic_lookup
ldid -S .theos/obj/arm64/libsmserver.dylib
S=/tmp/libsmserver-stage; rm -rf $S
mkdir -p $S/var/jb/Library/MobileSubstrate/DynamicLibraries $S/DEBIAN
install -m 755 .theos/obj/arm64/libsmserver.dylib $S/var/jb/Library/MobileSubstrate/DynamicLibraries/
install -m 644 libsmserver.plist $S/var/jb/Library/MobileSubstrate/DynamicLibraries/
install -m 644 control $S/DEBIAN/ && install -m 755 layout/DEBIAN/postinst $S/DEBIAN/
dpkg-deb --root-owner-group -b $S ../com.janshai.libsmserver_$(dpkg-deb -f $S/DEBIAN/control Version)_iphoneos-arm64.deb
```

`-undefined dynamic_lookup` lets the logos hooks resolve at load time; `-lroot_oldabi` is theos's rootless shim; `--root-owner-group` keeps the archive root-owned. Bump `libsmserver/control`'s `Version` on every change so dpkg treats it as an upgrade.

Install on the phone over SSH with `package/device-install.sh` (Sileo's dpkg trigger adds the new CDHash to the Dopamine trust cache even though the script's own parse prints `added=0`), then relaunch the SMServer app and start its server — the postinst respring kills the app as a background app.
