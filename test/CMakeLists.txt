add_library(collisionutils ${HL2SDK_DIR}/public/collisionutils.cpp)
target_include_directories(collisionutils PRIVATE ${HL2SDK_DIR}/public ${HL2SDK_DIR}/public/tier0 ${HL2SDK_DIR}/public/tier1)
target_compile_options(collisionutils PUBLIC ${my_compile_options})
target_compile_definitions(collisionutils PUBLIC ${shared_defs})

set(PLUGIN_NAME plugin_navmesh)
add_library(${PLUGIN_NAME} MODULE VSPlugin.cpp)

# --- REMOVED: This line is redundant. link_directories in the parent handles this. ---
# target_link_directories(${PLUGIN_NAME} PUBLIC ${HL2SDK_PUB_LIBS} ${HL2SDK_COM_LIBS})

target_link_options(${PLUGIN_NAME} PRIVATE ${my_link_options})

target_link_libraries(${PLUGIN_NAME}  
    valve_navmesh          # Links to your internal nav_mesh library
    plugin_util            # Links to your internal util library
    collisionutils         # Links to the collisionutils library built here
    tier0                  # Links to libtier0.a (should be found via link_directories)
    vstdlib                # Links to libvstdlib.a (should be found via link_directories)
    tier1                  # Links to libtier1.a (should be found via link_directories)
    mathlib                # Links to libmathlib.a (should be found via link_directories)
    lzma                   # Links to liblzma.a (should be found via link_directories)
)
 
#find installed mods on system.
set( steam_dir steamapps/common )
FILE(GLOB games ${steam_root}/${steam_dir}/*)
foreach(GAME ${games})
    foreach(mod ${mods})
        if (EXISTS ${GAME}/${mod})
            set( src_dir "${GAME}" )
            list(APPEND mod_dirs "${GAME}/${mod}")
        endif (EXISTS ${GAME}/${mod})
    endforeach(mod ${mods})
endforeach(GAME ${games})
if (NOT DEFINED src_dir)
    message (WARNING "No supported mods found.")
endif (NOT DEFINED src_dir)

foreach(mod_dir ${mod_dirs})
    if (EXISTS ${mod_dir}/addons)
        set( install_dir ${mod_dir}/addons )
        install (TARGETS ${PLUGIN_NAME} DESTINATION ${install_dir})
    endif (EXISTS ${mod_dir}/addons)
endforeach(mod_dir ${mod_dirs})
