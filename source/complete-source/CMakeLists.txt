cmake_minimum_required(VERSION 3.22)
message("Using toolchain file ${CMAKE_TOOLCHAIN_FILE}.")

project(
	DtryKeyUtil
	VERSION 0.0.2
	DESCRIPTION "A munted update to CommonLibSSE-NG"
	LANGUAGES CXX
)
set(CMAKE_CXX_STANDARD 23)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_INTERPROCEDURAL_OPTIMIZATION ON)

set(CMAKE_ARCHIVE_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/lib)
set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)

######
# File definitions (Could be nicer but I just want it all in one file right now)
######
configure_file(
	${CMAKE_CURRENT_SOURCE_DIR}/cmake/version.rc.in
	${CMAKE_CURRENT_BINARY_DIR}/version.rc
	@ONLY
)

set(headers
	#src/PCH.h
)

set(sources
	src/main.cpp
	src/PCH.h
	src/hasher.hpp
	src/hooks.cpp
	src/hooks.h
	src/inputEventHandler.cpp
	src/inputEventHandler.h
	src/settings.cpp
	src/settings.h
	src/utils.h
	src/lib/SimpleIni.h
	src/inputHandler/inputTracer.h
	src/inputHandler/inputTracer.cpp
	src/inputHandler/inputTrigger.h
	src/inputHandler/inputTrigger.cpp
	src/inputHandler/movementInputTracer.h
	src/inputHandler/movementInputTracer.cpp
	src/inputHandler/animEvent.cpp
	src/inputHandler/animEvent.h

	${CMAKE_CURRENT_BINARY_DIR}/version.rc
)

source_group(
        TREE ${CMAKE_CURRENT_SOURCE_DIR}
        FILES
        ${headers}
        ${sources})

######
# Library inclusions and target config
######
# CommonLibSSE-NG 6.7.0, built from source.
# The mod used the "commonlibsse-ng-flatrim" vcpkg port from the colorglass registry,
# which is far too old for Skyrim 1.7.99 and its Address Library format 5.
# "flatrim" meant SE+AE with no VR, so VR stays off here to match.
set(ENABLE_SKYRIM_VR OFF)
set(BUILD_TESTS OFF)
set(SKSE_SUPPORT_XBYAK ON)   # must be set BEFORE add_subdirectory to take effect
add_subdirectory(external/CommonLibNG CommonLibNG EXCLUDE_FROM_ALL)
# add_commonlibsse_plugin() is only auto-included on CommonLib's prebuilt path.
include("${CMAKE_CURRENT_SOURCE_DIR}/external/CommonLibNG/cmake/CommonLibSSE.cmake")

find_package(xbyak REQUIRED CONFIG)

add_commonlibsse_plugin(${PROJECT_NAME} SOURCES ${headers} ${sources})
add_library("${PROJECT_NAME}::${PROJECT_NAME}" ALIAS "${PROJECT_NAME}")

target_include_directories(${PROJECT_NAME}
    PRIVATE
        $<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src>
        $<BUILD_INTERFACE:${CMAKE_CURRENT_BINARY_DIR}/src>
        $<INSTALL_INTERFACE:src>
)

target_link_libraries(
	${PROJECT_NAME}
	PRIVATE
		xbyak::xbyak
)

target_precompile_headers(
	${PROJECT_NAME}
    PRIVATE
        src/PCH.h
)

# add_commonlibsse_plugin() generates a SKSEPluginInfo() declaration whose
# StructCompatibility field lands on versionIndependenceEx, so it can only write 0 or 1
# and can never set kVersionIndependentEx_AddressLibraryV5 (bit 1). Drop it and declare
# the plugin by hand in main.cpp instead, so the DLL advertises Address Library format 5.
# PluginDeclaration::GetSingleton() still works: it just reads the SKSEPlugin_Version
# export, and PluginVersionData shares its layout for the version and name fields.
get_target_property(_dtry_sources ${PROJECT_NAME} SOURCES)
list(REMOVE_ITEM _dtry_sources "${CMAKE_CURRENT_BINARY_DIR}/__${PROJECT_NAME}Plugin.cpp")
set_target_properties(${PROJECT_NAME} PROPERTIES SOURCES "${_dtry_sources}")

target_compile_definitions(${PROJECT_NAME} PRIVATE NOMINMAX WIN32_LEAN_AND_MEAN)


install(TARGETS ${PROJECT_NAME}
        DESTINATION "${CMAKE_INSTALL_LIBDIR}")


