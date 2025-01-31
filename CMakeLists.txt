cmake_minimum_required(VERSION 3.8 FATAL_ERROR)
project(SeriousProton LANGUAGES C CXX)

set(CMAKE_MODULE_PATH
    ${CMAKE_CURRENT_SOURCE_DIR}/cmake/
    ${CMAKE_MODULE_PATH}
)
if(POLICY CMP0074)
    cmake_policy(SET CMP0074 NEW)
endif()
if(POLICY CMP0077)
    cmake_policy(SET CMP0077 NEW)
endif()

# User-settings
option(SERIOUSPROTON_WITH_JSON "Use json library." OFF)
option(WARNING_IS_ERROR "Enable warning as errors." OFF)

#
set(EXTERNALS_DIR "${PROJECT_BINARY_DIR}/externals")
set(DOWNLOADS_DIR "${PROJECT_BINARY_DIR}/downloads")
file(MAKE_DIRECTORY "${EXTERNAL_DIR}" "${DOWNLOADS_DIR}")

# Crash Logger for MinGW
if(WIN32)
    option(ENABLE_CRASH_LOGGER "Enable the Dr. MinGW crash logging facilities" OFF)
    set(DRMINGW_ROOT DRMINGW_ROOT-NOTFOUND CACHE PATH "Path to Dr. MinGW")

    if(NOT ENABLE_CRASH_LOGGER)
        message(STATUS "Crash Logger is OFF")
    else()
        message(STATUS "Crash Logger is ON")

        if(NOT DRMINGW_ROOT)
            message(VERBOSE "Downloading Dr. MinGW")

            if(${CMAKE_SIZEOF_VOID_P} EQUAL 4)
                set(DRMINGW_ARCH "32")
            else()
                set(DRMINGW_ARCH "64")
            endif()

            # 0.9.x seems to give a hard time to people on Win7.
            # Sticking with 0.8 for that reason.
            set(DRMINGW_VERSION "0.8.2")
            set(DRMINGW_BASENAME "drmingw-${DRMINGW_VERSION}-win${DRMINGW_ARCH}")
            set(DRMINGW_ROOT "${CMAKE_CURRENT_BINARY_DIR}/${DRMINGW_BASENAME}" CACHE PATH "Path to Dr. MinGW" FORCE)

            if(NOT EXISTS "${DRMINGW_ROOT}/bin/exchndl.dll")
                set(DRMINGW_ZIP "${CMAKE_CURRENT_BINARY_DIR}/${DRMINGW_BASENAME}.7z")

                file(DOWNLOAD "https://github.com/jrfonseca/drmingw/releases/download/${DRMINGW_VERSION}/${DRMINGW_BASENAME}.7z" "${DRMINGW_ZIP}" TIMEOUT 60 TLS_VERIFY ON)
                execute_process(COMMAND ${CMAKE_COMMAND} -E tar -xf "${DRMINGW_ZIP}" WORKING_DIRECTORY "${CMAKE_CURRENT_BINARY_DIR}")
            endif()
        endif()
    endif()
endif()

#--------------------------------Dependencies----------------------------------
find_package(SDL2 REQUIRED)
if(NOT DEFINED SDL2_LIBRARIES)
    set(SDL2_LIBRARIES SDL2::SDL2 SDL2::SDL2main)
endif()

# GLM - OpenGL Mathematic library
SET(WITH_GLM "auto" CACHE STRING "Which GLM to use (possible values are 'bundled', 'system' or 'auto')")

if(NOT WITH_GLM STREQUAL "bundled")
    set(glm_find_args QUIET)

    if(WITH_GLM STREQUAL "system")
       set(glm_find_args REQUIRED)
    endif()

    # Not bundled - attempt to find system package first.
    find_package(glm ${glm_find_args})
endif()

if(glm_FOUND)
    message(STATUS "GLM version used: SYSTEM")
    if(NOT TARGET glm::glm)
        # Compatibility with older versions
        add_library(glm::glm ALIAS glm)
    endif()
