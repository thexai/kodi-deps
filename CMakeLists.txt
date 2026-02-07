cmake_minimum_required(VERSION 3.4)

project(KodiDependencies LANGUAGES C)

cmake_policy(SET CMP0114 NEW)

set_property(GLOBAL PROPERTY USE_FOLDERS ON)

option(FOLDER_PER_TARGET "install dependencies in separate folders" ON)

include(CheckSymbolExists)

check_symbol_exists(_X86_ "Windows.h" _X86_)
check_symbol_exists(_AMD64_ "Windows.h" _AMD64_)
check_symbol_exists(_ARM_ "Windows.h" _ARM_)
check_symbol_exists(_ARM64_ "Windows.h" _ARM64_)

if(_X86_)
   set(ARCH win32)
elseif(_AMD64_)
   set(ARCH x64)
elseif(_ARM_)
   set(ARCH arm)
elseif(_ARM64_)
   set(ARCH arm64)
else()
   set(ARCH unknown)
endif()

unset(_X86_)
unset(_AMD64_)
unset(_ARM_)
unset(_ARM64_)

if(CMAKE_SYSTEM_NAME STREQUAL WindowsStore)
    set(ADDITIONAL_ARGS "-DCMAKE_SYSTEM_NAME=${CMAKE_SYSTEM_NAME}" "-DCMAKE_SYSTEM_VERSION=${CMAKE_SYSTEM_VERSION}")
    set(ARCH "win10-${ARCH}")
endif()

if(NOT PREFIX)
    set(PREFIX "${CMAKE_SOURCE_DIR}/package/${ARCH}")
endif()

if(FOLDER_PER_TARGET)
    set(INSTALL_PREFIX "${PREFIX}/$(TargetName)")
else()
    set(INSTALL_PREFIX "${PREFIX}")
endif()

message(STATUS "  FOLDER_PER_TARGET: ${FOLDER_PER_TARGET}")
message(STATUS "             PREFIX: ${PREFIX}")
message(STATUS "    ADDITIONAL_ARGS: ${ADDITIONAL_ARGS}")
message(STATUS "     INSTALL_PREFIX: ${INSTALL_PREFIX}")

find_program(PATCH patch)
include(FindPackageHandleStandardArgs)
find_package_handle_standard_args(PATCH REQUIRED_VARS PATCH)
if(NOT ${PATCH_FOUND})
  message(FATAL_ERROR "patch not found")
endif()

include(ExternalProject)

add_custom_target(DependenciesRequired-zip)
function(add_dependency_project_package name version)
    # it's possible to pack dependency only if it was installed in separate folder
    if(FOLDER_PER_TARGET)
        set(hash_file "hashes")
        if(WINDOWS_STORE)
            set(hash_file "${hash_file}-win10")
        endif()
        set(hash_file "${hash_file}-$(Platform).txt")
        set(dep_name "${name}-${version}")
        string(TIMESTAMP date "%Y%m%d")
        if(NOT ANY IN_LIST ARGN)
            if(WINDOWS_STORE)
                set(dep_name "${dep_name}-win10")
            endif()
            set(dep_name "${dep_name}-$(Platform)-$(PlatformToolset)")
        endif()
        set(dep_name "${dep_name}-${date}")
        file(TO_NATIVE_PATH "${PREFIX}/../${dep_name}" zip_path)
        file(TO_NATIVE_PATH "${PREFIX}/${dep_name}" COPY_DESTINATION)
        file(TO_NATIVE_PATH "${PREFIX}/../${hash_file}" hash_path)

        ExternalProject_Add_Step(${name} zip
            COMMENT "Packaging '${name}'"
            DEPENDEES ${name}
            COMMAND ${CMAKE_COMMAND} -E copy_directory "${name}" "${COPY_DESTINATION}"
            COMMAND ${CMAKE_COMMAND} -E tar "cf" "${zip_path}.7z" --format=7zip -- "${COPY_DESTINATION}"
            COMMAND ${CMAKE_COMMAND} -E remove_directory "${COPY_DESTINATION}"
            COMMAND ${CMAKE_COMMAND} -Dhash_path=${hash_path} -Ddep_name=${dep_name} -Dzip_path=${zip_path} -P ${CMAKE_SOURCE_DIR}/zip.cmake
            WORKING_DIRECTORY ${PREFIX}
            ALWAYS TRUE
            EXCLUDE_FROM_MAIN TRUE
        )
        ExternalProject_Add_StepTargets(${name} zip)
        # ExternalProject_Add_StepDependencies(${name} zip ${name})
        add_dependencies(DependenciesRequired-zip ${name}-zip)
    endif()
