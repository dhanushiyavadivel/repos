# CMake build script for the libgit2 project
#
# Building (out of source build):
# > mkdir build && cd build
# > cmake .. [-DSETTINGS=VALUE]
# > cmake --build .
#
# Testing:
# > ctest -V
#
# Install:
# > cmake --build . --target install

PROJECT(libgit2 C)
CMAKE_MINIMUM_REQUIRED(VERSION 2.8.11)
CMAKE_POLICY(SET CMP0015 NEW)
IF (POLICY CMP0051)
	CMAKE_POLICY(SET CMP0051 NEW)
ENDIF()
IF (POLICY CMP0042)
	CMAKE_POLICY(SET CMP0042 NEW)
ENDIF()

# Add find modules to the path
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${libgit2_SOURCE_DIR}/cmake/Modules/")

INCLUDE(CheckLibraryExists)
INCLUDE(CheckFunctionExists)
INCLUDE(CheckSymbolExists)
INCLUDE(CheckStructHasMember)
INCLUDE(CheckPrototypeDefinition) # Added in CMake 3.0
INCLUDE(AddCFlagIfSupported)
INCLUDE(FindPkgLibraries)
INCLUDE(FindThreads)
INCLUDE(FindStatNsec)
INCLUDE(IdeSplitSources)
INCLUDE(FeatureSummary)
INCLUDE(EnableWarnings)

# Build options
#
OPTION(SONAME				"Set the (SO)VERSION of the target"			 ON)
OPTION(BUILD_SHARED_LIBS		"Build Shared Library (OFF for Static)"			 ON)
OPTION(THREADSAFE			"Build libgit2 as threadsafe"				 ON)
OPTION(BUILD_CLAR			"Build Tests using the Clar suite"			 ON)
OPTION(BUILD_EXAMPLES			"Build library usage example apps"			OFF)
OPTION(BUILD_FUZZERS			"Build the fuzz targets"				OFF)
OPTION(TAGS				"Generate tags"						OFF)
OPTION(PROFILE				"Generate profiling information"			OFF)
OPTION(ENABLE_TRACE			"Enables tracing support"				OFF)
OPTION(LIBGIT2_FILENAME			"Name of the produced binary"				OFF)

   SET(SHA1_BACKEND 			"CollisionDetection"			       CACHE STRING
       "Backend to use for SHA1. One of Generic, OpenSSL, Win32, CommonCrypto, mbedTLS, CollisionDetection.")
OPTION(USE_SSH				"Link with libssh to enable SSH support"		 ON)
OPTION(USE_HTTPS			"Enable HTTPS support. Can be set to a specific backend" ON)
OPTION(USE_GSSAPI			"Link with libgssapi for SPNEGO auth"			OFF)
OPTION(USE_STANDALONE_FUZZERS		"Enable standalone fuzzers (compatible with gcc)"	OFF)
OPTION(VALGRIND				"Configure build for valgrind"				OFF)
OPTION(CURL				"Use curl for HTTP if available"			 ON)
OPTION(USE_EXT_HTTP_PARSER		"Use system HTTP_Parser if available"			 ON)
OPTION(DEBUG_POOL			"Enable debug pool allocator"				OFF)
OPTION(ENABLE_WERROR			"Enable compilation with -Werror"			OFF)
OPTION(USE_BUNDLED_ZLIB    		"Use the bundled version of zlib"			OFF)
   SET(CLAR_XML 			"OFF"					       CACHE STRING
       "Writes test results in XML format.  One of ON, OFF or the directory to write to; this does not affect the output executables, this only affects the behavior of the ctest command.")

IF (UNIX AND NOT APPLE)
	OPTION(ENABLE_REPRODUCIBLE_BUILDS "Enable reproducible builds"				OFF)
ENDIF()

IF (APPLE)
	OPTION(USE_ICONV		"Link with and use iconv library"			 ON)
ENDIF()

IF(MSVC)
	# This option is only available when building with MSVC. By default, libgit2
	# is build using the cdecl calling convention, which is useful if you're
	# writing C. However, the CLR and Win32 API both expect stdcall.
	#
	# If you are writing a CLR program and want to link to libgit2, you'll want
	# to turn this on by invoking CMake with the "-DSTDCALL=ON" argument.
	OPTION(STDCALL			"Build libgit2 with the __stdcall convention"		OFF)

	# This option must match the settings used in your program, in particular if you
	# are linking statically
	OPTION(STATIC_CRT		"Link the static CRT libraries"		 		 ON)

	# If you want to embed a copy of libssh2 into libgit2, pass a
	# path to libssh2
	OPTION(EMBED_SSH_PATH		"Path to libssh2 to embed (Windows)"			OFF)
