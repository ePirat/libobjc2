cmake_minimum_required(VERSION 3.1)

project(libobjc C ASM CXX)

INCLUDE (CheckCXXSourceCompiles)

macro(install_symlink filepath sympath)
	if (UNIX)
		install(CODE "execute_process(COMMAND ${CMAKE_COMMAND} -E create_symlink ${filepath} ${sympath})")
		install(CODE "message(\"-- Symlinking: ${sympath} -> ${filepath}\")")
	endif ()
endmacro(install_symlink)

set(CMAKE_C_FLAGS_DEBUG "-O0 -Xclang -fno-inline ${CMAKE_C_FLAGS_DEBUG}")
set(CMAKE_C_FLAGS_RELEASE "-O3 ${CMAKE_C_FLAGS_RELEASE}")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")

set(libobjc_VERSION 4.6)

set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Xclang -fexceptions -Xclang -fobjc-exceptions")
if (MSVC)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /EHas")
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /EHas")
	set(CMAKE_C_FLAGS_DEBUG "/Z7 ${CMAKE_C_FLAGS_DEBUG}")
	set(CMAKE_SHARED_LINKER_FLAGS "/DEBUG /INCREMENTAL:NO ${CMAKE_SHARED_LINKER_FLAGS}")
	set(CMAKE_EXE_LINKER_FLAGS "/DEBUG /INCREMENTAL:NO ${CMAKE_EXE_LINKER_FLAGS}")
	set(objc_LINK_FLAGS "/DEBUG /INCREMENTAL:NO ${objc_LINK_FLAGS}")
endif()
# Build configuration
add_definitions( -DGNUSTEP -D__OBJC_RUNTIME_INTERNAL__=1)

set(libobjc_ASM_SRCS 
	block_trampolines.S
	objc_msgSend.S)
set(libobjc_OBJC_SRCS 
	NSBlocks.m
	Protocol2.m
	arc.m
	associate.m
	blocks_runtime.m
	properties.m)
set(libobjc_C_SRCS 
	alias_table.c
	block_to_imp.c
	caps.c
	category_loader.c
	class_table.c
	dtable.c
	encoding2.c
	hooks.c
	ivar.c
	loader.c
	mutation.m
	protocol.c
	runtime.c
	sarray2.c
	selector_table.c
	sendmsg2.c
	)
set(libobjc_HDRS
	objc/Availability.h
	objc/Object.h
	objc/Protocol.h
	objc/blocks_private.h
	objc/blocks_runtime.h
	objc/capabilities.h
	objc/developer.h
	objc/encoding.h
	objc/hooks.h
	objc/message.h
	objc/objc-api.h
	objc/objc-arc.h
	objc/objc-auto.h
	objc/objc-class.h
	objc/objc-runtime.h
	objc/objc.h
	objc/runtime-deprecated.h
	objc/runtime.h
	objc/slot.h)

# Windows does not use DWARF EH
if (WIN32)
	list(APPEND libobjc_CXX_SRCS eh_win32_msvc.cc)
else ()
	list(APPEND libobjc_C_SRCS eh_personality.c)
	set(libobjcxx_CXX_SRCS objcxx_eh.cc)
endif (WIN32)


# For release builds, we disable spamming the terminal with warnings about
# selector type mismatches
if (CMAKE_BUILD_TYPE STREQUAL Release)
	add_definitions(-DNO_SELECTOR_MISMATCH_WARNINGS)
else ()
	add_definitions(-DGC_DEBUG)
endif ()

set(TYPE_DEPENDENT_DISPATCH TRUE CACHE BOOL
	"Enable type-dependent dispatch")
if (TYPE_DEPENDENT_DISPATCH)
	add_definitions(-DTYPE_DEPENDENT_DISPATCH)
endif ()
set(ENABLE_TRACING FALSE CACHE BOOL
	"Enable tracing support (slower, not recommended for deployment)")
if (ENABLE_TRACING)
	add_definitions(-DWITH_TRACING=1)
