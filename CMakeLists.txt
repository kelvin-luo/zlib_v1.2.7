cmake_minimum_required(VERSION 3.15)
#set(CMAKE_ALLOW_LOOSE_LOOP_CONSTRUCTS ON)

project(zlib
     VERSION 1.2.7.0
   LANGUAGES  C CXX
    )

set(VERSION "1.2.7")

#set(CMAKE_INSTALL_PREFIX  "${CMAKE_BINARY_DIR}/../install__zlib__VC14_Ver_debug" )


###############################################
include(cmake_kelvin/ProjectSsetting.cmake)
###############################################


#
# version
#
message(STATUS "zlib Version: ${zlib_VERSION_MAJOR}.${zlib_VERSION_MINOR}.${zlib_VERSION_PATCH}.${zlib_VERSION_TWEAK}")
set( zlib_SOVERSION 202 )

configure_file (
    "${PROJECT_SOURCE_DIR}/config.h.in"
    "${PROJECT_BINARY_DIR}/config.h"
    )

include_directories(
    ${PROJECT_BINARY_DIR}
    )



#set(INSTALL_BIN_DIR "${CMAKE_INSTALL_PREFIX}/bin" CACHE PATH "Installation directory for executables")
#set(INSTALL_LIB_DIR "${CMAKE_INSTALL_PREFIX}/lib" CACHE PATH "Installation directory for libraries")
#set(INSTALL_INC_DIR "${CMAKE_INSTALL_PREFIX}/include" CACHE PATH "Installation directory for headers")
#set(INSTALL_MAN_DIR "${CMAKE_INSTALL_PREFIX}/share/man" CACHE PATH "Installation directory for manual pages")
#set(INSTALL_PKGCONFIG_DIR "${CMAKE_INSTALL_PREFIX}/share/pkgconfig" CACHE PATH "Installation directory for pkgconfig (.pc) files")

include(CheckTypeSize)
include(CheckFunctionExists)
include(CheckIncludeFile)
include(CheckCSourceCompiles)
enable_testing()

check_include_file(sys/types.h HAVE_SYS_TYPES_H)
check_include_file(stdint.h    HAVE_STDINT_H)
check_include_file(stddef.h    HAVE_STDDEF_H)

#
# Check to see if we have large file support
#
set(CMAKE_REQUIRED_DEFINITIONS -D_LARGEFILE64_SOURCE=1)
# We add these other definitions here because CheckTypeSize.cmake
# in CMake 2.4.x does not automatically do so and we want
# compatibility with CMake 2.4.x.
if(HAVE_SYS_TYPES_H)
    list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_SYS_TYPES_H)
endif()
if(HAVE_STDINT_H)
    list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_STDINT_H)
endif()
if(HAVE_STDDEF_H)
    list(APPEND CMAKE_REQUIRED_DEFINITIONS -DHAVE_STDDEF_H)
endif()
check_type_size(off64_t OFF64_T)
if(HAVE_OFF64_T)
   add_definitions(-D_LARGEFILE64_SOURCE=1)
endif()
set(CMAKE_REQUIRED_DEFINITIONS) # clear variable

#
# Check for fseeko
#
check_function_exists(fseeko HAVE_FSEEKO)
if(NOT HAVE_FSEEKO)
    add_definitions(-DNO_FSEEKO)
endif()

#
# Check for unistd.h
#
check_include_file(unistd.h Z_HAVE_UNISTD_H)

if(MSVC)
    set(CMAKE_DEBUG_POSTFIX "d")
    add_definitions(-D_CRT_SECURE_NO_DEPRECATE)
    add_definitions(-D_CRT_NONSTDC_NO_DEPRECATE)
    include_directories(${CMAKE_CURRENT_SOURCE_DIR})
endif()

if(NOT CMAKE_CURRENT_SOURCE_DIR STREQUAL CMAKE_CURRENT_BINARY_DIR)
    # If we're doing an out of source build and the user has a zconf.h
    # in their source tree...
    if(EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h)
        message(STATUS "Renaming")
        message(STATUS "    ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h")
        message(STATUS "to 'zconf.h.included' because this file is included with zlib")
        message(STATUS "but CMake generates it automatically in the build directory.")
        file(RENAME ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h ${CMAKE_CURRENT_SOURCE_DIR}/zconf.h.included)
  endif()
endif()

