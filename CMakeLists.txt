cmake_minimum_required(VERSION 2.6)

project(UriParser C)

set(VERSION_STRING "0.8.6")
IF(EXISTS "${CMAKE_SOURCE_DIR}/.git")
    FIND_PROGRAM(GIT git)
    IF(GIT)
        EXECUTE_PROCESS(
            WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
            COMMAND ${GIT} describe --tags
            OUTPUT_VARIABLE GIT_VERSION  OUTPUT_STRIP_TRAILING_WHITESPACE
        )
        if(NOT VERSION_STRING)
                EXECUTE_PROCESS(
                    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
                    COMMAND ${GIT} rev-parse --short HEAD
                    OUTPUT_VARIABLE GIT_VERSION  OUTPUT_STRIP_TRAILING_WHITESPACE
                )
        endif()
        SET(VERSION_STRING ${GIT_VERSION})
    ENDIF(GIT)
ENDIF()
MESSAGE("Version: ${VERSION_STRING}\n")
set(URIPARSER_VERSION ${VERSION_STRING})

option(BUILD_TESTS "Build demos." OFF)

if (BUILD_SHARED_LIBS)
    if (CMAKE_COMPILER_IS_GNUCXX)
       # Just setting CMAKE_POSITION_INDEPENDENT_CODE should be enough to set
       # -fPIC for GCC but sometimes it still doesn't get set, so make sure it
       # does.
       add_definitions("-fPIC")
    endif()
    set(CMAKE_POSITION_INDEPENDENT_CODE ON)
    if(WIN32)
        add_definitions(-DDLL_EXPORT)
    endif(WIN32)
endif()

IF(MSVC)
    # This option is to enable the /MP switch for Visual Studio 2005 and above compilers
    OPTION(WIN32_USE_MP "Set to ON to build OpenSceneGraph with the /MP option (Visual Studio 2005 and above)." ON)
    MARK_AS_ADVANCED(WIN32_USE_MP)
    IF(WIN32_USE_MP)
        SET(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} /MP")
    ENDIF(WIN32_USE_MP)
ENDIF(MSVC)

include (CheckIncludeFile)
include (CheckFunctionExists)

check_include_file("wchar.h" HAVE_WCHAR_H)
check_include_file("stdio.h" HAVE_STDIO_H)

check_function_exists(wprintf HAVE_WPRINTF)

set(CMAKE_MODULE_PATH ${CMAKE_SOURCE_DIR}/cmake ${CMAKE_MODULE_PATH})

option(ENABLE_SIZEDOWN  "enable size down: -DURI_SIZEDOWN" ON)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DURI_SIZEDOWN")

option(DISABLE_CHAR "disable char: -DURI_NO_ANSI" OFF)
if(DISABLE_CHAR)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DURI_NO_ANSI")
endif()

option(DISABLE_WCHAR_T "disable wchar_t: -DURI_NO_UNICODE " OFF)
if(DISABLE_WCHAR_T)
    set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -DURI_NO_UNICODE")
endif()

if(DISABLE_WCHAR_T AND DISABLE_CHAR)
   message(FATAL_ERROR "DISABLE_WCHAR_T and DISABLE_CHAR are mutually exclusive")
endif()

if(NOT (DISABLE_WCHAR_T OR DISABLE_CHAR) AND BUILD_TESTS)
    enable_testing()
endif()

set(URIPARSER_SRCS
  src/UriCommon.c
  src/UriCommon.h
  src/UriCompare.c
  src/UriEscape.c
  src/UriFile.c
  src/UriIp4.c
  src/UriIp4Base.c
  src/UriIp4Base.h
  src/UriNormalize.c
  src/UriNormalizeBase.c
  src/UriNormalizeBase.h
  src/UriParse.c
  src/UriParseBase.c
  src/UriParseBase.h
  src/UriQuery.c
  src/UriRecompose.c
  src/UriResolve.c
  src/UriShorten.c
  )

set(URIPARSER_INCS
  include/uriparser/Uri.h
  include/uriparser/UriBase.h
  include/uriparser/UriDefsAnsi.h
  include/uriparser/UriDefsConfig.h
  include/uriparser/UriDefsUnicode.h
  include/uriparser/UriIp4.h
  )

set(LIB_NAME uriparser)
add_library(${LIB_NAME} ${URIPARSER_SRCS})
set_target_properties(${LIB_NAME} PROPERTIES SOVERSION ${VERSION_STRING})
target_include_directories(${LIB_NAME} PUBLIC ${CMAKE_SOURCE_DIR}/include)

include(GNUInstallDirs OPTIONAL)

set(INSTALL_CMAKE_DIR CMake CACHE PATH
    "Installation directory for CMake files")

export(TARGETS ${LIB_NAME}
  FILE "${CMAKE_BINARY_DIR}/UriParserTargets.cmake")

# Install uriparser library
install(TARGETS ${LIB_NAME}
    EXPORT UriParserTargets
    RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}"
    ARCHIVE DESTINATION "${CMAKE_INSTALL_LIBDIR}"
    LIBRARY DESTINATION "${CMAKE_INSTALL_LIBDIR}" 
    )
install(FILES ${URIPARSER_INCS} DESTINATION "${CMAKE_INSTALL_INCLUDEDIR}/uriparser")

# PkgConfig configure file
if(NOT MSVC)
    configure_file(${CMAKE_CURRENT_SOURCE_DIR}/cmake/liburiparser.pc.in
        ${CMAKE_CURRENT_BINARY_DIR}/liburiparser.pc @ONLY)
    install(FILES ${CMAKE_CURRENT_BINARY_DIR}/liburiparser.pc
        DESTINATION
        "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
endif()

# Cmake configure files
configure_file(cmake/UriParserConfig.cmake.in
  "${CMAKE_BINARY_DIR}/UriParserConfig.cmake" @ONLY)
configure_file(cmake/UriParserConfigVersion.cmake.in
  "${CMAKE_BINARY_DIR}/UriParserConfigVersion.cmake" @ONLY)
install(FILES
  "${CMAKE_BINARY_DIR}/UriParserConfig.cmake"
  "${CMAKE_BINARY_DIR}/UriParserConfigVersion.cmake"
  DESTINATION ${INSTALL_CMAKE_DIR} )

# Install the export set for use with the install-tree
# install(EXPORT UriParserTargets DESTINATION ${INSTALL_CMAKE_DIR})

# Unistall
CONFIGURE_FILE(
    "${CMAKE_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
    "${CMAKE_BINARY_DIR}/cmake_uninstall.cmake"
    IMMEDIATE @ONLY)
ADD_CUSTOM_TARGET(uninstall
    "${CMAKE_COMMAND}" -P "${CMAKE_BINARY_DIR}/cmake_uninstall.cmake")