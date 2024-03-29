# MojoSetup; a portable, flexible installation application.
#
# Please see the file LICENSE.txt in the source's root directory.
#
#  This file written by Ryan C. Gordon.



# The "BINARY SIZE +=" comments note about how much bulk, in kilobytes, a
#  given option adds to the binary on x86 Linux (built with gcc 3.3.6
#  MinSizeRel options and stripped, uncompressed). These numbers will vary,
#  and even on the original test system, become incorrect over time.
# Only choose options you want/need to squeeze every byte off the download.


# !!! FIXME: this is stupid.
IF(NOT BEOS)
    IF(APPLE)
        PROJECT(MojoSetup)
    ELSE(APPLE)
        PROJECT(MojoSetup C)
    ENDIF(APPLE)
ELSE(NOT BEOS)
    PROJECT(MojoSetup CXX)
ENDIF(NOT BEOS)

CMAKE_MINIMUM_REQUIRED(VERSION 3.10)

EXECUTE_PROCESS(
    COMMAND hg tip --template hg-{rev}:{node|short}
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
    RESULT_VARIABLE HGVERSION_RC
    OUTPUT_VARIABLE MOJOSETUP_VERSION
    ERROR_QUIET
    OUTPUT_STRIP_TRAILING_WHITESPACE
)

IF(HGVERSION_RC)
    SET(MOJOSETUP_VERSION "???")
ENDIF(HGVERSION_RC)

# !!! FIXME: ditch this when Gary's curses patches go into a formal CMake
# !!! FIXME:  release, and just bump the minimum required version to it.
# Search our own cmakemodules first.
SET(CMAKE_MODULE_PATH ${CMAKE_MODULE_PATH} "${PROJECT_SOURCE_DIR}/misc")

# I hate that they define "WIN32" ... we're about to move to Win64...I hope!
IF(WIN32 AND NOT WINDOWS)
    SET(WINDOWS TRUE)
ENDIF(WIN32 AND NOT WINDOWS)

# Bleh, let's do it for "APPLE" too.
IF(APPLE AND NOT MACOSX)
    SET(MACOSX TRUE)
ENDIF(APPLE AND NOT MACOSX)

# And this might be wrong...
IF (CMAKE_SYSTEM MATCHES OS2)
    SET(OS2 TRUE)
ENDIF (CMAKE_SYSTEM MATCHES OS2)

IF(CMAKE_SYSTEM_NAME STREQUAL "SunOS")
    SET(SOLARIS TRUE)
ENDIF(CMAKE_SYSTEM_NAME STREQUAL "SunOS")

INCLUDE(CheckIncludeFile)
INCLUDE(CheckLibraryExists)
INCLUDE(CheckCSourceCompiles)
INCLUDE(CheckCCompilerFlag)
INCLUDE(TestBigEndian)

# See if C99 "restrict" keyword is available to us.
CHECK_C_SOURCE_COMPILES(
    "int test_restrict(const char * restrict ptr); int main(void) { return 0; }"
    MOJOSETUP_HAVE_RESTRICT_KEYWORD
)
IF(NOT MOJOSETUP_HAVE_RESTRICT_KEYWORD)
    # See if __restrict__ keyword is available to us.
    CHECK_C_SOURCE_COMPILES(
        "int test_restrict(const char * __restrict__ ptr); int main(void) { return 0; }"
        MOJOSETUP_HAVE_GNU_RESTRICT_KEYWORD
    )
    #IF(MOJOSETUP_HAVE_GNU_RESTRICT_KEYWORD)
    #    ADD_DEFINITIONS(-Drestrict=__restrict__)
    #ELSE(MOJOSETUP_HAVE_GNU_RESTRICT_KEYWORD)
        ADD_DEFINITIONS("-Drestrict=\"\"")
    #ENDIF(MOJOSETUP_HAVE_GNU_RESTRICT_KEYWORD)
ENDIF(NOT MOJOSETUP_HAVE_RESTRICT_KEYWORD)

ADD_DEFINITIONS(-D__MOJOSETUP__=1)
ADD_DEFINITIONS(-DAPPID=mojosetup)
ADD_DEFINITIONS(-DAPPREV="${MOJOSETUP_VERSION}")
ADD_DEFINITIONS(-D_REENTRANT)
ADD_DEFINITIONS(-D_THREAD_SAFE)

INCLUDE_DIRECTORIES(.)
INCLUDE_DIRECTORIES(lua/src)

IF(WINDOWS)
    ADD_DEFINITIONS(-DPLATFORM_WINDOWS=1)
    ADD_DEFINITIONS(-D_CRT_SECURE_NO_WARNINGS=1)
    SET(USES_WINMAIN WIN32)
    SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} shell32)
ENDIF(WINDOWS)

IF(MACOSX)
    ADD_DEFINITIONS(-DPLATFORM_MACOSX=1)
    IF(CMAKE_OSX_ARCHITECTURES MATCHES ppc)
        ADD_DEFINITIONS(-DMAC_OS_X_VERSION_MIN_REQUIRED=1020)
        SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} "-mmacosx-version-min=10.2")
    ENDIF(CMAKE_OSX_ARCHITECTURES MATCHES ppc)
    SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} "-framework Carbon")
ENDIF(MACOSX)

IF(BEOS)
    ADD_DEFINITIONS(-DPLATFORM_BEOS=1)
ENDIF(BEOS)

IF(UNIX)
    ADD_DEFINITIONS(-D_FILE_OFFSET_BITS=64)
    ADD_DEFINITIONS(-DPLATFORM_UNIX=1)
    ADD_DEFINITIONS(-DLUA_USE_MKSTEMP=1)  # as opposed to tmpnam(), yuck!
    # No _setjmp/_longjmp in BeOS 5 (or Haiku, at the moment).
    IF(NOT BEOS)
        ADD_DEFINITIONS(-DLUA_USE_ULONGJMP=1)
    ENDIF(NOT BEOS)
