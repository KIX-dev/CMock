###################################################################################
#                                                                                 #
# NAME: CMakeLists.txt                                                            #
#                                                                                 #
# AUTHOR: Mike Karlesky, Mark VanderVoord, Greg Williams.                         #
# WRITTEN BY: Alexey Artyushkov.                                                  #
#                                                                                 #
# License: MIT                                                                    #
#                                                                                 #
###################################################################################

cmake_minimum_required(VERSION 3.12)

# Read src/cmock.h file and get project version from it
set(CMOCK_HEADER "src/cmock.h")

file(STRINGS "${CMOCK_HEADER}" CMOCK_HEADER_CONTENT 
    REGEX "^#define CMOCK_VERSION_(MAJOR|MINOR|BUILD) +[0-9]+$"
)

set(CMOCK_HEADER_VERSION_MAJOR 0)
set(CMOCK_HEADER_VERSION_MINOR 0)
set(CMOCK_HEADER_VERSION_BUILD 0)

foreach(VERSION_LINE IN LISTS CMOCK_HEADER_CONTENT)
    foreach(VERSION_PART MAJOR MINOR BUILD)
        string(CONCAT REGEX_STRING "#define CMOCK_VERSION_" 
                                   "${VERSION_PART}" 
                                   " +([0-9]+)"
        )

        if(VERSION_LINE MATCHES "${REGEX_STRING}")
            set(CMOCK_HEADER_VERSION_${VERSION_PART} "${CMAKE_MATCH_1}")
        endif()
    endforeach()
endforeach()

function(git_submodule_init SUBMODULE_DIR)
    execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init --recursive -- ${SUBMODULE_DIR}
                    WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}
                    RESULT_VARIABLE GIT_SUBMODULE_RESULT)
    if (NOT GIT_SUBMODULE_RESULT EQUAL "0")
        message(FATAL_ERROR "git submodule update --init --recursive -- ${SUBMODULE_DIR} failed with ${GIT_SUBMOD_RESULT}, please checkout submodules")
    endif()
endfunction(git_submodule_init)

project(cmock
    VERSION     ${CMOCK_HEADER_VERSION_MAJOR}.${CMOCK_HEADER_VERSION_MINOR}.${CMOCK_HEADER_VERSION_BUILD}
    LANGUAGES   C
    DESCRIPTION "Mock and stub generator for unit testing C"
)

if (NOT TARGET unity)
    find_package(Git REQUIRED)
    set(UNITY_SUBMODULE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/vendor/unity)
    message(STATUS "Using Unity dependency from Git submodule ${UNITY_SUBMODULE_DIR}")
    git_submodule_init(${UNITY_SUBMODULE_DIR})
    add_subdirectory(${UNITY_SUBMODULE_DIR})
endif()

# Options to Build With Extras -------------------------------------------------
option(CMOCK_EXTENSION_CEXCEPTION "Compiles CMock with the \"CException\" extension." OFF)
set(CMOCK_EXTENSION_CEXCEPTION_ENABLED $<BOOL:${CMOCK_EXTENSION_CEXCEPTION}>)

if(${CMOCK_EXTENSION_CEXCEPTION})
    find_package(Git REQUIRED)
    set(CMOCK_EXTENSION_CEXCEPTION_SUBMODULE_DIR ${CMAKE_CURRENT_SOURCE_DIR}/vendor/c_exception)
    message(STATUS "CMock: Building with the CException extension from Git submodule ${CMOCK_EXTENSION_CEXCEPTION_SUBMODULE_DIR}")
    git_submodule_init(${CMOCK_EXTENSION_CEXCEPTION_SUBMODULE_DIR})
    set(CMOCK_EXTENSION_CEXCEPTION_SOURCES_DIR ${CMOCK_EXTENSION_CEXCEPTION_SUBMODULE_DIR}/lib)
    set(CMOCK_EXTENSION_CEXCEPTION_HEADERS_DIR ${CMOCK_EXTENSION_CEXCEPTION_SUBMODULE_DIR}/lib)
    set(CMOCK_EXTENSION_CEXCEPTION_SOURCE ${CMOCK_EXTENSION_CEXCEPTION_SOURCES_DIR}/CException.c)
    set(CMOCK_EXTENSION_CEXCEPTION_HEADER ${CMOCK_EXTENSION_CEXCEPTION_HEADERS_DIR}/CException.h)
endif()

