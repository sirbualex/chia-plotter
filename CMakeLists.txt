cmake_minimum_required(VERSION 3.10)
set(CMAKE_CXX_STANDARD 17)
project(chia-plotter C CXX ASM)


if (MSVC)
	set(CMAKE_CXX_FLAGS_RELEASE "${CMAKE_CXX_FLAGS_RELEASE} /MT")
	set(CMAKE_CXX_FLAGS_DEBUG "${CMAKE_CXX_FLAGS_DEBUG} /MTd")
    add_definitions(-D_CRT_SECURE_NO_WARNINGS)   
else()
	find_package(Threads REQUIRED)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wall -Wno-unused-function")
endif()

add_subdirectory(lib/bls-signatures)


set(BLAKE3_PATH lib/BLAKE3/c)
set(FSE_PATH lib/FSE/lib)

include_directories(
	lib
	include
	lib/bls-signatures/src
	${BLAKE3_PATH}
	${CMAKE_BINARY_DIR}/_deps/relic-src/include
	${CMAKE_BINARY_DIR}/_deps/relic-build/include
	${sodium_INCLUDE_DIR}
)

#add_definitions(-DBLAKE3_NO_SSE2=true)
#add_definitions(-DBLAKE3_NO_SSE41=true)
#add_definitions(-DBLAKE3_NO_AVX2=true)
#add_definitions(-DBLAKE3_NO_AVX512=true)

IF (WIN32)
set(BLAKE3_SRC
    ${BLAKE3_PATH}/blake3.c
    ${BLAKE3_PATH}/blake3_portable.c
    ${BLAKE3_PATH}/blake3_dispatch.c

    ${BLAKE3_PATH}/blake3_sse2.c
    ${BLAKE3_PATH}/blake3_sse41.c
    ${BLAKE3_PATH}/blake3_avx2.c
    ${BLAKE3_PATH}/blake3_avx512.c
)
ELSEIF(OSX_NATIVE_ARCHITECTURE STREQUAL "arm64")
set(BLAKE3_SRC
    ${BLAKE3_PATH}/blake3.c
    ${BLAKE3_PATH}/blake3_portable.c
    ${BLAKE3_PATH}/blake3_dispatch.c
)
ELSE()
set(BLAKE3_SRC
    ${BLAKE3_PATH}/blake3.c
    ${BLAKE3_PATH}/blake3_portable.c
    ${BLAKE3_PATH}/blake3_dispatch.c
    ${BLAKE3_PATH}/blake3_avx2_x86-64_unix.S
    ${BLAKE3_PATH}/blake3_avx512_x86-64_unix.S
    ${BLAKE3_PATH}/blake3_sse41_x86-64_unix.S
    ${BLAKE3_PATH}/blake3_sse2_x86-64_unix.S
)
ENDIF()

set(FSE_SRC
    ${FSE_PATH}/fse_compress.c
    ${FSE_PATH}/fse_decompress.c
    ${FSE_PATH}/entropy_common.c
    ${FSE_PATH}/hist.c
)

add_library(blake3 STATIC ${BLAKE3_SRC})

add_library(fse STATIC ${FSE_SRC})

add_library(chia_plotter STATIC
	lib/chacha8.c
	src/settings.cpp
)
if (MSVC)
	target_link_libraries(chia_plotter blake3 fse)
else()
	target_link_libraries(chia_plotter blake3 fse Threads::Threads)
endif()

add_executable(test_copy test/test_copy.cpp)
add_executable(test_disk_sort test/test_disk_sort.cpp)

add_executable(test_phase_1 test/test_phase_1.cpp)
add_executable(test_phase_2 test/test_phase_2.cpp)
add_executable(test_phase_3 test/test_phase_3.cpp)
add_executable(test_phase_4 test/test_phase_4.cpp)

add_executable(check_phase_1 test/check_phase_1.cpp)

add_executable(chia_plot src/chia_plot.cpp)

if (MSVC)

	add_library(uint128 STATIC uint128_t/uint128_t.cpp)
	target_include_directories(uint128 PUBLIC uint128_t)

	target_link_libraries(test_copy chia_plotter)
	target_link_libraries(test_disk_sort chia_plotter uint128)
	target_link_libraries(test_phase_1 chia_plotter uint128)
	target_link_libraries(test_phase_2 chia_plotter uint128)
	target_link_libraries(test_phase_3 chia_plotter uint128)
	target_link_libraries(test_phase_4 chia_plotter uint128)
	target_link_libraries(check_phase_1 chia_plotter uint128)

	target_link_libraries(chia_plot chia_plotter bls uint128)
	set_target_properties(chia_plot PROPERTIES LINK_OPTIONS -NODEFAULTLIB:LIBCMT)

else()
	
target_link_libraries(test_copy chia_plotter)
target_link_libraries(test_disk_sort chia_plotter)

target_link_libraries(test_phase_1 chia_plotter)
target_link_libraries(test_phase_2 chia_plotter)
target_link_libraries(test_phase_3 chia_plotter)
target_link_libraries(test_phase_4 chia_plotter)

target_link_libraries(check_phase_1 chia_plotter)

target_link_libraries(chia_plot chia_plotter bls)

endif()