ENDIF(UNIX)

IF(CMAKE_COMPILER_IS_GNUCC)
    ADD_DEFINITIONS(-pipe -Wall -fsigned-char)

    # See if -fvisibility=hidden is available to us.
    CHECK_C_SOURCE_COMPILES("
        #if ((defined(__GNUC__)) && (__GNUC__ >= 4))
        int main(int argc, char **argv) { int is_gcc4 = 1; return 0; }
        #else
        #error This is not gcc4.
        #endif
    " MOJOSETUP_IS_GCC4)

    IF(MOJOSETUP_IS_GCC4)
        IF(NOT OS2 AND NOT SOLARIS) # Not supported on OS/2 or Solaris.
            ADD_DEFINITIONS(-fvisibility=hidden)
        ENDIF(NOT OS2 AND NOT SOLARIS)
    ENDIF(MOJOSETUP_IS_GCC4)

    # See if -fno-stack-protector is available to us.
    # It doesn't seem to work well, and it adds bulk to the binary.
    CHECK_C_COMPILER_FLAG("-fno-stack-protector" MOJOSETUP_GCC_HAS_STACKPROT)
    IF(MOJOSETUP_GCC_HAS_STACKPROT)
        ADD_DEFINITIONS(-fno-stack-protector)
    ENDIF(MOJOSETUP_GCC_HAS_STACKPROT)

    # !!! FIXME: probably not safe long-term.
    # CMake mailing list had this hack for getting rid of -rdynamic:
    #   http://public.kitware.com/pipermail/cmake/2006-July/010404.html
    IF(CMAKE_SYSTEM_NAME STREQUAL "Linux")
        SET(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS)
        SET(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS)
    ENDIF(CMAKE_SYSTEM_NAME STREQUAL "Linux")

    # Don't use -rpath.
    SET(CMAKE_SKIP_RPATH ON CACHE BOOL "Skip RPATH" FORCE)
ENDIF(CMAKE_COMPILER_IS_GNUCC)

IF(CMAKE_C_COMPILER_ID STREQUAL "SunPro")
    ADD_DEFINITIONS(-erroff=E_EMPTY_TRANSLATION_UNIT)
    ADD_DEFINITIONS(-xldscope=hidden)
ENDIF(CMAKE_C_COMPILER_ID STREQUAL "SunPro")



TEST_BIG_ENDIAN(MOJOSETUP_IS_BIGENDIAN)
IF(MOJOSETUP_IS_BIGENDIAN)
    ADD_DEFINITIONS(-DPLATFORM_BIGENDIAN=1)
ELSE(MOJOSETUP_IS_BIGENDIAN)
    ADD_DEFINITIONS(-DPLATFORM_LITTLEENDIAN=1)
ENDIF(MOJOSETUP_IS_BIGENDIAN)

SET(BZLIB_DIR bzip2)
SET(BZLIB_SRCS
    ${BZLIB_DIR}/blocksort.c
    ${BZLIB_DIR}/huffman.c
    ${BZLIB_DIR}/crctable.c
    ${BZLIB_DIR}/randtable.c
    ${BZLIB_DIR}/compress.c
    ${BZLIB_DIR}/decompress.c
    ${BZLIB_DIR}/bzlib.c
)

SET(LIBLZMA_DIR liblzma)
SET(LIBLZMA_SRCS
    ${LIBLZMA_DIR}/check/check.c
    ${LIBLZMA_DIR}/common/block_buffer_decoder.c
    ${LIBLZMA_DIR}/common/block_decoder.c
    ${LIBLZMA_DIR}/common/block_header_decoder.c
    ${LIBLZMA_DIR}/common/block_util.c
    ${LIBLZMA_DIR}/common/common.c
    ${LIBLZMA_DIR}/common/easy_decoder_memusage.c
    ${LIBLZMA_DIR}/common/easy_preset.c
    ${LIBLZMA_DIR}/common/filter_buffer_decoder.c
    ${LIBLZMA_DIR}/common/filter_common.c
    ${LIBLZMA_DIR}/common/filter_decoder.c
    ${LIBLZMA_DIR}/common/filter_flags_decoder.c
    ${LIBLZMA_DIR}/common/index.c
    ${LIBLZMA_DIR}/common/index_decoder.c
    ${LIBLZMA_DIR}/common/index_hash.c
    ${LIBLZMA_DIR}/common/stream_decoder.c
    ${LIBLZMA_DIR}/common/stream_flags_common.c
    ${LIBLZMA_DIR}/common/stream_flags_decoder.c
    ${LIBLZMA_DIR}/common/vli_decoder.c
    ${LIBLZMA_DIR}/common/vli_size.c
    ${LIBLZMA_DIR}/delta/delta_common.c
    ${LIBLZMA_DIR}/delta/delta_decoder.c
    ${LIBLZMA_DIR}/lz/lz_decoder.c
    ${LIBLZMA_DIR}/lzma/lzma2_decoder.c
    ${LIBLZMA_DIR}/lzma/lzma_decoder.c
    ${LIBLZMA_DIR}/lzma/lzma_encoder_presets.c
    ${LIBLZMA_DIR}/simple/arm.c
    ${LIBLZMA_DIR}/simple/armthumb.c
    ${LIBLZMA_DIR}/simple/ia64.c
    ${LIBLZMA_DIR}/simple/powerpc.c
    ${LIBLZMA_DIR}/simple/simple_coder.c
    ${LIBLZMA_DIR}/simple/simple_decoder.c
    ${LIBLZMA_DIR}/simple/sparc.c
    ${LIBLZMA_DIR}/simple/x86.c
)

SET(LIBFETCH_DIR libfetch)
SET(LIBFETCH_SRCS
    ${LIBFETCH_DIR}/fetch.c
    ${LIBFETCH_DIR}/common.c
    ${LIBFETCH_DIR}/ftp.c
    ${LIBFETCH_DIR}/http.c
)

SET(LUA_DIR lua)
SET(LUA_SRCS
    ${LUA_DIR}/src/lapi.c
    ${LUA_DIR}/src/ldebug.c
    ${LUA_DIR}/src/ldo.c
    ${LUA_DIR}/src/ldump.c
    ${LUA_DIR}/src/lfunc.c
    ${LUA_DIR}/src/lgc.c
    ${LUA_DIR}/src/lmem.c
    ${LUA_DIR}/src/lobject.c
    ${LUA_DIR}/src/lopcodes.c
    ${LUA_DIR}/src/lstate.c
    ${LUA_DIR}/src/lstring.c
    ${LUA_DIR}/src/ltable.c
    ${LUA_DIR}/src/ltm.c
    ${LUA_DIR}/src/lundump.c
    ${LUA_DIR}/src/lvm.c
    ${LUA_DIR}/src/lzio.c
    ${LUA_DIR}/src/lauxlib.c
    ${LUA_DIR}/src/lbaselib.c
    ${LUA_DIR}/src/lstrlib.c
    ${LUA_DIR}/src/ltablib.c
    ${LUA_DIR}/src/lctype.c
)

SET(LUA_PARSER_SRCS
    ${LUA_DIR}/src/lparser.c
    ${LUA_DIR}/src/llex.c
    ${LUA_DIR}/src/lcode.c
)

SET(MOJOSETUP_SRCS
    buildver.c
    mojosetup.c
    gui.c
    fileio.c
    archive_zip.c
    archive_tar.c
    archive_uz2.c
    archive_pck.c
    archive_pkg.c
    checksum_crc32.c
    checksum_md5.c
    checksum_sha1.c
    platform_unix.c
    platform_windows.c
    lua_glue.c
    ${LUA_SRCS}
)

# Have to separate this, so CMake doesn't try to link in C++ support on other
#  platforms.
IF(BEOS)
    SET(MOJOSETUP_SRCS ${MOJOSETUP_SRCS} platform_beos.cpp)
ENDIF(BEOS)

SET(MOJOLUAC_SRCS
    ${LUA_SRCS}
    ${LUA_PARSER_SRCS}
    ${LUA_DIR}/src/luac.c
    ${LUA_DIR}/src/linit.c
    ${LUA_DIR}/src/ldblib.c
    ${LUA_DIR}/src/liolib.c
    ${LUA_DIR}/src/lmathlib.c
    ${LUA_DIR}/src/loslib.c
    ${LUA_DIR}/src/lbitlib.c
    ${LUA_DIR}/src/lcorolib.c
    ${LUA_DIR}/src/loadlib.c
)

SET(STBIMAGE_SRCS
    stb_image.c
)

# Disabling the parser cuts the Lua binary bits by about 35%, plus .luac files
#  are almost always smaller than the original scripts. The downside is you
#  (and end users in the field) can't just tweak a script without recompiling
#  it, but even that's not an unclimbable obstacle.
# In reality, you probably want to keep the parser, though, unless you REALLY
#  must save every single byte in the download.
# YOU NEED THE PARSER IF YOU WANT MANIFESTS WRITTEN OUT.
# YOU NEED THE PARSER IF YOU WANT THE UNINSTALLER TO WORK.
# DON'T DISABLE THIS NOW IF YOU DON'T ABSOLUTELY HAVE TO.
# BINARY SIZE += 19
OPTION(MOJOSETUP_LUA_PARSER "Bigger binary but scripts don't need to be compiled." TRUE)
IF(MOJOSETUP_LUA_PARSER)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LUA_PARSER_SRCS})
ELSE(MOJOSETUP_LUA_PARSER)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LUA_DIR}/etc/noparser.c)
    ADD_DEFINITIONS(-DDISABLE_LUA_PARSER=1)
