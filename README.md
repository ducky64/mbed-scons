# mbed-scons
SCons rules for compiling a mbed library with gcc-arm

## Usage
### SConscript-mbed
The `SConscript-mbed` file contains rules for a mbed-like library, where
the path to the target (like `TARGET_Freescale`, `TARGET_KLXX`) is defined in
the environment variable `MBED_TARGET`. The build system will automatically
recurse the mbed source directories accordingly. An example mbed
`StaticLibrary` build would be (assuming you submoduled this repository into
`mbed-scons` and the `mbedmicro/mbed` library into `mbed`:

```
Export('env')
SConscript('mbed-scons/SConscript-mbed')
mbed_lib = env.MbedLikeLibrary('mbed', 'mbed/libraries/mbed/',
                               ['api/', 'common/', 'hal/', 'targets/cmsis/', 'targets/hal/'])
```

where the arguments are (library name, mbed library root, and list of
directories from mbed library root to include headers and sources.

### Predefined target-specific environments
Environments for some common platforms (like the FRDM-KL25Z) are included:
- `SConscript-mbed-env-kl05z`
- `SConscript-mbed-env-kl25z`

These add the MBED\_TARGET, CPPDEFINES, and MBED\_CPU environment variables for
a particular target. The linker script needs to be specified separately.

### Quickstart
To actually build mbed firmware using this system, you will need to:
1. Set the correct compilers:

  ```
  env['AR'] = 'arm-none-eabi-ar'
  env['AS'] = 'arm-none-eabi-as'
  env['CC'] = 'arm-none-eabi-gcc'
  env['CXX'] = 'arm-none-eabi-g++'
  env['LINK'] = 'arm-none-eabi-g++'
  env['RANLIB'] = 'arm-none-eabi-ranlib'
  env['OBJCOPY'] = 'arm-none-eabi-objcopy'
  env['OBJDUMP'] = 'arm-none-eabi-objdump'
  env['PROGSUFFIX'] = '.elf'
  ```

1. Export your environment (as `env`) so the included scripts can do their magic:

  ```
  Export('env')
  ```

1. Import `SConscript-mbed`

  ```
  SConscript('mbed-scons/SConscript-mbed')
  ```

1. Import the target-specific environment, or write your own.

  ```
  SConscript('mbed-scons/SConscript-mbed-env-kl25z')
  ```

1. Add the linker script to `LINKFLAGS`:

  ```
  env.Append(LINKFLAGS = [
    '-Tmbed/libraries/mbed/targets/cmsis/TARGET_Freescale/TARGET_KLXX/TARGET_KL25Z/TOOLCHAIN_GCC_ARM/MKL25Z4.ld',
])
  ```

1. Build the mbed static library and add it to the global libraries:

  ```
  mbed_lib = env.MbedLikeLibrary('mbed', 'mbed/libraries/mbed/',
                                 ['api/', 'common/', 'hal/', 'targets/cmsis/', 'targets/hal/'])
  env.Append(LIBS=mbed_lib)
  ```

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

1. Optionally, also make a `.bin` version which can be flashed using the mbed MSD:

  ```
  env.Objcopy(myprog)
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
in their own `variant_dir`. For example:

`SConscript-env-kl25z`:
```
Import('env')

SConscript('mbed-scons/SConscript-mbed-env-kl25z', exports='env')

mbed_lib = env.MbedLikeLibrary('mbed', 'libs/mbed/libraries/mbed/',
                               ['api/', 'common/', 'hal/', 'targets/cmsis/', 'targets/hal/'])

env.Append(LIBS=mbed_lib)

env.Append(LINKFLAGS = [
  '-Tmbed/libraries/mbed/targets/cmsis/TARGET_Freescale/TARGET_KLXX/TARGET_KL25Z/TOOLCHAIN_GCC_ARM/MKL25Z4.ld',
])
```

`SConscript-env-kl05z`:
Similar as above

`Sconstruct` (top-level script):
```
env_orig = env

env = env_orig.Clone()
SConscript('SConscript-env-kl25z', variant_dir='build/kl25z', exports='env')
env_kl25z_bl = env

env = env_orig.Clone()
SConscript('SConscript-env-kl05z', variant_dir='build/kl05z', exports='env')
env_kl05z_bl = env
```
