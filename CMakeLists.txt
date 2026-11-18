cmake_minimum_required(VERSION 3.21)

if(NOT CMAKE_BUILD_TYPE AND NOT CMAKE_CONFIGURATION_TYPES)
    set(CMAKE_BUILD_TYPE "Debug" CACHE STRING "Choose build type") # or "Release"
endif()

project(cmakepresets-webp-sdl3-image-opengl-cpp LANGUAGES CXX C)

set(CMAKE_CXX_STANDARD 20)
set(CMAKE_CXX_STANDARD_REQUIRED ON)

# =============================================================================
# Android: Register ABI-specific library roots for Gradle's CMake invocation
# =============================================================================
if(ANDROID)
    list(APPEND CMAKE_FIND_ROOT_PATH
        "C:/libs/cglm-0.9.6"
        "C:/libs/sdl3-3.4.20-android/${ANDROID_ABI}"
        "C:/libs/sdl3_image-3.4.8-webp-android/${ANDROID_ABI}"
    )
endif()

# =============================================================================
# Sources & Target
# =============================================================================
set(PROJECT_SOURCES
    src/main.cpp
)

if(ANDROID)
    set(APP_TARGET main)
    add_library(${APP_TARGET} SHARED ${PROJECT_SOURCES})
else()
    set(APP_TARGET app)
    add_executable(${APP_TARGET} ${PROJECT_SOURCES})
endif()

target_include_directories(${APP_TARGET} PRIVATE
    ${CMAKE_CURRENT_SOURCE_DIR}/src
)

# =============================================================================
# Imported Libraries
# =============================================================================

# --- cglm (header-only on all platforms) -------------------------------------
find_path(CGLM_INCLUDE_DIR NAMES cglm/cglm.h REQUIRED)
add_library(cglm INTERFACE)
set_target_properties(cglm PROPERTIES
    INTERFACE_INCLUDE_DIRECTORIES "${CGLM_INCLUDE_DIR}"
)

# --- SDL3 & SDL3_image -------------------------------------------------------
find_path(SDL3_INCLUDE_DIR       NAMES SDL3/SDL.h REQUIRED)
find_path(SDL3_IMAGE_INCLUDE_DIR NAMES SDL3_image/SDL_image.h REQUIRED)

if(ANDROID)
    find_library(SDL3_LIBRARY       NAMES libSDL3.so SDL3 REQUIRED)
    find_library(SDL3_IMAGE_LIBRARY NAMES libSDL3_image.so SDL3_image REQUIRED)

    add_library(SDL3 SHARED IMPORTED)
    set_target_properties(SDL3 PROPERTIES
        IMPORTED_LOCATION "${SDL3_LIBRARY}"
        INTERFACE_INCLUDE_DIRECTORIES "${SDL3_INCLUDE_DIR}"
    )

    add_library(SDL3_image SHARED IMPORTED)
    set_target_properties(SDL3_image PROPERTIES
        IMPORTED_LOCATION "${SDL3_IMAGE_LIBRARY}"
        INTERFACE_INCLUDE_DIRECTORIES "${SDL3_IMAGE_INCLUDE_DIR}"
        INTERFACE_LINK_LIBRARIES "SDL3"
    )
