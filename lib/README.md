# Generated bindings live here

The Dart bindings are produced by UniFFI-Dart and are **not** checked into
source control. Run `scripts/generate_bindings.sh` to regenerate
`bdk.dart` before building or testing.

This placeholder file keeps the `lib/` directory present in the
repository so that `dart` tooling can resolve the package structure.

## Mobile build artifacts

Mobile consumers (Flutter/Dart) can build platform-specific binaries using the
scripts in `scripts/`.

### iOS (XCFramework)

1. Generate the required static libraries:
   ```bash
   ./scripts/generate_bindings.sh --target ios
   ```
2. Package them into an XCFramework:
   ```bash
   ./scripts/build-ios-xcframework.sh
   ```
   The framework is written to `ios/Release/bdkffi.xcframework/`. Keep it there to
   reuse across multiple apps, or direct the output into a Flutter project with the
   optional flag:

   ```bash
   ./scripts/build-ios-xcframework.sh --output bdk_demo/ios/ios
   ```

### Android (.so libraries)

1. Ensure `ANDROID_NDK_ROOT` points to an Android NDK r26c (or compatible) installation. For example:
   - macOS/Linux (adjust the NDK directory as needed):
     ```bash
     export ANDROID_NDK_ROOT="$HOME/Library/Android/sdk/ndk/29.0.14206865"
     ```
   - Windows PowerShell:
     ```powershell
     $Env:ANDROID_NDK_ROOT = "C:\\Users\\<you>\\AppData\\Local\\Android\\Sdk\\ndk\\29.0.14206865"
     ```
     (Use `setx ANDROID_NDK_ROOT <path>` if you want the variable persisted for future shells.)
2. Build the shared objects:
   ```bash
   ./scripts/generate_bindings.sh --target android
   ```
3. Stage the artifacts for inclusion in a Flutter project (make script executable or run with `bash`):
   ```bash
   chmod +x scripts/build-android.sh
   ./scripts/build-android.sh
   # or
   bash ./scripts/build-android.sh 
   ```

   By default the script stages artifacts in `android/libs/<abi>/libbdkffi.so` so the same
   slices can be reused across multiple apps; direct them into the demo’s `jniLibs`
   by passing `--output` as shown below. 

   ```bash
   chmod +x scripts/build-android.sh
   ./scripts/build-android.sh --output bdk_demo/android/app/src/main/jniLibs
   # or, without changing permissions
   bash ./scripts/build-android.sh --output bdk_demo/android/app/src/main/jniLibs

### Desktop tests (macOS/Linux)

- Run the base script without a target flag before executing `dart test`:
  ```bash
  ./scripts/generate_bindings.sh
  ```
  This regenerates the Dart bindings and drops the host dynamic library
  (`libbdkffi.dylib` on macOS, `libbdkffi.so` on Linux) in the project root.
  The generated loader expects those files when running in the VM, so tests fail if you only invoke the platform-specific targets.

### Verification

- On iOS, confirm the framework slices:
  ```bash
  lipo -info ios/Release/bdkffi.xcframework/ios-arm64/libbdkffi.a
  lipo -info ios/Release/bdkffi.xcframework/ios-arm64_x86_64-simulator/libbdkffi.a
  ```
- On Android, verify the shared objects:
  ```bash
  find android/libs -name "libbdkffi.so"
  ```

## Flutter demo (`bdk_demo/`)

Once the native artifacts are staged you can run the sample Flutter app to confirm
the bindings load end-to-end.

### Prerequisites

1. Generate bindings and host libraries:
   ```bash
   ./scripts/generate_bindings.sh
   ./scripts/generate_bindings.sh --target android
   ```
2. Stage Android shared objects into the app’s `jniLibs` (repeat per update):
   ```bash
   bash ./scripts/build-android.sh --output bdk_demo/android/app/src/main/jniLibs
   ```

### Run the app

```bash
cd bdk_demo
flutter pub get
flutter run       
```

The Android variant uses a small MethodChannel to discover where the system
actually deploys `libbdkffi.so` (some devices nest ABI folders under
`nativeLibraryDir`). That path is fed back into Dart so the generated loader in
`lib/bdk.dart` can `dlopen` the correct slice.

### What the demo verifies

- The native dynamic library loads on-device via the generated FFI loader.
- A BIP84 descriptor string is constructed through the Dart bindings and displayed to the UI.
- The UI badge switches between success and error states, so you immediately see
  if the bindings failed to load or threw during descriptor creation (delete the android artifacts and rerun the app to simulate a failure).

Use this screen as a smoke test after rebuilding bindings or regenerating artifacts; if it turns green and shows `Network: testnet` the demo is exercising the FFI surface successfully.