ENDIF()


IF(WIN32)
	# By default, libgit2 is built with WinHTTP.  To use the built-in
	# HTTP transport, invoke CMake with the "-DWINHTTP=OFF" argument.
	OPTION(WINHTTP			"Use Win32 WinHTTP routines"				 ON)
ENDIF()

IF(MSVC)
	# Enable MSVC CRTDBG memory leak reporting when in debug mode.
	OPTION(MSVC_CRTDBG 		"Enable CRTDBG memory leak reporting"			OFF)
ENDIF()

FILE(STRINGS "${libgit2_SOURCE_DIR}/include/git2/version.h" GIT2_HEADER REGEX "^#define LIBGIT2_VERSION \"[^\"]*\"$")

STRING(REGEX REPLACE "^.*LIBGIT2_VERSION \"([0-9]+).*$" "\\1" LIBGIT2_VERSION_MAJOR "${GIT2_HEADER}")
STRING(REGEX REPLACE "^.*LIBGIT2_VERSION \"[0-9]+\\.([0-9]+).*$" "\\1" LIBGIT2_VERSION_MINOR  "${GIT2_HEADER}")
STRING(REGEX REPLACE "^.*LIBGIT2_VERSION \"[0-9]+\\.[0-9]+\\.([0-9]+).*$" "\\1" LIBGIT2_VERSION_REV "${GIT2_HEADER}")
SET(LIBGIT2_VERSION_STRING "${LIBGIT2_VERSION_MAJOR}.${LIBGIT2_VERSION_MINOR}.${LIBGIT2_VERSION_REV}")

FILE(STRINGS "${libgit2_SOURCE_DIR}/include/git2/version.h" GIT2_HEADER_SOVERSION REGEX "^#define LIBGIT2_SOVERSION [0-9]+$")
STRING(REGEX REPLACE "^.*LIBGIT2_SOVERSION ([0-9]+)$" "\\1" LIBGIT2_SOVERSION "${GIT2_HEADER_SOVERSION}")