set(ZLIB_PC ${CMAKE_CURRENT_BINARY_DIR}/zlib.pc)
configure_file( ${CMAKE_CURRENT_SOURCE_DIR}/zlib.pc.cmakein
		${ZLIB_PC} @ONLY)
configure_file(	${CMAKE_CURRENT_SOURCE_DIR}/zconf.h.cmakein
		${CMAKE_CURRENT_BINARY_DIR}/zconf.h @ONLY)
include_directories(${CMAKE_CURRENT_BINARY_DIR} ${CMAKE_SOURCE_DIR})


#============================================================================
# zlib
#============================================================================

include_directories(include )

set(ZLIB_PUBLIC_HDRS
    ${CMAKE_CURRENT_BINARY_DIR}/zconf.h
    include/zlib.h
)
set(ZLIB_PRIVATE_HDRS
    crc32.h
    deflate.h
    gzguts.h
    inffast.h
    inffixed.h
    inflate.h
    inftrees.h
    trees.h
    zutil.h
)
set(ZLIB_SRCS
    adler32.c
    compress.c
    crc32.c
    deflate.c
    gzclose.c
    gzlib.c
    gzread.c
    gzwrite.c
    inflate.c
    infback.c
    inftrees.c
    inffast.c
    trees.c
    uncompr.c
    zutil.c
)

if(NOT MINGW)
    set(ZLIB_DLL_SRCS
        win32/zlib1.rc # If present will override custom build rule below.
    )
endif()

if(CMAKE_COMPILER_IS_GNUCC)
    if(ASM686)
        set(ZLIB_ASMS contrib/asm686/match.S)
    elseif (AMD64)
        set(ZLIB_ASMS contrib/amd64/amd64-match.S)
    endif ()

	if(ZLIB_ASMS)
		add_definitions(-DASMV)
		set_source_files_properties(${ZLIB_ASMS} PROPERTIES LANGUAGE C COMPILE_FLAGS -DNO_UNDERLINE)
	endif()
endif()

if(MSVC)
    if(ASM686)
		ENABLE_LANGUAGE(ASM_MASM)
        set(ZLIB_ASMS
			contrib/masmx86/inffas32.asm
			contrib/masmx86/match686.asm
		)
    elseif (AMD64)
		ENABLE_LANGUAGE(ASM_MASM)
        set(ZLIB_ASMS
			contrib/masmx64/gvmat64.asm
			contrib/masmx64/inffasx64.asm
		)
    endif()

	if(ZLIB_ASMS)
		add_definitions(-DASMV -DASMINF)
	endif()
endif()

# parse the full version number from zlib.h and include in ZLIB_FULL_VERSION
file(READ ${CMAKE_CURRENT_SOURCE_DIR}/include/zlib.h _zlib_h_contents)
string(REGEX REPLACE ".*#define[ \t]+ZLIB_VERSION[ \t]+\"([-0-9A-Za-z.]+)\".*"
    "\\1" ZLIB_FULL_VERSION ${_zlib_h_contents})

if(MINGW)
    # This gets us DLL resource information when compiling on MinGW.
    if(NOT CMAKE_RC_COMPILER)
        set(CMAKE_RC_COMPILER windres.exe)
    endif()

    add_custom_command(OUTPUT ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj
                       COMMAND ${CMAKE_RC_COMPILER}
                            -D GCC_WINDRES
                            -I ${CMAKE_CURRENT_SOURCE_DIR}
                            -I ${CMAKE_CURRENT_BINARY_DIR}
                            -o ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj
                            -i ${CMAKE_CURRENT_SOURCE_DIR}/win32/zlib1.rc)
    set(ZLIB_DLL_SRCS ${CMAKE_CURRENT_BINARY_DIR}/zlib1rc.obj)
endif(MINGW)

include_directories(include)

add_library(zlib SHARED ${ZLIB_SRCS} ${ZLIB_ASMS} ${ZLIB_DLL_SRCS} ${ZLIB_PUBLIC_HDRS} ${ZLIB_PRIVATE_HDRS})
add_library(zlibstatic STATIC ${ZLIB_SRCS} ${ZLIB_ASMS} ${ZLIB_PUBLIC_HDRS} ${ZLIB_PRIVATE_HDRS})
set_target_properties(zlib PROPERTIES DEFINE_SYMBOL ZLIB_DLL)
set_target_properties(zlib PROPERTIES SOVERSION 1)

