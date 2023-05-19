cmake_minimum_required(VERSION 3.21)
project(target-cortexm LANGUAGES C CXX)

# store defaults in cache so they can be overriden on the command line

set(CMAKE_C_STANDARD 11 CACHE STRING "Default C version")
set(CMAKE_CXX_STANDARD 17 CACHE STRING "Default C++ version")

set(CMAKE_C_EXTENSIONS OFF)
set(CMAKE_C_STANDARD_REQUIRED ON)
set(CMAKE_CXX_EXTENSIONS OFF)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

set(CMAKE_EXPORT_COMPILE_COMMANDS ON)

option(RTOS "Enable RTOS support" OFF)

message(STATUS "Toolchain file: ${CMAKE_TOOLCHAIN_FILE}")
message(STATUS "Language versions: C++${CMAKE_CXX_STANDARD} C${CMAKE_C_STANDARD}")

# clang-tidy config needs to include Arm cross compiler header dirs

cmake_path(GET TOOLCHAIN PARENT_PATH TOOLCHAIN_BASE)
cmake_path(SET TOOLCHAIN_INCLUDE ${TOOLCHAIN_BASE}/arm-none-eabi/include)
FILE(GLOB TOOLCHAIN_CXX_VER RELATIVE ${TOOLCHAIN_INCLUDE}/c++ ${TOOLCHAIN_INCLUDE}/c++/*)

message(STATUS "Toolchain include folder: ${TOOLCHAIN_INCLUDE}")

set(CLANG_TIDY_OPTIONS 
  --checks=-*,readability-* 
  --extra-arg-before=-I${TOOLCHAIN_INCLUDE}/c++/${TOOLCHAIN_CXX_VER}/arm-none-eabi
  --extra-arg-before=-I${TOOLCHAIN_INCLUDE}/c++/${TOOLCHAIN_CXX_VER}
  --extra-arg-before=-I${TOOLCHAIN_INCLUDE}
  CACHE STRING "clang-tidy options" 
)

# uncomment this line to include clang_tidy as part of 
# the compilation phase
# set(CMAKE_CXX_CLANG_TIDY clang-tidy ${CLANG_TIDY_OPTIONS})

add_compile_options(
    -Wall
    -Wextra
    -Wconversion
    -Wsign-conversion
    # -Werror
    # -save-temps
    $<$<AND:$<COMPILE_LANGUAGE:CXX>,$<EQUAL:${CMAKE_CXX_STANDARD},20>>:-Wno-deprecated>
    # $<$<AND:$<COMPILE_LANGUAGE:CXX>,$<VERSION_GREATER_EQUAL:${CMAKE_CXX_STANDARD},20>>:-fcoroutines>
    $<$<AND:$<COMPILE_LANGUAGE:CXX>,$<VERSION_GREATER_EQUAL:${CMAKE_CXX_STANDARD},20>>:-fmodules-ts>
    $<$<CONFIG:DEBUG>:-g3>
    $<$<CONFIG:DEBUG>:-Og>
    $<$<CONFIG:RELEASE>:-O3>
    $<IF:$<BOOL:${EXCEPTIONS}>,-fexceptions,-fno-exceptions>
#    $<IF:$<AND:$<BOOL:${RTOS}>,$<COMPILE_LANGUAGE:CXX>>,-fexceptions,>
)

add_compile_definitions(
    $<$<CONFIG:DEBUG>:DEBUG>
    $<$<CONFIG:DEBUG>:TRACE_ENABLED>
    $<$<CONFIG:RELEASE>:NDEBUG>            # GCC/G++ disables asserts
    $<$<BOOL:${RTOS}>:RTOS>
)

# Standard configuration - change with care

if (EXISTS ${CMAKE_SOURCE_DIR}/src/Modules.txt)
  file(STRINGS ${CMAKE_SOURCE_DIR}/src/Modules.txt MODULE_SRC)
  list(TRANSFORM MODULE_SRC STRIP)
  message(STATUS "Building modules: ${MODULE_SRC}")
  list(TRANSFORM MODULE_SRC PREPEND ${CMAKE_SOURCE_DIR}/src/)
  add_library(modules OBJECT
    ${MODULE_SRC}
  )

  target_include_directories(modules PRIVATE
    ${CMAKE_SOURCE_DIR}/src
    ${CMAKE_SOURCE_DIR}/include
  )
  target_compile_options(modules PRIVATE
    -xc++
  )
endif()

FILE (GLOB USER_SRC 
    ${CMAKE_SOURCE_DIR}/src/*.c 
    ${CMAKE_SOURCE_DIR}/src/*.cpp
)

add_executable(Application ${USER_SRC})

target_include_directories(Application PRIVATE
  ${CMAKE_SOURCE_DIR}/src
  ${CMAKE_SOURCE_DIR}/include
)

if (TARGET modules)
    target_link_libraries(Application PRIVATE modules)
endif()


set(LINKER_SCRIPTS 
    ${CMAKE_SOURCE_DIR}/ldscripts/mem.ld 
    ${CMAKE_SOURCE_DIR}/ldscripts/sections.ld
)

set_target_properties(Application PROPERTIES
    SUFFIX .elf
    LINK_DEPENDS "${LINKER_SCRIPTS}"
)

target_link_options(Application PRIVATE
  -T${CMAKE_SOURCE_DIR}/ldscripts/mem.ld
  -T${CMAKE_SOURCE_DIR}/ldscripts/sections.ld
  LINKER:--print-memory-usage
  LINKER:--sort-section=name
  LINKER:--cref
  LINKER:-Map,${CMAKE_CURRENT_BINARY_DIR}/Application.map
)

if (IS_DIRECTORY ${CMAKE_SOURCE_DIR}/system)
  add_subdirectory(system)
  target_link_libraries(Application PRIVATE system)
endif()

if (USE_RTOS AND IS_DIRECTORY ${CMAKE_SOURCE_DIR}/middleware)
  add_subdirectory(middleware)
  target_link_libraries(Application PRIVATE middleware)
endif()

if (EXISTS ${CMAKE_SOURCE_DIR}/src/main.c AND IS_DIRECTORY ${CMAKE_SOURCE_DIR}/drivers-c)
  add_subdirectory(drivers-c)
  target_link_libraries(Application PRIVATE drivers-c)
endif()

if (EXISTS ${CMAKE_SOURCE_DIR}/src/main.cpp  AND IS_DIRECTORY ${CMAKE_SOURCE_DIR}/drivers-cpp)
  add_subdirectory(drivers-cpp)
  target_link_libraries(Application PRIVATE drivers-cpp)
endif()

# Post build processing

if (EXISTS "${TOOLCHAIN_SIZE}")
  add_custom_command(
    TARGET Application
    POST_BUILD
    COMMAND ${TOOLCHAIN_SIZE} --format=berkeley $<TARGET_FILE:Application> 
            >${CMAKE_CURRENT_BINARY_DIR}/$<TARGET_NAME:Application>.bsz
  )

  add_custom_command(
    TARGET Application
    POST_BUILD
    COMMAND ${TOOLCHAIN_SIZE} --format=sysv -x $<TARGET_FILE:Application> 
            >${CMAKE_CURRENT_BINARY_DIR}/$<TARGET_NAME:Application>.ssz
  )
else()
  message(STATUS "'size' not found: cannot generate .[bs]sz files")
endif()

if (EXISTS ${CMAKE_OBJCOPY})
  add_custom_command(
    TARGET Application
    POST_BUILD
    COMMAND ${CMAKE_OBJCOPY} -O ihex $<TARGET_FILE:Application> 
            ${CMAKE_CURRENT_BINARY_DIR}/$<TARGET_NAME:Application>.hex
  )
else()
  message(STATUS "'objcopy' not found: cannot generate .hex file")
endif()

# static analysis targets

find_program(CLANG_TIDY clang-tidy)

if (CLANG_TIDY)
    add_custom_target(
        clang-tidy clang-tidy -p ${CMAKE_BINARY_DIR} ${CLANG_TIDY_OPTIONS} ${USER_SRC}
        DEPENDS ${USER_SRC}
    )
endif()

# optional testing

if (EXISTS ${CMAKE_SOURCE_DIR}/tests/CMakeLists.txt)
  enable_testing()
  add_subdirectory(tests)
endif()