# Main target ------------------------------------------------------------------
add_library(${PROJECT_NAME} STATIC)
add_library(${PROJECT_NAME}::framework ALIAS ${PROJECT_NAME})
target_link_libraries(${PROJECT_NAME} PUBLIC unity)

# Includes ---------------------------------------------------------------------
include(GNUInstallDirs)
include(CMakePackageConfigHelpers)

target_sources(${PROJECT_NAME}
    PRIVATE
        src/cmock.c
        $<$<BOOL:${CMOCK_EXTENSION_CEXCEPTION_ENABLED}>:${CMOCK_EXTENSION_CEXCEPTION_SOURCE}>
)

target_include_directories(${PROJECT_NAME}
    PUBLIC
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}>
        $<INSTALL_INTERFACE:${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}>
        $<BUILD_INTERFACE:$<$<BOOL:${CMOCK_EXTENSION_CEXCEPTION_ENABLED}>:${CMOCK_EXTENSION_CEXCEPTION_HEADERS_DIR}>>
)

set(${PROJECT_NAME}_PUBLIC_HEADERS
        src/cmock.h
        src/cmock_internals.h
        $<$<BOOL:${CMOCK_EXTENSION_CEXCEPTION_ENABLED}>:${CMOCK_EXTENSION_CEXCEPTION_HEADER}>
)

set_target_properties(${PROJECT_NAME}
    PROPERTIES 
        C_STANDARD          11
        C_STANDARD_REQUIRED ON
        C_EXTENSIONS        OFF
        PUBLIC_HEADER       "${${PROJECT_NAME}_PUBLIC_HEADERS}"
        EXPORT_NAME         framework
)

target_compile_options(${PROJECT_NAME}
    PRIVATE
        # Clang
        $<$<C_COMPILER_ID:Clang>:
            -Wcast-align
            -Wcast-qual
            -Wconversion
            -Wexit-time-destructors                            
            -Wglobal-constructors
            -Wmissing-noreturn
            -Wmissing-prototypes
            -Wno-missing-braces
            -Wold-style-cast
            -Wshadow
            -Wweak-vtables
            -Werror
            -Wall
            $<$<VERSION_GREATER_EQUAL:$<C_COMPILER_VERSION>,8.0.0>:-Wextra-semi-stmt>
        >
        
        # GCC
        $<$<C_COMPILER_ID:GNU>:
            -Waddress
            -Waggregate-return
            -Wformat-nonliteral
            -Wformat-security
            -Wformat
            -Winit-self
            -Wmissing-declarations
            -Wmissing-include-dirs
            -Wno-multichar
            -Wno-parentheses
            -Wno-type-limits
            -Wno-unused-parameter
            -Wunreachable-code
            -Wwrite-strings
            -Wpointer-arith
            -Werror
            -Wall
        >

        # MSVC
        $<$<C_COMPILER_ID:MSVC>:
            /Wall
        >
)

write_basic_package_version_file(${PROJECT_NAME}ConfigVersion.cmake
    VERSION       ${PROJECT_VERSION}
    COMPATIBILITY SameMajorVersion 
)

## Target installation
install(TARGETS   ${PROJECT_NAME}
    EXPORT        ${PROJECT_NAME}Targets
    ARCHIVE       DESTINATION ${CMAKE_INSTALL_LIBDIR}
    LIBRARY       DESTINATION ${CMAKE_INSTALL_LIBDIR}
    PUBLIC_HEADER DESTINATION ${CMAKE_INSTALL_INCLUDEDIR}/${PROJECT_NAME}
    COMPONENT     library
)

## Target's cmake files: targets export
install(EXPORT  ${PROJECT_NAME}Targets
    NAMESPACE   ${PROJECT_NAME}::
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

## Target's cmake files: config and version config for find_package()
install(FILES   ${PROJECT_NAME}Config.cmake
                ${CMAKE_CURRENT_BINARY_DIR}/${PROJECT_NAME}ConfigVersion.cmake
    DESTINATION ${CMAKE_INSTALL_LIBDIR}/cmake/${PROJECT_NAME}
)

## CMock scripts: content of lib directory
file(GLOB CMOCK_RUBY_SCRIPTS ${CMAKE_CURRENT_SOURCE_DIR}/lib/*)
install(FILES     ${CMOCK_RUBY_SCRIPTS}
    DESTINATION   ${CMAKE_INSTALL_LIBDIR}/${PROJECT_NAME}
)