endif (ENABLE_TRACING)

set(BOEHM_GC FALSE CACHE BOOL
	"Enable garbage collection support (not recommended)")
if (BOEHM_GC)
	include(FindPkgConfig)
	pkg_check_modules(GC REQUIRED bdw-gc)
	link_directories(${GC_LIBRARY_DIRS})
	# If there's a threaded version, use it
	find_library(LIBGC gc-threaded PATHS "${GC_LIBRARY_DIRS}")
	if (LIBGC)
	else ()
		find_library(LIBGC gc PATHS GC_LIBRARY_DIRS)
	endif ()
	message(STATUS "Using Boehm GC library: ${LIBGC}")
	include_directories(GC_INCLUDE_DIRS)
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${GC_CFLAGS}")
	set(CMAKE_OBJC_FLAGS "${CMAKE_OBJC_FLAGS} -fobjc-gc")
	set(objc_LINK_FLAGS "${objc_LINK_FLAGS} ${GC_CFLAGS}")
	add_definitions(-DENABLE_GC)
	list(APPEND libobjc_OBJC_SRCS gc_boehm.c)
else ()
	list(APPEND libobjc_OBJC_SRCS gc_none.c)
endif ()

set(OLDABI_COMPAT TRUE CACHE BOOL
	"Enable compatibility with GCC and old GNUstep ABIs")

set(LEGACY_COMPAT FALSE CACHE BOOL
	"Enable legacy compatibility features")

if (OLDABI_COMPAT)
	list(APPEND libobjc_C_SRCS legacy.c abi_version.c statics_loader.c)
	add_definitions(-DOLDABI_COMPAT=1)
endif()

if (LEGACY_COMPAT)
	list(APPEND libobjc_C_SRCS legacy_malloc.c)
else ()
	add_definitions(-DNO_LEGACY)
endif ()

set(LIBOBJC_NAME "objc" CACHE STRING 
	"Name of the Objective-C runtime library (e.g. objc2 for libobjc2)")

set(INCLUDE_DIRECTORY "objc" CACHE STRING 
	"Subdirectory of the include path to install the headers.")


if (${CMAKE_C_COMPILER_ID} MATCHES Clang*)
	set(CMAKE_OBJC_FLAGS "${CMAKE_OBJC_FLAGS} -Wno-deprecated-objc-isa-usage -Wno-objc-root-class")
	if (${CMAKE_C_COMPILER_VERSION} VERSION_GREATER 3.1)
		set(CMAKE_OBJC_FLAGS "${CMAKE_OBJC_FLAGS} -fobjc-runtime=gnustep-2.0")
	endif ()
else (${CMAKE_C_COMPILER_ID} MATCHES Clang*)
	MESSAGE("WARNING: It is strongly recommended that you compile with clang")
endif (${CMAKE_C_COMPILER_ID} MATCHES Clang*)

if (${CMAKE_SYSTEM_PROCESSOR} STREQUAL "i686")
	set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -march=i586")
endif ()

set(INSTALL_TARGETS objc)

# On Windows, CMake adds /TC to the clang-cl flags and doesn't provide a way to
# tell it not to.  We fix this by telling clang do disregard that option,
# unconditionally (which means that it still defaults to C for .c files).
set(ENV{CCC_OVERRIDE_OPTIONS} "x/TC x/Gm-")

set_source_files_properties(
	${libobjc_ASM_SRCS}
	LANGUAGE C
	COMPILE_FLAGS "${CMAKE_OBJC_FLAGS} -Xclang -x -Xclang assembler-with-cpp"
)

set_source_files_properties(
	${libobjc_CXX_SRCS}
	LANGUAGE CXX
	COMPILE_FLAGS "${CMAKE_CXX_FLAGS}"
)

set_source_files_properties(
	${libobjc_OBJC_SRCS}
	COMPILE_FLAGS "${CMAKE_OBJC_FLAGS} -Xclang -x -Xclang objective-c"
)