ENDIF(MOJOSETUP_LUA_PARSER)


# Kludge for Linux x86/amd64 bins...
IF(UNIX AND NOT MACOSX)  # Just use Mach-O Universal/"fat" binaries on OS X.
    OPTION(MOJOSETUP_MULTIARCH "Allow multiarch hack." FALSE)
    MARK_AS_ADVANCED(MOJOSETUP_MULTIARCH)
    IF(MOJOSETUP_MULTIARCH)
        ADD_DEFINITIONS(-DSUPPORT_MULTIARCH=1)
    ENDIF(MOJOSETUP_MULTIARCH)
ENDIF(UNIX AND NOT MACOSX)


# Optional bits of the Lua runtime library...

# BINARY SIZE += 4.5
OPTION(MOJOSETUP_LUALIB_IO "Add Lua 'io' library" TRUE)
MARK_AS_ADVANCED(MOJOSETUP_LUALIB_IO)
IF(MOJOSETUP_LUALIB_IO)
    ADD_DEFINITIONS(-DSUPPORT_LUALIB_IO=1)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LUA_DIR}/src/liolib.c)
ENDIF(MOJOSETUP_LUALIB_IO)

# BINARY SIZE += 2.5
OPTION(MOJOSETUP_LUALIB_OS "Add Lua 'os' library" TRUE)
MARK_AS_ADVANCED(MOJOSETUP_LUALIB_OS)
IF(MOJOSETUP_LUALIB_OS)
    ADD_DEFINITIONS(-DSUPPORT_LUALIB_OS=1)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LUA_DIR}/src/loslib.c)
ENDIF(MOJOSETUP_LUALIB_OS)