if(NOT CYGWIN)
    # This property causes shared libraries on Linux to have the full version
    # encoded into their final filename.  We disable this on Cygwin because
    # it causes cygz-${ZLIB_FULL_VERSION}.dll to be created when cygz.dll
    # seems to be the default.
    #
    # This has no effect with MSVC, on that platform the version info for
    # the DLL comes from the resource file win32/zlib1.rc
    set_target_properties(zlib PROPERTIES VERSION ${ZLIB_FULL_VERSION})
endif()

if(UNIX)
    # On unix-like platforms the library is almost always called libz
   set_target_properties(zlib zlibstatic PROPERTIES OUTPUT_NAME z)
   set_target_properties(zlib PROPERTIES LINK_FLAGS "-Wl,--version-script,${CMAKE_CURRENT_SOURCE_DIR}/zlib.map")
elseif(BUILD_SHARED_LIBS AND WIN32)
    # Creates zlib1.dll when building shared library version
    set_target_properties(zlib PROPERTIES SUFFIX "1.dll")
endif()

if(NOT SKIP_INSTALL_LIBRARIES AND NOT SKIP_INSTALL_ALL )
    install(TARGETS zlib zlibstatic
        RUNTIME DESTINATION "${INSTALL_BIN_DIR}"
        ARCHIVE DESTINATION "${INSTALL_LIB_DIR}"
        LIBRARY DESTINATION "${INSTALL_LIB_DIR}" )
endif()
if(NOT SKIP_INSTALL_HEADERS AND NOT SKIP_INSTALL_ALL )
#    install(FILES ${ZLIB_PUBLIC_HDRS} DESTINATION "${INSTALL_INC_DIR}")
endif()
if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL )
    install(FILES zlib.3 DESTINATION "${INSTALL_MAN_DIR}/man3")
endif()
if(NOT SKIP_INSTALL_FILES AND NOT SKIP_INSTALL_ALL )
#    install(FILES ${ZLIB_PC} DESTINATION "${INSTALL_PKGCONFIG_DIR}")
endif()


#//-----------------------------------------------------------------

target_include_directories(zlib PRIVATE  ${CMAKE_SOURCE_DIR}/include)
set_target_properties(zlib PROPERTIES PUBLIC_HEADER ${CMAKE_SOURCE_DIR}/include/zlib.h   )

# 写入库的版本信息
include(CMakePackageConfigHelpers)
write_basic_package_version_file(
        zlibConfigVersion.cmake
        VERSION ${PACKAGE_VERSION}
        COMPATIBILITY AnyNewerVersion
        )

# 将库文件，可执行文件，头文件安装到指定目录
install(TARGETS zlib  zlibstatic
        EXPORT zlibTargets
        LIBRARY DESTINATION lib
        ARCHIVE DESTINATION lib
        RUNTIME DESTINATION bin
        PUBLIC_HEADER DESTINATION include
        )

# 将库的相关信息输出为MyMathConfig.cmake，供find_package查找
install(EXPORT zlibTargets
        FILE zlibTargets.cmake
        NAMESPACE zlib::
        DESTINATION lib/cmake/zlib
        )

configure_file(zlibConfig.cmake.in  zlibConfig.cmake @ONLY)
install(FILES "${CMAKE_CURRENT_BINARY_DIR}/zlibConfig.cmake"
                "${CMAKE_CURRENT_BINARY_DIR}/zlibConfigVersion.cmake"
        DESTINATION lib/cmake/zlib
        )

install(FILES  ${CMAKE_CURRENT_BINARY_DIR}/zconf.h     DESTINATION include )



#============================================================================
# Example binaries
#============================================================================

add_executable(example test/example.c)
target_link_libraries(example zlib)
add_test(example example)

add_executable(minigzip test/minigzip.c)
target_link_libraries(minigzip zlib)