# Platform specific compilation flags
IF (MSVC)
	ADD_DEFINITIONS(-D_SCL_SECURE_NO_WARNINGS)
	ADD_DEFINITIONS(-D_CRT_SECURE_NO_DEPRECATE)
	ADD_DEFINITIONS(-D_CRT_NONSTDC_NO_DEPRECATE)

	STRING(REPLACE "/Zm1000" " " CMAKE_C_FLAGS "${CMAKE_C_FLAGS}")

	# /GF - String pooling
	# /MP - Parallel build
	SET(CMAKE_C_FLAGS "/GF /MP /nologo ${CMAKE_C_FLAGS}")

	IF (STDCALL)
		# /Gz - stdcall calling convention
		SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} /Gz")
	ENDIF ()

	IF (STATIC_CRT)
		SET(CRT_FLAG_DEBUG "/MTd")
		SET(CRT_FLAG_RELEASE "/MT")
	ELSE()
		SET(CRT_FLAG_DEBUG "/MDd")
		SET(CRT_FLAG_RELEASE "/MD")
	ENDIF()

	IF (MSVC_CRTDBG)
		SET(GIT_MSVC_CRTDBG 1)
		SET(CRT_FLAG_DEBUG "${CRT_FLAG_DEBUG}")
		SET(CMAKE_C_STANDARD_LIBRARIES "${CMAKE_C_STANDARD_LIBRARIES} Dbghelp.lib")
	ENDIF()

	# /Zi - Create debugging information
	# /Od - Disable optimization
	# /D_DEBUG - #define _DEBUG
	# /MTd - Statically link the multithreaded debug version of the CRT
	# /MDd - Dynamically link the multithreaded debug version of the CRT
	# /RTC1 - Run time checks
	SET(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} /Zi /Od /D_DEBUG /RTC1 ${CRT_FLAG_DEBUG}")

	# /DNDEBUG - Disables asserts
	# /MT - Statically link the multithreaded release version of the CRT
	# /MD - Dynamically link the multithreaded release version of the CRT
	# /O2 - Optimize for speed
	# /Oy - Enable frame pointer omission (FPO) (otherwise CMake will automatically turn it off)
	# /GL - Link time code generation (whole program optimization)
	# /Gy - Function-level linking
	SET(CMAKE_C_FLAGS_RELEASE "/DNDEBUG /O2 /Oy /GL /Gy ${CRT_FLAG_RELEASE}")

	# /Oy- - Disable frame pointer omission (FPO)
	SET(CMAKE_C_FLAGS_RELWITHDEBINFO "/DNDEBUG /Zi /O2 /Oy- /GL /Gy ${CRT_FLAG_RELEASE}")

	# /O1 - Optimize for size
	SET(CMAKE_C_FLAGS_MINSIZEREL "/DNDEBUG /O1 /Oy /GL /Gy ${CRT_FLAG_RELEASE}")

	# /DYNAMICBASE - Address space load randomization (ASLR)
	# /NXCOMPAT - Data execution prevention (DEP)
	# /LARGEADDRESSAWARE - >2GB user address space on x86
	# /VERSION - Embed version information in PE header
	SET(CMAKE_EXE_LINKER_FLAGS "/DYNAMICBASE /NXCOMPAT /LARGEADDRESSAWARE /VERSION:${LIBGIT2_VERSION_MAJOR}.${LIBGIT2_VERSION_MINOR}")

	# /DEBUG - Create a PDB
	# /LTCG - Link time code generation (whole program optimization)
	# /OPT:REF /OPT:ICF - Fold out duplicate code at link step
	# /INCREMENTAL:NO - Required to use /LTCG
	# /DEBUGTYPE:cv,fixup - Additional data embedded in the PDB (requires /INCREMENTAL:NO, so not on for Debug)
	SET(CMAKE_EXE_LINKER_FLAGS_DEBUG "/DEBUG")
	SET(CMAKE_EXE_LINKER_FLAGS_RELEASE "/RELEASE /LTCG /OPT:REF /OPT:ICF /INCREMENTAL:NO")
	SET(CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO "/DEBUG /RELEASE /LTCG /OPT:REF /OPT:ICF /INCREMENTAL:NO /DEBUGTYPE:cv,fixup")
	SET(CMAKE_EXE_LINKER_FLAGS_MINSIZEREL "/RELEASE /LTCG /OPT:REF /OPT:ICF /INCREMENTAL:NO")

	# Same linker settings for DLL as EXE
	SET(CMAKE_SHARED_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS}")
	SET(CMAKE_SHARED_LINKER_FLAGS_DEBUG "${CMAKE_EXE_LINKER_FLAGS_DEBUG}")
	SET(CMAKE_SHARED_LINKER_FLAGS_RELEASE "${CMAKE_EXE_LINKER_FLAGS_RELEASE}")
	SET(CMAKE_SHARED_LINKER_FLAGS_RELWITHDEBINFO "${CMAKE_EXE_LINKER_FLAGS_RELWITHDEBINFO}")
	SET(CMAKE_SHARED_LINKER_FLAGS_MINSIZEREL "${CMAKE_EXE_LINKER_FLAGS_MINSIZEREL}")