else()
    # We can only be coming here from:
    #    * 'auto', if we failed to find a suitable system package,
    #    * 'bundled'
    message(STATUS "GLM version used: BUNDLED")
    
    set(GLM_VERSION "0.9.9.8")
    set(GLM_URL "https://github.com/g-truc/glm")
    if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.11)
        include(FetchContent)
        FetchContent_Declare(
            glm
            GIT_REPOSITORY "${GLM_URL}"
            GIT_TAG "${GLM_VERSION}"
        )
        FetchContent_GetProperties(glm)
    else()
        set(GLM_BASE_PATH "${EXTERNALS_DIR}")
        set(glm_POPULATED FALSE)
        if(NOT EXISTS "${GLM_BASE_PATH}/glm/CMakeLists.txt")
            set(GLM_ZIP "${DOWNLOADS_DIR}/glm.zip")
            file(DOWNLOAD "${GLM_URL}/releases/download/${GLM_VERSION}/glm-${GLM_VERSION}.zip" "${GLM_ZIP}" TIMEOUT 60 TLS_VERIFY ON)

            file(MAKE_DIRECTORY "${GLM_BASE_PATH}/glm")
            execute_process(COMMAND ${CMAKE_COMMAND} -E tar -xf "${GLM_ZIP}" WORKING_DIRECTORY "${GLM_BASE_PATH}")
        endif()

        set(glm_SOURCE_DIR "${GLM_BASE_PATH}/glm")
        set(glm_BINARY_DIR "${PROJECT_BINARY_DIR}/externals/glm")
    endif()

    if(NOT glm_POPULATED)
        if(COMMAND FetchContent_Populate)
            FetchContent_Populate(glm)
        endif()
        add_subdirectory(${glm_SOURCE_DIR} ${glm_BINARY_DIR} EXCLUDE_FROM_ALL)
    endif()
endif()

add_subdirectory(libs/Box2D)
add_subdirectory(libs/glad)

# See cmake/GenerateGlDebug.cmake for details.
get_target_property(glad_source_dir glad SOURCE_DIR)
set(glDebug_inl "${CMAKE_CURRENT_BINARY_DIR}/include/graphics/glDebug.inl")
add_custom_command(
    OUTPUT "${glDebug_inl}"
    COMMAND ${CMAKE_COMMAND} -Dglad_h="${glad_source_dir}/glad/glad.h" -DglDebug_inl="${glDebug_inl}" -P cmake/GenerateGlDebug.cmake
    MAIN_DEPENDENCY "${glad_source_dir}/glad/glad.h"
    DEPENDS cmake/glDebug.inl.in cmake/GenerateGlDebug.cmake
    COMMENT "Generating glDebug.inl"
    WORKING_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}"
)

set_source_files_properties("${glDebug_inl}" PROPERTIES GENERATED TRUE)

add_subdirectory(libs/lua)

# json11
if(SERIOUSPROTON_WITH_JSON)
    SET(WITH_JSON "bundled" CACHE STRING "Which json11 to use (possible values are 'bundled' or 'system')")

    if(WITH_JSON STREQUAL "system")
        find_package(json11 REQUIRED)
        message(STATUS "json11 version used: SYSTEM")
    elseif(WITH_JSON STREQUAL "bundled")
        message(STATUS "json11 version used: BUNDLED")
        add_subdirectory(libs/json11)
    endif()
endif()

SET(WITH_OPUS "bundled" CACHE STRING "Which opus library to use (possible values are 'bundled' or 'system')")

if(WITH_OPUS STREQUAL "system")
    find_package(PkgConfig)
    pkg_check_modules(OPUS REQUIRED opus)
    message(STATUS "opus version used: SYSTEM")
    add_library(opus INTERFACE)
    target_link_libraries(opus ${OPUS_LIBRARIES})
    target_include_directories(opus PUBLIC ${OPUS_INCLUDE_DIRS})
    target_compile_options(opus PUBLIC ${OPUS_CFLAGS_OTHER})
elseif(WITH_OPUS STREQUAL "bundled")
    message(STATUS "opus version used: BUNDLED")
    add_subdirectory(libs/libopus)
endif()

# FreeType 2
set(WITH_FREETYPE2 "auto" CACHE STRING "Which FreeType2 to use (possible values are 'bundled', 'system' or 'auto')")

if(NOT WITH_FREETYPE2 STREQUAL "bundled")

    set(freetype2_find_args QUIET)

    if(WITH_FREETYPE2 STREQUAL "system")
       set(freetype2_find_args REQUIRED)
    endif()

    # Not bundled - attempt to find system package first.
    find_package(Freetype 2 ${freetype2_find_args})
endif()

if(Freetype_FOUND)
    message(STATUS "Freetype 2 version used: SYSTEM")
else()
    # We can only be coming here from:
    #    * 'auto', if we failed to find a suitable system package,
    #    * 'bundled'
    set(Freetype_DIR libs/cmake)
    find_package(Freetype 2 REQUIRED NO_DEFAULT_PATH)
    message(STATUS "Freetype 2 version used: BUNDLED")
