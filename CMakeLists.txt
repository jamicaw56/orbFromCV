
cmake_minimum_required(VERSION 3.10)

project(StandaloneORB VERSION 1.0 LANGUAGES CXX)

# Require C++14 standard or higher
set(CMAKE_CXX_STANDARD 14)
set(CMAKE_CXX_STANDARD_REQUIRED True)

# Add maximum optimization flags for performance
if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
    set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -O3 -Wall -Wextra")
endif()

# Build every target with AddressSanitizer + UndefinedBehaviorSanitizer.
# Off by default; CI runs the test suite with it on (see .github/workflows/ci.yml).
option(ORB_SANITIZE "Build with AddressSanitizer and UndefinedBehaviorSanitizer" OFF)

if(ORB_SANITIZE)
    if(CMAKE_CXX_COMPILER_ID MATCHES "Clang|GNU")
        add_compile_options(-fsanitize=address,undefined -fno-omit-frame-pointer -fno-sanitize-recover=all)
        add_link_options(-fsanitize=address,undefined)
    elseif(MSVC)
        # MSVC has ASan only (no UBSan); it is incompatible with /RTC1 in Debug.
        add_compile_options(/fsanitize=address)
        string(REGEX REPLACE "/RTC(su|[1su])" "" CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG}")
    else()
        message(WARNING "ORB_SANITIZE is not supported for ${CMAKE_CXX_COMPILER_ID}; ignoring")
    endif()
endif()

# Core algorithm sources, shared by the CLI executable and the test suite
set(LIB_SOURCES
    orb_core.cpp
    fast_detector.cpp
    image_utils.cpp
    orb.cpp
)

# Define the header files (useful to make them appear in IDEs)
set(HEADERS
    orb_core.hpp
    fast_detector.hpp
    image_utils.hpp
    orb_pattern.hpp
    orb.hpp
    stb_image.h
)

add_library(orb_lib STATIC ${LIB_SOURCES} ${HEADERS})
target_include_directories(orb_lib PUBLIC ${CMAKE_CURRENT_SOURCE_DIR})

# Create the main executable
add_executable(orb_test main.cpp)
target_link_libraries(orb_test PRIVATE orb_lib)

# Per-stage timing diagnostic (pyramid, FAST, Harris, orientation,
# descriptors, ...). No external deps, built by default like orb_test.
add_executable(profile_orb tools/profile_orb.cpp)
target_link_libraries(profile_orb PRIVATE orb_lib)

enable_testing()
add_subdirectory(tests)

# Optional diagnostic tool comparing this repo's ORB against OpenCV's cv::ORB.
# Off by default: orb_lib/orb_test/orb_tests never require OpenCV.
# See docs/design/compare-orb-design.md.
option(BUILD_OPENCV_COMPARISON "Build the compare_orb tool (requires OpenCV)" OFF)

if(BUILD_OPENCV_COMPARISON)
    find_package(OpenCV REQUIRED)
    add_executable(compare_orb tools/compare_orb.cpp)
    target_include_directories(compare_orb PRIVATE
        ${OpenCV_INCLUDE_DIRS}
        # Fallback for the official OpenCV Windows package: its per-toolset
        # OpenCVConfig.cmake (under build/x64/vcXX/lib) bakes in absolute
        # include paths from the OpenCV build machine, which don't exist
        # locally. The real headers live at the standard build/include path.
        "${OpenCV_DIR}/../../../include"
    )
    # Prefer the single "world" target when the OpenCV package only ships
    # that (its OpenCV_LIBS can otherwise list per-module libs that don't
    # exist in a world-only build).
    if(TARGET opencv_world)
        target_link_libraries(compare_orb PRIVATE orb_lib opencv_world)
    else()
        target_link_libraries(compare_orb PRIVATE orb_lib ${OpenCV_LIBS})
    endif()
endif()
