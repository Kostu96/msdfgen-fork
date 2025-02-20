
cmake_minimum_required(VERSION 3.15)
include(cmake/version.cmake)

option(MSDFGEN_CORE_ONLY "Only build the core library with no dependencies" OFF)
option(MSDFGEN_USE_CPP11 "Build with C++11 enabled" ON)
option(MSDFGEN_INSTALL "Generate installation target" OFF)
option(MSDFGEN_DYNAMIC_RUNTIME "Link dynamic runtime library instead of static" OFF)
option(BUILD_SHARED_LIBS "Generate dynamic library files instead of static" OFF)
option(MSDFGEN_DISABLE_SVG "" ON)
option(MSDFGEN_DISABLE_PNG "" ON)

get_property(MULTI_CONFIG GLOBAL PROPERTY GENERATOR_IS_MULTI_CONFIG)
if(NOT MULTI_CONFIG AND NOT CMAKE_BUILD_TYPE)
    message(STATUS "CMAKE_BUILD_TYPE not set, defaulting to Release")
    set(CMAKE_BUILD_TYPE Release)
endif()

if(MSDFGEN_DYNAMIC_RUNTIME)
    set(MSDFGEN_MSVC_RUNTIME "MultiThreaded$<$<CONFIG:Debug>:Debug>DLL")
else()
    set(MSDFGEN_MSVC_RUNTIME "MultiThreaded$<$<CONFIG:Debug>:Debug>")
endif()

if(BUILD_SHARED_LIBS)
    set(CMAKE_WINDOWS_EXPORT_ALL_SYMBOLS ON)
endif()

# Version is specified in vcpkg.json
project(msdfgen VERSION ${MSDFGEN_VERSION} LANGUAGES CXX)

if(MAX_WARNING_LEVEL)
    if (MSVC)
        add_compile_options(/W4)
    else()
        add_compile_options(-Wall -Wextra -Wpedantic)
    endif()
endif()

file(GLOB_RECURSE MSDFGEN_CORE_HEADERS RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "core/*.h" "core/*.hpp")
file(GLOB_RECURSE MSDFGEN_CORE_SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "core/*.cpp")
file(GLOB_RECURSE MSDFGEN_EXT_HEADERS RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "ext/*.h" "ext/*.hpp")
file(GLOB_RECURSE MSDFGEN_EXT_SOURCES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "ext/*.cpp" "lib/*.cpp")

# Core library
if (NOT DEFINED CMAKE_DEBUG_POSTFIX)
  set(CMAKE_DEBUG_POSTFIX "d")
endif()

add_library(msdfgen-core "${CMAKE_CURRENT_SOURCE_DIR}/msdfgen.h" ${MSDFGEN_CORE_HEADERS} ${MSDFGEN_CORE_SOURCES})
add_library(msdfgen::msdfgen-core ALIAS msdfgen-core)
set_target_properties(msdfgen-core PROPERTIES PUBLIC_HEADER "${MSDFGEN_CORE_HEADERS}" DEBUG_POSTFIX "${CMAKE_DEBUG_POSTFIX}")
set_property(TARGET msdfgen-core PROPERTY MSVC_RUNTIME_LIBRARY "${MSDFGEN_MSVC_RUNTIME}")
target_compile_definitions(msdfgen-core PUBLIC
    MSDFGEN_VERSION=${MSDFGEN_VERSION}
    MSDFGEN_VERSION_MAJOR=${MSDFGEN_VERSION_MAJOR}
    MSDFGEN_VERSION_MINOR=${MSDFGEN_VERSION_MINOR}
    MSDFGEN_VERSION_REVISION=${MSDFGEN_VERSION_REVISION}
    MSDFGEN_COPYRIGHT_YEAR=${MSDFGEN_COPYRIGHT_YEAR}
)
target_include_directories(msdfgen-core INTERFACE
    $<INSTALL_INTERFACE:include/msdfgen>
    $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/>
)
set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT msdfgen-core)

if(MSDFGEN_USE_CPP11)
    target_compile_features(msdfgen-core PUBLIC cxx_std_11)
    target_compile_definitions(msdfgen-core PUBLIC MSDFGEN_USE_CPP11)
endif()

if(BUILD_SHARED_LIBS AND WIN32)
    target_compile_definitions(msdfgen-core PRIVATE "MSDFGEN_PUBLIC=__declspec(dllexport)")
    target_compile_definitions(msdfgen-core INTERFACE "MSDFGEN_PUBLIC=__declspec(dllimport)")
else()
    target_compile_definitions(msdfgen-core PUBLIC MSDFGEN_PUBLIC=)
endif()