endif()

#---------------------------------File lists-----------------------------------
set(source_files #All SeriousProton's objects to compile
    src/audio/source.cpp
    src/audio/sound.cpp
    src/audio/music.cpp
    src/clipboard.cpp
    src/collisionable.cpp
    src/engine.cpp
    src/event.cpp
    src/graphics/font.cpp
    src/graphics/freetypefont.cpp
    src/graphics/image.cpp
    src/graphics/renderTarget.cpp
    src/graphics/texture.cpp
    src/graphics/textureAtlas.cpp
    src/graphics/shader.cpp
    src/graphics/opengl.cpp
    src/i18n.cpp
    src/logging.cpp
    src/multiplayer.cpp
    src/multiplayer_client.cpp
    src/multiplayer_proxy.cpp
    src/multiplayer_server.cpp
    src/multiplayer_server_scanner.cpp
    src/networkAudioStream.cpp
    src/networkRecorder.cpp
    src/P.cpp
    src/postProcessManager.cpp
    src/random.cpp
    src/Renderable.cpp
    src/resources.cpp
    src/scriptInterface.cpp
    src/scriptInterfaceMagic.cpp
    src/shaderManager.cpp
    src/soundManager.cpp
    src/stringImproved.cpp
    src/stringutil/base64.cpp
    src/stringutil/sha1.cpp
    src/textureManager.cpp
    src/timer.cpp
    src/tween.cpp
    src/Updatable.cpp
    src/windowManager.cpp
    src/io/keybinding.cpp
    src/io/network/address.cpp
    src/io/network/selector.cpp
    src/io/network/socketBase.cpp
    src/io/network/tcpListener.cpp
    src/io/network/tcpSocket.cpp
    src/io/network/udpSocket.cpp
    src/io/http/request.cpp
    src/io/http/server.cpp
    src/io/http/websocket.cpp

    src/audio/source.h
    src/audio/sound.h
    src/audio/music.h
    src/clipboard.h
    src/collisionable.h
    src/dynamicLibrary.h
    src/engine.h
    src/event.h
    src/graphics/alignment.h
    src/graphics/font.h
    src/graphics/freetypefont.h
    src/graphics/image.h
    src/graphics/renderTarget.h
    src/graphics/texture.h
    src/graphics/textureAtlas.h
    src/graphics/shader.h
    src/graphics/opengl.h
    src/i18n.h
    src/io/keybinding.h
    src/io/dataBuffer.h
    src/io/pointer.h
    src/io/textinput.h
    src/io/http/request.h
    src/io/http/server.h
    src/io/http/websocket.h
    src/io/network/address.h
    src/io/network/selector.h
    src/io/network/socketBase.h
    src/io/network/tcpListener.h
    src/io/network/tcpSocket.h
    src/io/network/udpSocket.h
    src/logging.h
    src/multiplayer_client.h
    src/multiplayer.h
    src/multiplayer_internal.h
    src/multiplayer_proxy.h
    src/multiplayer_server.h
    src/multiplayer_server_scanner.h
    src/networkAudioStream.h
    src/networkRecorder.h
    src/nonCopyable.h
    src/P.h
    src/postProcessManager.h
    src/random.h
    src/rect.h
    src/Renderable.h
    src/resources.h
    src/scriptInterface.h
    src/scriptInterfaceMagic.h
    src/shaderManager.h
    src/soundManager.h
    src/stringImproved.h
    src/stringutil/base64.h
    src/stringutil/sha1.h
    src/textureManager.h
    src/tween.h
    src/timer.h
    src/Updatable.h
    src/vectorUtils.h
    src/windowManager.h

    cmake/glDebug.inl.in
    "${glDebug_inl}"
)

if(NOT ANDROID)
    list(APPEND source_files src/dynamicLibrary.cpp)
endif()

#----------------------------------Compiling-----------------------------------


# Set our optimization flags.
set(OPTIMIZER_FLAGS)
if(CMAKE_COMPILER_IS_GNUCC)
    # On gcc, we want some general optimalizations that improve speed a lot.
    set(OPTIMIZER_FLAGS ${OPTIMIZER_FLAGS} -O3 -flto -funsafe-math-optimizations)

    # If we are compiling for a rasberry pi, we want to aggressively optimize for the CPU we are running on.
    # Note that this check only works if we are compiling directly on the pi, as it is a dirty way of checkif if we are on the pi.
    if(EXISTS /opt/vc/include/bcm_host.h OR COMPILE_FOR_PI)
        set(OPTIMIZER_FLAGS ${OPTIMIZER_FLAGS} -mcpu=native -mfpu=neon-vfpv4 -mfloat-abi=hard)
    endif()
