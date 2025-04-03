cmake_minimum_required(VERSION 3.21)

# Project configuration
project(BigWhiteTool)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_BUILD_TYPE Release)

# NDK path configuration (using a variable with clear name)
set(ANDROID_NDK_PATH "${CMAKE_SOURCE_DIR}/android-ndk")
# Allow overriding NDK path via environment variable or command line
if(DEFINED ENV{ANDROID_NDK_PATH})
    set(ANDROID_NDK_PATH $ENV{ANDROID_NDK_PATH})
endif()
if(DEFINED ANDROID_NDK_PATH_OVERRIDE)
    set(ANDROID_NDK_PATH ${ANDROID_NDK_PATH_OVERRIDE})
endif()

# Android configuration
set(CMAKE_SYSTEM_NAME ANDROID)
set(CMAKE_SYSTEM_VERSION 21)
set(ANDROID_PLATFORM 21)
set(ANDROID_SDK_TOOLS_VERSION 20)
set(ANDROID_ABI arm64-v8a) # Default architecture
set(ANDROID_NDK ${ANDROID_NDK_PATH})
set(CMAKE_TOOLCHAIN_FILE ${ANDROID_NDK_PATH}/build/cmake/android.toolchain.cmake)
set(ANDROID_SDK_ROOT ${ANDROID_NDK_PATH})

# Support multiple architectures in CI/CD environment
if(DEFINED ENV{TARGET_ABI})
    set(ANDROID_ABI $ENV{TARGET_ABI})
endif()

# Output file directories - organized by architecture
set(OUTPUT_DIR ${CMAKE_SOURCE_DIR}/outputs/${ANDROID_ABI})
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${OUTPUT_DIR}/bin)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${OUTPUT_DIR}/lib)
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${OUTPUT_DIR}/lib)

# Version information from Git (for CI/CD)
find_package(Git QUIET)
if(GIT_FOUND)
    execute_process(
        COMMAND ${GIT_EXECUTABLE} describe --tags --always
        OUTPUT_VARIABLE GIT_VERSION
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    execute_process(
        COMMAND ${GIT_EXECUTABLE} rev-parse --short HEAD
        OUTPUT_VARIABLE GIT_COMMIT_HASH
        OUTPUT_STRIP_TRAILING_WHITESPACE
    )
    add_definitions(-DGIT_VERSION="${GIT_VERSION}")
    add_definitions(-DGIT_COMMIT_HASH="${GIT_COMMIT_HASH}")
    message(STATUS "Git version: ${GIT_VERSION}")
    message(STATUS "Git commit: ${GIT_COMMIT_HASH}")
endif()

# Header files
set(INCLUDE_DIRS
    include/
    include/ImGui
    include/Android_touch
    include/Android_shm
    include/native_surface
    include/Android_draw
    include/Android_dump
    include/request
    include/tools
)
include_directories(${INCLUDE_DIRS})

# Gather all header and source files
file(GLOB_RECURSE FILE_INCLUDES include/*.h*)
file(GLOB_RECURSE FILE_SOURCES src/*.c*)

# Third-party libraries
link_directories(ffmpeg/lib/${ANDROID_ABI})

# Uncomment to use static libraries
# add_library(biu STATIC IMPORTED)
# set_target_properties(biu PROPERTIES IMPORTED_LOCATION ${CMAKE_SOURCE_DIR}/libs/libbiu.a)

# Optional components
option(BUILD_TESTS "Build test suite" OFF)
option(ENABLE_DUMPER "Enable dumper functionality" OFF)

# Main executable target
add_executable(BigWhiteTool
    ${FILE_INCLUDES}
    ${FILE_SOURCES}
    src/BigWhiteTool.cpp
)

# Link libraries
target_link_libraries(BigWhiteTool PRIVATE
    EGL
    GLESv3
    log
    android
    # Add any conditional libraries below
)

# Conditionally include the dumper SDK
if(ENABLE_DUMPER)
    add_subdirectory(DumperSDK)
    target_link_libraries(BigWhiteTool PRIVATE UE5Dumper)
endif()

# Installation rules for CI/CD
install(TARGETS BigWhiteTool
    RUNTIME DESTINATION bin
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)

# Add tests if enabled
if(BUILD_TESTS)
    enable_testing()
    add_subdirectory(tests)
endif()
