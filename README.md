# mbed-scons
SCons helpers for configuring mbed targets with gcc-arm.

## Usage
### SConscript-mbed
The `SConscript-mbed` file contains helper functions to set up a environment
that can build mbed targets with gcc-arm.

Example mbed StaticLibrary build for the NUCLEO_L432KC target:

```
Export('env')
SConscript('mbed-scons/SConscript-mbed')

env.ConfigureMbedTarget('NUCLEO_L432KC', 'mbed/hal/targets.json')
mbed_paths = env.GetMbedSourceDirectories('mbed/hal')
env.Append(CPPPATH=mbed_paths)
env['MBED_LINKSCRIPT'] = env.GetMbedLinkscript(mbed_paths)
mbed_lib = env.StaticLibrary('mbed', env.GetMbedSources(mbed_paths))
env.Prepend(LIBS = mbed_lib)
```

The available environment actions are:
- `ConfigureMbedTarget`: configures `env` based on the target name (first
  argument) found in mbed's `targets.json` file (second argument).
- `GetMbedSourceDirectories`: given a configured `env` (above), returns a
  list of all source directories for the configured target.
- `GetMbedSources`: from a list of all source directories, returns a list of
  all source files (to build into a StaticLibrary, for example).
- `GetMbedLinkscript`: from a list of all source directories, returns the
  linker script to use. 
- `Binary`: translates `.elf` files into `.bin` files that can be directly
  flashed onto a mbed mass-storage device. The only input is the source file,
  and it will generate a `.bin` file of the same name.
- `SmybolsSize`: from a `.elf` file, generates a `.nm_size.txt` file which
  contains a list of symbols in the executable, sorted by size. Useful for
  determining what is bloating your binary when optimizing for space.
- `Objdump`: from a `.elf` file, generates a `.dump.txt` file which shows a
  disassembled view of the compiled program.

### Quickstart
To actually build mbed firmware using this system, you will need to:
1. Export your environment (as `env`) so the included scripts can do their magic:

  ```
  Export('env')
  ```
  
1. Set the correct compilers.

  ```
  SConscript('mbed-scons/SConscript-env-gcc-arm')
  ```

1. Import `SConscript-mbed`

  ```
  SConscript('mbed-scons/SConscript-mbed')
  ```

1. Configure the environment for your target (the Nucleo L432KC, in this example).

  ```
  env.ConfigureMbedTarget('NUCLEO_L432KC', 'mbed/hal/targets.json')
  ```

1. Find the source directories for your target, and add them to `CPPPATH`.

  ```
  mbed_paths = env.GetMbedSourceDirectories('mbed/hal')
  env.Append(CPPPATH=mbed_paths)
  ```

1. Set the linker script.

  ```
  env['MBED_LINKSCRIPT'] = env.GetMbedLinkscript(mbed_paths)
  ```

1. Build the static library and add it to `LIBS`.

  ```
  mbed_lib = env.StaticLibrary('mbed', env.GetMbedSources(mbed_paths))
  env.Prepend(LIBS = mbed_lib)
  ```

  `Prepend` is used since GCC's linker discards unused parts of libraries as
  they are searched, so the "top-level" library must come first. If libraries
  are specified in dependency order (with "base" libraries first), then using
  `Prepend` consistently can avoid nastiness like `--Wl,--whole-archive`.
  
  However, `--Wl,--whole-archive` may be necessary for targets that use weak
  symbols.

1. Optionally, turn on compiler optimizations:
  - optimizing for size:

    ```
    env.Append(CCFLAGS = '-Os')
    ```

  - or optimizing for speed:

    ```
    env.Append(CCFLAGS = '-O3')
    ```

1. Build your program (`.elf`):

  ```
  myprog = env.Program('myprog', [list_of_your_sources])
  ```

1. Optionally, also make a `.bin` file:

  ```
  env.Binary(myprog)
  ```

### Advanced: Multi-target builds
It is also possible to set up multiple environments, each building for a
different target platform. This is useful if you have a single project with
components on different targets, or cross-platform code that is meant to run
on multiple targets.

If using a shared mbed library source directory, make sure the mbed static
libraries are built in their own VariantDirs, so that the same files from
different platforms don't conflict. This can be accomplished by defining the
mbed library for each environment in its own SConscript file, then calling them
in their own `variant_dir`.

## Misc
### Troubleshooting
- You may need to `.srcnode()` all the `CPPPATH`s if using in a SConscript
  invoked with `duplicate=0`.
  
  ```
  mbed_paths = env.GetMbedSourceDirectories('mbed/hal')
  env.Append(CPPPATH=[x.srcnode() for x in mbed_paths])
  ```
  
- Some code may require using `nosys.specs` to compile. In that case, add this
  linker flag:

  ```
  env.Append(LINKFLAGS='--specs=nosys.specs')
  ```

- C++11 is supported by GCC but not the default in the environments here (which
  try to be consistent with mbed generated projects). The default `gnu++98` can
  be overridden by adding this compiler flag *after* importing the
  target-specific environment:

  ```
  env.Append(CXXFLAGS='-std=c++11')
  ```
