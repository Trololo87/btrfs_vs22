cmake_minimum_required(VERSION 3.15)

project(btrfs VERSION 1.9.0)

include_directories("/usr/x86_64-w64-mingw32/include/ddk")

option(WITH_TEST "Compile test program" OFF)

if(MSVC) # cmake bug 15170
    if(MSVC_C_ARCHITECTURE_ID STREQUAL "X86")
        set(CMAKE_SYSTEM_PROCESSOR "x86")
    elseif(MSVC_C_ARCHITECTURE_ID STREQUAL "x64")
        set(CMAKE_SYSTEM_PROCESSOR "x86_64")
    elseif(MSVC_C_ARCHITECTURE_ID STREQUAL "ARMV7")
        set(CMAKE_SYSTEM_PROCESSOR "arm")
    elseif(MSVC_C_ARCHITECTURE_ID STREQUAL "ARM64")
        set(CMAKE_SYSTEM_PROCESSOR "aarch64")
    endif()
endif()

# zstd

set(ZSTD_SRC_FILES src/zstd/lib/common/entropy_common.c
    src/zstd/lib/common/error_private.c
    src/zstd/lib/compress/fse_compress.c
    src/zstd/lib/common/fse_decompress.c
    src/zstd/lib/compress/hist.c
    src/zstd/lib/compress/huf_compress.c
    src/zstd/lib/decompress/huf_decompress.c
    src/zstd/lib/common/zstd_common.c
    src/zstd/lib/compress/zstd_compress.c
    src/zstd/lib/compress/zstd_compress_literals.c
    src/zstd/lib/compress/zstd_compress_sequences.c
    src/zstd/lib/compress/zstd_compress_superblock.c
    src/zstd/lib/compress/zstd_preSplit.c
    src/zstd/lib/compress/zstd_preSplit.h
    src/zstd/lib/decompress/zstd_ddict.c
    src/zstd/lib/decompress/zstd_decompress.c
    src/zstd/lib/decompress/zstd_decompress_block.c
    src/zstd/lib/compress/zstd_double_fast.c
    src/zstd/lib/compress/zstd_fast.c
    src/zstd/lib/compress/zstd_lazy.c
    src/zstd/lib/compress/zstd_ldm.c
    src/zstd/lib/compress/zstd_opt.c
    src/zstd/lib/common/xxhash.c)

add_library(zstd STATIC ${ZSTD_SRC_FILES})
target_compile_definitions(zstd PRIVATE -DZSTD_DEPS_MALLOC -DXXH_NO_STDLIB)

if(NOT MSVC)
    target_compile_options(zstd PRIVATE -ffunction-sections -include ${CMAKE_SOURCE_DIR}/src/zstd-shim.h)
