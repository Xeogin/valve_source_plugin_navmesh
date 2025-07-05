cmake_minimum_required(VERSION 3.10)
project(valve_source_plugin_navmesh)

# Set common compile options and definitions
set(my_compile_options -g -Wall)
set(shared_defs PLUGIN_EXPORTS)

# Add platform-specific definitions
if (CMAKE_SYSTEM_NAME STREQUAL "Linux")
    add_definitions(-D_LINUX)
    # Use CMAKE_SIZEOF_VOID_P to determine 32-bit or 64-bit for Source SDK definitions
    if (CMAKE_SIZEOF_VOID_P EQUAL 8)
        add_definitions(-D_X64) # Define for 64-bit Linux
        add_definitions(-DCOMPILER_GCC) # Explicitly define GCC compiler
    elseif (CMAKE_SIZEOF_VOID_P EQUAL 4)
        add_definitions(-D_X32) # Define for 32-bit Linux
        add_definitions(-DCOMPILER_GCC) # Explicitly define GCC compiler
    endif()
elseif (CMAKE_SYSTEM_NAME STREQUAL "Windows")
    add_definitions(-D_WIN32)
    if (CMAKE_SIZEOF_VOID_P EQUAL 8)
        add_definitions(-D_WIN64) # Define for 64-bit Windows
    endif()
    add_definitions(-DCOMPILER_MSVC) # Explicitly define MSVC compiler
endif()

# Define output directories
set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/lib")
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY "${CMAKE_BINARY_DIR}/bin")

# Define HL2SDK_DIR relative to your project's source directory
# IMPORTANT: This path assumes hl2sdk is checked out as a sibling directory
# to your 'valve_source_plugin_navmesh' project folder in the workspace root.
# For example, if your workspace is /home/runner/work/your_repo/your_repo/
# then HL2SDK would be in /home/runner/work/your_repo/hl2sdk/
set(HL2SDK_DIR "${CMAKE_CURRENT_SOURCE_DIR}/../hl2sdk") 

# Determine architecture-specific directory for HL2SDK libraries
# This logic will correctly set HL2SDK_ARCH_DIR to "linux64" for 64-bit builds.
if (CMAKE_SIZEOF_VOID_P EQUAL 8) # 64-bit
    set(HL2SDK_ARCH_DIR "linux64") # For Linux 64-bit
elseif (CMAKE_SIZEOF_VOID_P EQUAL 4) # 32-bit
    set(HL2SDK_ARCH_DIR "linux32") # For Linux 32-bit
else()
    message(FATAL_ERROR "Unsupported architecture size")
endif()

# Define the paths for HL2SDK common and public libraries
# These paths are where the *compiled* HL2SDK libraries (.a files) are expected to be.
set(HL2SDK_COM_LIBS "${HL2SDK_DIR}/lib/common/${HL2SDK_ARCH_DIR}")
set(HL2SDK_PUB_LIBS "${HL2SDK_DIR}/lib/public/${HL2SDK_ARCH_DIR}")

# Add these directories to the linker search path globally
link_directories(${HL2SDK_PUB_LIBS})
link_directories(${HL2SDK_COM_LIBS})

# Define the base names for HL2SDK libraries (no :.a syntax)
set(tier0 tier0)
set(vstdlib vstdlib)
set(tier1 tier1)
set(mathlib mathlib)
set(lzma lzma)

# Add subdirectories for plugin and test builds
add_subdirectory(test)

# You can also add more subdirectories here if your project has them