# BINARY SIZE += 3.5
OPTION(MOJOSETUP_LUALIB_MATH "Add Lua 'math' library" TRUE)
MARK_AS_ADVANCED(MOJOSETUP_LUALIB_MATH)
IF(MOJOSETUP_LUALIB_MATH)
    ADD_DEFINITIONS(-DSUPPORT_LUALIB_MATH=1)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LUA_DIR}/src/lmathlib.c)
ENDIF(MOJOSETUP_LUALIB_MATH)

# BINARY SIZE += 3.5
OPTION(MOJOSETUP_LUALIB_DB "Add Lua 'db' library" TRUE)
MARK_AS_ADVANCED(MOJOSETUP_LUALIB_DB)
IF(MOJOSETUP_LUALIB_DB)
    ADD_DEFINITIONS(-DSUPPORT_LUALIB_DB=1)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LUA_DIR}/src/ldblib.c)
ENDIF(MOJOSETUP_LUALIB_DB)

# BINARY SIZE += 4
OPTION(MOJOSETUP_LUALIB_PACKAGE "Add Lua 'package' library" TRUE)
MARK_AS_ADVANCED(MOJOSETUP_LUALIB_PACKAGE)
IF(MOJOSETUP_LUALIB_PACKAGE)
    ADD_DEFINITIONS(-DSUPPORT_LUALIB_PACKAGE=1)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LUA_DIR}/src/loadlib.c)
ENDIF(MOJOSETUP_LUALIB_PACKAGE)

# BINARY SIZE += ?
OPTION(MOJOSETUP_LUALIB_BIT "Add Lua 'bit' library" TRUE)
MARK_AS_ADVANCED(MOJOSETUP_LUALIB_BIT)
IF(MOJOSETUP_LUALIB_BIT)
    ADD_DEFINITIONS(-DSUPPORT_LUALIB_BIT=1)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LUA_DIR}/src/lbitlib.c)
ENDIF(MOJOSETUP_LUALIB_BIT)

# BINARY SIZE += ?
OPTION(MOJOSETUP_LUALIB_CORO "Add Lua 'coro' library" TRUE)
MARK_AS_ADVANCED(MOJOSETUP_LUALIB_CORO)
IF(MOJOSETUP_LUALIB_CORO)
    ADD_DEFINITIONS(-DSUPPORT_LUALIB_CORO=1)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LUA_DIR}/src/lcorolib.c)
ENDIF(MOJOSETUP_LUALIB_CORO)


# Checksums...

# BINARY SIZE += !!! FIXME: check this.
OPTION(MOJOSETUP_CHECKSUM_CRC32 "Enable CRC-32 checksum support" TRUE)
IF(MOJOSETUP_CHECKSUM_CRC32)
    ADD_DEFINITIONS(-DSUPPORT_CRC32=1)
ENDIF(MOJOSETUP_CHECKSUM_CRC32)

# BINARY SIZE += !!! FIXME: check this.
OPTION(MOJOSETUP_CHECKSUM_MD5 "Enable MD5 checksum support" TRUE)
IF(MOJOSETUP_CHECKSUM_MD5)
    ADD_DEFINITIONS(-DSUPPORT_MD5=1)
ENDIF(MOJOSETUP_CHECKSUM_MD5)

# BINARY SIZE += !!! FIXME: check this.
OPTION(MOJOSETUP_CHECKSUM_SHA1 "Enable SHA-1 checksum support" TRUE)
IF(MOJOSETUP_CHECKSUM_SHA1)
    ADD_DEFINITIONS(-DSUPPORT_SHA1=1)
ENDIF(MOJOSETUP_CHECKSUM_SHA1)


# GUI plugins...

MACRO(MOJOSETUP_ADD_LIBRARY _TARGET _SRCS)
    ADD_LIBRARY(${_TARGET} SHARED ${_SRCS})
    SET(MOJOSETUP_TARGETS "${MOJOSETUP_TARGETS};${_TARGET}")
ENDMACRO(MOJOSETUP_ADD_LIBRARY)


# BINARY SIZE += 2.5
OPTION(MOJOSETUP_GUI_STDIO "Enable stdio GUI" TRUE)
IF(MOJOSETUP_GUI_STDIO)
    ADD_DEFINITIONS(-DSUPPORT_GUI_STDIO=1)
    OPTION(MOJOSETUP_GUI_STDIO_STATIC "Statically link stdio GUI" TRUE)
    IF(MOJOSETUP_GUI_STDIO_STATIC)
        ADD_DEFINITIONS(-DGUI_STATIC_LINK_STDIO=1)
        SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} gui_stdio.c)
    ELSE(MOJOSETUP_GUI_STDIO_STATIC)
        MOJOSETUP_ADD_LIBRARY(mojosetupgui_stdio gui_stdio.c)
    ENDIF(MOJOSETUP_GUI_STDIO_STATIC)
ENDIF(MOJOSETUP_GUI_STDIO)

