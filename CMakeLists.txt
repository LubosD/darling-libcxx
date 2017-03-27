project(libcxx)

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -nostdinc -std=c++11")

include_directories(
	BEFORE ${CMAKE_CURRENT_SOURCE_DIR}/include
)
include_directories(
	${CMAKE_SOURCE_DIR}/src/libc/locale
	${CMAKE_SOURCE_DIR}/src/libc/locale/FreeBSD
	${CMAKE_SOURCE_DIR}/src/libc/stdtime/FreeBSD
	${CMAKE_SOURCE_DIR}/src/libc/pthreads
	${CMAKE_SOURCE_DIR}/src/libc/include/FreeBSD
	${CMAKE_SOURCE_DIR}/src/external/libcxxabi/include
	${CMAKE_SOURCE_DIR}/src/libunwind/include
)

add_definitions(-D_LIBCPP_BUILDING_LIBRARY=1)

set(cxx_sources
	src/algorithm.cpp
	src/any.cpp
	src/bind.cpp
	src/condition_variable.cpp
	src/debug.cpp
	src/exception.cpp
	src/future.cpp
	src/hash.cpp
	src/chrono.cpp
	src/ios.cpp
	src/iostream.cpp
	src/locale.cpp
	src/memory.cpp
	src/mutex.cpp
	src/new.cpp
	src/optional.cpp
	src/random.cpp
	src/regex.cpp
	src/shared_mutex.cpp
	src/stdexcept.cpp
	src/string.cpp
	src/strstream.cpp
	src/system_error.cpp
	src/thread.cpp
	src/typeinfo.cpp
	src/utility.cpp
	src/valarray.cpp
	src/variant.cpp
)

add_library(cxx_obj OBJECT ${cxx_sources})
make_fat(cxx_obj)

set(DYLIB_INSTALL_NAME "/usr/lib/libc++.1.dylib")
set(DYLIB_COMPAT_VERSION "1.0.0")
set(DYLIB_CURRENT_VERSION "1.0.0")

add_darling_library(cxx SHARED $<TARGET_OBJECTS:cxx_obj>)
make_fat(cxx)
target_link_libraries(cxx PRIVATE system cxxabi)
set_target_properties(cxx PROPERTIES OUTPUT_NAME "c++.1")
set_property(TARGET cxx APPEND_STRING PROPERTY LINK_FLAGS " -Wl,-reexport_library,${CMAKE_BINARY_DIR}/src/external/libcxxabi/libc++abi.dylib \
      -Wl,-unexported_symbols_list,${CMAKE_CURRENT_SOURCE_DIR}/lib/libc++unexp.exp ")

install(TARGETS cxx DESTINATION libexec/darling/usr/lib)
InstallSymlink("libc++.1.dylib" "${CMAKE_INSTALL_PREFIX}/libexec/darling/usr/lib/libc++.dylib")

# For dyld only
set(CMAKE_AR "${CMAKE_BINARY_DIR}/src/external/cctools-port/cctools/ar/x86_64-apple-darwin11-ar")
set(CMAKE_RANLIB "${CMAKE_BINARY_DIR}/src/external/cctools-port/cctools/misc/ranlib")
add_library(cxx_static STATIC src/stdexcept.cpp)
make_fat(cxx_static)
add_dependencies(cxxabi_static ranlib x86_64-apple-darwin11-ar)

