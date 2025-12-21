# Setting Up CLion for Rust + C++ FFI Debugging

Here's how to configure CLion to debug your mixed Rust/C++ FFI project like `libquicr-rust`:

## Step 1: Install CLion and Rust Plugin

1. Download and install [CLion](https://www.jetbrains.com/clion/) if you haven't already
2. Open CLion and go to **Settings/Preferences** → **Plugins**
3. Search for **"Rust"** in the Marketplace tab
4. Install the **Rust** plugin and restart CLion

## Step 2: Open Your Project

1. In CLion, go to **File** → **Open**
2. Navigate to your `libquicr-rust` project directory and open it
3. CLion should detect both the `Cargo.toml` (Rust) and any C++ code in your project

## Step 3: Configure the Toolchain

1. Go to **Settings/Preferences** → **Build, Execution, Deployment** → **Toolchains**
2. Ensure you have a valid C++ toolchain configured (GCC, Clang, or MSVC)
3. Go to **Settings/Preferences** → **Languages & Frameworks** → **Rust**
4. Make sure the Rust toolchain path is correctly set

## Step 4: Ensure Debug Symbols in Your Build

Your `build.rs` should already be configured for debug symbols. Make sure you're building in debug mode (not release):

```shell script
cargo build
```


## Step 5: Create a Run/Debug Configuration

1. Go to **Run** → **Edit Configurations**
2. Click **+** and select **Cargo**
3. Configure it:
   - **Name**: Give it a descriptive name (e.g., "Debug FFI")
   - **Command**: `run` (or `test` if debugging tests)
   - **Working directory**: Your project root
4. Click **Apply** and **OK**

## Step 6: Set Breakpoints and Debug

1. Open your C++ file (e.g., in the `ffi/` or `vendor/` directory)
2. Click the gutter to set a breakpoint on an executable line - this should now work!
3. Open your Rust file and set breakpoints there too if needed
4. Click the **Debug** button (bug icon) or press `Shift` `F9`

## Troubleshooting Tips

- **Breakpoint not hit in C++**: Ensure the C++ code is actually being called and linked with debug symbols (`-g` flag)
- **Symbols not loading**: Check that your `build.rs` passes debug flags correctly to the C++ compiler (e.g., via `cc` crate)
- **Mixed debugging issues**: Make sure you're using LLDB as the debugger (preferred for Rust) - configure in **Settings** → **Build, Execution, Deployment** → **Debugger**

---

Would you like me to look at your `build.rs` file to verify the debug symbols are being set correctly for the C++ compilation?