endif()

# Targets setup
# =============
#
# * seriousproton - this is the consumer library, the one you use in a target_link_libraries() call as a consumer.
#
# * seriousproton_objects: SP source files.
#   MinGW has... troubles when LTO is enabled and linking a static library: https://stackoverflow.com/q/27372667
#   To work around that, sources are exposed to the consumer (through the interface library) and built in 'their' space.
#
# * seriousproton_deps: SP dependencies / setup (include dirs, linked libraries etc)
#
# Both seriousproton_objects and seriousproton 'link' against seriousproton_deps:
# This allows compile flags/settings to be forwarded to consumer, as well as proper configuration to compile individual SP objects.

## Common settings / dependencies (c++ standards, headers, dependencies etc)
add_library(seriousproton_deps INTERFACE)

target_include_directories(seriousproton_deps
    INTERFACE
        "$<BUILD_INTERFACE:${CMAKE_CURRENT_SOURCE_DIR}/src;${CMAKE_CURRENT_BINARY_DIR}/include;${SDL2_INCLUDE_DIRS};${FREETYPE_INCLUDE_DIRS}>"
)
target_compile_features(seriousproton_deps INTERFACE cxx_std_17)

find_package(Threads REQUIRED)

target_compile_definitions(seriousproton_deps
    INTERFACE
        $<$<BOOL:${MSVC}>:NOMINMAX;_CRT_SECURE_NO_WARNINGS>
        $<$<CONFIG:Debug>:DEBUG>
        # Windows: Backwards compatibility with Win7 SP1: https://docs.microsoft.com/en-us/cpp/porting/modifying-winver-and-win32-winnt
        $<$<BOOL:${WIN32}>:WINVER=0x0601;_WIN32_WINNT=0x0601>
        $<$<AND:$<PLATFORM_ID:Android>,$<NOT:$<CONFIG:Release>>>:NDK_DEBUG=1>
)

target_compile_options(seriousproton_deps
    INTERFACE
        "$<$<AND:$<NOT:$<BOOL:${MSVC}>>,$<CONFIG:RelWithDebInfo>>:-g1>"
        "$<$<AND:$<BOOL:${CMAKE_COMPILER_IS_GNUCC}>,$<OR:$<CONFIG:RelWithDebInfo>,$<CONFIG:Release>>>:${OPTIMIZER_FLAGS}>"
        "$<$<NOT:$<BOOL:${MSVC}>>:-Wall;-Wextra;-Woverloaded-virtual;-Wdouble-promotion;-Wsuggest-override;-Werror=return-type;-Wno-unused-parameter;-Wno-unused-but-set-parameter>"
        "$<$<BOOL:${MSVC}>:/MP;/permissive->"
)

target_link_libraries(seriousproton_deps
    INTERFACE 
        box2d glad lua ${SDL2_LIBRARIES} glm::glm Threads::Threads ${FREETYPE_LIBRARIES}
        $<$<BOOL:${WIN32}>:wsock32 ws2_32 crypt32>
        # LTO flag must be on the linker's list as well.
        "$<$<AND:$<BOOL:${CMAKE_COMPILER_IS_GNUCC}>,$<OR:$<CONFIG:RelWithDebInfo>,$<CONFIG:Release>>>:-flto>"
        "$<BUILD_INTERFACE:opus;$<$<BOOL:${SERIOUSPROTON_WITH_JSON}>:json11>;$<$<NOT:$<BOOL:${ANDROID}>>:${CMAKE_DL_LIBS}>>"
)

if(NOT ANDROID)
    # Necessary for some older compilers (except on android, where the fs api isn't used)
    if(CMAKE_COMPILER_IS_GNUCC AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9.1)
        target_link_libraries(seriousproton_deps INTERFACE $<BUILD_INTERFACE:stdc++fs>)
    endif()
    if(CMAKE_CXX_COMPILER_ID STREQUAL "Clang" AND CMAKE_CXX_COMPILER_VERSION VERSION_LESS 9.0)
        target_link_libraries(seriousproton_deps INTERFACE $<BUILD_INTERFACE:c++fs>)
    endif()
endif()