# BINARY SIZE += !!! FIXME: check this.
SET(CURSES_NEED_WIDE TRUE)
SET(CURSES_NEED_NCURSES TRUE)
FIND_PACKAGE(Curses)
IF(CURSES_FOUND)
    OPTION(MOJOSETUP_GUI_NCURSES "Enable ncurses GUI" TRUE)
    IF(MOJOSETUP_GUI_NCURSES)
        ADD_DEFINITIONS(-DSUPPORT_GUI_NCURSES=1)
        INCLUDE_DIRECTORIES(${CURSES_INCLUDE_DIR})

        IF(CURSES_HAVE_NCURSESW_NCURSES_H)
            ADD_DEFINITIONS(-DHAVE_NCURSESW_NCURSES_H)
        ELSEIF(CURSES_HAVE_NCURSESW_CURSES_H)
            ADD_DEFINITIONS(-DHAVE_NCURSESW_CURSES_H)
        ELSEIF(CURSES_HAVE_NCURSESW_H)
            ADD_DEFINITIONS(-DHAVE_NCURSESW_H)
        ELSEIF(CURSES_HAVE_CURSES_H)
            ADD_DEFINITIONS(-DHAVE_CURSES_H)
        ENDIF(CURSES_HAVE_NCURSESW_NCURSES_H)

        OPTION(MOJOSETUP_GUI_NCURSES_STATIC "Statically link ncurses GUI" FALSE)
        IF(MOJOSETUP_GUI_NCURSES_STATIC)
            ADD_DEFINITIONS(-DGUI_STATIC_LINK_NCURSES=1)
            SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} gui_ncurses.c)
            SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} ${CURSES_LIBRARIES})
        ELSE(MOJOSETUP_GUI_NCURSES_STATIC)
            MOJOSETUP_ADD_LIBRARY(mojosetupgui_ncurses gui_ncurses.c)
            TARGET_LINK_LIBRARIES(mojosetupgui_ncurses ${CURSES_LIBRARIES})
        ENDIF(MOJOSETUP_GUI_NCURSES_STATIC)
    ENDIF(MOJOSETUP_GUI_NCURSES)
ENDIF(CURSES_FOUND)

IF(MACOSX)
    OPTION(MOJOSETUP_GUI_COCOA "Enable Cocoa GUI" TRUE)
    IF(MOJOSETUP_GUI_COCOA)
        ADD_DEFINITIONS(-DSUPPORT_GUI_COCOA=1)
        OPTION(MOJOSETUP_GUI_COCOA_STATIC "Statically link Cocoa GUI" TRUE)
        IF(MOJOSETUP_GUI_COCOA_STATIC)
            ADD_DEFINITIONS(-DGUI_STATIC_LINK_COCOA=1)
            SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} gui_cocoa.m)
            SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} "-framework Cocoa")
        ELSE(MOJOSETUP_GUI_COCOA_STATIC)
            MOJOSETUP_ADD_LIBRARY(mojosetupgui_cocoa gui_cocoa.m)
            TARGET_LINK_LIBRARIES(mojosetupgui_cocoa
                "-framework Carbon -framework Cocoa -mmacosx-version-min=10.2"
            )
        ENDIF(MOJOSETUP_GUI_COCOA_STATIC)
    ENDIF(MOJOSETUP_GUI_COCOA)
ENDIF(MACOSX)

IF(UNIX)
IF(NOT BEOS)
IF(NOT MACOSX)
FIND_PACKAGE(GTK2 COMPONENTS gtk)
IF(NOT GTK2_FOUND)
    MESSAGE(STATUS "Can't find GTK+v2 headers/libraries. Can't build GTK+ GUI.")
ELSE(NOT GTK2_FOUND)
    # In order to reduce the GTK2 library dependencies at link time, we only link against 'gtk-x11-2.0'.
    # This is more portable, as the dynamic linker/loader will take care of the other library dependencies at run time.
    SET(LIBGTK_LINK_FLAGS "-lgtk-x11-2.0")
    OPTION(MOJOSETUP_GUI_GTKPLUS2 "Enable GTK+ 2.0 GUI" TRUE)
    IF(MOJOSETUP_GUI_GTKPLUS2)
        ADD_DEFINITIONS(-DSUPPORT_GUI_GTKPLUS2=1)
        INCLUDE_DIRECTORIES(${GTK2_INCLUDE_DIRS})
        OPTION(MOJOSETUP_GUI_GTKPLUS2_STATIC "Statically link GTK+ GUI" FALSE)
        IF(MOJOSETUP_GUI_GTKPLUS2_STATIC)
            ADD_DEFINITIONS(-DGUI_STATIC_LINK_GTKPLUS2=1)
            SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} gui_gtkplus2.c)
            SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} ${LIBGTK_LINK_FLAGS})
        ELSE(MOJOSETUP_GUI_GTKPLUS2_STATIC)
            MOJOSETUP_ADD_LIBRARY(mojosetupgui_gtkplus2 gui_gtkplus2.c)
            TARGET_LINK_LIBRARIES(mojosetupgui_gtkplus2 ${LIBGTK_LINK_FLAGS})
        ENDIF(MOJOSETUP_GUI_GTKPLUS2_STATIC)
    ENDIF(MOJOSETUP_GUI_GTKPLUS2)
ENDIF(NOT GTK2_FOUND)
ENDIF(NOT MACOSX)
ENDIF(NOT BEOS)
ENDIF(UNIX)

# BINARY SIZE += !!! FIXME: check this.
OPTION(MOJOSETUP_GUI_WWW "Enable www GUI" FALSE)  # !!! FIXME: make TRUE
IF(MOJOSETUP_GUI_WWW)
    ADD_DEFINITIONS(-DSUPPORT_GUI_WWW=1)
    OPTION(MOJOSETUP_GUI_WWW_STATIC "Statically link www GUI" FALSE)
    IF(MOJOSETUP_GUI_WWW_STATIC)
        ADD_DEFINITIONS(-DGUI_STATIC_LINK_WWW=1)
        SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} gui_www.c)
        IF(WINDOWS)
            SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} wsock32)
        ENDIF(WINDOWS)
    ELSE(MOJOSETUP_GUI_WWW_STATIC)
        MOJOSETUP_ADD_LIBRARY(mojosetupgui_www gui_www.c)
        IF(WINDOWS)
            TARGET_LINK_LIBRARIES(mojosetupgui_www wsock32)
        ENDIF(WINDOWS)
    ENDIF(MOJOSETUP_GUI_WWW_STATIC)
    SET(MOJOSETUP_USES_SOCKETS TRUE)
ENDIF(MOJOSETUP_GUI_WWW)


# Archivers...

