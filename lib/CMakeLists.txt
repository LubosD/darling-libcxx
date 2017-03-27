set(LIBCXX_LIB_CMAKEFILES_DIR "${CMAKE_CURRENT_BINARY_DIR}${CMAKE_FILES_DIRECTORY}"  PARENT_SCOPE)

# Get sources
file(GLOB LIBCXX_SOURCES ../src/*.cpp ../src/*.c)
if(WIN32)
  file(GLOB LIBCXX_WIN32_SOURCES ../src/support/win32/*.cpp)
  list(APPEND LIBCXX_SOURCES ${LIBCXX_WIN32_SOURCES})
elseif("${CMAKE_SYSTEM_NAME}" STREQUAL "SunOS")
  file(GLOB LIBCXX_SOLARIS_SOURCES ../src/support/solaris/*.cpp)
  list(APPEND LIBCXX_SOURCES ${LIBCXX_SOLARIS_SOURCES})
endif()

# Add all the headers to the project for IDEs.
if (LIBCXX_CONFIGURE_IDE)
  file(GLOB_RECURSE LIBCXX_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/../include/*)
  if(WIN32)
    file( GLOB LIBCXX_WIN32_HEADERS ${CMAKE_CURRENT_SOURCE_DIR}/../include/support/win32/*.h)
    list(APPEND LIBCXX_HEADERS ${LIBCXX_WIN32_HEADERS})
  endif()
  # Force them all into the headers dir on MSVC, otherwise they end up at
  # project scope because they don't have extensions.
  if (MSVC_IDE)
    source_group("Header Files" FILES ${LIBCXX_HEADERS})
  endif()
endif()

set(APPLE TRUE)
add_darling_library(cxx SHARED ${LIBCXX_SOURCES} ${LIBCXX_HEADERS})
set_target_properties(cxx PROPERTIES OUTPUT_NAME "c++.1")

if(NOT LIBCXX_INSTALL_LIBRARY)
  set(exclude_from_all EXCLUDE_FROM_ALL)
endif()

# If LIBCXX_CXX_ABI_LIBRARY_PATH is defined we want to add it to the search path.
add_link_flags_if(LIBCXX_CXX_ABI_LIBRARY_PATH
                  "${CMAKE_LIBRARY_PATH_FLAG}${LIBCXX_CXX_ABI_LIBRARY_PATH}")


if (LIBCXX_GENERATE_COVERAGE AND NOT LIBCXX_COVERAGE_LIBRARY)
  find_compiler_rt_library(profile LIBCXX_COVERAGE_LIBRARY)
endif()
add_library_flags_if(LIBCXX_COVERAGE_LIBRARY "${LIBCXX_COVERAGE_LIBRARY}")

if (APPLE AND (LIBCXX_CXX_ABI_LIBNAME STREQUAL "libcxxabi" OR
               LIBCXX_CXX_ABI_LIBNAME STREQUAL "default"))
  set(LIBCXX_OSX_REEXPORT_SYSTEM_ABI_LIBRARY ON)
endif()

if (LIBCXX_ENABLE_STATIC_ABI_LIBRARY)
  add_library_flags("-Wl,--whole-archive" "-Wl,-Bstatic")
  add_library_flags("${LIBCXX_CXX_ABI_LIBRARY}")
  add_library_flags("-Wl,-Bdynamic" "-Wl,--no-whole-archive")
elseif (LIBCXX_OSX_REEXPORT_SYSTEM_ABI_LIBRARY)
  add_library_flags("${LIBCXX_CXX_ABI_LIBRARY}")
else ()
 add_interface_library("${LIBCXX_CXX_ABI_LIBRARY}")
endif()

if (APPLE AND LLVM_USE_SANITIZER)
  if (("${LLVM_USE_SANITIZER}" STREQUAL "Address") OR
      ("${LLVM_USE_SANITIZER}" STREQUAL "Address;Undefined") OR
      ("${LLVM_USE_SANITIZER}" STREQUAL "Undefined;Address"))
    set(LIBFILE "libclang_rt.asan_osx_dynamic.dylib")
  elseif("${LLVM_USE_SANITIZER}" STREQUAL "Undefined")
    set(LIBFILE "libclang_rt.ubsan_osx_dynamic.dylib")
  elseif("${LLVM_USE_SANITIZER}" STREQUAL "Thread")
    set(LIBFILE "libclang_rt.tsan_osx_dynamic.dylib")
  else()
    message(WARNING "LLVM_USE_SANITIZER=${LLVM_USE_SANITIZER} is not supported on OS X")
  endif()
  if (LIBFILE)
    find_compiler_rt_dir(LIBDIR)
    if (NOT IS_DIRECTORY "${LIBDIR}")
      message(FATAL_ERROR "Cannot find compiler-rt directory on OS X required for LLVM_USE_SANITIZER")
    endif()
    set(LIBCXX_SANITIZER_LIBRARY "${LIBDIR}/${LIBFILE}")
    set(LIBCXX_SANITIZER_LIBRARY "${LIBCXX_SANITIZER_LIBRARY}" PARENT_SCOPE)
    message(STATUS "Manually linking compiler-rt library: ${LIBCXX_SANITIZER_LIBRARY}")
    add_library_flags("${LIBCXX_SANITIZER_LIBRARY}")
    add_link_flags("-Wl,-rpath,${LIBDIR}")
  endif()
endif()

# Generate private library list.
#add_library_flags_if(LIBCXX_HAS_PTHREAD_LIB pthread)
#add_library_flags_if(LIBCXX_HAS_C_LIB c)
#add_library_flags_if(LIBCXX_HAS_M_LIB m)
#add_library_flags_if(LIBCXX_HAS_RT_LIB rt)
#add_library_flags_if(LIBCXX_HAS_GCC_S_LIB gcc_s)
#add_library_flags_if(LIBCXX_HAVE_CXX_ATOMICS_WITH_LIB atomic)

# Setup flags.
if (NOT WIN32)
  add_flags_if_supported(-fPIC)
endif()

add_link_flags_if_supported(-nodefaultlibs)

if (LIBCXX_TARGETING_MSVC)
  if (LIBCXX_DEBUG_BUILD)
    set(LIB_SUFFIX "d")
  else()
    set(LIB_SUFFIX "")
  endif()
  add_compile_flags(/Zl)
  add_link_flags(/nodefaultlib)

  add_library_flags(ucrt${LIB_SUFFIX}) # Universal C runtime
  add_library_flags(vcruntime${LIB_SUFFIX}) # C++ runtime
  add_library_flags(msvcrt${LIB_SUFFIX}) # C runtime startup files
  # Required for standards-complaint wide character formatting functions
  # (e.g. `printfw`/`scanfw`)
  add_library_flags(iso_stdio_wide_specifiers)
endif()

if (LIBCXX_OSX_REEXPORT_SYSTEM_ABI_LIBRARY)
  if (NOT DEFINED LIBCXX_LIBCPPABI_VERSION)
    set(LIBCXX_LIBCPPABI_VERSION "2") # Default value
    execute_process(
      COMMAND xcrun --show-sdk-version
      OUTPUT_VARIABLE sdk_ver
      RESULT_VARIABLE res
      OUTPUT_STRIP_TRAILING_WHITESPACE)
    if (res EQUAL 0)
      message(STATUS "Found SDK version ${sdk_ver}")
      string(REPLACE "10." "" sdk_ver "${sdk_ver}")
      if (sdk_ver LESS 9)
        set(LIBCXX_LIBCPPABI_VERSION "")
      else()
        set(LIBCXX_LIBCPPABI_VERSION "2")
      endif()
    endif()
  endif()

  if ( CMAKE_OSX_DEPLOYMENT_TARGET STREQUAL "10.6" )
    add_definitions(-D__STRICT_ANSI__)
    add_link_flags(
      "-compatibility_version 1"
      "-current_version 1"
      "-install_name /usr/lib/libc++.1.dylib"
      "-Wl,-reexport_library,/usr/lib/libc++abi.dylib"
      "-Wl,-unexported_symbols_list,${CMAKE_CURRENT_SOURCE_DIR}/libc++unexp.exp"
      "/usr/lib/libSystem.B.dylib")
  else()
    if (DEFINED CMAKE_OSX_SYSROOT AND NOT CMAKE_OSX_SYSROOT STREQUAL "")
      list(FIND CMAKE_OSX_ARCHITECTURES "armv7" OSX_HAS_ARMV7)
      if (NOT OSX_HAS_ARMV7 EQUAL -1)
        set(OSX_RE_EXPORT_LINE
          "${CMAKE_OSX_SYSROOT}/usr/lib/libc++abi.dylib"
          "-Wl,-reexported_symbols_list,${CMAKE_CURRENT_SOURCE_DIR}/libc++sjlj-abi.exp")
      else()
        set(OSX_RE_EXPORT_LINE
          "-Wl,-reexport_library,${CMAKE_OSX_SYSROOT}/usr/lib/libc++abi.dylib")
      endif()
    else()
      set(OSX_RE_EXPORT_LINE "-Wl,-reexported_symbols_list,${CMAKE_CURRENT_SOURCE_DIR}/libc++abi${LIBCXX_LIBCPPABI_VERSION}.exp")
    endif()

    add_link_flags(
      "-compatibility_version 1"
      "-install_name /usr/lib/libc++.1.dylib"
      "-Wl,-unexported_symbols_list,${CMAKE_CURRENT_SOURCE_DIR}/libc++unexp.exp"
      "${OSX_RE_EXPORT_LINE}"
      "-Wl,-force_symbols_not_weak_list,${CMAKE_CURRENT_SOURCE_DIR}/notweak.exp"
      "-Wl,-force_symbols_weak_list,${CMAKE_CURRENT_SOURCE_DIR}/weak.exp")
  endif()
endif()

target_link_libraries(cxx PRIVATE ${LIBCXX_LIBRARIES} system cxxabi_shared)
split_list(LIBCXX_COMPILE_FLAGS)
split_list(LIBCXX_LINK_FLAGS)

install(TARGETS cxx DESTINATION libexec/darling/usr/lib)
InstallSymlink("libc++.1.dylib" "${CMAKE_INSTALL_PREFIX}/libexec/darling/usr/lib/libc++.dylib")

set_target_properties(cxx
  PROPERTIES
  #    COMPILE_FLAGS "${LIBCXX_COMPILE_FLAGS}"
#    LINK_FLAGS    "${LIBCXX_LINK_FLAGS}"
#    OUTPUT_NAME   "c++"
    VERSION       "1.0"
    SOVERSION     "1"
  )

if (LIBCXX_BUILD_EXTERNAL_THREAD_LIBRARY)
  file(GLOB LIBCXX_EXTERNAL_THREADING_SUPPORT_SOURCES ../test/support/external_threads.cpp)

  if (LIBCXX_ENABLE_SHARED)
    add_library(cxx_external_threads SHARED ${LIBCXX_EXTERNAL_THREADING_SUPPORT_SOURCES})
  else()
    add_library(cxx_external_threads STATIC ${LIBCXX_EXTERNAL_THREADING_SUPPORT_SOURCES})
  endif()

  set_target_properties(cxx_external_threads
    PROPERTIES
      LINK_FLAGS    "${LIBCXX_LINK_FLAGS}"
      COMPILE_FLAGS "${LIBCXX_COMPILE_FLAGS}"
      OUTPUT_NAME   "c++external_threads"
  )
endif()

# Generate a linker script inplace of a libc++.so symlink. Rerun this command
# after cxx builds.
if (LIBCXX_ENABLE_SHARED AND LIBCXX_ENABLE_ABI_LINKER_SCRIPT)
  # Get the name of the ABI library and handle the case where CXXABI_LIBNAME
  # is a target name and not a library. Ex cxxabi_shared.
  set(LIBCXX_INTERFACE_LIBRARY_NAMES)
  foreach(lib ${LIBCXX_INTERFACE_LIBRARIES})
    # FIXME: Handle cxxabi_static and unwind_static.
    if (TARGET ${lib} OR
        (${lib} MATCHES "cxxabi(_static|_shared)?" AND HAVE_LIBCXXABI) OR
        (${lib} MATCHES "unwind(_static|_shared)?" AND HAVE_LIBUNWIND))
      list(APPEND LIBCXX_INTERFACE_LIBRARY_NAMES "$<TARGET_PROPERTY:${lib},OUTPUT_NAME>")
    else()
      list(APPEND LIBCXX_INTERFACE_LIBRARY_NAMES "${lib}")
    endif()
  endforeach()
  #split_list(LIBCXX_INTERFACE_LIBRARY_NAMES)
  # Generate a linker script inplace of a libc++.so symlink. Rerun this command
  # after cxx builds.
  add_custom_command(TARGET cxx_shared POST_BUILD
    COMMAND
      ${PYTHON_EXECUTABLE} ${LIBCXX_SOURCE_DIR}/utils/gen_link_script.py
    ARGS
      "$<TARGET_LINKER_FILE:cxx_shared>"
      ${LIBCXX_INTERFACE_LIBRARY_NAMES}
    WORKING_DIRECTORY ${LIBCXX_BUILD_DIR}
  )
endif()

set_property(TARGET cxx APPEND_STRING PROPERTY LINK_FLAGS    " ${LIBCXX_LINK_FLAGS} -Wl,-no_arch_warnings -Wl,-reexport_library,${CMAKE_BINARY_DIR}/src/external/libcxxabi/src/libc++abi.dylib ")
set_property(TARGET cxx APPEND_STRING PROPERTY COMPILE_FLAGS " ${LIBCXX_COMPILE_FLAGS}")
make_fat(cxx)

if (LIBCXX_INSTALL_LIBRARY)
  if (LIBCXX_INSTALL_EXPERIMENTAL_LIBRARY)
    set(experimental_lib cxx_experimental)
  endif()
  install(TARGETS ${LIBCXX_TARGETS} ${experimental_lib}
    LIBRARY DESTINATION lib${LIBCXX_LIBDIR_SUFFIX} COMPONENT cxx
    ARCHIVE DESTINATION lib${LIBCXX_LIBDIR_SUFFIX} COMPONENT cxx
    )
  # NOTE: This install command must go after the cxx install command otherwise
  # it will not be executed after the library symlinks are installed.
  if (LIBCXX_ENABLE_SHARED AND LIBCXX_ENABLE_ABI_LINKER_SCRIPT)
    # Replace the libc++ filename with $<TARGET_LINKER_FILE:cxx>
    # after we required CMake 3.0.
    install(FILES "${LIBCXX_LIBRARY_DIR}/libc++${CMAKE_SHARED_LIBRARY_SUFFIX}"
      DESTINATION lib${LIBCXX_LIBDIR_SUFFIX}
      COMPONENT libcxx)
  endif()
endif()

if (NOT CMAKE_CONFIGURATION_TYPES AND (LIBCXX_INSTALL_LIBRARY OR
                                       LIBCXX_INSTALL_HEADERS))
    if(LIBCXX_INSTALL_LIBRARY)
      set(lib_install_target cxx)
    endif()
    if (LIBCXX_INSTALL_EXPERIMENTAL_LIBRARY)
      set(experimental_lib_install_target cxx_experimental)
    endif()
    if(LIBCXX_INSTALL_HEADERS)
      set(header_install_target install-cxx-headers)
    endif()
    add_custom_target(install-cxx
                      DEPENDS ${lib_install_target}
                              ${experimental_lib_install_target}
                              ${header_install_target}
                      COMMAND "${CMAKE_COMMAND}"
                      -DCMAKE_INSTALL_COMPONENT=cxx
                      -P "${LIBCXX_BINARY_DIR}/cmake_install.cmake")
    add_custom_target(install-libcxx DEPENDS install-cxx)
endif()