ELSE ()
	IF (ENABLE_REPRODUCIBLE_BUILDS)
		SET(CMAKE_C_ARCHIVE_CREATE "<CMAKE_AR> Dqc <TARGET> <LINK_FLAGS> <OBJECTS>")
		SET(CMAKE_C_ARCHIVE_APPEND "<CMAKE_AR> Dq  <TARGET> <LINK_FLAGS> <OBJECTS>")
		SET(CMAKE_C_ARCHIVE_FINISH "<CMAKE_RANLIB> -D <TARGET>")
	ENDIF()

	SET(CMAKE_C_FLAGS "-D_GNU_SOURCE ${CMAKE_C_FLAGS}")

	ENABLE_WARNINGS(all)
	ENABLE_WARNINGS(extra)

	IF (CMAKE_SYSTEM_NAME MATCHES "(Solaris|SunOS)")
		SET(CMAKE_C_FLAGS "-D_POSIX_C_SOURCE=200112L -D__EXTENSIONS__ -D_POSIX_PTHREAD_SEMANTICS ${CMAKE_C_FLAGS}")
	ENDIF()

	SET(CMAKE_C_FLAGS_DEBUG "${CMAKE_C_FLAGS_DEBUG} -D_DEBUG -O0")

	IF (MINGW OR MSYS) # MinGW and MSYS always do PIC and complain if we tell them to
		STRING(REGEX REPLACE "-fPIC" "" CMAKE_SHARED_LIBRARY_C_FLAGS "${CMAKE_SHARED_LIBRARY_C_FLAGS}")
	ELSEIF (BUILD_SHARED_LIBS)
		ADD_C_FLAG_IF_SUPPORTED(-fvisibility=hidden)

		SET(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -fPIC")
	ENDIF ()

	IF (MINGW)
		# MinGW >= 3.14 uses the C99-style stdio functions
		# automatically, but forks like mingw-w64 still want
		# us to define this in order to use them
		ADD_DEFINITIONS(-D__USE_MINGW_ANSI_STDIO=1)
	ENDIF ()

	ENABLE_WARNINGS(documentation)
	DISABLE_WARNINGS(missing-field-initializers)
	ENABLE_WARNINGS(strict-aliasing)
	ENABLE_WARNINGS(strict-prototypes)
	ENABLE_WARNINGS(declaration-after-statement)
	ENABLE_WARNINGS(shift-count-overflow)
	DISABLE_WARNINGS(unused-const-variable)
	DISABLE_WARNINGS(unused-function)
	ENABLE_WARNINGS(format)
	ENABLE_WARNINGS(format-security)

	IF (APPLE) # Apple deprecated OpenSSL
	    DISABLE_WARNINGS(deprecated-declarations)
	ENDIF()

	IF (PROFILE)
		SET(CMAKE_C_FLAGS "-pg ${CMAKE_C_FLAGS}")
		SET(CMAKE_EXE_LINKER_FLAGS "-pg ${CMAKE_EXE_LINKER_FLAGS}")
	ENDIF ()
ENDIF()

IF( NOT CMAKE_CONFIGURATION_TYPES )
	# Build Debug by default
	IF (NOT CMAKE_BUILD_TYPE)
		SET(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Choose the type of build, options are: Debug Release RelWithDebInfo MinSizeRel." FORCE)
	ENDIF ()
ELSE()
	# Using a multi-configuration generator eg MSVC or Xcode
	# that uses CMAKE_CONFIGURATION_TYPES and not CMAKE_BUILD_TYPE
ENDIF()

IF(BUILD_FUZZERS AND NOT USE_STANDALONE_FUZZERS)
	# The actual sanitizer link target will be added when linking the fuzz
	# targets.
	SET(CMAKE_REQUIRED_FLAGS "-fsanitize=fuzzer-no-link")
	ADD_C_FLAG(-fsanitize=fuzzer-no-link)
	UNSET(CMAKE_REQUIRED_FLAGS)
ENDIF ()

ADD_SUBDIRECTORY(src)

# Tests
IF (NOT MSVC)
   IF (NOT BUILD_SHARED_LIBS)
       SET(CMAKE_FIND_LIBRARY_SUFFIXES ".a")
   ENDIF()
ENDIF ()

IF (BUILD_CLAR)
	ENABLE_TESTING()
	ADD_SUBDIRECTORY(tests)
ENDIF ()

IF (TAGS)
	FIND_PROGRAM(CTAGS ctags)
	IF (NOT CTAGS)
		MESSAGE(FATAL_ERROR "Could not find ctags command")
	ENDIF ()

	FILE(GLOB_RECURSE SRC_ALL *.[ch])

	ADD_CUSTOM_COMMAND(
		OUTPUT tags
		COMMAND ${CTAGS} -a ${SRC_ALL}
		DEPENDS ${SRC_ALL}
	)
	ADD_CUSTOM_TARGET(
		do_tags ALL
		DEPENDS tags
	)
ENDIF ()

IF (BUILD_EXAMPLES)
	ADD_SUBDIRECTORY(examples)
ENDIF ()

IF(BUILD_FUZZERS)
	IF(NOT USE_STANDALONE_FUZZERS)
		IF(BUILD_EXAMPLES)
			MESSAGE(FATAL_ERROR "Cannot build the fuzzer targets and the examples together")
		ENDIF()
		IF(BUILD_CLAR)
			MESSAGE(FATAL_ERROR "Cannot build the fuzzer targets and the tests together")
		ENDIF()
	ENDIF()
	ADD_SUBDIRECTORY(fuzzers)
ENDIF()

IF(CMAKE_VERSION VERSION_GREATER 3)
	FEATURE_SUMMARY(WHAT ENABLED_FEATURES DESCRIPTION "Enabled features:")
	FEATURE_SUMMARY(WHAT DISABLED_FEATURES DESCRIPTION "Disabled features:")
ELSE()
	PRINT_ENABLED_FEATURES()
	PRINT_DISABLED_FEATURES()
ENDIF()
