# Add collisionutils as a source file, assuming it's part of the SDK that your
# project compiles directly. The paths must point to the actual files in Valve's repo.
# The 'src/public' prefix is crucial for ValveSoftware/source-sdk-2013.
add_library(collisionutils STATIC ${HL2SDK_DIR}/src/public/collisionutils.cpp) # Added STATIC to ensure a library is created
target_include_directories(collisionutils PRIVATE
    ${HL2SDK_DIR}/src/public
    ${HL2SDK_DIR}/src/public/tier0
    ${HL2SDK_DIR}/src/public/tier1
)
target_compile_options(collisionutils PUBLIC ${my_compile_options})
target_compile_definitions(collisionutils PUBLIC ${shared_defs})

set(PLUGIN_NAME plugin_navmesh)
# IMPORTANT POTENTIAL ISSUE:
# If 'plugin_navmesh' is also defined as a target (e.g., using 'add_library' or 'add_executable')
# in your top-level CMakeLists.txt or another subdirectory, this line will cause a
# "target already exists" error. Typically, a main plugin target is defined once,
# usually at the top level, and subdirectories might add sources or define *their own*
# sub-libraries/executables that the main plugin links against.
add_library(${PLUGIN_NAME} MODULE VSPlugin.cpp)

# Link directories for Source SDK libraries.
# These paths need to be where the *compiled* SDK libraries reside.
# If your main CMakeLists.txt (or other subdirectories like 'util' or 'nav_mesh')
# are building these SDK common libs (tier0, tier1, etc.), then the link directories
# should point to *their* output locations (e.g., build_dir/Release or similar).
# Valve's SDK repo does not ship with pre-built binaries in 'lib/public' or 'lib/common'.
# The 'buildallprojects' script or building the VS solution populates these locations
# or generates the necessary project files to build them alongside your plugin.
# For now, keeping the original, but be aware this is a common point of failure.
target_link_directories(${PLUGIN_NAME} PUBLIC
    ${HL2SDK_PUB_LIBS} # This variable comes from main CMakeLists.txt, might need verification if it's correct
    ${HL2SDK_COM_LIBS} # Same as above
)

target_link_options(${PLUGIN_NAME} PRIVATE ${my_link_options})

# --- FIX: Changed to use the keyword signature (PRIVATE) ---
target_link_libraries(${PLUGIN_NAME}
    PRIVATE # <-- ADDED THIS KEYWORD HERE TO ENSURE CONSISTENCY
    # These dependencies should be defined and built by your other add_subdirectory calls
    # (e.g., plugin_util, valve_navmesh) or come from the SDK itself.
    valve_navmesh
    plugin_util
    
    # These are Source SDK internal libraries.
    # On Linux, they often resolve to :tier1.a etc. after buildallprojects.
    # On Windows, they are library files (tier1.lib, mathlib.lib) that need to be linked.
    # The 'tier1', 'mathlib', 'lzma' variables are defined in your main CMakeLists.txt.
    ${tier1}
    ${mathlib}
    ${lzma}
    vstdlib # Standard Valve library
    collisionutils # The library we defined above
    tier0 # Tier0 library
)

# --- REMOVED INSTALLATION LOGIC FOR CI/CD ---
# The following section, which attempts to find installed mods and install the plugin,
# is typically not suitable for a CI/CD build pipeline. In CI/CD, the goal is to
# produce build artifacts (the .so/.dll file) and upload them.
# The `upload-artifact` step in your GitHub Actions workflow already handles this.
# Leaving this commented out to prevent errors related to finding local steam installations
# on the GitHub Actions runners.
#
# #find installed mods on system.
# set( steam_dir steamapps/common )
# FILE(GLOB games ${steam_root}/${steam_dir}/*)
# foreach(GAME ${games})
#     foreach(mod ${mods})
#         if (EXISTS ${GAME}/${mod})
#             set( src_dir "${GAME}" )
#             list(APPEND mod_dirs "${GAME}/${mod}")
#         endif (EXISTS ${GAME}/${mod})
#     endforeach(mod ${mods})
# endforeach(GAME ${games})
# if (NOT DEFINED src_dir)
#     message (WARNING "No supported mods found.")
# endif (NOT DEFINED src_dir)
#
# foreach(mod_dir ${mod_dirs})
#     if (EXISTS ${mod_dir}/addons)
#         set( install_dir ${mod_dir}/addons )
#         install (TARGETS ${PLUGIN_NAME} DESTINATION ${install_dir})
#     endif (EXISTS ${mod_dir}/addons)
# endforeach(mod_dir ${mod_dirs})
