# Build system for the common Sanitizer runtime support library components.
# These components are shared between AddressSanitizer and ThreadSanitizer.

set(SANITIZER_SOURCES
  sanitizer_allocator.cc
  sanitizer_common.cc
  sanitizer_deadlock_detector1.cc
  sanitizer_deadlock_detector2.cc
  sanitizer_flags.cc
  sanitizer_flag_parser.cc
  sanitizer_libc.cc
  sanitizer_libignore.cc
  sanitizer_linux.cc
  sanitizer_mac.cc
  sanitizer_persistent_allocator.cc
  sanitizer_platform_limits_linux.cc
  sanitizer_platform_limits_posix.cc
  sanitizer_posix.cc
  sanitizer_printf.cc
  sanitizer_procmaps_common.cc
  sanitizer_procmaps_freebsd.cc
  sanitizer_procmaps_linux.cc
  sanitizer_procmaps_mac.cc
  sanitizer_stackdepot.cc
  sanitizer_stacktrace.cc
  sanitizer_stacktrace_printer.cc
  sanitizer_suppressions.cc
  sanitizer_symbolizer.cc
  sanitizer_symbolizer_libbacktrace.cc
  sanitizer_symbolizer_mac.cc
  sanitizer_symbolizer_win.cc
  sanitizer_tls_get_addr.cc
  sanitizer_thread_registry.cc
  sanitizer_win.cc)

# Libc functions stubs. These sources should be linked instead of
# SANITIZER_LIBCDEP_SOURCES when sanitizer_common library must not depend on
# libc.
set(SANITIZER_NOLIBC_SOURCES
  sanitizer_common_nolibc.cc)

set(SANITIZER_LIBCDEP_SOURCES
  sanitizer_common_libcdep.cc
  sanitizer_coverage_libcdep.cc
  sanitizer_coverage_mapping_libcdep.cc
  sanitizer_linux_libcdep.cc
  sanitizer_posix_libcdep.cc
  sanitizer_stacktrace_libcdep.cc
  sanitizer_stoptheworld_linux_libcdep.cc
  sanitizer_symbolizer_libcdep.cc
  sanitizer_symbolizer_posix_libcdep.cc
  sanitizer_symbolizer_process_libcdep.cc
  sanitizer_unwind_linux_libcdep.cc)

# Explicitly list all sanitizer_common headers. Not all of these are
# included in sanitizer_common source files, but we need to depend on
# headers when building our custom unit tests.
set(SANITIZER_HEADERS
  sanitizer_addrhashmap.h
  sanitizer_allocator.h
  sanitizer_allocator_interface.h
  sanitizer_allocator_internal.h
  sanitizer_atomic.h
  sanitizer_atomic_clang.h
  sanitizer_atomic_msvc.h
  sanitizer_bitvector.h
  sanitizer_bvgraph.h
  sanitizer_common.h
  sanitizer_common_interceptors.inc
  sanitizer_common_interceptors_ioctl.inc
  sanitizer_common_interceptors_format.inc
  sanitizer_common_syscalls.inc
  sanitizer_deadlock_detector.h
  sanitizer_deadlock_detector_interface.h
  sanitizer_flag_parser.h
  sanitizer_flags.h
  sanitizer_flags.inc
  sanitizer_interface_internal.h
  sanitizer_internal_defs.h
  sanitizer_lfstack.h
  sanitizer_libc.h
  sanitizer_libignore.h
  sanitizer_linux.h
  sanitizer_list.h
  sanitizer_mac.h
  sanitizer_mutex.h
  sanitizer_persistent_allocator.h
  sanitizer_placement_new.h
  sanitizer_platform.h
  sanitizer_platform_interceptors.h
  sanitizer_platform_limits_posix.h
  sanitizer_posix.h
  sanitizer_procmaps.h
  sanitizer_quarantine.h
  sanitizer_report_decorator.h
  sanitizer_stackdepot.h
  sanitizer_stackdepotbase.h
  sanitizer_stacktrace.h
  sanitizer_stacktrace_printer.h
  sanitizer_stoptheworld.h
  sanitizer_suppressions.h
  sanitizer_symbolizer.h
  sanitizer_symbolizer_internal.h
  sanitizer_symbolizer_libbacktrace.h
  sanitizer_symbolizer_mac.h
  sanitizer_symbolizer_win.h
  sanitizer_syscall_generic.inc
  sanitizer_syscall_linux_x86_64.inc
  sanitizer_thread_registry.h)

