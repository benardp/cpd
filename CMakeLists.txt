cmake_minimum_required(VERSION 3.13)
set(CPD_LANGUAGES CXX)
set(CPD_VERSION 0.5.5)
set(CPD_SOVERSION 0)
set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

project(cpd LANGUAGES ${CPD_LANGUAGES} VERSION ${CPD_VERSION})

# Upstream
set(CMAKE_MODULE_PATH "${CMAKE_CURRENT_SOURCE_DIR}/cmake")
find_package(Eigen3 REQUIRED CONFIG)
find_package(Fgt QUIET CONFIG)
option(WITH_FGT "Build with fgt" ${Fgt_FOUND})
if(WITH_FGT)
    find_package(Fgt REQUIRED CONFIG)
endif()


# Configuration
#if ("${CMAKE_CXX_COMPILER_ID}" STREQUAL "MSVC")
#    option(BUILD_SHARED_LIBS "Build shared libraries" OFF)
#else()
#    option(BUILD_SHARED_LIBS "Build shared libraries" ON)
#endif()

include(CMakePackageConfigHelpers)
write_basic_package_version_file("${PROJECT_BINARY_DIR}/cmake/cpd-config-version.cmake" VERSION ${CPD_VERSION} COMPATIBILITY AnyNewerVersion)
configure_file(cmake/cpd-config.cmake.in "${PROJECT_BINARY_DIR}/cmake/cpd-config.cmake" @ONLY)
install(FILES "${PROJECT_BINARY_DIR}/cmake/cpd-config.cmake" "${PROJECT_BINARY_DIR}/cmake/cpd-config-version.cmake" DESTINATION lib/cmake/cpd)
configure_file(src/version.cpp.in "${PROJECT_BINARY_DIR}/src/version.cpp")


# C++ library
set(library-src
    src/affine.cpp
    src/gauss_transform.cpp
    src/matrix.cpp
    src/nonrigid.cpp
    src/normalization.cpp
    src/rigid.cpp
    src/transform.cpp
    src/utils.cpp
    "${PROJECT_BINARY_DIR}/src/version.cpp"
    )
if(WITH_FGT)
    list(APPEND library-src src/gauss_transform_fgt.cpp)
else()
    list(APPEND library-src src/gauss_transform_make_default.cpp)
endif()

add_library(cpd STATIC ${library-src})
set_target_properties(cpd PROPERTIES
    OUTPUT_NAME cpd
    VERSION ${CPD_VERSION}
    SOVERSION ${CPD_SOVERSION}
    )
target_include_directories(cpd
    PUBLIC
    $<BUILD_INTERFACE:${PROJECT_SOURCE_DIR}/include>
    $<INSTALL_INTERFACE:include>
    ${EIGEN3_INCLUDE_DIR}
    )
if(WITH_FGT)
    target_link_libraries(cpd PRIVATE Fgt::fgt)
    target_compile_definitions(cpd PRIVATE CPD_WITH_FGT)
endif()

option(WITH_STRICT_WARNINGS "Build with stricter warnings" ON)
if(WITH_STRICT_WARNINGS)
    if("${CMAKE_CXX_COMPILER_ID}" MATCHES "Clang")
        target_compile_options(cpd PRIVATE -Wall -pedantic -Wno-gnu-zero-variadic-macro-arguments)
    endif()
endif()


# Tests
option(WITH_TESTS "Build test suite" ON)
if(WITH_TESTS)
    enable_testing()
    set(GOOGLETEST_VERSION 1.14.0)
    add_subdirectory("${PROJECT_SOURCE_DIR}/vendor/googletest-1.14.0/googletest")
    add_subdirectory(tests)
endif()


# Install
# install(TARGETS cpd DESTINATION lib EXPORT cpd-targets)
# install(DIRECTORY include/cpd DESTINATION include)
# install(EXPORT cpd-targets NAMESPACE Cpd:: DESTINATION lib/cmake/cpd)


# Docs
find_package(Doxygen QUIET)
option(WITH_DOCS "Add documentation target" ${Doxygen_FOUND})
if(WITH_DOCS)
    find_package(Doxygen REQUIRED)
    configure_file(docs/Doxyfile.in "${PROJECT_BINARY_DIR}/docs/Doxyfile")
    add_custom_target(doc COMMAND ${DOXYGEN_EXECUTABLE} "${PROJECT_BINARY_DIR}/docs/Doxyfile")
endif()


# Components
find_package(jsoncpp QUIET)
option(WITH_JSONCPP "Build with jsoncpp" ${jsoncpp_FOUND})
if(WITH_JSONCPP)
    find_package(jsoncpp REQUIRED)
    add_subdirectory(components/jsoncpp)
endif()
