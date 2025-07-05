add_library(plugin_util
    SimpleException.cpp
    BaseEntity.cpp
    UtilTrace.cpp
    EntityUtils.cpp
    Handle.cpp
    common_util.cpp
)

target_compile_definitions(plugin_util PUBLIC ${shared_defs})

# Ensure PIC is on for shared libraries, especially on Linux
set_property(TARGET plugin_util PROPERTY POSITION_INDEPENDENT_CODE ON)

# This line is correctly applying the shared_includes.
# Now that 'shared_includes' in the top-level CMakeLists.txt
# contains the SDK header paths, this should resolve the include errors
# for source files compiled as part of 'plugin_util'.
target_include_directories(plugin_util PUBLIC ${shared_includes})

target_compile_options(plugin_util PUBLIC ${my_compile_options})

# Note: If 'plugin_util' itself needs to link against any Source SDK libraries
# directly (e.g., tier0.lib, vstdlib.lib), you would add a target_link_libraries
# command here for 'plugin_util'. Based on your original top-level,
# the main 'plugin_navmesh' links against them, which implies 'plugin_util'
# might not need direct linking here if it's only a static library that
# 'plugin_navmesh' ultimately consumes.
