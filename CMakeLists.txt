cmake_minimum_required (VERSION 3.28.3) # Keep this version or upgrade carefully
project (plugin_navmesh)

# !!! CRITICAL CHANGE HERE !!!
# Use the SOURCE_SDK_PATH variable passed from the GitHub Action.
# This variable points directly to the 'source-sdk-2013' checkout.
set(HL2SDK_DIR "${SOURCE_SDK_PATH}")
# Ensure HL2SDK_DIR uses forward slashes to prevent backslash escape issues on Windows
string(REPLACE "\\" "/" HL2SDK_DIR "${HL2SDK_DIR}") # Add this line

# These will be dynamically set based on OS and build configuration
set(SDK_COMMON_LIB_PATH "")
set(SDK_PUBLIC_LIB_PATH "")

# Existing initializations (will be overridden)
set (tier1 tier1) # These will be overridden by the if(LINUX) block for Linux
set (mathlib mathlib)
set (lzma lzma)

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    list(APPEND shared_defs __clang__)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    list(APPEND shared_defs GNUC) # GNUC is often important for Source SDK
endif()

# --- START: MODIFIED SECTION FOR SHARED INCLUDES ---
# Define Source SDK include directories to be used by all targets
list(APPEND shared_includes
    "${HL2SDK_DIR}/src/public"
    "${HL2SDK_DIR}/src/public/tier0"
    "${HL2SDK_DIR}/src/public/tier1"
    "${HL2SDK_DIR}/src/common"
    "${HL2SDK_DIR}/src/public/mathlib"
    "${HL2SDK_DIR}/src/public/vgui"
    "${HL2SDK_DIR}/src/public/engine"     # Added for IEngineTrace.h, ivdebugoverlay.h
    "${HL2SDK_DIR}/src/public/game/server" # Added for iplayerinfo.h (common location)
    "${HL2SDK_DIR}/src/public/player"      # Another possible location for iplayerinfo.h
    "${HL2SDK_DIR}/src/public/game"
    "${HL2SDK_DIR}/src/game/shared" # <--- ADDED THIS LINE: Very common SDK include path
)
# Make these globally available for all source files in the project
include_directories(${shared_includes})
# --- END: MODIFIED SECTION FOR SHARED INCLUDES ---


if (LINUX)
    set (tier1 :tier1.a)
    set (mathlib :mathlib.a)
    set (lzma :lzma.a)
    
    # Linux specific SDK library paths
    set (SDK_COMMON_LIB_PATH "${HL2SDK_DIR}/lib/linux64/common")
    set (SDK_PUBLIC_LIB_PATH "${HL2SDK_DIR}/lib/linux64/public")

    set (SRC_HOME $ENV{HOME})
    set( steam_root "$ENV{HOME}/.steam/steam" )
    set(my_compile_options $<$<CONFIG:Debug>:-Wall> -fPIC -m64)
    list(APPEND shared_defs _LINUX LINUX POSIX)
    set(my_link_options -static-libgcc -static-libstdc++)

else() # This block applies to Windows
    # Windows specific SDK library paths
    # BASED ON YOUR LOG: The libraries are directly under x64, not in a Release/Debug subfolder.
    set(SDK_PUBLIC_LIB_PATH "${HL2SDK_DIR}/src/lib/public/${CMAKE_VS_PLATFORM_NAME}")
    
    # !!! ADDING THIS PATH BASED ON YOUR LOG !!!
    # It appears some common libs like tier1 and mathlib might be output here.
    set(SDK_COMMON_LIB_PATH "${HL2SDK_DIR}/src/lib/common/${CMAKE_VS_PLATFORM_NAME}/2015/release") 

    set(my_compile_options
        $<$<CONFIG:>:/MT>
        $<$<CONFIG:Debug>:/MTd>
        $<$<CONFIG:Release>:/MT>
    )
    list(APPEND shared_defs COMPILER_MSVC COMPILER_MSVC64 _CRT_NO_VA_START_VALIDATION _CRT_SECURE_NO_DEPRECATE
        _CRT_NONSTDC_NO_DEPRECATE)
    
    # --- MODIFIED: Explicitly setting linker paths for Windows ---
    set(my_link_options "/NODEFAULTLIB:libcmtd /SAFESEH:NO /LIBPATH:${SDK_PUBLIC_LIB_PATH} /LIBPATH:${SDK_COMMON_LIB_PATH}")
endif ()


# Add source files from subdirectories
# CMake's 'include_directories' command above makes these paths available globally.
# However, for stricter modern CMake, you'd use target_include_directories on each specific target,
# passing ${shared_includes} to them. For now, this global include_directories should work.
add_subdirectory(${PROJECT_SOURCE_DIR}/util)
add_subdirectory(${PROJECT_SOURCE_DIR}/nav_mesh)
add_subdirectory(${PROJECT_SOURCE_DIR}/test)

# --- Find and Link Source SDK Libraries ---

# --- COMMENTED OUT: Explicit /LIBPATH is now used in my_link_options ---
# link_directories(
#     "${SDK_PUBLIC_LIB_PATH}"
#     "${SDK_COMMON_LIB_PATH}" # Now includes the new common path
# )

# Find specific SDK libraries
find_library(TIER0_LIB tier0 PATHS "${SDK_PUBLIC_LIB_PATH}" "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH)
find_library(TIER1_LIB tier1 PATHS "${SDK_PUBLIC_LIB_PATH}" "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH)
find_library(VSTDLIB_LIB vstdlib PATHS "${SDK_PUBLIC_LIB_PATH}" "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH)
find_library(MATHLIB_LIB mathlib PATHS "${SDK_PUBLIC_LIB_PATH}" "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH)
# Also add LZMA from the common path, as your build output shows it there
find_library(LZMA_LIB lzma PATHS "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH)


# Check if required libraries were found
if(NOT TIER0_LIB)
    message(FATAL_ERROR "Could not find tier0 library. Searched in: ${SDK_PUBLIC_LIB_PATH}, ${SDK_COMMON_LIB_PATH}")
endif()
if(NOT TIER1_LIB)
    message(FATAL_ERROR "Could not find tier1 library. Searched in: ${SDK_PUBLIC_LIB_PATH}, ${SDK_COMMON_LIB_PATH}")
endif()
if(NOT VSTDLIB_LIB)
    message(FATAL_ERROR "Could not find vstdlib library. Searched in: ${SDK_PUBLIC_LIB_PATH}, ${SDK_COMMON_LIB_PATH}")
endif()
if(NOT MATHLIB_LIB)
    message(FATAL_ERROR "Could not find mathlib library. Searched in: ${SDK_PUBLIC_LIB_PATH}, ${SDK_COMMON_LIB_PATH}")
endif()
if(NOT LZMA_LIB)
    message(FATAL_ERROR "Could not find lzma library. Searched in: ${SDK_COMMON_LIB_PATH}")
endif()

# --- NEW: Apply the custom link options to your target ---
target_link_options(plugin_navmesh PUBLIC ${my_link_options})

target_link_libraries(
    plugin_navmesh # This should be the target name for your final plugin DLL/SO
    PUBLIC
    ${TIER0_LIB}
    ${TIER1_LIB}
    ${VSTDLIB_LIB}
    ${MATHLIB_LIB}
    ${LZMA_LIB} # Added LZMA
    # Add any other core SDK libraries your plugin requires (e.g., public.lib, server.lib)
)

# For Linux, you might need to add system libraries explicitly if not handled by your toolchain file
if (LINUX)
    target_link_libraries(plugin_navmesh PUBLIC dl) # For dlopen/dlsym
endif()

# Removed redundant 'include_directories(${shared_includes})' comment