# Extensions library
if(NOT MSDFGEN_CORE_ONLY)
    if(NOT TARGET Freetype::Freetype)
        find_package(Freetype REQUIRED)
    endif()
    if(NOT MSDFGEN_DISABLE_SVG AND NOT TARGET tinyxml2::tinyxml2)
        find_package(tinyxml2 REQUIRED)
    endif()
    if(NOT MSDFGEN_DISABLE_PNG AND NOT TARGET PNG::PNG)
        find_package(PNG REQUIRED)
    endif()

    add_library(msdfgen-ext "${CMAKE_CURRENT_SOURCE_DIR}/msdfgen-ext.h" ${MSDFGEN_EXT_HEADERS} ${MSDFGEN_EXT_SOURCES})
    add_library(msdfgen::msdfgen-ext ALIAS msdfgen-ext)
    set_target_properties(msdfgen-ext PROPERTIES PUBLIC_HEADER "${MSDFGEN_EXT_HEADERS}")
    set_property(TARGET msdfgen-ext PROPERTY MSVC_RUNTIME_LIBRARY "${MSDFGEN_MSVC_RUNTIME}")
    target_compile_definitions(msdfgen-ext INTERFACE MSDFGEN_EXTENSIONS)
    if(NOT MSDFGEN_DISABLE_SVG)
        target_link_libraries(msdfgen-ext PRIVATE tinyxml2::tinyxml2)
    else()
        target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_DISABLE_SVG)
    endif()
    if(NOT MSDFGEN_DISABLE_PNG)
        target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_USE_LIBPNG)
        target_link_libraries(msdfgen-ext PRIVATE PNG::PNG)
    else()
        target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_DISABLE_PNG)
    endif()
    if(MSDFGEN_DISABLE_VARIABLE_FONTS)
        target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_DISABLE_VARIABLE_FONTS)
    endif()
    target_link_libraries(msdfgen-ext PRIVATE Freetype::Freetype msdfgen::msdfgen-core)
    target_include_directories(msdfgen-ext
        PUBLIC
            $<INSTALL_INTERFACE:include/msdfgen>
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}>
        PRIVATE
            ${CMAKE_CURRENT_SOURCE_DIR}/include
    )
    set_property(DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR} PROPERTY VS_STARTUP_PROJECT msdfgen-ext)

    if(BUILD_SHARED_LIBS AND WIN32)
        target_compile_definitions(msdfgen-ext PRIVATE "MSDFGEN_EXT_PUBLIC=__declspec(dllexport)")
        target_compile_definitions(msdfgen-ext INTERFACE "MSDFGEN_EXT_PUBLIC=__declspec(dllimport)")
    else()
        target_compile_definitions(msdfgen-ext PUBLIC MSDFGEN_EXT_PUBLIC=)
    endif()

    add_library(msdfgen-full INTERFACE)
    add_library(msdfgen::msdfgen ALIAS msdfgen-full)
    target_link_libraries(msdfgen-full INTERFACE msdfgen::msdfgen-core msdfgen::msdfgen-ext)
else()
    add_library(msdfgen::msdfgen ALIAS msdfgen-core)
endif()

# Hide ZERO_CHECK and ALL_BUILD targets
set_property(GLOBAL PROPERTY USE_FOLDERS ON)
set_property(GLOBAL PROPERTY PREDEFINED_TARGETS_FOLDER meta)

# Installation
if(MSDFGEN_INSTALL)
    include(GNUInstallDirs)
    include(CMakePackageConfigHelpers)
    set(MSDFGEN_CONFIG_PATH "lib/cmake/msdfgen")

    # install tree package config
    write_basic_package_version_file(
        "${CMAKE_CURRENT_BINARY_DIR}/msdfgenConfigVersion.cmake"
        VERSION ${PROJECT_VERSION}
        COMPATIBILITY SameMajorVersion
    )

    configure_package_config_file(
        cmake/msdfgenConfig.cmake.in
        ${MSDFGEN_CONFIG_PATH}/msdfgenConfig.cmake
        INSTALL_DESTINATION ${MSDFGEN_CONFIG_PATH}
        NO_CHECK_REQUIRED_COMPONENTS_MACRO
    )

    # build tree package config
    configure_file(
        cmake/msdfgenConfig.cmake.in
        msdfgenConfig.cmake
        @ONLY
    )

    install(TARGETS msdfgen-core EXPORT msdfgenTargets
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
        ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        FRAMEWORK DESTINATION ${CMAKE_INSTALL_LIBDIR}
        PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/msdfgen/core
    )
    install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/msdfgen.h" DESTINATION include/msdfgen)
    if(MSVC AND BUILD_SHARED_LIBS)
        install(FILES $<TARGET_PDB_FILE:msdfgen-core> DESTINATION ${CMAKE_INSTALL_BINDIR} OPTIONAL)
    endif()

    if(NOT MSDFGEN_CORE_ONLY)
        install(TARGETS msdfgen-ext EXPORT msdfgenTargets
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
            FRAMEWORK DESTINATION ${CMAKE_INSTALL_LIBDIR}
            PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/msdfgen/ext
        )
        install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/msdfgen-ext.h" DESTINATION include/msdfgen)
        if(MSVC AND BUILD_SHARED_LIBS)
            install(FILES $<TARGET_PDB_FILE:msdfgen-ext> DESTINATION ${CMAKE_INSTALL_BINDIR} OPTIONAL)
        endif()
        install(TARGETS msdfgen-full EXPORT msdfgenTargets)
    endif()

    export(EXPORT msdfgenTargets NAMESPACE msdfgen:: FILE "${CMAKE_CURRENT_BINARY_DIR}/msdfgenTargets.cmake")
    install(EXPORT msdfgenTargets FILE msdfgenTargets.cmake NAMESPACE msdfgen:: DESTINATION ${MSDFGEN_CONFIG_PATH})

    install(
        FILES
            "${CMAKE_CURRENT_BINARY_DIR}/${MSDFGEN_CONFIG_PATH}/msdfgenConfig.cmake"
            "${CMAKE_CURRENT_BINARY_DIR}/msdfgenConfigVersion.cmake"
        DESTINATION ${MSDFGEN_CONFIG_PATH}
    )
endif()