# BINARY SIZE += 8
OPTION(MOJOSETUP_ARCHIVE_ZIP "Enable ZIP support" TRUE)
IF(MOJOSETUP_ARCHIVE_ZIP)
    ADD_DEFINITIONS(-DSUPPORT_ZIP=1)
    SET(MOJOSETUP_NEED_ZLIB TRUE)
ENDIF(MOJOSETUP_ARCHIVE_ZIP)

# BINARY SIZE += 2
OPTION(MOJOSETUP_ARCHIVE_TAR "Enable TAR support" TRUE)
IF(MOJOSETUP_ARCHIVE_TAR)
    # !!! FIXME: the gzip/bzip2/xz support doesn't require .tar archives now.
    # !!! FIXME: Maybe not ask here, so the questions look the same everywhere.
    ADD_DEFINITIONS(-DSUPPORT_TAR=1)
    OPTION(MOJOSETUP_ARCHIVE_TAR_GZ "Enable TAR.GZ support" TRUE)
    IF(MOJOSETUP_ARCHIVE_TAR_GZ)
        SET(MOJOSETUP_INPUT_GZIP TRUE)
    ENDIF(MOJOSETUP_ARCHIVE_TAR_GZ)

    OPTION(MOJOSETUP_ARCHIVE_TAR_BZ2 "Enable TAR.BZ2 support" TRUE)
    IF(MOJOSETUP_ARCHIVE_TAR_BZ2)
        SET(MOJOSETUP_INPUT_BZIP2 TRUE)
    ENDIF(MOJOSETUP_ARCHIVE_TAR_BZ2)

    OPTION(MOJOSETUP_ARCHIVE_TAR_XZ "Enable TAR.XZ support" TRUE)
    IF(MOJOSETUP_ARCHIVE_TAR_XZ)
        SET(MOJOSETUP_INPUT_XZ TRUE)
    ENDIF(MOJOSETUP_ARCHIVE_TAR_XZ)
ENDIF(MOJOSETUP_ARCHIVE_TAR)

OPTION(MOJOSETUP_ARCHIVE_UZ2 "Enable UZ2 support" FALSE)
IF(MOJOSETUP_ARCHIVE_UZ2)
    ADD_DEFINITIONS(-DSUPPORT_UZ2=1)
    SET(MOJOSETUP_NEED_ZLIB TRUE)
ENDIF(MOJOSETUP_ARCHIVE_UZ2)

OPTION(MOJOSETUP_ARCHIVE_PCK "Enable PCK support" FALSE)
IF(MOJOSETUP_ARCHIVE_PCK)
    ADD_DEFINITIONS(-DSUPPORT_PCK=1)
	SET(MOJOSETUP_INPUT_GZIP TRUE)
ENDIF(MOJOSETUP_ARCHIVE_PCK)

OPTION(MOJOSETUP_ARCHIVE_PKG "Enable PKG support" FALSE)
IF(MOJOSETUP_ARCHIVE_PKG)
    ADD_DEFINITIONS(-DSUPPORT_PKG=1)
ENDIF(MOJOSETUP_ARCHIVE_PKG)

# Input decoders...

# BINARY SIZE += 1.5
IF(NOT MOJOSETUP_INPUT_GZIP)  # optional if something didn't force it.
    OPTION(MOJOSETUP_INPUT_GZIP "Enable GZIP support" FALSE)
ENDIF(NOT MOJOSETUP_INPUT_GZIP)

IF(MOJOSETUP_INPUT_GZIP)
    ADD_DEFINITIONS(-DSUPPORT_GZIP=1)
    SET(MOJOSETUP_NEED_ZLIB TRUE)
ENDIF(MOJOSETUP_INPUT_GZIP)

# BINARY SIZE += 1.5
IF(NOT MOJOSETUP_INPUT_BZIP2)  # optional if something didn't force it.
    OPTION(MOJOSETUP_INPUT_BZIP2 "Enable BZIP2 support" FALSE)
ENDIF(NOT MOJOSETUP_INPUT_BZIP2)

IF(MOJOSETUP_INPUT_BZIP2)
    ADD_DEFINITIONS(-DSUPPORT_BZIP2=1)
    ADD_DEFINITIONS(-DBZ_NO_STDIO=1)
    SET(MOJOSETUP_NEED_BZLIB TRUE)
ENDIF(MOJOSETUP_INPUT_BZIP2)

# BINARY SIZE += ???
IF(NOT MOJOSETUP_INPUT_XZ)  # optional if something didn't force it.
    OPTION(MOJOSETUP_INPUT_XZ "Enable XZ support" FALSE)
ENDIF(NOT MOJOSETUP_INPUT_XZ)

IF(MOJOSETUP_INPUT_XZ)
    ADD_DEFINITIONS(-DSUPPORT_XZ=1)
    SET(MOJOSETUP_NEED_LIBLZMA TRUE)
ENDIF(MOJOSETUP_INPUT_XZ)


# Image decoders for GUIs...

OPTION(MOJOSETUP_IMAGE_JPG "Enable JPG support" TRUE)
IF(MOJOSETUP_IMAGE_JPG)
    ADD_DEFINITIONS(-DSUPPORT_JPG=1)
    SET(MOJOSETUP_NEED_STBIMAGE TRUE)
ENDIF(MOJOSETUP_IMAGE_JPG)

OPTION(MOJOSETUP_IMAGE_PNG "Enable PNG support" TRUE)
IF(MOJOSETUP_IMAGE_PNG)
    ADD_DEFINITIONS(-DSUPPORT_PNG=1)
    SET(MOJOSETUP_NEED_STBIMAGE TRUE)
ENDIF(MOJOSETUP_IMAGE_PNG)

OPTION(MOJOSETUP_IMAGE_TGA "Enable TGA support" FALSE)
IF(MOJOSETUP_IMAGE_TGA)
    ADD_DEFINITIONS(-DSUPPORT_TGA=1)
    SET(MOJOSETUP_NEED_STBIMAGE TRUE)