if(HAVE_OFF64_T)
    add_executable(example64 test/example.c)
    target_link_libraries(example64 zlib)
    set_target_properties(example64 PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64")
    add_test(example64 example64)

    add_executable(minigzip64 test/minigzip.c)
    target_link_libraries(minigzip64 zlib)
    set_target_properties(minigzip64 PROPERTIES COMPILE_FLAGS "-D_FILE_OFFSET_BITS=64")
endif()


include(${CMAKE_CURRENT_SOURCE_DIR}/OCMS-post.cmake)



#
#
#
#
##
#
#-------------------------------------- install   ------------------------------
#set(CMAKE_INSTALL_PREFIX  "${CMAKE_BINARY_DIR}/../install__zlib__VC14_Ver_debug" )

#
##
#
#//--------------------------------install config----------------------------
#测试
option(zlib_WITH_TESTS "Compile and (for LibBlackBorder_check) run LibBlackBorder test executables" OFF)
#测试
option(zlib_WITH_POST_BUILD_UNITTEST "Automatically run unit-tests as a post build step" ON)
# 警告作为错误，这个要关闭
option(zlib_WITH_WARNING_AS_ERROR "Force compilation to fail if a warning occurs" OFF)
#标准c cpp
option(zlib_WITH_STRICT_ISO "Issue all the warnings demanded by strict ISO C and ISO C++" ON)
#//设置安装选项为打开
option(zlib_WITH_PKGCONFIG_SUPPORT "Generate and install .pc files" ON)
option(zlib_WITH_CMAKE_PACKAGE "Generate and install cmake package files" ON)
# 设置编译动态库选项打开
option(BUILD_SHARED_LIBS "Build LibBlackBorder as a shared library." ON)

# Adhere to GNU filesystem layout conventions
include(GNUInstallDirs)

set(DEBUG_LIBNAME_SUFFIX "" CACHE STRING "Optional suffix to append to the library name for a debug build")
set(zlib_USE_SECURE_MEMORY "0" CACHE STRING "-D...=1 to use memory-wiping allocator for STL" )


##//-------------------------安装 pkg-config-------------------------------
#if(LibBlackBorder_WITH_PKGCONFIG_SUPPORT)
#    configure_file(
#        "pkg-config/libBlackBd.pc.in"
#        "pkg-config/libBlackBd.pc"
#        @ONLY)
#    install(FILES "${CMAKE_CURRENT_BINARY_DIR}/pkg-config/libBlackBd.pc"
#        DESTINATION "${CMAKE_INSTALL_LIBDIR}/pkgconfig")
#endif()

#//---------------------------安装 cmake 目录 MathFunctions ----------------------------------
if(zlib_WITH_CMAKE_PACKAGE)
#    if( 0  )
#        # 安装 libBlackBdConfig.cmake  同时安装   libBlackBdConfig-release.cmake
#        install(EXPORT zlib
#            FILE zlibConfig.cmake
#            DESTINATION lib/cmake/MathFunctions
#            #        CONFIGURATIONS Debug Release
#            EXPORT_LINK_INTERFACE_LIBRARIES
#            #EXCLUDE_FROM_ALL
#            )
#        include (CMakePackageConfigHelpers)
#        # generate the config file that is includes the exports
#        configure_package_config_file(${CMAKE_CURRENT_SOURCE_DIR}/Config.cmake.in
#            "${CMAKE_CURRENT_BINARY_DIR}/libBlackBdConfig.cmake"
#            INSTALL_DESTINATION "lib/cmake/MathFunctions"
#            NO_SET_AND_CHECK_MACRO
#            NO_CHECK_REQUIRED_COMPONENTS_MACRO
#            )
#        # generate 版本号文件
#        write_basic_package_version_file(
#            "${CMAKE_CURRENT_BINARY_DIR}/libBlackBdConfigVersion.cmake"
#            VERSION "${LIBBLACKBORDER_VERSION_MAJOR}.${LIBBLACKBORDER_VERSION_MINOR}"
#            COMPATIBILITY AnyNewerVersion
#            )
#        # install the configuration file
#        install(FILES
#            ${CMAKE_CURRENT_BINARY_DIR}/libBlackBdConfig.cmake
#            DESTINATION lib/cmake/MathFunctions
#            )
#        # 安装 版本号文件
#        install(FILES ${CMAKE_CURRENT_BINARY_DIR}/libBlackBdConfigVersion.cmake
#            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/MathFunctions  )
#    endif()

    if( 1  )
        include (CMakePackageConfigHelpers)
        # 安装 libBlackBdConfig.cmake  同时安装   libBlackBdConfig-release.cmake   LIBBLACKBORDER
        install(EXPORT zlib
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/zlib
            FILE zlibConfig.cmake)
        # 创建 版本号文件
        write_basic_package_version_file (
            "${CMAKE_CURRENT_BINARY_DIR}/zlibConfigVersion.cmake"
            VERSION ${PROJECT_VERSION}
            COMPATIBILITY SameMajorVersion)
        # 安装 版本号文件
        install(FILES ${CMAKE_CURRENT_BINARY_DIR}/zlibConfigVersion.cmake
            DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/zlib  )
    endif()
endif()



#
##
#
#---------------- 第 1 种安装方式------------------------
if(  0  )
    #//---------------install------------------- LibBlackBorder
    ##    if(LibBlackBorder_WITH_CMAKE_PACKAGE)
    #        set(INSTALL_EXPORT EXPORT libBlackBd  )
    ##    endif()
    #    set_target_properties( LibBlackBorder PROPERTIES OUTPUT_NAME libBlackBd
    #        DEBUG_OUTPUT_NAME libBlackBd${DEBUG_LIBNAME_SUFFIX} )
    # 系统库的包含 主要是vs2015
    INCLUDE(InstallRequiredSystemLibraries)

    INSTALL( TARGETS zlib ARCHIVE DESTINATION lib )
    INSTALL( DIRECTORY ./ DESTINATION include FILES_MATCHING PATTERN "*.h"  )
    INSTALL( TARGETS zlib RUNTIME DESTINATION bin )
    install( TARGETS zlib DESTINATION bin)

    set (CMAKE_RUNTIME_OUTPUT_DIRECTORY "bin")
    set (CMAKE_LIBRARY_OUTPUT_DIRECTORY "lib")
    set (CMAKE_ARCHIVE_OUTPUT_DIRECTORY "lib")
    set (RUNTIME_INSTALL_DIR "bin")
    set (LIBRARY_INSTALL_DIR "lib")
    set (INCLUDE_INSTALL_DIR "include")
    set (CONFIG_INSTALL_DIR  "lib/cmake/${PACKAGE_NAME}")
    #    INSTALL( DIRECTORY cmake DESTINATION ${CONFIG_INSTALL_DIR} FILES_MATCHING  )
    set (PKGCONFIG_INSTALL_DIR)
    INSTALL(DIRECTORY cmake/ DESTINATION lib/cmake)

    # 安装 头文件
    file(GLOB INCLUDE_FILES "include/*.h")
    install(FILES ${INCLUDE_FILES}  DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/zlib)

    #---------------- 第 2 种安装方式-----------------------------
else(  )
    set(zlib_WITH_CMAKE_PACKAGE true  )
    # Install instructions for this target
    if(zlib_WITH_CMAKE_PACKAGE)
        set(INSTALL_EXPORT EXPORT zlib  )
    else(zlib_WITH_CMAKE_PACKAGE)
        set(INSTALL_EXPORT)
    endif()

    if(BUILD_SHARED_LIBS)
        if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.12.0)
            add_compile_definitions( zlib_DLL_BUILD )
        else()
            add_definitions( -Dzlib_DLL_BUILD )
        endif()
    endif()

    set_target_properties( zlib PROPERTIES VERSION ${QroilibLibApp_VERSION} SOVERSION ${QroilibLibApp_SOVERSION})
    set_target_properties( zlib PROPERTIES OUTPUT_NAME zlib
        DEBUG_OUTPUT_NAME zlib${DEBUG_LIBNAME_SUFFIX} )
    set_target_properties( zlib PROPERTIES POSITION_INDEPENDENT_CODE ON)
    # Specify compiler features required when compiling a given target.
    # See https://cmake.org/cmake/help/v3.1/prop_gbl/CMAKE_CXX_KNOWN_FEATURES.html#prop_gbl:CMAKE_CXX_KNOWN_FEATURES
    # for complete list of features available
