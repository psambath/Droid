# Introduction #

This document describes how to build the system from scratch, apply the smart card related patches and flash a development device with the new images. It is essential that the device supports the necessary RIL and baseband modifications to allow APDU access to the SIM card, otherwise SIM access will not work!

## Getting the Android sources ##

A detailed description how to set up the local work environment can be found  [here](http://source.android.com/source/building.html).
```
$ mkdir seek-for-android && cd seek-for-android
$ repo init -u https://android.googlesource.com/platform/manifest -b android-5.0.0_r3.0.1
$ repo sync
```
After a successful sync it is recommended to build the Android platform once before applying the SEEK patch files to ensure the source code compiles properly.


## Patching the sources ##
Apply the necessary patches in the root directory of the Android source tree to enable SmartCard API support.

Please make sure which patch files should be applied as the different use cases (SmartCard API, SCWS/BIP, PC/SC support) are separated in different download packages. Do not apply patches multiple times!

#### SmartCard API support ####
Download the [smartcard-api-4\_0\_0.tar.gz](https://drive.google.com/file/d/0B63jMJOYc2l3SjZXSThtMkprNEk/view) patch and extract the content:
  * `smartcard-api.patch` - patch the Java sources for SmartCard API support - always required
  * `uicc.patch` - patch the Android Telephony framework with the required UICC methods and for the SmartCard API UiccTerminal support within the emulator - for emulator builds only
  * `emulator.patch` - patch the qemu module to support a UICC connected through the PC/SC host interface - for emulator builds only

#### BIP support - _currently not maintained_ ####
  * Patch the sources according to [BIP Extensions](https://github.com/sunyer/seek-for-android/wiki/BIP_Extensions) - only for BIP support

#### PC/SC support - _currently not maintained_ ####
  * Patch the sources according to [PCSC Support](https://github.com/sunyer/seek-for-android/wiki/PCSCSmartCardServiceIntro) - only for (native) Android applications requesting the PC/SC interface.

#### Apply the patches ####
```
$ cd <ANDROID_ROOT_DIR>
$ patch -p1 < <path_to_my_patches>/smartcard-api.patch
$ patch -p1 < <path_to_my_patches>/uicc.patch
$ patch -p1 < <path_to_my_patches>/emulator.patch
```

After applying the smartcard-api.patch patch the android source tree will contain the following:
  *  SmartCard Service source files in `packages/apps/SmartCardService`:
    * `src` - contains the sources of the SmartCard API Service
    * `openmobileapi` - contains the Open Mobile API shared library
    * `common` - contains classes that are shared in both projects
  * SIM Terminal source files in `packages/services/UiccTerminal`
  * eSE Terminal source files in `packages/services/eSETerminal`
  * SD Terminal source files in `packages/services/AssdTerminal`
    * `jni` - contains the native sources for ASSD support

In addition, the smartcard-api.patch extends `build/target/product/core.mk` to include the the SmartcardService and the 3 terminals in the build process.

Also, the "org.simalliance.openmobileapi.SYSTEM_TERMINAL" and "org.simalliance.openmobileapi.BIND_TERMINAL" permissions are declared in `frameworks/base/core/res/AndroidManifest.xml`.

If the optional `uicc.patch` is not applied, the SmartCard API will not compile as the `UiccTerminal` project requires those changes in the Android Telephony framework. In such case, there are two options:
  * Remove the file `UiccTerminal` project and any reference to it in `build/target/product/core.mk` -- this will remove support for the SIM card.
  * Fix the UiccTerminal. Basically, this can be done by removing any call to `mTelephonyManager.iccOpenLogicalChannel(aid, p2)` and `mTelephonyManager.iccGetAtr()`. You can throw an `UnsupportedOperationException` instead.

Update `current.xml` as the system needs to known the new IDs - required only once:
```
$ make update-api
```

## Alternative: clone from our repo! ##
With the migration to GitHub we've set up the Git repositories "the Android way" so they can be easily used with the `repo` tool.
You can download the Android code including the Smartcard API by using our manifest:
```
$ mkdir seek-for-android && cd seek-for-android
$ repo init -u https://github.com/seek-for-android/platform_manifest.git -b scapi-4.0.0
$ repo sync
```
This manifest is provided as a reference so third parties can:
* Integrate our projects into their manifests, and
* Collaborate to the SEEK projects by forking them and creating pull requests.

## Extract the vendor specific libraries ##
Development devices will not support all hardware modules without proprietary (closed-source) libraries. Make sure to have downloaded the correct [vendor specific libraries](https://developers.google.com/android/nexus/drivers) and extracted the files accordingly, e.g.
```
$ cd <ANDROID_ROOT_DIR>
$ ./extract-lge-mako.sh
$ ./extract-qcom-mako.sh
$ ./extract-broadcom-mako.sh
```
and follow the steps in the shell scripts.

## Optional: _embedded SE support_ - _currently not maintained_ ##
Embedded SE access on Ice Cream Sandwich (and above) is protected with the Android permission `com.android.nfc.permission.NFCEE_ADMIN` in addition to the client whitelist file `/system/etc/nfcee_access.xml`.
This file claims to include _signatures_ of applications that can retrieve the NFCEE\_ADMIN permission. However, it seems to be the signer  certificate chain instead of the application signature.

The content of `nfcee_access.xml` can be created with:
```
$ cd <ANDROID_ROOT_DIR>
$ tail -n+2 build/target/product/security/platform.x509.pem | head -n-1 | base64 -d | hexdump
```
Any system integrator who is including the SmartCard API in his own build system needs to adapt the access control file located under `device/<vendor>/<product>/nfcee_access.xml` with their own certificate to grant embedded SE access.

## Building the system ##
Compile the sources for the target device by either using `lunch` or setting the environment variables manually. Replace `maguro` with `grouper`, `manta` or `mako` depending on the actual development device:
```
$ cd <ANDROID_ROOT_DIR>
$ . build/envsetup.sh 
$ lunch full_mako-eng
============================================
PLATFORM_VERSION_CODENAME=REL
PLATFORM_VERSION=4.4
TARGET_PRODUCT=full_mako
TARGET_BUILD_VARIANT=eng
TARGET_BUILD_TYPE=release
TARGET_BUILD_APPS=
TARGET_ARCH=arm
TARGET_ARCH_VARIANT=armv7-a-neon
TARGET_CPU_VARIANT=krait
HOST_ARCH=x86
HOST_OS=linux
HOST_OS_EXTRA=Linux-3.4.9-gentoo-x86_64-Intel-R-_Xeon-R-_CPU_E5-2687W_0_@_3.10GHz-with-gentoo-2.2
HOST_BUILD_TYPE=release
BUILD_ID=JSS15Q
OUT_DIR=out
============================================
```
Finally, compile the system with
```
$ make -jXX
```
where XX is the number of parallel jobs and have a (quick) break.

**Note 1:** Using `lunch` without parameters provides a list of build variants with _userdebug_ enabled, it is recommended to use _eng_ instead for adb root access without su.

**Note 2:** The Internet might provide more up-to-date information about how to build the system for a real devices, especially when non SmartCard API related problems arise.

## Flashing the device ##
**Note** You'll loose all data on the device if you proceed - ALL data!

To flash the device, execute:
```
$ ./out/host/linux-x86/bin/adb reboot bootloader
$ ANDROID_PRODUCT_OUT=out/target/product/mako ./out/host/linux-x86/bin/fastboot -w flashall
```
Alternatively, flash each partition manually:
```
$ ./out/host/linux-x86/bin/fastboot erase userdata
$ ./out/host/linux-x86/bin/fastboot erase cache
$ ./out/host/linux-x86/bin/fastboot flash boot out/target/product/maguro/boot.img
$ ./out/host/linux-x86/bin/fastboot flash system out/target/product/maguro/system.img
$ ./out/host/linux-x86/bin/fastboot flash userdata out/target/product/maguro/userdata.img
```
Again, replace `mako` with `grouper`, `manta` or `maguro` depending on the development device. Skip flashing the boot partition if you don't want to replace the existing kernel and ramdisk by executing `fastboot boot boot.img` as the last command.

Reboot into Android with:
```
$ ./out/host/linux-x86/bin/fastboot reboot
```

## Deprecated: _Building the SDK_ ##
Since SmartCard API provided as shared library it is not required to rebuild the SDK but just include the org.simalliance.openmobileapi shared library in the Android project as  described in [Using SmartCard API](UsingSmartCardAPI).

If required, the SDK for Linux or Mac-OS X can still be build with
```
$ make PRODUCT-sdk-sdk -j32
```
The new SDK is located under `out/host/linux-x86/sdk` if required.

## Generating the documentation ##
Execute
```
$ make docs
```
to find the offline documentation under `out/target/common/docs/offline-sdk/reference` or use the [ApiDoc](https://github.com/sunyer/Seek-for-Android/tree/master/doc/org/simalliance/openmobileapi) instead.

## Continue ##
with the instructions as described in section [EmulatorExtension](EmulatorExtension) and/or [UsingSmartCardAPI](UsingSmartCardAPI)