else()
    find_library(SDL3_LIBRARY       NAMES libSDL3.a SDL3 REQUIRED)
    find_library(SDL3_IMAGE_LIBRARY NAMES libSDL3_image.a SDL3_image REQUIRED)

    # --- WebP static bundle (used by static SDL3_image on Web & Desktop) -----
    find_path(WEBP_INCLUDE_DIR NAMES webp/decode.h REQUIRED)
    find_library(WEBP_DEMUX_LIB NAMES libwebpdemux.a webpdemux REQUIRED)
    find_library(WEBP_MUX_LIB   NAMES libwebpmux.a   webpmux   REQUIRED)
    find_library(WEBP_CORE_LIB  NAMES libwebp.a      webp      REQUIRED)
    find_library(WEBP_SHARP_LIB NAMES libsharpyuv.a  sharpyuv  REQUIRED)

    add_library(webp_demux    STATIC IMPORTED)
    add_library(webp_mux      STATIC IMPORTED)
    add_library(webp_core     STATIC IMPORTED)
    add_library(webp_sharpyuv STATIC IMPORTED)
    set_target_properties(webp_demux    PROPERTIES IMPORTED_LOCATION "${WEBP_DEMUX_LIB}")
    set_target_properties(webp_mux      PROPERTIES IMPORTED_LOCATION "${WEBP_MUX_LIB}")
    set_target_properties(webp_core     PROPERTIES IMPORTED_LOCATION "${WEBP_CORE_LIB}")
    set_target_properties(webp_sharpyuv PROPERTIES IMPORTED_LOCATION "${WEBP_SHARP_LIB}")

    add_library(webp INTERFACE)
    set_target_properties(webp PROPERTIES
        INTERFACE_INCLUDE_DIRECTORIES "${WEBP_INCLUDE_DIR}"
        INTERFACE_LINK_LIBRARIES "webp_demux;webp_mux;webp_core;webp_sharpyuv"
    )

    # --- Platform system dependencies for static SDL3 ------------------------
    if(EMSCRIPTEN)
        set(SDL3_SYS_LIBS "")
    elseif(WIN32)
        set(SDL3_SYS_LIBS "winmm;imm32;version;setupapi;dinput8;ole32;oleaut32;uuid")
    elseif(UNIX AND NOT APPLE)
        set(SDL3_SYS_LIBS "dl;pthread;m")
    endif()

    add_library(SDL3_Static STATIC IMPORTED)
    set_target_properties(SDL3_Static PROPERTIES
        IMPORTED_LOCATION "${SDL3_LIBRARY}"
        INTERFACE_INCLUDE_DIRECTORIES "${SDL3_INCLUDE_DIR}"
        INTERFACE_LINK_LIBRARIES "${SDL3_SYS_LIBS}"
    )

    add_library(SDL3_image_Static STATIC IMPORTED)
    set_target_properties(SDL3_image_Static PROPERTIES
        IMPORTED_LOCATION "${SDL3_IMAGE_LIBRARY}"
        INTERFACE_INCLUDE_DIRECTORIES "${SDL3_IMAGE_INCLUDE_DIR}"
        INTERFACE_LINK_LIBRARIES "SDL3_Static;webp"
    )
endif()

# --- GLAD + Desktop OpenGL (Windows & Linux only) ----------------------------
if(NOT ANDROID AND NOT EMSCRIPTEN)
    find_path(GLAD_ROOT_DIR NAMES include/glad/glad.h PATHS ${CMAKE_PREFIX_PATH} NO_DEFAULT_PATH REQUIRED)

    add_library(glad STATIC "${GLAD_ROOT_DIR}/src/glad.c")
    target_include_directories(glad PUBLIC "${GLAD_ROOT_DIR}/include")
    if(WIN32)
        target_link_libraries(glad PUBLIC opengl32)
    else()
        target_link_libraries(glad PUBLIC GL)
    endif()
endif()

# =============================================================================
# Linking & Platform Options
# =============================================================================
if(ANDROID)
    find_library(GLES3_LIB   GLESv3  REQUIRED)
    find_library(EGL_LIB     EGL     REQUIRED)
    find_library(ANDROID_LIB android REQUIRED)
    find_library(LOG_LIB     log     REQUIRED)

    target_link_libraries(${APP_TARGET} PRIVATE
        SDL3_image
        cglm
        ${GLES3_LIB}
        ${EGL_LIB}
        ${ANDROID_LIB}
        ${LOG_LIB}
    )

elseif(EMSCRIPTEN)
    set_target_properties(${APP_TARGET} PROPERTIES
        OUTPUT_NAME "app"
        SUFFIX ".js"
    )

    target_link_libraries(${APP_TARGET} PRIVATE
        SDL3_image_Static
        cglm
    )

    target_link_options(${APP_TARGET} PRIVATE
        "-sUSE_WEBGL2=1"
        "-sFULL_ES3=1"
        "-sALLOW_MEMORY_GROWTH=1"
        "--preload-file=${CMAKE_CURRENT_SOURCE_DIR}/assets@assets"
    )

else()
    target_link_libraries(${APP_TARGET} PRIVATE
        SDL3_image_Static
        cglm
        glad
    )

    if(EXISTS "${CMAKE_CURRENT_SOURCE_DIR}/assets")
        add_custom_command(TARGET ${APP_TARGET} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_directory
                "${CMAKE_CURRENT_SOURCE_DIR}/assets"
                "$<TARGET_FILE_DIR:${APP_TARGET}>/assets"
        )
    endif()

    target_link_options(${APP_TARGET} PRIVATE -static-libgcc -static-libstdc++)

    if(CMAKE_BUILD_TYPE STREQUAL "Release" OR CMAKE_BUILD_TYPE STREQUAL "MinSizeRel")
        target_compile_options(${APP_TARGET} PRIVATE -Os)
        target_link_options(${APP_TARGET} PRIVATE -s)
    endif()

    if(WIN32)
        target_link_options(${APP_TARGET} PRIVATE -static)
        if(CMAKE_BUILD_TYPE STREQUAL "Debug")
            target_link_options(${APP_TARGET} PRIVATE -mconsole)
        else()
            target_link_options(${APP_TARGET} PRIVATE -mwindows)
        endif()
    endif()
endif()