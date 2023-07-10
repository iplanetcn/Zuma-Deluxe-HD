cmake_minimum_required(VERSION 3.25)
project(
        ZumaHD
        LANGUAGES C
        VERSION 0.1.0
)

set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
list(APPEND CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake/sdl2)

# Download bass & bass-fx libraries for windows
if (WIN32)
    find_library(Bass bass NO_CACHE PATHS ${PROJECT_SOURCE_DIR}/lib/bass/c/x64)
    if (${Bass} STREQUAL Bass-NOTFOUND)
        message(STATUS "Downloading bass...")
        file(DOWNLOAD "https://www.un4seen.com/files/bass24.zip" "lib/bass-win.zip" SHOW_PROGRESS)
        file(ARCHIVE_EXTRACT INPUT "lib/bass-win.zip" DESTINATION lib/bass)
    endif ()


    find_library(BassFx bass_fx NO_CACHE PATHS ${PROJECT_SOURCE_DIR}/lib/bass-fx/C/x64)
    if (${BassFx} STREQUAL BassFx-NOTFOUND)
        message(STATUS "Downloading bass fx...")
        file(DOWNLOAD "https://www.un4seen.com/files/z/0/bass_fx24.zip" "lib/bass-fx-win.zip" SHOW_PROGRESS)
        file(ARCHIVE_EXTRACT INPUT "lib/bass-fx-win.zip" DESTINATION lib/bass-fx)
    endif ()
endif ()

# Download bass & bass-fx libraries for macOSX
if (APPLE)
    find_library(isBassFound bass NO_CACHE PATHS ${PROJECT_SOURCE_DIR}/lib/bass)
    if (${isBassFound} STREQUAL isBassFound-NOTFOUND)
        message(STATUS "Downloading bass...")
        file(DOWNLOAD "https://www.un4seen.com/files/bass24-osx.zip" "lib/bass-osx.zip" SHOW_PROGRESS)
        file(ARCHIVE_EXTRACT INPUT "lib/bass-osx.zip" DESTINATION lib/bass)
        file(ARCHIVE_EXTRACT INPUT "lib/bass-osx.zip" DESTINATION ${PROJECT_SOURCE_DIR}/lib/bass)
    endif ()

    find_library(isBassFxFound bass_fx NO_CACHE PATHS ${PROJECT_SOURCE_DIR}/lib/bass-fx)
    if (${isBassFxFound} STREQUAL isBassFxFound-NOTFOUND)
        message(STATUS "Downloading bass fx...")
        file(DOWNLOAD "https://www.un4seen.com/files/z/0/bass_fx24-osx.zip" "lib/bass-fx-osx.zip" SHOW_PROGRESS)
        file(ARCHIVE_EXTRACT INPUT "lib/bass-fx-osx.zip" DESTINATION lib/bass-fx)
        file(ARCHIVE_EXTRACT INPUT "lib/bass-fx-osx.zip" DESTINATION ${PROJECT_SOURCE_DIR}/lib/bass-fx)
    endif ()
endif ()

include(FetchContent)

# Fetching expat
message("Fetching expat")
FetchContent_Declare(
        expat
        GIT_REPOSITORY https://github.com/libexpat/libexpat.git
        GIT_TAG d8f43d6757af5ca5aae6cc185d6481f37f6c3be2
)

FetchContent_MakeAvailable(expat)

set(EXPAT_BUILD_DOCS OFF)
set(EXPAT_BUILD_TOOLS OFF)
add_subdirectory(${expat_SOURCE_DIR}/expat ${expat_BINARY_DIR})

# Fetching SDL2 libraries
if (WIN32)
    message("Fetching sdl2")
    FetchContent_Declare(
            sdl2
            GIT_REPOSITORY https://github.com/libsdl-org/SDL.git
            GIT_TAG SDL2
    )

    FetchContent_MakeAvailable(sdl2)

    message("Fetching sdl2 image")
    FetchContent_Declare(
            sdl2_image
            GIT_REPOSITORY https://github.com/libsdl-org/SDL_image.git
            GIT_TAG SDL2
    )

    set(SDL2IMAGE_SAMPLES OFF)
    FetchContent_MakeAvailable(sdl2_image)

    message("Fetching sdl2 ttf")
    FetchContent_Declare(
            sdl2_ttf
            GIT_REPOSITORY https://github.com/libsdl-org/SDL_ttf.git
            GIT_TAG SDL2
    )
    set(SDL2TTF_SAMPLES OFF)
    FetchContent_MakeAvailable(sdl2_ttf)
endif ()

if (APPLE)
    find_package(SDL2 REQUIRED)
    find_package(SDL2_image REQUIRED)
    find_package(SDL2_ttf REQUIRED)
endif ()

FILE(GLOB_RECURSE SOURCES RELATIVE ${PROJECT_SOURCE_DIR} "src/*.c")

add_executable(
        ${PROJECT_NAME}
        ${SOURCES}
)

target_include_directories(
        ${PROJECT_NAME} PUBLIC
        "${PROJECT_BINARY_DIR}"
        "${PROJECT_SOURCE_DIR}/src"
        "${PROJECT_SOURCE_DIR}/lib/bass/"
        "${PROJECT_SOURCE_DIR}/lib/bass-fx/"

        "${expat_SOURCE_DIR}/expat/lib"
        "${sdl2_SOURCE_DIR}/include"
        "${sdl2_image_SOURCE_DIR}"
        "${sdl2_ttf_SOURCE_DIR}"
)

link_directories(
        ${PROJECT_SOURCE_DIR}/lib/bass/
        ${PROJECT_SOURCE_DIR}/lib/bass-fx/
)

if (Win32)

    target_link_libraries(
            ${PROJECT_NAME} PRIVATE
            SDL2
            SDL2main
            SDL2_image
            SDL2_ttf

            ${PROJECT_SOURCE_DIR}/lib/bass/libbass.dylib
            ${PROJECT_SOURCE_DIR}/lib/bass-fx/libbass_fx.dylib

            expat
    )
endif ()

if (APPLE)
    target_link_libraries(
            ${PROJECT_NAME} PRIVATE
            SDL2::Core
            SDL2::Main
            SDL2::Image
            SDL2::TTF

            ${PROJECT_SOURCE_DIR}/lib/bass/libbass.dylib
            ${PROJECT_SOURCE_DIR}/lib/bass-fx/libbass_fx.dylib

            expat
    )
endif ()


message(STATUS ${Bass})
message(STATUS ${BassFx})

# Copy DLLs needed for runtime on Windows
if (WIN32)
    if (BUILD_SHARED_LIBS)
        add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
                COMMAND ${CMAKE_COMMAND} -E copy_if_different
                ${PROJECT_SOURCE_DIR}/lib/bass/x64/bass.dll
                ${PROJECT_SOURCE_DIR}/lib/bass-fx/x64/bass_fx.dll
                $<TARGET_FILE_DIR:ZumaHD>)
    endif ()
elseif (APPLE)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy_if_different
            ${PROJECT_SOURCE_DIR}/lib/bass/libbass.dylib
            ${PROJECT_SOURCE_DIR}/lib/bass-fx/libbass_fx.dylib
            $<TARGET_FILE_DIR:ZumaHD>)
endif ()

add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory
        ${CMAKE_CURRENT_SOURCE_DIR}/assets
        ${CMAKE_CURRENT_BINARY_DIR}/bin)
