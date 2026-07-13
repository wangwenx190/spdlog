cmake_minimum_required(VERSION 4.0)

file(STRINGS "${CMAKE_CURRENT_SOURCE_DIR}/include/spdlog/version.h" version_definitions
    REGEX "^#define[ 	]+SPDLOG_VER_(MAJOR|MINOR|PATCH)[ 	]+[0-9]+")
string(REGEX MATCH "SPDLOG_VER_MAJOR[ 	]+([0-9]+)" version_match "${version_definitions}")
set(version_major "${CMAKE_MATCH_1}")
string(REGEX MATCH "SPDLOG_VER_MINOR[ 	]+([0-9]+)" version_match "${version_definitions}")
set(version_minor "${CMAKE_MATCH_1}")
string(REGEX MATCH "SPDLOG_VER_PATCH[ 	]+([0-9]+)" version_match "${version_definitions}")
set(version_patch "${CMAKE_MATCH_1}")

project(spdlog
    VERSION ${version_major}.${version_minor}.${version_patch}
    DESCRIPTION "Fast C++ logging library"
    LANGUAGES CXX
)

if(NOT DEFINED CMAKE_BUILD_TYPE AND NOT DEFINED CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE "Release")
endif()
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_POSITION_INDEPENDENT_CODE ON)
set(CMAKE_VISIBILITY_INLINES_HIDDEN ON)
set(CMAKE_CXX_VISIBILITY_PRESET "hidden")
if(NOT "x${CMAKE_CXX_FLAGS}" STREQUAL "x")
    if(MSVC)
        string(REGEX REPLACE " [/-]W[01234] " " " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
    else()
        string(REGEX REPLACE " -W(all)?(extra)? " " " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
        string(REGEX REPLACE " -[W]?pedantic " " " CMAKE_CXX_FLAGS ${CMAKE_CXX_FLAGS})
    endif()
endif()
string(APPEND CMAKE_CXX_FLAGS " -w ")

set(__top_level ON)
if("x${CMAKE_SOURCE_DIR}" STREQUAL "x${PROJECT_SOURCE_DIR}")
    message("SPDLOG: Building as a top-level project.")
else()
    set(__top_level OFF)
    message("SPDLOG: Building as a subproject.")
endif()

option(SPDLOG_BUILD_SHARED "Build shared library edition of spdlog." ON)
option(SPDLOG_BUILD_STATIC "Build static library edition of spdlog." ON)
option(SPDLOG_INSTALL "Enable installation for spdlog." OFF)
if(NOT SPDLOG_BUILD_SHARED AND NOT SPDLOG_BUILD_STATIC)
    message(SEND_ERROR "You MUST build either the shared library or the static library or both of them!")
endif()

find_package(Threads REQUIRED)
include(GNUInstallDirs)
file(GLOB_RECURSE __sources LIST_DIRECTORIES false CONFIGURE_DEPENDS
    "./include/spdlog/*.h" "./src/*.h" "./src/*.cpp"
)

if(WIN32 AND SPDLOG_BUILD_SHARED)
    enable_language(RC)
    configure_file(spdlog.rc.in spdlog.rc @ONLY)
endif()

set(__targets)
if(SPDLOG_BUILD_SHARED)
    add_library(${PROJECT_NAME}_shared SHARED)
    add_library(${PROJECT_NAME}::${PROJECT_NAME}_shared ALIAS ${PROJECT_NAME}_shared)
    set_target_properties(${PROJECT_NAME}_shared PROPERTIES OUTPUT_NAME ${PROJECT_NAME})
    target_compile_definitions(${PROJECT_NAME}_shared
        PUBLIC SPDLOG_COMPILED_LIB SPDLOG_SHARED_LIB FMT_SHARED
        PRIVATE SPDLOG_EXPORTS FMT_LIB_EXPORT
    )
    if(WIN32)
        target_sources(${PROJECT_NAME}_shared PRIVATE "${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}.rc")
    endif()
    list(APPEND __targets ${PROJECT_NAME}_shared)
endif()
if(SPDLOG_BUILD_STATIC)
    add_library(${PROJECT_NAME}_static STATIC)
    add_library(${PROJECT_NAME}::${PROJECT_NAME}_static ALIAS ${PROJECT_NAME}_static)
    target_compile_definitions(${PROJECT_NAME}_static PUBLIC SPDLOG_COMPILED_LIB)
    list(APPEND __targets ${PROJECT_NAME}_static)
endif()
if(SPDLOG_BUILD_SHARED AND NOT SPDLOG_BUILD_STATIC)
    add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME}_shared)
endif()
if(SPDLOG_BUILD_STATIC AND NOT SPDLOG_BUILD_SHARED)
    add_library(${PROJECT_NAME}::${PROJECT_NAME} ALIAS ${PROJECT_NAME}_static)
endif()

foreach(__target IN LISTS __targets)
    target_sources(${__target} PRIVATE ${__sources})
    target_include_directories(${__target}
        PUBLIC
            $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/include>
            $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}>
    )
    target_link_libraries(${__target} PUBLIC Threads::Threads)
    if(__top_level)
        if(WIN32)
            target_compile_definitions(${__target} PRIVATE
                UNICODE _UNICODE
                NOMINMAX
                WIN32_LEAN_AND_MEAN
            )
        endif()
        if(MSVC)
            target_compile_options(${__target} PRIVATE
                /utf-8 /permissive- /Zc:__cplusplus
                $<$<NOT:$<CONFIG:Debug>>:/Gw /Gy /Zc:inline>
            )
            target_link_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:/OPT:REF /OPT:ICF>
            )
        else()
            target_compile_options(${__target} PRIVATE
                $<$<NOT:$<CONFIG:Debug>>:-ffunction-sections -fdata-sections>
            )
            if(APPLE)
                target_link_options(${__target} PRIVATE
                    $<$<NOT:$<CONFIG:Debug>>:-Wl,-dead_strip>
                )
            else()
                target_link_options(${__target} PRIVATE
                    $<$<NOT:$<CONFIG:Debug>>:-Wl,--gc-sections>
                )
            endif()
        endif()
    endif()
    set_target_properties(${__target} PROPERTIES VERSION ${PROJECT_VERSION})
    get_target_property(__type ${__target} TYPE)
    if(NOT "x${__type}" STREQUAL "xSTATIC_LIBRARY")
        set_target_properties(${__target} PROPERTIES SOVERSION ${PROJECT_VERSION_MAJOR})
    endif()
    if(SPDLOG_INSTALL)
        install(TARGETS ${__target}
            RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
            LIBRARY DESTINATION ${CMAKE_INSTALL_LIBDIR}
            ARCHIVE DESTINATION ${CMAKE_INSTALL_LIBDIR}
        )
    endif()
endforeach()

if(SPDLOG_INSTALL)
    install(
        DIRECTORY include/spdlog
        DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}
    )
endif()
