# CycloneDDS Cross-Compilation Guide for Cortex-A53 (AArch64)

This guide will walk you through the process of cross-compiling CycloneDDS for Cortex-A53 hardware platform using the ARM GNU toolchain.

## Prerequisites

Before starting, ensure you have the following installed:

- **CMake** (version 3.16 or later) - Already installed ✓
- **ARM GNU Toolchain** - Already installed at `C:\compiler\aarch64-none-linux-gnu` ✓
- **Git** (optional, for cloning if you don't have the source)
- **PowerShell** or Command Prompt

## Project Overview

CycloneDDS is a high-performance implementation of the OMG DDS specification. It includes:
- Core DDS library (`ddsc`)
- IDL compiler (`idlc`) for generating C code from IDL files
- Performance testing tool (`ddsperf`)
- Various examples and tests

## Step 1: Verify Your Environment

First, let's verify that your cross-compiler is accessible:

```powershell
# Test if the compiler is accessible
C:\compiler\aarch64-none-linux-gnu\bin\aarch64-none-linux-gnu-gcc.exe --version
```

You should see output indicating the GCC version and target architecture.

## Step 2: Configure Cross-Compilation Settings

For this setup, we'll configure the cross-compilation directly via command-line parameters instead of using a separate toolchain file. This approach gives you direct control over each setting and makes the configuration more explicit.

## Step 3: Prepare Build Directories

Create separate build directories for host and target builds:

```powershell
# Navigate to your CycloneDDS directory
cd d:\11_web\cyclonedds

# Create build directories
mkdir build-host
mkdir build-aarch64
```

## Step 4: Build Host Tools First

CycloneDDS requires some tools (like the IDL compiler) to be built for the host system first, as they are needed during the cross-compilation process.

```powershell
# Build host tools
cd build-host
cmake -G "Visual Studio 16 2019" -A x64 -DBUILD_EXAMPLES=OFF -DBUILD_TESTING=OFF ..
cmake --build . --config Release
cmake --install . --prefix ../install-host
cd ..
```

**Note:** You can use different build systems according to your preference:

**Option 1: Visual Studio (if available):**
- Visual Studio 2017: `"Visual Studio 15 2017"`
- Visual Studio 2019: `"Visual Studio 16 2019"`
- Visual Studio 2022: `"Visual Studio 17 2022"`

**Option 2: MinGW Makefiles (recommended for your setup):**

```powershell
cd build-host
cmake -G "MinGW Makefiles" -DCMAKE_MAKE_PROGRAM=mingw32-make.exe -DCMAKE_BUILD_TYPE=Release -DBUILD_EXAMPLES=OFF -DBUILD_TESTING=OFF ..
cmake --build .
cmake --install . --prefix ../install-host
cd ..
```

**Option 3: Ninja (alternative):**

```powershell
cd build-host
cmake -G "Ninja" -DCMAKE_BUILD_TYPE=Release -DBUILD_EXAMPLES=OFF -DBUILD_TESTING=OFF ..
cmake --build .
cmake --install . --prefix ../install-host
cd ..
```

## Step 5: Cross-Compile for AArch64

Now perform the actual cross-compilation for your target platform:

```powershell
cd build-aarch64

# Configure with cross-compilation using MinGW Makefiles
cmake -G "MinGW Makefiles" ^
  -DCMAKE_MAKE_PROGRAM=mingw32-make.exe ^
  -DCMAKE_C_COMPILER=C:/compiler/aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-gcc.exe ^
  -DCMAKE_CXX_COMPILER=C:/compiler/aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-g++.exe ^
  -DCMAKE_SYSTEM_NAME=Linux ^
  -DCMAKE_SYSTEM_PROCESSOR=aarch64 ^
  -DBUILD_TESTING=ON ^
  ..

# Build the project
cmake --build .

# Install to a target directory
cmake --install . --prefix ../install-aarch64
```



### Configuration Options Explanation:

- `CMAKE_C_COMPILER`: Specifies the cross-compiler for C files
- `CMAKE_CXX_COMPILER`: Specifies the cross-compiler for C++ files
- `CMAKE_SYSTEM_NAME=Linux`: Tells CMake we're targeting Linux
- `CMAKE_SYSTEM_PROCESSOR=aarch64`: Specifies the target processor architecture
- `BUILD_TESTS=ON`: Enable building of test suite

## Step 6: Verify the Build

After successful compilation, verify your build:

```powershell
# Check the generated libraries and executables
ls install-aarch64\lib
ls install-aarch64\bin

# Verify the target architecture
file install-aarch64\lib\libddsc.so
```

The output should indicate AArch64 architecture.

## Step 7: Optional Optimizations

### For Static Libraries (if needed):

```powershell
# Configure for static build (modify your base command by adding):
cmake -G "MinGW Makefiles" ^
  -DCMAKE_MAKE_PROGRAM=mingw32-make.exe ^
  -DCMAKE_C_COMPILER=C:/compiler/aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-gcc.exe ^
  -DCMAKE_CXX_COMPILER=C:/compiler/aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-g++.exe ^
  -DCMAKE_SYSTEM_NAME=Linux ^
  -DCMAKE_SYSTEM_PROCESSOR=aarch64 ^
  -DBUILD_TESTING=ON ^
  -DBUILD_SHARED_LIBS=OFF ^
  ..
```

### For Debug Build:

```powershell
# Configure for debug build (modify your base command by adding):
cmake -G "MinGW Makefiles" ^
  -DCMAKE_MAKE_PROGRAM=mingw32-make.exe ^
  -DCMAKE_C_COMPILER=C:/compiler/aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-gcc.exe ^
  -DCMAKE_CXX_COMPILER=C:/compiler/aarch64-none-linux-gnu/bin/aarch64-none-linux-gnu-g++.exe ^
  -DCMAKE_SYSTEM_NAME=Linux ^
  -DCMAKE_SYSTEM_PROCESSOR=aarch64 ^
  -DCMAKE_BUILD_TYPE=Debug ^
  -DBUILD_TESTING=ON ^
  ..
```

## Step 8: Deploy to Target System

Copy the following files to your Cortex-A53 target system:

1. **Libraries**: `install-aarch64/lib/` - Copy all `.so` files
2. **Executables**: `install-aarch64/bin/` - Copy example programs and tools
3. **Headers**: `install-aarch64/include/` - For developing applications on target

## Troubleshooting

### Common Issues and Solutions:

1. **Compiler not found:**
   - Verify the path in your toolchain file matches your installation
   - Ensure `.exe` extension is included on Windows

2. **Missing host tools:**
   - Make sure you built and installed the host tools first
   - Verify `CMAKE_PREFIX_PATH` points to the correct host installation

3. **Linker errors:**
   - Check that your target system has compatible glibc version
   - Consider using static linking if deployment is problematic

4. **Performance issues:**
   - Use Release build type for production
   - Enable Cortex-A53 specific optimizations in the toolchain file

## Testing Your Build

Once deployed to your Cortex-A53 system, you can test with the provided examples:

```bash
# On your target system (Cortex-A53 Linux)
# Test with HelloWorld example
./RoundtripPong &
./RoundtripPing 0 0 0

# Test with performance tool
./ddsperf ping
```
Study the examples in repo, use below command to build the examples, for Windows End.
```
cmake -G "MinGW Makefiles" -DCMAKE_INSTALL_PREFIX="../install" -DBUILD_EXAMPLES=ON ..
```
## add build directory
```
cmake --install . --prefix ../install-aarch64
```

## Next Steps

1. **Develop Applications**: Use the installed headers and libraries to develop your own DDS applications
2. **Configuration**: Create custom CycloneDDS XML configuration files for your specific network setup
3. **Integration**: Integrate CycloneDDS into your existing application framework

## References

- [CycloneDDS Documentation](https://cyclonedx.io/docs)
- [CycloneDDS GitHub Repository](https://github.com/eclipse-cyclonedds/cyclonedds)
- [ARM GNU Toolchain Documentation](https://developer.arm.com/tools-and-software/open-source-software/developer-tools/gnu-toolchain)

---

**Note:** This guide assumes your Cortex-A53 system runs Linux. If you're targeting a different OS (like QNX or VxWorks), you'll need to modify the toolchain file accordingly and refer to the existing examples in the `ports/` directory.