ENDIF(MOJOSETUP_IMAGE_TGA)

OPTION(MOJOSETUP_IMAGE_BMP "Enable BMP support" FALSE)
IF(MOJOSETUP_IMAGE_BMP)
    ADD_DEFINITIONS(-DSUPPORT_BMP=1)
    SET(MOJOSETUP_NEED_STBIMAGE TRUE)
ENDIF(MOJOSETUP_IMAGE_BMP)

OPTION(MOJOSETUP_IMAGE_PSD "Enable PSD support" FALSE)
IF(MOJOSETUP_IMAGE_PSD)
    ADD_DEFINITIONS(-DSUPPORT_PSD=1)
    SET(MOJOSETUP_NEED_STBIMAGE TRUE)
ENDIF(MOJOSETUP_IMAGE_PSD)

OPTION(MOJOSETUP_IMAGE_HDR "Enable HDR support" FALSE)
IF(MOJOSETUP_IMAGE_HDR)
    ADD_DEFINITIONS(-DSUPPORT_HDR=1)
    SET(MOJOSETUP_NEED_STBIMAGE TRUE)
ENDIF(MOJOSETUP_IMAGE_HDR)

IF(MOJOSETUP_NEED_STBIMAGE)
    ADD_DEFINITIONS(-DSUPPORT_STBIMAGE=1)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${STBIMAGE_SRCS})
ENDIF(MOJOSETUP_NEED_STBIMAGE)


# Networking...

# BINARY SIZE += 5  ...plus libfetch
OPTION(MOJOSETUP_URL_HTTP "Enable http:// support" TRUE)
IF(MOJOSETUP_URL_HTTP)
    ADD_DEFINITIONS(-DSUPPORT_URL_HTTP=1)
    SET(MOJOSETUP_NEED_LIBFETCH TRUE)
ENDIF(MOJOSETUP_URL_HTTP)

# BINARY SIZE += 9  ...plus libfetch
OPTION(MOJOSETUP_URL_FTP "Enable ftp://  support" TRUE)
IF(MOJOSETUP_URL_FTP)
    ADD_DEFINITIONS(-DSUPPORT_URL_FTP=1)
    SET(MOJOSETUP_NEED_LIBFETCH TRUE)
ENDIF(MOJOSETUP_URL_FTP)

# BINARY SIZE += 10
IF(MOJOSETUP_NEED_LIBFETCH)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LIBFETCH_SRCS})
    # Had to spin up some threads in libfetch...lame.
    # !!! FIXME: CMake will do -lpthread on Mac OS X, but it doesn't need it.
    IF(NOT MACOSX)
        FIND_PACKAGE(Threads)
    ENDIF(NOT MACOSX)
    SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} ${CMAKE_THREAD_LIBS_INIT})
    SET(MOJOSETUP_USES_SOCKETS TRUE)
ENDIF(MOJOSETUP_NEED_LIBFETCH)

IF(MOJOSETUP_USES_SOCKETS)
    IF(SOLARIS)
        SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} "socket")
    ENDIF(SOLARIS)
ENDIF(MOJOSETUP_USES_SOCKETS)


# Middleware...

IF(MOJOSETUP_NEED_ZLIB)
    SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} miniz.c)
ENDIF(MOJOSETUP_NEED_ZLIB)

IF(MOJOSETUP_NEED_BZLIB)
    SET(HAVE_SYSTEM_BZLIB FALSE)
    CHECK_INCLUDE_FILE(bzlib.h HAVE_BZLIB_H)
    IF(HAVE_BZLIB_H)
        CHECK_LIBRARY_EXISTS("bz2" "BZ2_bzDecompress" "" HAVE_LIBBZ2)
        IF(HAVE_LIBBZ2)
            SET(HAVE_SYSTEM_BZLIB TRUE)
        ENDIF(HAVE_LIBBZ2)
    ENDIF(HAVE_BZLIB_H)

    IF(HAVE_SYSTEM_BZLIB)
        OPTION(MOJOSETUP_INTERNAL_BZLIB "Link own bzlib instead of system library" FALSE)
    ELSE(HAVE_SYSTEM_BZLIB)
        SET(MOJOSETUP_INTERNAL_BZLIB TRUE)
    ENDIF(HAVE_SYSTEM_BZLIB)

    # BINARY SIZE += 46
    IF(MOJOSETUP_INTERNAL_BZLIB)
        ADD_DEFINITIONS(-DMOJOSETUP_INTERNAL_BZLIB=1)
        INCLUDE_DIRECTORIES(${BZLIB_DIR})
        SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${BZLIB_SRCS})
    ELSE(MOJOSETUP_INTERNAL_BZLIB)
        SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} bz2)
    ENDIF(MOJOSETUP_INTERNAL_BZLIB)
ENDIF(MOJOSETUP_NEED_BZLIB)

