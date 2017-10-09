# 移植fftw-3.3.6到Android上

## 0x00 源码的选择
这里有个坑，如果是直接在github上下载的代码是不能直接用的，因为这个库有很多自动生成的代码，需要另外的工具来生成，[具体请见github仓库的README.md](https://github.com/FFTW/fftw3/blob/master/README.md)。之前我直接用这个代码来编译，总是会出现一些莫名其妙的错误，找不到某些函数符号，然后一直找不到解决办法，偶然才看到GitHub的readme才发现是怎么回事。

如何生成代码不在本文的讨论范围。为了降低难度我们直接使用官网下载的最新稳定版，可以直接编译的。

到[官网](http://fftw.org/download.html)下载最新稳定版备用，我的是3.3.6

## 0x01 使用autotools系列工具
这个的话最简单，官网我没找到相关教程，google了一下发现了[这篇文章][1]，于是依葫芦画瓢。

1. 设置环境变量ANDROID_NDK指向NDK的位置（一般搞Android开发的都有了就不需要这个步骤了） 
2. 在源码根目录新建一个文件build_android.sh，文件里面内容如下:

```
#!/bin/sh
# FourierTest/build.sh
# Compiles fftw3 for Android
# Make sure you have ANDROID_NDK defined in .bashrc or .bash_profile

INSTALL_DIR="`pwd`/build/android"
SRC_DIR=.

cd $SRC_DIR

export
PATH="$ANDROID_NDK/toolchains/arm-linux-androideabi-4.9/prebuilt/linux-x86_64/bin/:$PATH"
export SYS_ROOT="$ANDROID_NDK/platforms/android-16/arch-arm/"
export CC="arm-linux-androideabi-gcc --sysroot=$SYS_ROOT"
export LD="arm-linux-androideabi-ld"
export AR="arm-linux-androideabi-ar"
export RANLIB="arm-linux-androideabi-ranlib"
export STRIP="arm-linux-androideabi-strip"

mkdir -p $INSTALL_DIR
./configure --host=arm-eabi --prefix=$INSTALL_DIR LIBS="-lc -lgcc"

make
make install

exit 0
```

然后直接运行./build_android.sh就好了，就会生成libfftw3.a在build/android/libs目录。如果没生成多半是权限问题没创建目录成功，但实际上目录已经有了，再运行一次就好了。

所有需要的东西都在build/android目录，包括头文件等，直接就能用了。

## 0x02 使用cmake
cmake复杂点，因为下载的源码里面没带cmake的文件，但是github里面的却有。没办法，那就直接从GitHub把CMakeLists.txt下载下来，放在fftw的源码根目录。cmake的交叉编译接触不多，Android Studio直接集成了cmake来做NDK编译，如果抛开Android Studio来使用cmake交叉编译，会比较麻烦，要指定很多东西，而且不知道会不会漏掉什么。Google了一番也发现没人会抛开Android Studio来做cmake的交叉编译，基于此，于是考虑直接使用Android Studio。

在Android Studio新建Module，命名为fftw，然后在fftw项目文件夹下新建文件夹fftw，然后把下载的源码连同那个CMakeLists.txt全部copy过来。在main目录下新建文件夹cpp，放一个fftw_jni.cpp进去，内容是空的就行。然后在fftw项目根目录新建一个CMakeLists.txt，内容如下：
```
cmake_minimum_required (VERSION 3.4.1)

add_subdirectory(fftw)

include_directories(
    fftw
)

add_library( fftw_jni
             SHARED
             src/main/cpp/fftw_jni.cpp)

target_link_libraries(fftw_jni android log fftw)
```

然后Link C++ Project with Gradle，指定上面这个新建的CMakeLists.txt文件。

打开fftw项目里面的fftw文件夹里的CMakeLists.txt文件，然后修改里面的内容，主要是删掉一些我们不需要的，修改后文件内容如下：
```
if (NOT DEFINED CMAKE_BUILD_TYPE)
  set (CMAKE_BUILD_TYPE Release CACHE STRING "Build type")
endif ()

if (POLICY CMP0042)
  cmake_policy (SET CMP0042 NEW)
endif ()

option (BUILD_SHARED_LIBS "Build shared libraries" ON)
option (BUILD_TESTS "Build tests" OFF)

option (ENABLE_OPENMP "Use OpenMP for multithreading" OFF)
option (ENABLE_THREADS "Use pthread for multithreading" OFF)
option (WITH_COMBINED_THREADS "Merge thread library" OFF)

option (ENABLE_FLOAT "single-precision" OFF)
option (ENABLE_LONG_DOUBLE "long-double precision" OFF)
option (ENABLE_QUAD_PRECISION "quadruple-precision" OFF)

option (ENABLE_SSE "Compile with SSE instruction set support" OFF)
option (ENABLE_SSE2 "Compile with SSE2 instruction set support" OFF)
option (ENABLE_AVX "Compile with AVX instruction set support" OFF)
option (ENABLE_AVX2 "Compile with AVX2 instruction set support" OFF)

include (CheckIncludeFile)
check_include_file (alloca.h         HAVE_ALLOCA_H)
check_include_file (altivec.h        HAVE_ALTIVEC_H)
check_include_file (c_asm.h          HAVE_C_ASM_H)
check_include_file (intrinsics.h     HAVE_INTRINSICS_H)
check_include_file (inttypes.h       HAVE_INTTYPES_H)
check_include_file (libintl.h        HAVE_LIBINTL_H)
check_include_file (limits.h         HAVE_LIMITS_H)
check_include_file (mach/mach_time.h HAVE_MACH_MACH_TIME_H)
check_include_file (malloc.h         HAVE_MALLOC_H)
check_include_file (memory.h         HAVE_MEMORY_H)
check_include_file (stddef.h         HAVE_STDDEF_H)
check_include_file (stdint.h         HAVE_STDINT_H)
check_include_file (stdlib.h         HAVE_STDLIB_H)
check_include_file (string.h         HAVE_STRING_H)
check_include_file (strings.h        HAVE_STRINGS_H)
check_include_file (sys/types.h      HAVE_SYS_TYPES_H)
check_include_file (sys/time.h       HAVE_SYS_TIME_H)
check_include_file (sys/stat.h       HAVE_SYS_STAT_H)
check_include_file (sys/sysctl.h     HAVE_SYS_SYSCTL_H)
check_include_file (time.h           HAVE_TIME_H)
check_include_file (uintptr.h        HAVE_UINTPTR_H)
check_include_file (unistd.h         HAVE_UNISTD_H)
if (HAVE_TIME_H AND HAVE_SYS_TIME_H)
  set (TIME_WITH_SYS_TIME TRUE)
endif ()

include (CheckPrototypeDefinition) 
check_prototype_definition (drand48 "double drand48 (void)" "0" stdlib.h HAVE_DECL_DRAND48)
check_prototype_definition (srand48 "void srand48(long int seedval)" "0" stdlib.h HAVE_DECL_SRAND48)
check_prototype_definition (cosl "long double cosl( long double arg )" "0" math.h HAVE_DECL_COSL)
check_prototype_definition (sinl "long double sinl( long double arg )" "0" math.h HAVE_DECL_SINL)
check_prototype_definition (memalign "void *memalign(size_t alignment, size_t size)" "0" malloc.h HAVE_DECL_MEMALIGN)
check_prototype_definition (posix_memalign "int posix_memalign(void **memptr, size_t alignment, size_t size)" "0" stdlib.h HAVE_DECL_POSIX_MEMALIGN)

include (CheckSymbolExists)
check_symbol_exists (clock_gettime time.h HAVE_CLOCK_GETTIME)
check_symbol_exists (gettimeofday sys/time.h HAVE_GETTIMEOFDAY)
check_symbol_exists (drand48 stdlib.h HAVE_DRAND48)
check_symbol_exists (srand48 stdlib.h HAVE_SRAND48)
check_symbol_exists (memalign malloc.h HAVE_MEMALIGN)
check_symbol_exists (posix_memalign stdlib.h HAVE_POSIX_MEMALIGN)
check_symbol_exists (mach_absolute_time mach/mach_time.h HAVE_MACH_ABSOLUTE_TIME)
check_symbol_exists (alloca alloca.h HAVE_ALLOCA)
check_symbol_exists (isnan math.h HAVE_ISNAN)
check_symbol_exists (snprintf stdio.h HAVE_SNPRINTF)

if (UNIX)
  set (CMAKE_REQUIRED_LIBRARIES m)
endif ()
check_symbol_exists (cosl math.h HAVE_COSL)
check_symbol_exists (sinl math.h HAVE_SINL)

include (CheckTypeSize)
check_type_size ("float" SIZEOF_FLOAT)
check_type_size ("double" SIZEOF_DOUBLE)
check_type_size ("int" SIZEOF_INT)
check_type_size ("long" SIZEOF_LONG)
check_type_size ("long long" SIZEOF_LONG_LONG)
check_type_size ("unsigned int" SIZEOF_UNSIGNED_INT)
check_type_size ("unsigned long" SIZEOF_UNSIGNED_LONG)
check_type_size ("unsigned long long" SIZEOF_UNSIGNED_LONG_LONG)
check_type_size ("size_t" SIZEOF_SIZE_T)
check_type_size ("ptrdiff_t" SIZEOF_PTRDIFF_T)

if (UNIX)
  set (HAVE_LIBM TRUE)
endif ()


if (ENABLE_THREADS)
  find_package (Threads)
endif ()
if (Threads_FOUND)
  if(CMAKE_USE_PTHREADS_INIT)
    set (USING_POSIX_THREADS 1)
  endif ()
  set (HAVE_THREADS TRUE)
endif ()

if (ENABLE_OPENMP)
  find_package (OpenMP)
endif ()
if (OPENMP_FOUND)
  set (HAVE_OPENMP TRUE)
endif ()

include (CheckCCompilerFlag)

if (ENABLE_SSE)
  foreach (FLAG "-msse" "/arch:SSE")
    unset (HAVE_SSE CACHE)
    check_c_compiler_flag (${FLAG} HAVE_SSE)
    if (HAVE_SSE)
      set (SSE_FLAG ${FLAG})
      break()
    endif ()
  endforeach ()
endif ()

if (ENABLE_SSE2)
  foreach (FLAG "-msse2" "/arch:SSE2")
    unset (HAVE_SSE2 CACHE)
    check_c_compiler_flag (${FLAG} HAVE_SSE2)
    if (HAVE_SSE2)
      set (SSE2_FLAG ${FLAG})
      break()
    endif ()
  endforeach ()
endif ()

if (ENABLE_AVX)
  foreach (FLAG "-mavx" "/arch:AVX")
    unset (HAVE_AVX CACHE)
    check_c_compiler_flag (${FLAG} HAVE_AVX)
    if (HAVE_AVX)
      set (AVX_FLAG ${FLAG})
      break()
    endif ()
  endforeach ()
endif ()

if (ENABLE_AVX2)
  foreach (FLAG "-mavx2" "/arch:AVX2")
    unset (HAVE_AVX2 CACHE)
    check_c_compiler_flag (${FLAG} HAVE_AVX2)
    if (HAVE_AVX2)
      set (AVX2_FLAG ${FLAG})
      break()
    endif ()
  endforeach ()
endif ()

# AVX2 codelets require FMA support as well
if (ENABLE_AVX2)
  foreach (FLAG "-mfma" "/arch:FMA")
    unset (HAVE_FMA CACHE)
    check_c_compiler_flag (${FLAG} HAVE_FMA)
    if (HAVE_FMA)
      set (FMA_FLAG ${FLAG})
      break()
    endif ()
  endforeach ()
endif ()

if (HAVE_SSE2 OR HAVE_AVX)
  set (HAVE_SIMD TRUE)
endif ()
file(GLOB           fftw_api_SOURCE                 api/*.c             api/*.h     kernel/*.h)
file(GLOB           fftw_dft_SOURCE                 dft/*.c             dft/*.h )
file(GLOB           fftw_dft_scalar_SOURCE          dft/scalar/*.c      dft/scalar/*.h kernel/*.h)
file(GLOB           fftw_dft_scalar_codelets_SOURCE dft/scalar/codelets/*.c     dft/scalar/codelets/*.h kernel/*.h)
file(GLOB           fftw_dft_simd_SOURCE            dft/simd/*.c        dft/simd/*.h)

file(GLOB           fftw_dft_simd_sse2_SOURCE       dft/simd/sse2/*.c   dft/simd/sse2/*.h)
file(GLOB           fftw_dft_simd_avx_SOURCE        dft/simd/avx/*.c    dft/simd/avx/*.h)
file(GLOB           fftw_dft_simd_avx2_SOURCE       dft/simd/avx2/*.c   dft/simd/avx2/*.h dft/simd/avx2-128/*.c   dft/simd/avx2-128/*.h)
file(GLOB           fftw_kernel_SOURCE              kernel/*.c          kernel/*.h)
file(GLOB           fftw_rdft_SOURCE                rdft/*.c            rdft/*.h)
file(GLOB           fftw_rdft_scalar_SOURCE         rdft/scalar/*.c     rdft/scalar/*.h)

file(GLOB           fftw_rdft_scalar_r2cb_SOURCE    rdft/scalar/r2cb/*.c
                                                    rdft/scalar/r2cb/*.h)
file(GLOB           fftw_rdft_scalar_r2cf_SOURCE    rdft/scalar/r2cf/*.c
                                                    rdft/scalar/r2cf/*.h)
file(GLOB           fftw_rdft_scalar_r2r_SOURCE     rdft/scalar/r2r/*.c
                                                    rdft/scalar/r2r/*.h)

file(GLOB           fftw_rdft_simd_SOURCE           rdft/simd/*.c       rdft/simd/*.h)
file(GLOB           fftw_rdft_simd_sse2_SOURCE      rdft/simd/sse2/*.c  rdft/simd/sse2/*.h)
file(GLOB           fftw_rdft_simd_avx_SOURCE       rdft/simd/avx/*.c   rdft/simd/avx/*.h)
file(GLOB           fftw_rdft_simd_avx2_SOURCE      rdft/simd/avx2/*.c  rdft/simd/avx2/*.h rdft/simd/avx2-128/*.c  rdft/simd/avx2-128/*.h)

file(GLOB           fftw_reodft_SOURCE              reodft/*.c          reodft/*.h)
file(GLOB           fftw_simd_support_SOURCE        simd-support/*.c    simd-support/*.h)
file(GLOB           fftw_libbench2_SOURCE           libbench2/*.c       libbench2/*.h)
list (REMOVE_ITEM   fftw_libbench2_SOURCE ${CMAKE_CURRENT_SOURCE_DIR}/libbench2/useropt.c)

set(SOURCEFILES
    ${fftw_api_SOURCE}
    ${fftw_dft_SOURCE}
    ${fftw_dft_scalar_SOURCE}
    ${fftw_dft_scalar_codelets_SOURCE}
    ${fftw_dft_simd_SOURCE}
    ${fftw_kernel_SOURCE}
    ${fftw_rdft_SOURCE}
    ${fftw_rdft_scalar_SOURCE}

    ${fftw_rdft_scalar_r2cb_SOURCE}
    ${fftw_rdft_scalar_r2cf_SOURCE}
    ${fftw_rdft_scalar_r2r_SOURCE}

    ${fftw_rdft_simd_SOURCE}
    ${fftw_reodft_SOURCE}
    ${fftw_simd_support_SOURCE}
    ${fftw_threads_SOURCE}
)

set(fftw_par_SOURCE
    threads/api.c
    threads/conf.c
    threads/ct.c
    threads/dft-vrank-geq1.c
    threads/f77api.c
    threads/hc2hc.c
    threads/rdft-vrank-geq1.c
    threads/vrank-geq1-rdft2.c)

set (fftw_threads_SOURCE ${fftw_par_SOURCE} threads/threads.c)
set (fftw_omp_SOURCE ${fftw_par_SOURCE} threads/openmp.c)


include_directories(.)

if (WITH_COMBINED_THREADS)
  list (APPEND SOURCEFILES ${fftw_threads_SOURCE})
endif ()


if (HAVE_SSE2)
  list (APPEND SOURCEFILES ${fftw_dft_simd_sse2_SOURCE} ${fftw_rdft_simd_sse2_SOURCE})
endif ()

if (HAVE_AVX)
  list (APPEND SOURCEFILES ${fftw_dft_simd_avx_SOURCE} ${fftw_rdft_simd_avx_SOURCE})
endif ()

if (HAVE_AVX2)
  list (APPEND SOURCEFILES ${fftw_dft_simd_avx2_SOURCE} ${fftw_rdft_simd_avx2_SOURCE})
endif ()

set (FFTW_VERSION 3.3.7)

set (PREC_SUFFIX)
if (ENABLE_FLOAT)
  set (FFTW_SINGLE TRUE)
  set (BENCHFFT_SINGLE TRUE)
  set (PREC_SUFFIX f)
endif ()

if (ENABLE_LONG_DOUBLE)
  set (FFTW_LDOUBLE TRUE)
  set (BENCHFFT_LDOUBLE TRUE)
  set (PREC_SUFFIX l)
endif ()

if (ENABLE_QUAD_PRECISION)
  set (FFTW_QUAD TRUE)
  set (BENCHFFT_QUAD TRUE)
  set (PREC_SUFFIX q)
endif ()
set (fftw3_lib fftw3${PREC_SUFFIX})

configure_file (cmake.config.h.in config.h @ONLY)
include_directories (${CMAKE_CURRENT_BINARY_DIR})

if (BUILD_SHARED_LIBS)
  add_definitions (-DFFTW_DLL)
endif ()


add_library (${fftw3_lib} ${SOURCEFILES})
if (MSVC)
  target_compile_definitions (${fftw3_lib} PRIVATE /bigobj)
endif ()
if (HAVE_SSE)
  target_compile_options (${fftw3_lib} PRIVATE ${SSE_FLAG})
endif ()
if (HAVE_SSE2)
  target_compile_options (${fftw3_lib} PRIVATE ${SSE2_FLAG})
endif ()
if (HAVE_AVX)
  target_compile_options (${fftw3_lib} PRIVATE ${AVX_FLAG})
endif ()
if (HAVE_AVX2)
  target_compile_options (${fftw3_lib} PRIVATE ${AVX2_FLAG})
endif ()
if (HAVE_FMA)
  target_compile_options (${fftw3_lib} PRIVATE ${FMA_FLAG})
endif ()
if (HAVE_LIBM)
  target_link_libraries (${fftw3_lib} m)
endif ()

set (subtargets ${fftw3_lib})

if (Threads_FOUND)
  if (WITH_COMBINED_THREADS)
    target_link_libraries (${fftw3_lib} ${CMAKE_THREAD_LIBS_INIT})
  else ()
    add_library (${fftw3_lib}_threads ${fftw_threads_SOURCE})
    target_link_libraries (${fftw3_lib}_threads ${fftw3_lib})
    target_link_libraries (${fftw3_lib}_threads ${CMAKE_THREAD_LIBS_INIT})
    list (APPEND subtargets ${fftw3_lib}_threads)
  endif ()
endif ()

if (OPENMP_FOUND)
  add_library (${fftw3_lib}_omp ${fftw_omp_SOURCE})
  target_link_libraries (${fftw3_lib}_omp ${fftw3_lib})
  target_link_libraries (${fftw3_lib}_omp ${CMAKE_THREAD_LIBS_INIT})
  list (APPEND subtargets ${fftw3_lib}_omp)
  target_compile_options (${fftw3_lib}_omp PRIVATE ${OpenMP_C_FLAGS})
endif ()


```

然后按Android Studio菜单上的Build->Make Module 'fftw'，出现编译错误，貌似是找不到头文件。
于是在把原来的include_directories(.)改成如下内容：
```
include_directories(.
     kernel
     api
     dft
     dft/scalar
     dft/scalar/codelets # really needed?
     dft/simd
     dft/simd/sse2
     dft/simd/avx
     rdft
     rdft/scalar
     rdft/simd
     reodft
     simd-support
     libbench2
)
```

再次编译，成功。

## 0x03 测试
之前在网上找资料的时候发现已经有人移植过了fftw到Android上，还有完整的测试项目，直接下回来用。

[项目地址][2]

直接Android Studio clone这个地址，然后编译运行，看看人家的效果。

额……他也没做正确性测试，只是看看能不能跑而已。。。好吧！

直接替换他的库为我编译出来的libfftw3.a。不知道为啥他比我多两个。先不管，应该是fftw的不同特性。

能正确运行，说明咱们的移植过程应该没有错误。这里说应该，是因为这里只能证明库链接正确，但不能证明运算是否正确。

fftw这个库应该还有需要优化的编译选项，后续再研究。

完。

[1]:http://blog.jimjh.com/compiling-open-source-libraries-with-android-ndk-part-2.html
[2]:https://github.com/iyezhou/Android-FFTW