#
# C++ Runtime interaction
#


function(test_cxx CXX_RUNTIME_NAME IS_STDLIB)
	set(CXX_RUNTIME_NAME "${CMAKE_SHARED_LIBRARY_PREFIX}${CXX_RUNTIME_NAME}${CMAKE_SHARED_LIBRARY_SUFFIX}")
	find_library(CXX_RUNTIME_LIB NAMES ${CXX_RUNTIME_NAME})
	if (CXX_RUNTIME_LIB)
		message(STATUS "Testing ${CXX_RUNTIME_LIB} as the C++ runtime library")
		try_compile(USERUNTIME 
			"${CMAKE_BINARY_DIR}/CMake"
			"${CMAKE_SOURCE_DIR}/CMake"
			test_cxx_runtime
			CMAKE_FLAGS "-DCXX_RUNTIME=${CXX_RUNTIME_LIB}")
		if (USERUNTIME)
			set(CXX_RUNTIME ${CXX_RUNTIME_LIB} PARENT_SCOPE)
		endif()
	endif()
endfunction()

set(ENABLE_OBJCXX true CACHE BOOL
	"Enable support for Objective-C++")

set(CXXRT_IS_STDLIB false)

if(WIN32)
	if(CMAKE_SIZEOF_VOID_P EQUAL 8)
		set(ASM_TARGET -m64)
	else()
		set(ASM_TARGET -m32)
	endif()
endif()


if (MSVC)
	# CMake is completely broken when you try to build assembly files on Windows.
	add_custom_command(OUTPUT block_trampolines.obj
		COMMAND echo ${CMAKE_C_COMPILER} ${ASM_TARGET} -c "${CMAKE_SOURCE_DIR}/block_trampolines.S" -o "${CMAKE_BINARY_DIR}/block_trampolines.obj"
		COMMAND ${CMAKE_C_COMPILER} ${ASM_TARGET} -c "${CMAKE_SOURCE_DIR}/block_trampolines.S" -o "${CMAKE_BINARY_DIR}/block_trampolines.obj"
		MAIN_DEPENDENCY block_trampolines.S
	)
	add_custom_command(OUTPUT objc_msgSend.obj
		COMMAND echo ${CMAKE_C_COMPILER} ${ASM_TARGET} -c "${CMAKE_SOURCE_DIR}/objc_msgSend.S" -o "${CMAKE_BINARY_DIR}/objc_msgSend.obj"
		COMMAND ${CMAKE_C_COMPILER} ${ASM_TARGET} -c "${CMAKE_SOURCE_DIR}/objc_msgSend.S" -o "${CMAKE_BINARY_DIR}/objc_msgSend.obj"
		MAIN_DEPENDENCY objc_msgSend.S
		DEPENDS objc_msgSend.aarch64.S objc_msgSend.arm.S objc_msgSend.mips.S objc_msgSend.x86-32.S objc_msgSend.x86-64.S
	)
	set(libobjc_ASM_OBJS block_trampolines.obj objc_msgSend.obj)
endif()


add_library(objc SHARED ${libobjc_C_SRCS} ${libobjc_ASM_SRCS} ${libobjc_OBJC_SRCS} ${libobjc_ASM_OBJS})

if (ENABLE_OBJCXX)
	if (WIN32)
		message(STATUS "Using MSVC-compatible exception model")
	else ()
		message(STATUS "Testing C++ interop")
		# Try to find libcxxrt.so.  We can link to this to provide the C++ ABI
		# layer, if it exists.
		test_cxx(cxxrt false)
		# If it doesn't, then look for GNU libsupc++.so instead (either works,
		# they're ABI compatible).
		if (NOT CXX_RUNTIME)
			test_cxx(supc++ false)
		endif (NOT CXX_RUNTIME)
		if (NOT CXX_RUNTIME)
			test_cxx(c++abi false)
		endif (NOT CXX_RUNTIME)

		# If we have a C++ ABI library, then we can produce a single libobjc that
		# works for Objective-C and Objective-C++.  If not, then we need to provide
		# a separate libobjcxx.
		if (CXX_RUNTIME)
			message(STATUS "Using ${CXX_RUNTIME} as the C++ runtime library")
		else()
			message(STATUS "Testing C++ standard library")
			try_compile(USERUNTIME 
				"${CMAKE_BINARY_DIR}/CMake"
				"${CMAKE_SOURCE_DIR}/CMake"
				test_cxx_runtime)
			if (${USERUNTIME})
				message(STATUS "libobjc will depend on C++ standard library")
				set(CXXRT_IS_STDLIB true)
			else()
				message(STATUS "No useable C++ runtime found")
				set(ENABLE_OBJCXX false)
			endif()
		endif ()
	endif ()
