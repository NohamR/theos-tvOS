# tvOS Theos Build & Installation Fixes

To successfully build and install tweaks for tvOS on modern firmware (up to Apple TV 18.3) using Theos (particularly under the `rootless` scheme), several modifications to Theos and the project files were required. 

Here is a summary of the changes made:

## 1. Theos Framework Search Path Fix
**File:** `/Users/noham/theos/makefiles/common.mk`

**Issue:** When building with `THEOS_PACKAGE_SCHEME=rootless`, Theos was omitting the base `vendor/lib` fallback from `_THEOS_INTERNAL_SEARCHPATHS` if the scheme-specific directory did not exist.
**Fix:** We appended the base `vendor/lib` and `lib` directories into the inclusion list so that the linker could safely fall back and find the necessary frameworks if scheme-specific ones are missing.

```bash
diff --git a/makefiles/common.mk b/Users/noham/theos/makefiles/common.mk
index 854ecf6..8d22a39 100644
--- a/makefiles/common.mk
+++ b/Users/noham/theos/makefiles/common.mk
@@ -191,7 +191,11 @@ _THEOS_INTERNAL_SEARCHPATHS += \
 else
 _THEOS_INTERNAL_SEARCHPATHS += \
 	$(THEOS_VENDOR_LIBRARY_PATH)/$(THEOS_TARGET_NAME)/$(THEOS_PACKAGE_SCHEME) \
-	$(THEOS_LIBRARY_PATH)/$(THEOS_TARGET_NAME)/$(THEOS_PACKAGE_SCHEME)
+        $(THEOS_VENDOR_LIBRARY_PATH) \
+        $(THEOS_LIBRARY_PATH)/$(THEOS_TARGET_NAME)/$(THEOS_PACKAGE_SCHEME) \
+        $(THEOS_LIBRARY_PATH) \
+        $(THEOS_TARGET_LIBRARY_PATH)/$(THEOS_PACKAGE_SCHEME) \
+        $(THEOS_TARGET_LIBRARY_PATH)
 endif
 _THEOS_INTERNAL_LDFLAGS = $(foreach path,$(_THEOS_INTERNAL_SEARCHPATHS),$(if $(call __exists,$(path)),-L$(path) -F$(path)))
```

## 2. CydiaSubstrate Architecture & Platform Mismatch
**Location:** `/Users/noham/theos/vendor/lib/appletv/rootless/CydiaSubstrate.framework`

**Issue:** The compilation failed with a linker error because `CydiaSubstrate.tbd` internally declared its platform as `ios`, which broke tvOS static linking. 
**Fix:** 
1. Created the directory `vendor/lib/appletv/rootless/CydiaSubstrate.framework`.
2. Copied the headers from the base CydiaSubstrate framework.
3. Cloned `CydiaSubstrate.tbd` and replaced `platform: ios` with `platform: tvos`.

```bash
theos/vendor/lib/appletv/
└── rootless
    └── CydiaSubstrate.framework
        ├── CydiaSubstrate.tbd
        └── Headers
            └── CydiaSubstrate.h
```

`theos/vendor/lib/appletv/rootless/CydiaSubstrate.framework/CydiaSubstrate.tbd`:
```
---
archs:           [ armv7, armv7s, arm64, arm64e ]
platform:        tvos
install-name:    /Library/Frameworks/CydiaSubstrate.framework/CydiaSubstrate
current-version: 0.0.0
compatibility-version: 0.0.0
exports:
  - archs:            [ armv7, armv7s, arm64, arm64e ]
    symbols:          [ _MSCloseImage, _MSDebug, _MSFindAddress, _MSFindSymbol,
                        _MSGetImageByName, _MSHookClassPair, _MSHookFunction,
                        _MSHookMemory, _MSHookMessageEx, _MSImageAddress,
                        _MSMapImage ]
...
```


## 3. Package Architecture Hardcode Fix
**File:** `/Users/noham/theos/vendor/mod/rootless/package/deb.mk`

**Issue:** The rootless module explicitly forced all packages to build with the `iphoneos-arm64` architecture, overriding the `control` file's target variables. 
**Fix:** Modified the override condition from an arbitrary rewrite to only apply when the detected architecture is `iphoneos-arm` (targeting old 32-bit platforms), allowing tvOS architecture overrides to pass unaffected.

```bash
diff --git a/vendor/mod/rootless/package/deb.mk b/Users/noham/theos/vendor/mod/rootless/package/deb.mk
index fa6ee66..5905018 100644
--- a/vendor/mod/rootless/package/deb.mk
+++ b/Users/noham/theos/vendor/mod/rootless/package/deb.mk
@@ -1,3 +1,3 @@
-ifneq ($(THEOS_PACKAGE_ARCH),iphoneos-arm64)
+ifeq ($(THEOS_PACKAGE_ARCH),iphoneos-arm)
 	THEOS_PACKAGE_ARCH := iphoneos-arm64
 endif
```