else()
    target_compile_options(zstd PRIVATE /Gy /FI ${CMAKE_SOURCE_DIR}/src/zstd-shim.h)
    set_property(TARGET zstd PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

if(MSVC AND CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
    target_compile_options(zstd PUBLIC /Gz) # stdcall
endif()

# zlib

set(ZLIB_SRC_FILES src/zlib/adler32.c
    src/zlib/deflate.c
    src/zlib/inffast.c
    src/zlib/inflate.c
    src/zlib/inftrees.c
    src/zlib/trees.c
    src/zlib/zutil.c)

add_library(zlib STATIC ${ZLIB_SRC_FILES})
target_compile_definitions(zlib PRIVATE -DNO_GZIP -DZ_SOLO)

if(NOT MSVC)
    target_compile_options(zlib PRIVATE -ffunction-sections)
else()
    target_compile_options(zlib PRIVATE /Gy)
endif()

if(MSVC AND CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
    target_compile_options(zlib PUBLIC /Gz) # stdcall
endif()

# btrfs.sys

set(SRC_FILES src/zstd/lib/decompress/huf_decompress_amd64.S
    src/balance.c
    src/blake2b-ref.c
    src/boot.c
    src/btrfs.c
    src/cache.c
    src/calcthread.c
    src/compress.c
    src/crc32c.c
    src/create.c
    src/devctrl.c
    src/dirctrl.c
    src/extent-tree.c
    src/fastio.c
    src/fileinfo.c
    src/flushthread.c
    src/free-space.c
    src/fsctl.c
    src/fsrtl.c
    src/galois.c
    src/pnp.c
    src/read.c
    src/registry.c
    src/reparse.c
    src/scrub.c
    src/search.c
    src/security.c
    src/send.c
    src/sha256.c
    src/treefuncs.c
    src/volume.c
    src/worker-thread.c
    src/write.c
    ${CMAKE_CURRENT_BINARY_DIR}/btrfs.rc)

# Work around bug in MSVC version of cmake - see https://gitlab.kitware.com/cmake/cmake/-/merge_requests/4257
set(CMAKE_ASM_MASM_COMPILE_OPTIONS_MSVC_RUNTIME_LIBRARY_MultiThreaded         "")
set(CMAKE_ASM_MASM_COMPILE_OPTIONS_MSVC_RUNTIME_LIBRARY_MultiThreadedDLL      "")
set(CMAKE_ASM_MASM_COMPILE_OPTIONS_MSVC_RUNTIME_LIBRARY_MultiThreadedDebug    "")
set(CMAKE_ASM_MASM_COMPILE_OPTIONS_MSVC_RUNTIME_LIBRARY_MultiThreadedDebugDLL "")

set(CMAKE_ASM_MASM_FLAGS "/Zd")

if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
    if(MSVC)
        enable_language(ASM_MASM)
        set(SRC_FILES ${SRC_FILES}
            src/crc32c-masm.asm
            src/xor-masm.asm)
    else()
        enable_language(ASM)
        set(SRC_FILES ${SRC_FILES}
            src/crc32c-gas.S
            src/xor-gas.S)
    endif()
endif()

if(MSVC AND (CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64"))
    # see cmake bug 24317 if armasm64 fails (should be fixed in CMake 3.29)
    enable_language(ASM_MARMASM)
    set(SRC_FILES ${SRC_FILES}
        src/crc32c-aarch64.asm)
endif()

configure_file(src/btrfs.rc.in btrfs.rc)

if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64")
    add_definitions(-D_AMD64_)
    set(MS_ARCH "x64")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
    add_definitions(-D_X86_)
    set(MS_ARCH "x86")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "arm")
    add_definitions(-D_ARM_)
    set(MS_ARCH "arm")
elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
    add_definitions(-D_ARM64_)
    set(MS_ARCH "arm64")
endif()

if(MSVC)
    include_directories("$ENV{WindowsSdkDir}Include\\$ENV{WindowsSDKLibVersion}km")
    link_directories("$ENV{WindowsSdkDir}Lib\\$ENV{WindowsSDKLibVersion}km\\${MS_ARCH}")
elseif(CMAKE_CXX_COMPILER_ID STREQUAL "GNU" AND WIN32)
    include_directories("${CMAKE_FIND_ROOT_PATH}/usr/include/ddk")
endif()

add_library(btrfs SHARED ${SRC_FILES})
target_link_libraries(btrfs zstd zlib)

if(CMAKE_BUILD_TYPE STREQUAL "Debug")
    add_definitions(-D_DEBUG)
endif()

if(NOT MSVC)
    target_compile_options(btrfs PUBLIC -U__NO_INLINE__)
    add_definitions(-D__USE_MINGW_ANSI_STDIO=0)
endif()

target_compile_definitions(btrfs PUBLIC _KERNEL_MODE WIN9X_COMPAT_SPINLOCK)

if(MSVC)
    if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
        target_compile_options(btrfs PUBLIC /Gz) # stdcall
    endif()

    target_link_libraries(btrfs ntoskrnl hal)

    if(CMAKE_SYSTEM_PROCESSOR STREQUAL "arm" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
        target_link_libraries(btrfs bufferoverflowfastfailk)
    else()
        target_link_libraries(btrfs BufferOverflowK)
    endif()

    if(CMAKE_SYSTEM_PROCESSOR STREQUAL "arm")
        target_link_libraries(btrfs armrt)
    elseif(CMAKE_SYSTEM_PROCESSOR STREQUAL "aarch64")
        target_link_libraries(btrfs arm64rt)
    endif()

    target_link_libraries(btrfs rtlver)
    target_link_options(btrfs PUBLIC /SUBSYSTEM:NATIVE /NODEFAULTLIB /MANIFEST:NO /Driver /ENTRY:DriverEntry)

    # strip out flags for MSVC's runtime checks
    string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")
    string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG}")
else()
    target_compile_options(btrfs PUBLIC -Wall -Werror-implicit-function-declaration -Werror=incompatible-pointer-types -Wno-expansion-to-defined -Wunused-parameter -Wtype-limits -Wextra)

    if (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
        target_compile_options(btrfs PUBLIC -Werror=cast-function-type -Wold-style-declaration)
    elseif (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
        target_compile_options(btrfs PUBLIC -Wno-pragma-pack) # ignore warning in mingw headers
    endif()

    target_link_libraries(btrfs ntoskrnl hal gcc)
    target_link_options(btrfs PUBLIC -nostdlib -Wl,--subsystem,native -Wl,--file-alignment,0x1000 -Wl,--section-alignment,0x1000 -Wl,--exclude-all-symbols)

    if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
        target_link_options(btrfs PUBLIC -Wl,--entry,_DriverEntry@8)
    else()
        target_link_options(btrfs PUBLIC -Wl,--entry,DriverEntry)
    endif()
endif()

set_target_properties(btrfs PROPERTIES PREFIX "")
set_target_properties(btrfs PROPERTIES SUFFIX ".sys")

# --------------------------------------

# shellbtrfs.dll

set(SHELLEXT_SRC_FILES src/shellext/balance.cpp
    src/shellext/contextmenu.cpp
    src/shellext/devices.cpp
    src/shellext/factory.cpp
    src/shellext/iconoverlay.cpp
    src/shellext/main.cpp
    src/shellext/mappings.cpp
    src/shellext/mountmgr.cpp
    src/shellext/propsheet.cpp
    src/shellext/recv.cpp
    src/shellext/scrub.cpp
    src/shellext/send.cpp
    src/shellext/volpropsheet.cpp
    src/crc32c.c
    src/shellext/shellbtrfs.def
    ${CMAKE_CURRENT_BINARY_DIR}/shellbtrfs.rc)

if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
    if(MSVC)
        enable_language(ASM_MASM)
        set(SHELLEXT_SRC_FILES ${SHELLEXT_SRC_FILES} src/crc32c-masm.asm)
    else()
        enable_language(ASM)
        set(SHELLEXT_SRC_FILES ${SHELLEXT_SRC_FILES} src/crc32c-gas.S)
    endif()
endif()

set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_VISIBILITY_PRESET hidden)

configure_file(src/shellext/shellbtrfs.rc.in shellbtrfs.rc)

add_library(shellbtrfs SHARED ${SHELLEXT_SRC_FILES})

if(NOT MSVC)
    target_link_options(shellbtrfs PUBLIC -static -static-libgcc)
    target_link_libraries(shellbtrfs pthread)

    target_compile_options(shellbtrfs PUBLIC -Wall -Wno-expansion-to-defined -Wunused-parameter -Wtype-limits -Wextra)
else()
    target_compile_options(shellbtrfs PUBLIC /EHsc)
    target_link_options(shellbtrfs PUBLIC /MANIFEST:NO)
endif()

target_link_libraries(shellbtrfs comctl32 ntdll setupapi uxtheme shlwapi windowscodecs gdi32 advapi32 shell32 ole32)

if(MSVC AND CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
    target_compile_options(shellbtrfs PUBLIC /Gz) # stdcall
endif()

set_target_properties(shellbtrfs PROPERTIES PREFIX "")

if(MSVC)
    set_property(TARGET shellbtrfs PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

# --------------------------------------

# ubtrfs.dll

set(UBTRFS_SRC_FILES src/ubtrfs/ubtrfs.c
    src/crc32c.c
    src/sha256.c
    src/blake2b-ref.c
    src/ubtrfs/ubtrfs.def
    ${CMAKE_CURRENT_BINARY_DIR}/ubtrfs.rc)

if(CMAKE_SYSTEM_PROCESSOR STREQUAL "x86_64" OR CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
    if(MSVC)
        enable_language(ASM_MASM)
        set(UBTRFS_SRC_FILES ${UBTRFS_SRC_FILES} src/crc32c-masm.asm)
    else()
        enable_language(ASM)
        set(UBTRFS_SRC_FILES ${UBTRFS_SRC_FILES} src/crc32c-gas.S)
    endif()
endif()

configure_file(src/ubtrfs/ubtrfs.rc.in ubtrfs.rc)

add_library(ubtrfs SHARED ${UBTRFS_SRC_FILES})

if(MSVC)
    set_property(TARGET ubtrfs PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

target_compile_definitions(ubtrfs PUBLIC _USRDLL)
target_link_libraries(ubtrfs ntdll advapi32)
target_link_libraries(ubtrfs zstd)

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    target_compile_options(ubtrfs PUBLIC -Wno-pragma-pack) # ignore warning in mingw headers
endif()

if(MSVC AND CMAKE_SYSTEM_PROCESSOR STREQUAL "x86")
    target_compile_options(ubtrfs PUBLIC /Gz) # stdcall
endif()

if(NOT MSVC)
    target_compile_options(ubtrfs PUBLIC -Werror-implicit-function-declaration)
    target_link_options(ubtrfs PUBLIC -static -static-libgcc -static-libstdc++)
endif()

set_target_properties(ubtrfs PROPERTIES PREFIX "")

# --------------------------------------

# mkbtrfs.exe

set(MKBTRFS_SRC_FILES src/mkbtrfs/mkbtrfs.c
    ${CMAKE_CURRENT_BINARY_DIR}/mkbtrfs.rc)

configure_file(src/mkbtrfs/mkbtrfs.rc.in mkbtrfs.rc)

add_executable(mkbtrfs ${MKBTRFS_SRC_FILES})

if(MSVC)
    set_property(TARGET mkbtrfs PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
else()
    target_link_options(mkbtrfs PUBLIC -static -static-libgcc)
endif()

# --------------------------------------

# test.exe

if(WITH_TEST)
    set(CMAKE_CXX_STANDARD 20)
    set(CMAKE_CXX_STANDARD_REQUIRED ON)

    configure_file(src/tests/test.rc.in test.rc)

    set(TEST_SRC_FILES src/tests/test.cpp
        ${CMAKE_CURRENT_BINARY_DIR}/test.rc
        src/tests/create.cpp
        src/tests/supersede.cpp
        src/tests/overwrite.cpp
        src/tests/io.cpp
        src/tests/mmap.cpp
        src/tests/rename.cpp
        src/tests/delete.cpp
        src/tests/links.cpp
        src/tests/oplock.cpp
        src/tests/cs.cpp
        src/tests/reparse.cpp
        src/tests/streams.cpp
        src/tests/ea.cpp
        src/tests/fileinfo.cpp
        src/tests/security.cpp)

    add_executable(test ${TEST_SRC_FILES})

    if(MSVC)
        set_property(TARGET test PROPERTY MSVC_RUNTIME_LIBRARY "MultiThreaded$<$<CONFIG:Debug>:Debug>")
    else()
        target_link_options(test PUBLIC -static -static-libgcc -municode)
        target_compile_options(test PUBLIC -Wall -Wno-expansion-to-defined -Wunused-parameter -Wtype-limits -Wextra)
    endif()

    target_link_libraries(test ntdll version advapi32)
endif()

# --------------------------------------

# install

install(TARGETS btrfs DESTINATION bin)
install(TARGETS shellbtrfs DESTINATION bin)
install(TARGETS ubtrfs DESTINATION bin)
install(TARGETS mkbtrfs DESTINATION bin)