endif (ENABLE_OBJCXX)


if (ENABLE_OBJCXX)
	if (NOT CXXRT_IS_STDLIB)
		# We don't want to link the STL implementation (e.g. libstdc++) if
		# we have a separate C++ runtime.
		set(CMAKE_CXX_IMPLICIT_LINK_LIBRARIES "")
		target_link_libraries(objc ${CXX_RUNTIME})
	endif()
	list(APPEND libobjc_CXX_SRCS ${libobjcxx_CXX_SRCS})
	target_sources(objc PRIVATE ${libobjc_CXX_SRCS})
endif()


# Currently, we actually need pthreads, but we should use the platform's native
# threading implementation (we do for everything except thread-local storage)
set(CMAKE_THREAD_PREFER_PTHREAD)
include(FindThreads)
set(objc_LINK_FLAGS "${objc_LINK_FLAGS} ${CMAKE_THREAD_LIBS_INIT}")




set_target_properties(objc PROPERTIES
	LINKER_LANGUAGE C
	SOVERSION ${libobjc_VERSION}
	OUTPUT_NAME ${LIBOBJC_NAME}
	LINK_FLAGS "${objc_LINK_FLAGS}"
	)

set_property(TARGET PROPERTY NO_SONAME true)

set(BUILD_STATIC_LIBOBJC false CACHE BOOL
	"Build the static version of libobjc")
if (BUILD_STATIC_LIBOBJC)
	add_library(objc-static STATIC ${libobjc_C_SRCS} ${libobjc_ASM_SRCS} ${libobjc_OBJC_SRCS} ${libobjc_CXX_SRCS})
	set_target_properties(objc-static PROPERTIES
		POSITION_INDEPENDENT_CODE true
		OUTPUT_NAME ${LIBOBJC_NAME})
	list(APPEND INSTALL_TARGETS objc-static)
endif ()

# Explicitly link libgc if we are compiling with gc support.
if (LIBGC)
	target_link_libraries(objc ${LIBGC})
endif ()

# Make weak symbols work on OS X
if (APPLE)
	set(CMAKE_SHARED_LIBRARY_CREATE_C_FLAGS
		"${CMAKE_SHARED_LIBRARY_CREATE_C_FLAGS} -undefined dynamic_lookup")
	set(CMAKE_C_LINK_FLAGS "${CMAKE_C_LINK_FLAGS} -Wl,-undefined,dynamic_lookup")
	set(CMAKE_CXX_LINK_FLAGS "${CMAKE_CXX_LINK_FLAGS} -Wl,-undefined,dynamic_lookup")
endif ()

#
# Installation
#


find_program(GNUSTEP_CONFIG gnustep-config)
if (GNUSTEP_CONFIG)
	EXEC_PROGRAM(gnustep-config
		ARGS "--installation-domain-for=libobjc2"
		OUTPUT_VARIABLE DEFAULT_INSTALL_TYPE)
endif ()


# If we have GNUstep environment variables, then default to installing in the
# GNUstep local environment.
if (DEFAULT_INSTALL_TYPE)
else ()
	set(DEFAULT_INSTALL_TYPE "NONE")
endif ()

if (NOT CMAKE_INSTALL_LIBDIR)
	set(CMAKE_INSTALL_LIBDIR lib)