endfunction()


# Executables for build

if(NOT WINDOWS_STORE)
ExternalProject_Add(flatc
    GIT_REPOSITORY https://github.com/Paxxi/flatbuffers
    GIT_TAG 46d447759eb599ed7ea8a2ab297db6ac40e2bbd1
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DFLATBUFFERS_BUILD_FLATHASH:BOOL=OFF
        -DFLATBUFFERS_BUILD_FLATLIB:BOOL=OFF
        -DFLATBUFFERS_BUILD_TESTS:BOOL=OFF
)
add_dependency_project_package(flatc 2.0.0)
endif()

# dependencies required by others

ExternalProject_Add(bzip2
    GIT_REPOSITORY https://github.com/Paxxi/bzip2
    GIT_TAG 6aba9bd98f1c67c9266a15884695ca04b7633445
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(bzip2 1.0.8)

ExternalProject_Add(expat
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/libexpat/libexpat/archive/R_2_7_3.tar.gz
    URL_HASH SHA256=7c853a76f7c2a562c3edfd84ca47e487d9eff3966786191dfb60cc441daf82d6
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    UPDATE_DISCONNECTED ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DEXPAT_BUILD_TESTS:BOOL=OFF
        -DEXPAT_BUILD_DOCS:BOOL=OFF
        -DEXPAT_BUILD_EXAMPLES:BOOL=OFF
        -DEXPAT_BUILD_TOOLS:BOOL=OFF
        -DEXPAT_SHARED_LIBS:BOOL=OFF
  SOURCE_SUBDIR expat
)
add_dependency_project_package(expat 2.7.3)

ExternalProject_Add(libiconv
    GIT_REPOSITORY https://github.com/Paxxi/libiconv
    GIT_TAG 88316873fbaa24c7dee0bbb739dbfaa2a51ddf20
    GIT_SHALLOW ON
    UPDATE_DISCONNECTED ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DBUILD_SHARED_LIBS:BOOL=OFF
)
add_dependency_project_package(libiconv 1.16)

ExternalProject_Add(openssl
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://www.openssl.org/source/openssl-3.0.19.tar.gz
    URL_HASH SHA256=fa5a4143b8aae18be53ef2f3caf29a2e0747430b8bc74d32d88335b94ab63072
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(openssl 3.0.19)

ExternalProject_Add(zlib
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/madler/zlib/releases/download/v1.3.1/zlib-1.3.1.tar.gz
    URL_HASH SHA256=9a93b2b7dfdac77ceba5a558a580e74667dd6fede4585b91eefb60f03b72df23
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DZLIB_BUILD_EXAMPLES:BOOL=OFF
        -DSKIP_INSTALL_FILES:BOOL=ON
)
add_dependency_project_package(zlib 1.3.1)

ExternalProject_Add(xz
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/tukaani-project/xz/releases/download/v5.8.1/xz-5.8.1.tar.gz
    URL_HASH SHA256=507825b599356c10dca1cd720c9d0d0c9d5400b9de300af00e4d1ea150795543
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DXZ_TOOL_XZ:BOOL=OFF
        -DXZ_TOOL_LZMAINFO:BOOL=OFF
        -DXZ_TOOL_XZDEC:BOOL=OFF
        -DXZ_TOOL_LZMADEC:BOOL=OFF
        -DXZ_DOC:BOOL=OFF
)
add_dependency_project_package(xz 5.8.1)

ExternalProject_Add(miniwdk
    GIT_REPOSITORY https://github.com/Paxxi/miniwdk
    GIT_TAG 2561ff58d73f16c4ee4beb6e25fa9c49026da115
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
# Causes miniwdk zip target to be added and it can't be built on github
#add_dependency_project_package(miniwdk 10.0.18362 ANY)

ExternalProject_Add(flatbuffers
    GIT_REPOSITORY https://github.com/Paxxi/flatbuffers
    GIT_TAG 46d447759eb599ed7ea8a2ab297db6ac40e2bbd1
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DFLATBUFFERS_BUILD_FLATC:BOOL=OFF
        -DFLATBUFFERS_BUILD_FLATHASH:BOOL=OFF
        -DFLATBUFFERS_BUILD_FLATLIB:BOOL=OFF
        -DFLATBUFFERS_BUILD_TESTS:BOOL=OFF
)
add_dependency_project_package(flatbuffers 2.0.0)

ExternalProject_Add(freetype
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://download.savannah.gnu.org/releases/freetype/freetype-2.14.1.tar.xz
    URL_HASH SHA256=32427e8c471ac095853212a37aef816c60b42052d4d9e48230bab3bdf2936ccc
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(freetype 2.14.1)

ExternalProject_Add(fstrcmp
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL http://fstrcmp.sourceforge.net/fstrcmp-0.7.D001.tar.gz
    URL_HASH SHA256=e4018e850f80700acee8da296e56e15b1eef711ab15157e542e7d7e1237c3476
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(fstrcmp 0.7)

ExternalProject_Add(libcdio
    DEPENDS libiconv
    GIT_REPOSITORY https://github.com/Paxxi/libcdio
    GIT_TAG 312d5b1ff684960095521be5e287eaeb5aaad4a5
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DBUILD_SHARED_LIBS:BOOL=ON
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/libiconv
)
add_dependency_project_package(libcdio 2.1.0)

ExternalProject_Add(libfribidi
    GIT_REPOSITORY https://github.com/thexai/fribidi
    GIT_TAG 29ca7f90e6117ccc8159ac751a5e533329e04530
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(libfribidi 1.0.8)

ExternalProject_Add(harfbuzz
    DEPENDS freetype
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/harfbuzz/harfbuzz/releases/download/12.3.2/harfbuzz-12.3.2.tar.xz
    URL_HASH SHA256=6f6db164359a2da5a84ef826615b448b33e6306067ad829d85d5b0bf936f1bb8
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DHB_HAVE_DIRECTWRITE:BOOL=ON
        -DHB_HAVE_FREETYPE:BOOL=ON
        -DHB_BUILD_SUBSET:BOOL=OFF
        -DBUILD_SHARED_LIBS:BOOL=ON
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/freetype
)
add_dependency_project_package(harfbuzz 12.3.2)

ExternalProject_Add(sqlite
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://www.sqlite.org/2025/sqlite-amalgamation-3500400.zip
    URL_HASH SHA256=1d3049dd0f830a025a53105fc79fd2ab9431aea99e137809d064d8ee8356b032
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DBUILD_SHARED_LIBS:BOOL=ON
)
add_dependency_project_package(sqlite 3500400)

ExternalProject_Add(tinyxml
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://sourceforge.net/projects/tinyxml/files/tinyxml/2.6.2/tinyxml_2_6_2.tar.gz
    URL_HASH SHA256=15bdfdcec58a7da30adc87ac2b078e4417dbe5392f3afb719f9ba6d062645593
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(tinyxml 2.6.2)


# Required, dyloaded deps
if(NOT WINDOWS_STORE)
ExternalProject_Add(detours
    GIT_REPOSITORY https://github.com/Paxxi/Detours
    GIT_TAG 195b39b960cd8106ceec90b3bf9c426bda034d12
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(detours 64ec13)
endif()

if(NOT WINDOWS_STORE)
ExternalProject_Add(dnssd
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://opensource.apple.com/tarballs/mDNSResponder/mDNSResponder-2881.60.4.tar.gz
    URL_HASH SHA256=192b74fa5f4fdd985d6347bc2b39fed9b194c633f7cf7392f0a4c4d8b64c96a7
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DBUILD_SHARED_LIBS:BOOL=ON
)
add_dependency_project_package(dnssd 2881.60.4)
endif()

ExternalProject_Add(lcms2
    GIT_REPOSITORY https://github.com/Paxxi/Little-CMS
    GIT_TAG 5c40d2402b4db97f6b91f83cfead6f28e14935f4
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DBUILD_SHARED_LIBS:BOOL=ON
)
add_dependency_project_package(lcms2 2.9)

ExternalProject_Add(libgpg-error
    DEPENDS libiconv
    GIT_REPOSITORY https://github.com/Paxxi/libgpg-error
    GIT_TAG 36ba2419cd42ca3b16b45d317fecb531ab23e3e0
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/libiconv
)
add_dependency_project_package(libgpg-error 1.36.0)

ExternalProject_Add(libgcrypt
    DEPENDS libgpg-error libiconv
    GIT_REPOSITORY https://github.com/Paxxi/libgcrypt
    GIT_TAG 7bb6e291fb112125d827cd1a75071ad4f6c3b704
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/libgpg-error%3B%3B${PREFIX}/libiconv
)
add_dependency_project_package(libgcrypt 1.8.5)

ExternalProject_Add(winflexbison
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/lexxmark/winflexbison/releases/download/v2.5.20/win_flex_bison-2.5.20.zip
    URL_HASH SHA256=8d751f31eb8d79251d809f37bb0d853d72f58a21c5f672d5e31f35b4d97b31fe
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/winflexbison.diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(winflexbison 2.5.20)

ExternalProject_Add(libaacs
    DEPENDS libgcrypt libgpg-error winflexbison libiconv
    GIT_REPOSITORY https://github.com/Paxxi/libaacs
    GIT_TAG 8fc07cd78e47f276fc7f19220409adb31ec9ccb4
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DBUILD_SHARED_LIBS:BOOL=ON
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/libgcrypt%3B%3B${PREFIX}/libgpg-error%3B%3B${PREFIX}/winflexbison%3B%3B${PREFIX}/libiconv
)
add_dependency_project_package(libaacs 0.9.0)

ExternalProject_Add(libbdplus
    DEPENDS libgcrypt libgpg-error libiconv
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://download.videolan.org/pub/videolan/libbdplus/0.2.0/libbdplus-0.2.0.tar.bz2
    URL_HASH SHA256=b93eea3eaef33d6e9155d2c34b068c505493aa5a4936e63274f4342ab0f40a58
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/libgcrypt%3B%3B${PREFIX}/libgpg-error%3B%3B${PREFIX}/libiconv
)
add_dependency_project_package(libbdplus 0.2.0)

ExternalProject_Add(libxml2
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://download.gnome.org/sources/libxml2/2.15/libxml2-2.15.1.tar.xz
    URL_HASH SHA256=c008bac08fd5c7b4a87f7b8a71f283fa581d80d80ff8d2efd3b26224c39bc54c
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DBUILD_SHARED_LIBS:BOOL=ON
        -DLIBXML2_WITH_DEBUG:BOOL=OFF
        -DLIBXML2_WITH_ICONV:BOOL=OFF
        -DLIBXML2_WITH_MODULES:BOOL=OFF
        -DLIBXML2_WITH_PROGRAMS:BOOL=OFF
        -DLIBXML2_WITH_TESTS:BOOL=OFF
)
add_dependency_project_package(libxml2 2.15.1)

if(NOT WINDOWS_STORE)
ExternalProject_Add(libplist
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/libimobiledevice/libplist/releases/download/2.7.0/libplist-2.7.0.tar.bz2
    URL_HASH SHA256=7ac42301e896b1ebe3c654634780c82baa7cb70df8554e683ff89f7c2643eb8b
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DBUILD_SHARED_LIBS:BOOL=ON
)
add_dependency_project_package(libplist 2.7.0)
endif()

ExternalProject_Add(giflib
    GIT_REPOSITORY https://github.com/Paxxi/giflib
    GIT_TAG 9dfce00d4de675ee813d4bb95b59a5b189bb1f1c
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(giflib 5.2.1)

ExternalProject_Add(libpng
    DEPENDS zlib
    GIT_REPOSITORY https://github.com/Paxxi/libpng
    GIT_TAG b6910d90aa4b0c9c7a18fc7d140cd5ed405f278e
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/zlib
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DPNG_TESTS:BOOL=OFF
        -DPNG_SHARED:BOOL=OFF
)
add_dependency_project_package(libpng 1.6.37)

ExternalProject_Add(libwebp
    DEPENDS zlib
    GIT_REPOSITORY https://github.com/Paxxi/libwebp
    GIT_TAG 9b8da9837f804149394960b9a4e3226762960723
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/zlib
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DWEBP_BUILD_ANIM_UTILS=OFF
        -DWEBP_BUILD_CWEBP=OFF
        -DWEBP_BUILD_DWEBP=OFF
        -DWEBP_BUILD_GIF2WEBP=OFF
        -DWEBP_BUILD_IMG2WEBP=OFF
        -DWEBP_BUILD_VWEBP=OFF
        -DWEBP_BUILD_WEBPINFO=OFF
        -DWEBP_BUILD_WEBPMUX=OFF
        -DWEBP_BUILD_EXTRAS=OFF
)
add_dependency_project_package(libwebp 1.0.3)

ExternalProject_Add(libffi
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/libffi/libffi/archive/refs/tags/v3.4.4.tar.gz
    URL_HASH SHA256=828639972716ed18833df7b659b32060591fe0eb625a8d34078920d33c2dc867
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(libffi 3.4.4)

ExternalProject_Add(zstd
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/facebook/zstd/releases/download/v1.5.7/zstd-1.5.7.tar.gz
    URL_HASH SHA256=eb33e51f49a15e023950cd7825ca74a4a2b43db8354825ac24fc1b7ee09e6fa3
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    SOURCE_SUBDIR build/cmake
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DZSTD_BUILD_STATIC:BOOL=ON
        -DZSTD_BUILD_DYNAMIC:BOOL=OFF
)
add_dependency_project_package(zstd 1.5.7)

ExternalProject_Add(python
    DEPENDS bzip2 openssl sqlite zlib expat libffi xz zstd
    GIT_REPOSITORY https://github.com/thexai/cpython
    GIT_TAG e170687208abb826b566ea132987f5441af14285
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/bzip2%3B%3B${PREFIX}/openssl%3B%3B${PREFIX}/sqlite%3B%3B${PREFIX}/zlib%3B%3B${PREFIX}/libffi%3B%3B${PREFIX}/xz%3B%3B${PREFIX}/expat%3B%3B${PREFIX}/zstd
)
add_dependency_project_package(python 3.14.2)

ExternalProject_Add(libjpeg-turbo
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/libjpeg-turbo/libjpeg-turbo/releases/download/3.1.3/libjpeg-turbo-3.1.3.tar.gz
    URL_HASH SHA256=075920b826834ac4ddf97661cc73491047855859affd671d52079c6867c1c6c0
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DENABLE_SHARED:BOOL=OFF
        -DENABLE_STATIC:BOOL=ON
        -DWITH_JPEG8:BOOL=ON
        -DWITH_SIMD:BOOL=ON
        -DWITH_CRT_DLL:BOOL=ON
        -DWITH_TOOLS:BOOL=OFF
)
add_dependency_project_package(libjpeg-turbo 3.1.3)

ExternalProject_Add(pillow
    DEPENDS freetype libjpeg-turbo python zlib
    GIT_REPOSITORY https://github.com/thexai/Pillow
    GIT_TAG e214c7bd69eb375773265a9d64c08dd2f95a7a02
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/freetype%3B%3B${PREFIX}/libjpeg-turbo%3B%3B${PREFIX}/python%3B%3B${PREFIX}/zlib
)
add_dependency_project_package(pillow 12.1.0)

ExternalProject_Add(pycryptodome
    DEPENDS python
    GIT_REPOSITORY https://github.com/thexai/pycryptodome
    GIT_TAG 3801f6a9614724b7965352068bf6d4643a171707
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/python
        -DBUILD_SHARED_LIBS:BOOL=ON
        -DSEPARATE_NAMESPACE:BOOL=ON
)
add_dependency_project_package(pycryptodome 3.23.0)

if(NOT WINDOWS_STORE)
ExternalProject_Add(shairplay
    DOWNLOAD_NAME shairplay-ce80e00.tar.gz
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/juhovh/shairplay/archive/ce80e00.tar.gz
    URL_HASH SHA256=49ac0e0470ec6367f720c7b79a09165138f57b60f5949a2d9c38be5823d13294
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DBUILD_SHARED_LIBS:BOOL=ON
)
add_dependency_project_package(shairplay ce80e00)
endif()

ExternalProject_Add(libudfread
    GIT_REPOSITORY https://github.com/Paxxi/libudfread
    GIT_TAG 29fcdb5280e5d1778786795b594ec9fc60bf75da
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(libudfread 1.1.2)

ExternalProject_Add(libbluray
    DEPENDS freetype libiconv libxml2 libudfread libaacs
    GIT_REPOSITORY https://code.videolan.org/videolan/libbluray.git
    GIT_TAG bb5bc108ec695889855f06df338958004ff289ef
    GIT_SHALLOW ON
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/freetype%3B%3B${PREFIX}/libiconv%3B%3B${PREFIX}/libxml2%3B%3B${PREFIX}/libudfread
)
add_dependency_project_package(libbluray 1.3.4)

ExternalProject_Add(GoogleTest
    GIT_REPOSITORY https://github.com/Google/GoogleTest
    GIT_TAG 703bd9caab50b139428cea1aaff9974ebee5742e
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -Dgtest_force_shared_crt:BOOL=ON
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(GoogleTest 1.10.0)

ExternalProject_Add(date
    GIT_REPOSITORY https://github.com/Paxxi/date
    GIT_TAG e9943526c2ba23746b3fe4fb0522be85b1efcd00
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DUSE_SYSTEM_TZ_DB:BOOL=OFF
        -DUSE_TZ_DB_IN_DOT:BOOL=OFF
        -DBUILD_SHARED_LIBS:BOOL=OFF
        -DENABLE_DATE_TESTING:BOOL=OFF
        -DDISABLE_STRING_VIEW:BOOL=ON
        -DBUILD_TZ_LIB:BOOL=ON
        -DMANUAL_TZ_DB:BOOL=ON
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}
)
add_dependency_project_package(date 3.0.1)

add_custom_target(DependenciesRequired
    DEPENDS
        bzip2
        date
        expat
        flatbuffers
        freetype
        fstrcmp
        giflib
        GoogleTest
        harfbuzz
        lcms2
        libaacs
        libbdplus
        libbluray
        libcdio
        libffi
        libfribidi
        libgpg-error
        libgcrypt
        libjpeg-turbo
        libiconv
        libpng
        libudfread
        libwebp
        libxml2
        #miniwdk # WDK is not installed on github hosted runners
        openssl
        python
        pillow
        pycryptodome
        sqlite
        tinyxml
        winflexbison
        xz
        zlib
        zstd
)

add_custom_target(DependenciesRequiredDebug
    DEPENDS
        date
        detours
        GoogleTest
        libudfread
        tinyxml
        zlib
)

if(NOT WINDOWS_STORE)
add_dependencies(DependenciesRequired
    detours
    dnssd
    flatc
    libplist
    shairplay
)
endif()
