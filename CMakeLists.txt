cmake_minimum_required (VERSION 3.28.3)

project (plugin_navmesh)

set(DRIVE c CACHE STRING "User specified drive for Windows")
set( steam_root "${DRIVE}:/Program Files (x86)/steam" )

# HL2SDK_DIR is correctly set here relative to the project source directory
# (assuming hl2sdk is a sibling directory in the GitHub Actions workspace)
set(HL2SDK_DIR "${PROJECT_SOURCE_DIR}/../hl2sdk")

# These will be dynamically set inside the LINUX block
set (HL2SDK_PUB_LIBS) 
set (HL2SDK_COM_LIBS)

set (mods "tf;dod;hl2mp")
set(shared_defs  PLATFORM_64BITS X64BITS RAD_TELEMETRY_DISABLED $<$<CONFIG:Debug>:_DEBUG>)

set (tier1 tier1)
set (mathlib mathlib)
set (lzma lzma)

if (CMAKE_CXX_COMPILER_ID STREQUAL "Clang")
	list(APPEND shared_defs __clang__)
elseif (CMAKE_CXX_COMPILER_ID STREQUAL "GNU")
	list(APPEND shared_defs GNUC)
endif()

if (LINUX)
    # --- NEW LOGIC FOR LINUX ARCHITECTURE SELECTION ---
    # Determine the target architecture directory for HL2SDK libraries
    # CMAKE_SIZEOF_VOID_P is 4 for 32-bit and 8 for 64-bit compilation
    if (CMAKE_SIZEOF_VOID_P EQUAL 4)
        set(HL2SDK_ARCH_DIR "linux32")
        message(STATUS "Building for 32-bit Linux - Using HL2SDK libs from: ${HL2SDK_ARCH_DIR}")
    elseif (CMAKE_SIZEOF_VOID_P EQUAL 8)
        set(HL2SDK_ARCH_DIR "linux64")
        message(STATUS "Building for 64-bit Linux - Using HL2SDK libs from: ${HL2SDK_ARCH_DIR}")
    else()
        message(FATAL_ERROR "Unsupported target architecture. CMAKE_SIZEOF_VOID_P is ${CMAKE_SIZEOF_VOID_P}")
    endif()

    # Set library names with specific linker syntax for Linux static archives
	set (tier1 :tier1.a)
	set (mathlib :mathlib.a)
	set (lzma :lzma.a)
	
    # Use the dynamically determined architecture directory for HL2SDK library paths
	set (HL2SDK_COM_LIBS ${HL2SDK_DIR}/lib/common/${HL2SDK_ARCH_DIR})
	set (HL2SDK_PUB_LIBS ${HL2SDK_DIR}/lib/public/${HL2SDK_ARCH_DIR})

	set (SRC_HOME $ENV{HOME})
	set( steam_root "$ENV{HOME}/.steam/steam" )
	
    # Removed -m64 here; the -m32 flag is passed by the build system from action.yml
	set(my_compile_options $<$<CONFIG:Debug>:-Wall> -fPIC) 
	list(APPEND shared_defs _LINUX LINUX POSIX)
	set(my_link_options -static-libgcc -static-libstdc++)
else()
    # Windows paths (remain x64 as Windows builds are 64-bit)
	set (HL2SDK_PUB_LIBS ${HL2SDK_PUB_LIBS}/x64) # This will be ${HL2SDK_DIR}/lib/public/x64
	set (HL2SDK_COM_LIBS ${HL2SDK_COM_LIBS}/x64/2015/release) # This will be ${HL2SDK_DIR}/lib/common/x64/2015/release
	set(my_compile_options
		$<$<CONFIG:>:/MT> #---------|
		$<$<CONFIG:Debug>:/MTd> #---|-- Statically link the runtime libraries
		$<$<CONFIG:Release>:/MT> #--|
	)
	list(APPEND shared_defs COMPILER_MSVC COMPILER_MSVC64 _CRT_NO_VA_START_VALIDATION _CRT_SECURE_NO_DEPRECATE 
		_CRT_NONSTDC_NO_DEPRECATE)
	set(my_link_options /NODEFAULTLIB:libcmtd /SAFESEH:NO)
endif ()

# Add common include directories for the entire project
set(shared_includes 
    ${HL2SDK_DIR}/public 
    ${HL2SDK_DIR}/public/tier0 
	${HL2SDK_DIR}/public/engine 
    ${HL2SDK_DIR}/public/tier1 
    ${HL2SDK_DIR}/public/mathlib
	${HL2SDK_DIR}/public/game/server  
	${HL2SDK_DIR}/game/shared 
    ${HL2SDK_DIR}/game/server)

# Add the determined HL2SDK library directories to the linker search path
# This allows target_link_libraries to find them by just their names (e.g., tier1) or by their archive name (:tier1.a)
link_directories(${HL2SDK_PUB_LIBS})
link_directories(${HL2SDK_COM_LIBS})

# Add subdirectories for building their respective components
add_subdirectory(${PROJECT_SOURCE_DIR}/util)
add_subdirectory(${PROJECT_SOURCE_DIR}/nav_mesh)
add_subdirectory(${PROJECT_SOURCE_DIR}/test) # This will then use the updated HL2SDK_PUB_LIBS and HL2SDK_COM_LIBS for linking