IF(MOJOSETUP_NEED_LIBLZMA)
    SET(HAVE_SYSTEM_LIBLZMA FALSE)
    CHECK_INCLUDE_FILE(lzma.h HAVE_LZMA_H)
    IF(HAVE_LZMA_H)
        CHECK_LIBRARY_EXISTS("lzma" "lzma_stream_decoder" "" HAVE_LIBLZMA)
        IF(HAVE_LIBLZMA)
            SET(HAVE_SYSTEM_LIBLZMA TRUE)
        ENDIF(HAVE_LIBLZMA)
    ENDIF(HAVE_LZMA_H)

    IF(HAVE_SYSTEM_LIBLZMA)
        OPTION(MOJOSETUP_INTERNAL_LIBLZMA "Link own liblzma instead of system library" FALSE)
    ELSE(HAVE_SYSTEM_LIBLZMA)
        SET(MOJOSETUP_INTERNAL_LIBLZMA TRUE)
    ENDIF(HAVE_SYSTEM_LIBLZMA)

    # BINARY SIZE += ???
    IF(MOJOSETUP_INTERNAL_LIBLZMA)
        ADD_DEFINITIONS(-DMOJOSETUP_INTERNAL_LIBLZMA=1)
        INCLUDE_DIRECTORIES(${LIBLZMA_DIR}/api)
        INCLUDE_DIRECTORIES(${LIBLZMA_DIR}/common)
        INCLUDE_DIRECTORIES(${LIBLZMA_DIR}/check)
        INCLUDE_DIRECTORIES(${LIBLZMA_DIR}/delta)
        INCLUDE_DIRECTORIES(${LIBLZMA_DIR}/lz)
        INCLUDE_DIRECTORIES(${LIBLZMA_DIR}/lzma)
        INCLUDE_DIRECTORIES(${LIBLZMA_DIR}/rangecoder)
        INCLUDE_DIRECTORIES(${LIBLZMA_DIR}/simple)
        SET(OPTIONAL_SRCS ${OPTIONAL_SRCS} ${LIBLZMA_SRCS})
    ELSE(MOJOSETUP_INTERNAL_LIBLZMA)
        SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} lzma)
    ENDIF(MOJOSETUP_INTERNAL_LIBLZMA)
ENDIF(MOJOSETUP_NEED_LIBLZMA)

IF(UNIX)
    CHECK_INCLUDE_FILE(sys/ucred.h HAVE_UCRED_H)
    IF(HAVE_UCRED_H)
        ADD_DEFINITIONS(-DMOJOSETUP_HAVE_SYS_UCRED_H=1)
    ENDIF(HAVE_UCRED_H)

    CHECK_INCLUDE_FILE(mntent.h HAVE_MNTENT_H)
    IF(HAVE_MNTENT_H)
        ADD_DEFINITIONS(-DMOJOSETUP_HAVE_MNTENT_H=1)
    ENDIF(HAVE_MNTENT_H)

    # !!! FIXME: Solaris fails this, because mnttab.h implicitly
    # !!! FIXME:  depends on other system headers.  :(
    #CHECK_INCLUDE_FILE(sys/mnttab.h HAVE_SYS_MNTTAB_H)
    CHECK_C_SOURCE_COMPILES("
        #include <stdio.h>
        #include <sys/mnttab.h>
        int main(int argc, char **argv) { return 0; }
    " HAVE_SYS_MNTTAB_H)

    IF(HAVE_SYS_MNTTAB_H)
        ADD_DEFINITIONS(-DMOJOSETUP_HAVE_SYS_MNTTAB_H=1)
    ENDIF(HAVE_SYS_MNTTAB_H)

    IF(NOT MACOSX)
        CHECK_LIBRARY_EXISTS("dl" "dlopen" "" HAVE_LIBDL)
        IF(HAVE_LIBDL)
            SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} dl)
        ENDIF(HAVE_LIBDL)
        CHECK_LIBRARY_EXISTS("m" "sin" "" HAVE_LIBM)
        IF(HAVE_LIBM)
            SET(OPTIONAL_LIBS ${OPTIONAL_LIBS} m)
        ENDIF(HAVE_LIBM)
    ENDIF(NOT MACOSX)
ENDIF(UNIX)

OPTION(MOJOSETUP_BUILD_LUAC "Build separate Lua compiler" TRUE)
IF(MOJOSETUP_BUILD_LUAC)
    ADD_EXECUTABLE(mojoluac ${MOJOLUAC_SRCS})
    TARGET_LINK_LIBRARIES(mojoluac ${OPTIONAL_LIBS})
    # !!! FIXME: actually compile this.
    ADD_CUSTOM_TARGET(lua mojoluac -p ${CMAKE_CURRENT_SOURCE_DIR}/scripts/*.lua)
ENDIF(MOJOSETUP_BUILD_LUAC)

ADD_EXECUTABLE(mojosetup ${USES_WINMAIN} ${MOJOSETUP_SRCS} ${OPTIONAL_SRCS})
TARGET_LINK_LIBRARIES(mojosetup ${OPTIONAL_LIBS})
SET(MOJOSETUP_TARGETS "${MOJOSETUP_TARGETS};mojosetup")

ADD_EXECUTABLE(make_self_extracting misc/make_self_extracting.c)

# For cobbling together a skeleton installer...

# !!! FIXME: get rid of the custom misc/cp.cmake file, if CMake ever adds wildcard support for the 'copy' command
ADD_CUSTOM_TARGET(skeleton
    COMMENT "Assembling Skeleton..."
    COMMAND ${CMAKE_COMMAND} -E remove_directory ${CMAKE_BINARY_DIR}/skeleton
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/skeleton
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/skeleton/scripts
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/skeleton/guis
    COMMAND ${CMAKE_COMMAND} -E make_directory ${CMAKE_BINARY_DIR}/skeleton/data
    COMMAND ${CMAKE_COMMAND} -DFROM=${CMAKE_SOURCE_DIR}/scripts/*.lua -DTO=${CMAKE_BINARY_DIR}/skeleton/scripts -P ${CMAKE_SOURCE_DIR}/misc/cp.cmake
    COMMAND ${CMAKE_COMMAND} -DFROM=${CMAKE_BINARY_DIR}/*mojosetupgui*.* -DTO=${CMAKE_BINARY_DIR}/skeleton/guis -P ${CMAKE_SOURCE_DIR}/misc/cp.cmake
    COMMAND ${CMAKE_COMMAND} -E copy mojosetup ${CMAKE_BINARY_DIR}/skeleton
)

ADD_DEPENDENCIES(skeleton ${MOJOSETUP_TARGETS})

# end of CMakeLists.txt ...