endif ()


set(GNUSTEP_INSTALL_TYPE ${DEFAULT_INSTALL_TYPE} CACHE STRING
	"GNUstep installation type.  Options are NONE, SYSTEM, NETWORK or LOCAL.")
if (${GNUSTEP_INSTALL_TYPE} STREQUAL "NONE")
	SET(LIB_INSTALL_PATH "${CMAKE_INSTALL_LIBDIR}" CACHE STRING
		"Subdirectory of the root prefix where libraries are installed.")
	SET(HEADER_INSTALL_PATH "include")
else ()
	EXEC_PROGRAM(gnustep-config
		ARGS "--variable=GNUSTEP_${GNUSTEP_INSTALL_TYPE}_LIBRARIES"
		OUTPUT_VARIABLE LIB_INSTALL_PATH)
	EXEC_PROGRAM(gnustep-config
		ARGS "--variable=GNUSTEP_${GNUSTEP_INSTALL_TYPE}_HEADERS"
		OUTPUT_VARIABLE HEADER_INSTALL_PATH)
endif ()
message(STATUS "GNUstep install type set to ${GNUSTEP_INSTALL_TYPE}")

install(TARGETS ${INSTALL_TARGETS}
	LIBRARY DESTINATION ${LIB_INSTALL_PATH}
	ARCHIVE DESTINATION ${LIB_INSTALL_PATH})
install(FILES ${libobjc_HDRS}
	DESTINATION "${HEADER_INSTALL_PATH}/${INCLUDE_DIRECTORY}")

install_symlink(${HEADER_INSTALL_PATH}/${INCLUDE_DIRECTORY}/blocks_runtime.h ${HEADER_INSTALL_PATH}/Block.h)
install_symlink(${HEADER_INSTALL_PATH}/${INCLUDE_DIRECTORY}/blocks_private.h ${HEADER_INSTALL_PATH}/Block_private.h)

set(CPACK_GENERATOR TGZ CACHE STRING
	"Installer types to generate.  Sensible options include TGZ, RPM and DEB")

set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "GNUstep Objective-C Runtime")
set(CPACK_PACKAGE_VENDOR "The GNUstep Project")
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/COPYING")
set(CPACK_PACKAGE_VERSION_MAJOR "2")
set(CPACK_PACKAGE_VERSION_MINOR "0")
set(CPACK_PACKAGE_VERSION_PATCH "0")
set(CPACK_PACKAGE_CONTACT "GNUstep Developer <gnustep-dev@gnu.org>")
set(CPACK_PACKAGE_INSTALL_DIRECTORY "CMake ${CMake_VERSION_MAJOR}.${CMake_VERSION_MINOR}")
if (UNIX)
	set(CPACK_STRIP_FILES true CACHE BOOL "Strip libraries when packaging")
endif ()
include (CPack)

# uninstall target
configure_file(
	"${CMAKE_CURRENT_SOURCE_DIR}/cmake_uninstall.cmake.in"
	"${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
	IMMEDIATE @ONLY)

add_custom_target(uninstall
	COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)


set(TESTS TRUE CACHE BOOL
	"Enable building the tests")

if (TESTS)
	enable_testing()
	add_subdirectory(Test)
endif (TESTS)

CHECK_CXX_SOURCE_COMPILES("
	#include <stdlib.h>
	extern \"C\" {
	__attribute__((weak))
	void *__cxa_allocate_exception(size_t thrown_size) noexcept;
	}
	#include <exception>
	int main() { return 0; }" CXA_ALLOCATE_EXCEPTION_NOEXCEPT_COMPILES)

if (CXA_ALLOCATE_EXCEPTION_NOEXCEPT_COMPILES)
	add_definitions(-DCXA_ALLOCATE_EXCEPTION_SPECIFIER=noexcept)
else ()
	add_definitions(-DCXA_ALLOCATE_EXCEPTION_SPECIFIER=)
endif ()