#    target_compile_features(zlib PUBLIC
#        cxx_std_11 # Compiler mode is aware of C++ 11.
#        #MSVC 1900 cxx_alignas # Alignment control alignas, as defined in N2341.
#        #MSVC 1900 cxx_alignof # Alignment control alignof, as defined in N2341.
#        #MSVC 1900 cxx_attributes # Generic attributes, as defined in N2761.
#        cxx_auto_type # Automatic type deduction, as defined in N1984.
#        #MSVC 1900 cxx_constexpr # Constant expressions, as defined in N2235.
#        cxx_decltype # Decltype, as defined in N2343.
#        cxx_default_function_template_args # Default template arguments for function templates, as defined in DR226
#        cxx_defaulted_functions # Defaulted functions, as defined in N2346.
#        #MSVC 1900 cxx_defaulted_move_initializers # Defaulted move initializers, as defined in N3053.
#        cxx_delegating_constructors # Delegating constructors, as defined in N1986.
#        #MSVC 1900 cxx_deleted_functions # Deleted functions, as defined in N2346.
#        cxx_enum_forward_declarations # Enum forward declarations, as defined in N2764.
#        cxx_explicit_conversions # Explicit conversion operators, as defined in N2437.
#        cxx_extended_friend_declarations # Extended friend declarations, as defined in N1791.
#        cxx_extern_templates # Extern templates, as defined in N1987.
#        cxx_final # Override control final keyword, as defined in N2928, N3206 and N3272.
#        #MSVC 1900 cxx_func_identifier # Predefined __func__ identifier, as defined in N2340.
#        #MSVC 1900 cxx_generalized_initializers # Initializer lists, as defined in N2672.
#        #MSVC 1900 cxx_inheriting_constructors # Inheriting constructors, as defined in N2540.
#        #MSVC 1900 cxx_inline_namespaces # Inline namespaces, as defined in N2535.
#        cxx_lambdas # Lambda functions, as defined in N2927.
#        #MSVC 1900 cxx_local_type_template_args # Local and unnamed types as template arguments, as defined in N2657.
#        cxx_long_long_type # long long type, as defined in N1811.
#        #MSVC 1900 cxx_noexcept # Exception specifications, as defined in N3050.
#        #MSVC 1900 cxx_nonstatic_member_init # Non-static data member initialization, as defined in N2756.
#        cxx_nullptr # Null pointer, as defined in N2431.
#        cxx_override # Override control override keyword, as defined in N2928, N3206 and N3272.
#        cxx_range_for # Range-based for, as defined in N2930.
#        cxx_raw_string_literals # Raw string literals, as defined in N2442.
#        #MSVC 1900 cxx_reference_qualified_functions # Reference qualified functions, as defined in N2439.
#        cxx_right_angle_brackets # Right angle bracket parsing, as defined in N1757.
#        cxx_rvalue_references # R-value references, as defined in N2118.
#        #MSVC 1900 cxx_sizeof_member # Size of non-static data members, as defined in N2253.
#        cxx_static_assert # Static assert, as defined in N1720.
#        cxx_strong_enums # Strongly typed enums, as defined in N2347.
#        #MSVC 1900 cxx_thread_local # Thread-local variables, as defined in N2659.
#        cxx_trailing_return_types # Automatic function return type, as defined in N2541.
#        #MSVC 1900 cxx_unicode_literals # Unicode string literals, as defined in N2442.
#        cxx_uniform_initialization # Uniform initialization, as defined in N2640.
#        #MSVC 1900 cxx_unrestricted_unions # Unrestricted unions, as defined in N2544.
#        #MSVC 1900 cxx_user_literals # User-defined literals, as defined in N2765.
#        cxx_variadic_macros # Variadic macros, as defined in N1653.
#        cxx_variadic_templates # Variadic templates, as defined in N2242.
#        )

    install( TARGETS  zlib ${INSTALL_EXPORT}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR})

    # 从安装目录进行引用
    if(NOT CMAKE_VERSION VERSION_LESS 2.8.11)
        target_include_directories( zlib  PUBLIC
            $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_LIST_DIR}/${QROILIBLIBAPP_INCLUDE_DIR_EXPORT}>
            $<BUILD_INTERFACE:${PROJECT_BINARY_DIR}>
            )
    endif()

    #  安装 头文件
    #${CMAKE_INSTALL_INCLUDEDIR}
    file(GLOB INCLUDE_FILES "./*.h"   "${PROJECT_BINARY_DIR}/zconf.h")
    install(FILES ${INCLUDE_FILES} DESTINATION include/zlib )

##    message("___________________________________________")
#    #    message("INCLUDE_FILES = ")
#    #    message(${INCLUDE_FILES})
#        if( 0 )
#            #    install(TARGETS ${targetName}
#            #        #  mySharedLib myStaticLib
#            #        RUNTIME DESTINATION bin
#            #        LIBRARY DESTINATION lib
#            #        ARCHIVE DESTINATION lib/static)
#        endif()
#        if( 0 )
#            INSTALL( DIRECTORY ./qroilib DESTINATION include FILES_MATCHING PATTERN "*.h"  )
#            INSTALL(FILES COPYRIGHT README DESTINATION share/doc/QroilibLibApp)
#            INSTALL(DIRECTORY doc/ DESTINATION share/doc/QroilibLibApp)
#            INSTALL(DIRECTORY cmake/ DESTINATION lib)
#        endif()

#    if( 0 )
#        #  安装 头文件
#        add_subdirectory( include )
#    endif()
endif()