set(SANITIZER_COMMON_DEFINITIONS)

if(MSVC)
  list(APPEND SANITIZER_COMMON_DEFINITIONS
    SANITIZER_NEEDS_SEGV=0)
else()
  list(APPEND SANITIZER_COMMON_DEFINITIONS
    SANITIZER_NEEDS_SEGV=1)
endif()

include(CheckIncludeFile)
append_have_file_definition(rpc/xdr.h HAVE_RPC_XDR_H SANITIZER_COMMON_DEFINITIONS)
append_have_file_definition(tirpc/rpc/xdr.h HAVE_TIRPC_RPC_XDR_H SANITIZER_COMMON_DEFINITIONS)

set(SANITIZER_CFLAGS ${SANITIZER_COMMON_CFLAGS})
append_no_rtti_flag(SANITIZER_CFLAGS)

append_list_if(SANITIZER_LIMIT_FRAME_SIZE -Wframe-larger-than=570
               SANITIZER_CFLAGS)
append_list_if(COMPILER_RT_HAS_WGLOBAL_CONSTRUCTORS_FLAG -Wglobal-constructors
               SANITIZER_CFLAGS)

add_custom_target(sanitizer_common)
set(SANITIZER_RUNTIME_LIBRARIES)
if(APPLE)
  # Build universal binary on APPLE.

  add_compiler_rt_object_libraries(RTSanitizerCommon
    OS ${SANITIZER_COMMON_SUPPORTED_OS}
    ARCHS ${SANITIZER_COMMON_SUPPORTED_ARCH}
    SOURCES ${SANITIZER_SOURCES} ${SANITIZER_LIBCDEP_SOURCES}
    CFLAGS ${SANITIZER_CFLAGS}
    DEFS ${SANITIZER_COMMON_DEFINITIONS})
  foreach(os ${SANITIZER_COMMON_SUPPORTED_OS})
    list(APPEND SANITIZER_RUNTIME_LIBRARIES RTSanitizerCommon.${os})
  endforeach()
else()
  # Otherwise, build separate libraries for each target.
  
  add_compiler_rt_object_libraries(RTSanitizerCommon
    ARCHS ${SANITIZER_COMMON_SUPPORTED_ARCH}
    SOURCES ${SANITIZER_SOURCES} CFLAGS ${SANITIZER_CFLAGS}
    DEFS ${SANITIZER_COMMON_DEFINITIONS})
  add_compiler_rt_object_libraries(RTSanitizerCommonNoLibc
    ARCHS ${SANITIZER_COMMON_SUPPORTED_ARCH}
    SOURCES ${SANITIZER_NOLIBC_SOURCES} CFLAGS ${SANITIZER_CFLAGS}
    DEFS ${SANITIZER_COMMON_DEFINITIONS})
  add_compiler_rt_object_libraries(RTSanitizerCommonLibc
    ARCHS ${SANITIZER_COMMON_SUPPORTED_ARCH}
    SOURCES ${SANITIZER_LIBCDEP_SOURCES} CFLAGS ${SANITIZER_CFLAGS}
    DEFS ${SANITIZER_COMMON_DEFINITIONS})
  foreach(arch ${SANITIZER_COMMON_SUPPORTED_ARCH})
    list(APPEND SANITIZER_RUNTIME_LIBRARIES RTSanitizerCommon.${arch}
                                            RTSanitizerCommonLibc.${arch})
  endforeach()
endif()

add_dependencies(compiler-rt sanitizer_common)

# Unit tests for common sanitizer runtime.
if(COMPILER_RT_INCLUDE_TESTS)
  add_subdirectory(tests)
endif()
