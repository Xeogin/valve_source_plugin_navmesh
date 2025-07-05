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

if (LINUX)
    set (tier1 :tier1.a)
    set (mathlib :mathlib.a)
    set (lzma :lzma.a)
    
    # Linux specific SDK library paths
    # These paths are where the Source SDK's buildallprojects for Linux would place libraries.
    # It's common for them to be in a flat 'lib' directory within the SDK itself, or specific to linux64.
    set (SDK_COMMON_LIB_PATH "${HL2SDK_DIR}/lib/linux64/common")
    set (SDK_PUBLIC_LIB_PATH "${HL2SDK_DIR}/lib/linux64/public")
    # Add other potential paths if needed, e.g., "${HL2SDK_DIR}/lib"

    set (SRC_HOME $ENV{HOME})
    set( steam_root "$ENV{HOME}/.steam/steam" )
    set(my_compile_options $<$<CONFIG:Debug>:-Wall> -fPIC -m64)
    list(APPEND shared_defs _LINUX LINUX POSIX)
    set(my_link_options -static-libgcc -static-libstdc++)

else() # This block applies to Windows
    # Windows specific SDK library paths
    # The build in the action.yml produces libraries in this structure:
    # <SDK_ROOT>/src/lib/public/<Platform>/<Configuration>/
    # Where <Platform> is "x64" and <Configuration> is "Release" (from the msbuild call)
    set(SDK_PUBLIC_LIB_PATH "${HL2SDK_DIR}/src/lib/public/${CMAKE_VS_PLATFORM_NAME}/$<$<CONFIG:Debug>:Debug>$<$<CONFIG:Release>:Release>")
    # Some older or common libraries might be in src/lib/public directly
    set(SDK_COMMON_LIB_PATH "${HL2SDK_DIR}/src/lib/public/") # General public path
    # If there are common libs outside of public, e.g., src/lib/common/<platform>/<config>, add them here too.

    set(my_compile_options
        $<$<CONFIG:>:/MT>
        $<$<CONFIG:Debug>:/MTd>
        $<$<CONFIG:Release>:/MT>
    )
    list(APPEND shared_defs COMPILER_MSVC COMPILER_MSVC64 _CRT_NO_VA_START_VALIDATION _CRT_SECURE_NO_DEPRECATE
        _CRT_NONSTDC_NO_DEPRECATE)
    set(my_link_options /NODEFAULTLIB:libcmtd /SAFESEH:NO)
endif ()

# Set global include directories
include_directories(${shared_includes})

# Add source files from subdirectories
add_subdirectory(${PROJECT_SOURCE_DIR}/util)
add_subdirectory(${PROJECT_SOURCE_DIR}/nav_mesh)
add_subdirectory(${PROJECT_SOURCE_DIR}/test)

# --- Find and Link Source SDK Libraries ---
# This block should come AFTER all add_subdirectory calls for your plugin's components
# so that the targets (e.g., plugin_navmesh) are already defined.

# Add the determined SDK library paths to link_directories
link_directories(
    "${SDK_PUBLIC_LIB_PATH}"
    "${SDK_COMMON_LIB_PATH}"
    # You might also need to add the parent directory of these if libraries are directly in 'lib/public' or 'lib/common'
    # Example: "${HL2SDK_DIR}/src/lib/public"
)

# Find specific SDK libraries
# Make sure these library names (e.g., tier0, tier1, vstdlib) match the actual .lib/.a file names
# (e.g., tier0.lib or libtier0.a without the lib/extension part in find_library)
find_library(TIER0_LIB tier0 PATHS "${SDK_PUBLIC_LIB_PATH}" "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH)
find_library(TIER1_LIB tier1 PATHS "${SDK_PUBLIC_LIB_PATH}" "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH)
find_library(VSTDLIB_LIB vstdlib PATHS "${SDK_PUBLIC_LIB_PATH}" "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH)
find_library(MATHLIB_LIB mathlib PATHS "${SDK_PUBLIC_LIB_PATH}" "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH)
# Add other common SDK libraries you know you need, e.g.:
# find_library(PUBLIC_LIB public PATHS "${SDK_PUBLIC_LIB_PATH}" "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH)
# find_library(SERVER_LIB server PATHS "${SDK_PUBLIC_LIB_PATH}" "${SDK_COMMON_LIB_PATH}" NO_DEFAULT_PATH) # For game/server.lib

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

# !!! Assuming your main plugin target is named 'plugin_navmesh' (from add_library or add_executable in subdirs)
# You might need to adjust the target name 'plugin_navmesh' below
# if your main DLL/SO target is named something else within the 'test' subdirectory.
# For example, if 'test' adds `add_library(my_plugin SHARED VSPlugin.cpp)`, then the target is `my_plugin`.
# Looking at your last log: `plugin_navmesh.vcxproj` in `test` subdir, so `plugin_navmesh` is likely correct.

target_link_libraries(
    plugin_navmesh # This should be the target name for your final plugin DLL/SO
    PUBLIC
    ${TIER0_LIB}
    ${TIER1_LIB}
    ${VSTDLIB_LIB}
    ${MATHLIB_LIB}
    # Add any other core SDK libraries your plugin requires (e.g., public.lib, server.lib)
    # The subdirectories util, nav_mesh, test likely create their own static libs that get linked:
    # So, ensure you link against the libraries created by these subdirectories if they are static libs
    # Example:
    # util_lib # if util creates a static library named 'util_lib'
    # nav_mesh_lib # if nav_mesh creates a static library named 'nav_mesh_lib'
)

# For Linux, you might need to add system libraries explicitly if not handled by your toolchain file
if (LINUX)
    target_link_libraries(plugin_navmesh PUBLIC dl) # For dlopen/dlsym
endif()

# Set global include directories (re-added here as a reminder, good practice to keep them at the top)
# include_directories(${shared_includes}) # This was already present, just ensure it's still there.
