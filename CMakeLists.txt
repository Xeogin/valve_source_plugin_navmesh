cmake_minimum_required (VERSION 3.28.3) # Keep this version or upgrade carefully
project (plugin_navmesh)

# Remove or comment out these lines as they are not needed when SOURCE_SDK_PATH is passed
# set(DRIVE c CACHE STRING "User specified drive for Windows")
# set( steam_root "${DRIVE}:/Program Files (x86)/steam" )

# !!! CRITICAL CHANGE HERE !!!
# Use the SOURCE_SDK_PATH variable passed from the GitHub Action.
# This variable points directly to the 'source-sdk-2013' checkout.
set(HL2SDK_DIR "${SOURCE_SDK_PATH}")
# Ensure HL2SDK_DIR uses forward slashes to prevent backslash escape issues on Windows
string(REPLACE "\\" "/" HL2SDK_DIR "${HL2SDK_DIR}") # Add this line

set (HL2SDK_PUB_LIBS ${HL2SDK_DIR}/lib/public)
set (HL2SDK_COM_LIBS ${HL2SDK_DIR}/lib/common)
set (mods "tf;dod;hl2mp") # You can remove these if not needed by your build logic
set(shared_defs PLATFORM_64BITS X64BITS RAD_TELEMETRY_DISABLED $<$<CONFIG:Debug>:_DEBUG>)

set (tier1 tier1) # These will be overridden by the if(LINUX) block for Linux
set (mathlib mathlib)
set (lzma lzma)

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
    list(APPEND shared_defs __clang__)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
    list(APPEND shared_defs GNUC) # GNUC is often important for Source SDK
endif()

if (LINUX)
    set (tier1 :tier1.a)
    set (mathlib :mathlib.a)
    set (lzma :lzma.a)
    # The 'lib' directory structure in Valve's SDK is a bit tricky.
    # The Linux 64-bit libraries might be in source-sdk-2013/lib/linux64
    # rather than under HL2SDK_PUB_LIBS/linux64. Adjust as needed.
    # If source-sdk-2013 has pre-compiled libraries, ensure these paths point correctly.
    # For now, let's point them to the general lib dir if the specific /public or /common subdirs don't exist under 'lib'.
    set (HL2SDK_COM_LIBS ${HL2SDK_DIR}/lib/linux64/common) # Assuming this structure
    set (HL2SDK_PUB_LIBS ${HL2SDK_DIR}/lib/linux64/public) # Assuming this structure
    # If the above don't exist, you might need to adjust or rely on your project compiling them.
    
    set (SRC_HOME $ENV{HOME})
    set( steam_root "$ENV{HOME}/.steam/steam" )
    set(my_compile_options $<$<CONFIG:Debug>:-Wall> -fPIC -m64)
    list(APPEND shared_defs _LINUX LINUX POSIX)
    set(my_link_options -static-libgcc -static-libstdc++)
else() # This block applies to Windows
    # Windows specific settings (keep these for cross-platform compatibility)
    # For now, comment them out if they refer to non-existent pre-compiled binaries
    # and rely on your subdirectories (tier1, mathlib) to build necessary components.

    set(my_compile_options
        $<$<CONFIG:>:/MT>
        $<$<CONFIG:Debug>:/MTd>
        $<$<CONFIG:Release>:/MT>
    )
    list(APPEND shared_defs COMPILER_MSVC COMPILER_MSVC64 _CRT_NO_VA_START_VALIDATION _CRT_SECURE_NO_DEPRECATE
        _CRT_NONSTDC_NO_DEPRECATE)
    set(my_link_options /NODEFAULTLIB:libcmtd /SAFESEH:NO)
endif ()

# !!! CRITICAL CHANGE HERE !!!
# Update shared_includes to point to the actual SDK structure.
# Headers for `public`, `tier0`, `engine`, `tier1`, `mathlib`, `game/server`, `game/shared`
# are usually under the `src` directory of the ValveSoftware/source-sdk-2013 repo.
set(shared_includes
    "${HL2SDK_DIR}/src/public"
    "${HL2SDK_DIR}/src/public/tier0"
    "${HL2SDK_DIR}/src/public/engine"
    "${HL2SDK_DIR}/src/public/tier1"
    "${HL2SDK_DIR}/src/public/mathlib"
    "${HL2SDK_DIR}/src/public/game/server"
    "${HL2SDK_DIR}/src/game/shared"
    "${HL2SDK_DIR}/src/game/server"
)

# Your subdirectories must exist relative to your plugin's CMakeLists.txt
add_subdirectory(${PROJECT_SOURCE_DIR}/util)
add_subdirectory(${PROJECT_SOURCE_DIR}/nav_mesh)
add_subdirectory(${PROJECT_SOURCE_DIR}/test)