## Object list
add_library(seriousproton_objects OBJECT ${source_files})
target_compile_options(seriousproton_objects
    PRIVATE
        "$<$<AND:$<BOOL:${CMAKE_COMPILER_IS_GNUCC}>,$<BOOL:${WARNING_IS_ERROR}>>:-Werror>"
        "$<$<AND:$<BOOL:${MSVC}>,$<BOOL:${WARNING_IS_ERROR}>>:/WX>"
)

if(CMAKE_VERSION VERSION_GREATER_EQUAL 3.12)
    # target_link_libraries() support for object libraries only exists since 3.12.
    target_link_libraries(seriousproton_objects PUBLIC $<BUILD_INTERFACE:seriousproton_deps>)
else()
    # This is mainly for compatibility with Ubuntu 18.04, which still uses CMake 3.10.
    set_target_properties(seriousproton_objects PROPERTIES LINK_LIBRARIES $<BUILD_INTERFACE:seriousproton_deps>)
endif()

## Public libraries that 'consumers' link against.
add_library(seriousproton INTERFACE)

# Expose sources to consumer (necessary for LTO on some compilers, mingw)
set_target_properties(seriousproton PROPERTIES INTERFACE_SOURCES $<TARGET_OBJECTS:seriousproton_objects>)

# Forward SP settings to consumer.
target_link_libraries(seriousproton INTERFACE $<BUILD_INTERFACE:seriousproton_deps>)

#--------------------------------Installation----------------------------------
install(
    TARGETS seriousproton
    EXPORT seriousproton
    LIBRARY DESTINATION lib
    ARCHIVE DESTINATION lib
)
install(
    DIRECTORY ${seriousproton_include_dir}
    DESTINATION include/seriousproton
    FILES_MATCHING PATTERN "*.h"
    
)
install(
    DIRECTORY ${seriousproton_include_dir}
    DESTINATION include/seriousproton
    FILES_MATCHING PATTERN "*.hpp"
)
install(
    EXPORT seriousproton
    DESTINATION share/seriousproton
)

if(WIN32)
    if(MINGW)
        # For the MinGW library, the 'imported location' looks at the dll in the wrong folder (lib instead of bin)
        # So we have to get it by hand.
        # Also covers backwards compatibility with SDL version < 2.0.12.
        install(FILES "${SDL2_PREFIX}/bin/SDL2.dll" DESTINATION . COMPONENT runtime)
    else()
        install(PROGRAMS "$<TARGET_PROPERTY:SDL2::SDL2,IMPORTED_LOCATION>" DESTINATION . COMPONENT runtime)
    endif()

    if (MINGW)
        macro(get_mingw_dll dll location)
            execute_process(COMMAND ${CMAKE_CXX_COMPILER} -print-file-name=${dll} OUTPUT_VARIABLE ${location} OUTPUT_STRIP_TRAILING_WHITESPACE)
        endmacro()
        
        # In the wild, MinGW come in different flavor
        # (Nuwen is fully statically linked, exception handlers varies sjlj / seh / dw2, threading model can be posix or windows, ...)
        # Grab anything possible, the correct ones will stick for the given installation.
        get_mingw_dll(libstdc++-6.dll MINGW_STDCPP_DLL)
        get_mingw_dll(libgcc_s_sjlj-1.dll MINGW_LIBGCCSJLJ_DLL)
        get_mingw_dll(libgcc_s_seh-1.dll MINGW_LIBGCCSEH_DLL)
        get_mingw_dll(libgcc_s_dw2-1.dll MINGW_LIBGCCDW2_DLL)
        get_mingw_dll(libwinpthread-1.dll MINGW_PTHREAD_DLL)

        install(
            FILES
                ${MINGW_STDCPP_DLL}
                ${MINGW_LIBGCCSJLJ_DLL}
                ${MINGW_LIBGCCSEH_DLL}
                ${MINGW_LIBGCCDW2_DLL}
                ${MINGW_PTHREAD_DLL}
            DESTINATION .
            COMPONENT runtime
        OPTIONAL)
    endif()

    if(ENABLE_CRASH_LOGGER)
        install(
            FILES
            ${DRMINGW_ROOT}/bin/dbghelp.dll
            ${DRMINGW_ROOT}/bin/exchndl.dll
            ${DRMINGW_ROOT}/bin/mgwhelp.dll
            ${DRMINGW_ROOT}/bin/symsrv.dll
            ${DRMINGW_ROOT}/bin/symsrv.yes
            DESTINATION .
            COMPONENT runtime
        )
    endif()
endif()
