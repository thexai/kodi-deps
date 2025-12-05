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

ExternalProject_Add(uwp_compat
    GIT_REPOSITORY https://github.com/Paxxi/uwp_compat
    GIT_TAG feb56bc2c4573b8657bba2b40648f0b2e2ecf72b
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DBUILD_SHARED_LIBS:BOOL=OFF
)
add_dependency_project_package(uwp_compat 0.1)

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
    URL https://www.openssl.org/source/openssl-3.0.18.tar.gz
    URL_HASH SHA256=d80c34f5cf902dccf1f1b5df5ebb86d0392e37049e5d73df1b3abae72e4ffe8b
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(openssl 3.0.18)

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
    URL https://netix.dl.sourceforge.net/project/lzmautils/xz-5.2.5.tar.gz
    URL_HASH SHA256=f6f4910fd033078738bd82bfba4f49219d03b17eb0794eb91efbae419f4aba10
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(xz 5.2.5)

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

ExternalProject_Add(crossguid
    GIT_REPOSITORY https://github.com/Paxxi/crossguid
    GIT_TAG 76116f6794a5c582a8b38b8e02651b8065d81c39
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DXG_TESTS:BOOL=OFF
)
add_dependency_project_package(crossguid 0.2.2)

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
    GIT_REPOSITORY https://github.com/Paxxi/freetype2
    GIT_TAG c5832e61dbec993c84ca4996b0ad4f73d3794a98
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(freetype 2.10.1)

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
    GIT_REPOSITORY https://github.com/Paxxi/harfbuzz
    GIT_TAG bcd28eaf3339b6b6762a7914ee7a9e734dccfa42
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DHB_HAVE_DIRECTWRITE=ON
        -DHB_HAVE_FREETYPE=ON
        -DBUILD_SHARED_LIBS=ON
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/freetype
)
add_dependency_project_package(harfbuzz 2.8.0)

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
    URL https://opensource.apple.com/tarballs/mDNSResponder/mDNSResponder-878.260.1.tar.gz
    URL_HASH SHA256=d6ba7bdfa06bee1255a67ac0a4fff92f48255360f8326ade92efe66d85068e1d
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DBUILD_SHARED_LIBS:BOOL=ON
)
add_dependency_project_package(dnssd 878.260.1)
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
    DEPENDS libgcrypt libgpg-error libiconv uwp_compat
    GIT_REPOSITORY https://github.com/Paxxi/libbdplus
    GIT_TAG c37157774aaaf5a1002474c17d966c4d0996b889
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/libgcrypt%3B%3B${PREFIX}/libgpg-error%3B%3B${PREFIX}/libiconv%3B%3B${PREFIX}/uwp_compat
)
add_dependency_project_package(libbdplus 0.1.2)

ExternalProject_Add(libmicrohttpd
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://ftpmirror.gnu.org/libmicrohttpd/libmicrohttpd-1.0.1.tar.gz
    URL_HASH SHA256=a89e09fc9b4de34dde19f4fcb4faaa1ce10299b9908db1132bbfa1de47882b94
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(libmicrohttpd 1.0.1)

ExternalProject_Add(libxml2
    DEPENDS libiconv
    GIT_REPOSITORY https://github.com/Paxxi/libxml2
    GIT_TAG c9e6dae59f576f0e40dbef5d52f11b3de00e8a7d
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/libiconv
        -DBUILD_SHARED_LIBS:BOOL=ON
)
add_dependency_project_package(libxml2 2.9.9)

ExternalProject_Add(libxslt
    DEPENDS libiconv libxml2 uwp_compat
    GIT_REPOSITORY https://github.com/Paxxi/libxslt
    GIT_TAG f63869008f803398f86697031201f83812d65732
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/libiconv%3B%3B${PREFIX}/libxml2%3B%3B${PREFIX}/uwp_compat
        -DBUILD_SHARED_LIBS:BOOL=ON
)
add_dependency_project_package(libxslt 1.1.34)

if(NOT WINDOWS_STORE)
ExternalProject_Add(libplist
    DEPENDS libiconv libxml2
    GIT_REPOSITORY https://github.com/Paxxi/libplist
    GIT_TAG 80d48b2bbec8dfb0c81db2074dedfa5b92fb2593
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/libiconv%3B%3B${PREFIX}/libxml2
        -DBUILD_SHARED_LIBS:BOOL=ON
)
add_dependency_project_package(libplist 2.1.0)
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

ExternalProject_Add(mariadb-connector-c
    DEPENDS openssl
    GIT_REPOSITORY https://github.com/thexai/mariadb-connector-c
    GIT_TAG e5dc7e86a0fc2dfd3f36efd5522570db245df57e
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/openssl
        -DBUILD_SHARED_LIBS:BOOL=ON
        -DPLUGIN_AUTH_GSSAPI_CLIENT:STRING=OFF
        -DPLUGIN_DIALOG:STRING=STATIC
        -DPLUGIN_MYSQL_CLEAR_PASSWORD:STRING=STATIC
        -DPLUGIN_PVIO_NPIPE:STRING=STATIC
        -DPLUGIN_PVIO_SHMEM:STRING=STATIC
        -DPLUGIN_SHA256_PASSWORD:STRING=STATIC
        -DPLUGIN_CLIENT_ED25519:STRING=STATIC
        -DPLUGIN_CACHING_SHA2_PASSWORD:STRING=STATIC
        -DWITH_CURL:BOOL=OFF
        -DWITH_MYSQLCOMPAT:BOOL=ON
        -DWITH_SSL:STRING=OPENSSL
        -DWITH_UNIT_TESTS:BOOL=OFF
)
add_dependency_project_package(mariadb-connector-c 3.3.17)

ExternalProject_Add(libffi
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://github.com/libffi/libffi/archive/refs/tags/v3.3-rc1.tar.gz
    URL_HASH SHA256=87cbd612fa20f8eb7b91a3e996c5032b813644deee6c2d96724b50752296737e
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(libffi 3.3.0)

ExternalProject_Add(python
    DEPENDS bzip2 openssl sqlite zlib expat libffi xz
    GIT_REPOSITORY https://github.com/thexai/cpython
    GIT_TAG 09051503302b44a5df9f174154b98c5607542719
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/bzip2%3B%3B${PREFIX}/openssl%3B%3B${PREFIX}/sqlite%3B%3B${PREFIX}/zlib%3B%3B${PREFIX}/libffi%3B%3B${PREFIX}/xz%3B%3B${PREFIX}/expat
)
add_dependency_project_package(python 3.13.9)

ExternalProject_Add(libjpeg-turbo
    GIT_REPOSITORY https://github.com/Paxxi/libjpeg-turbo
    GIT_TAG 38bff2bf9e3e183f06da5b02b5b69d8eecdf7335
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DENABLE_SHARED:BOOL=OFF
        -DENABLE_STATIC:BOOL=ON
        -DWITH_JPEG8:BOOL=ON
        -DWITH_SIMD:BOOL=OFF
        -DWITH_CRT_DLL:BOOL=ON
)
add_dependency_project_package(libjpeg-turbo 2.0.3)

ExternalProject_Add(pillow
    DEPENDS freetype libjpeg-turbo python zlib
    GIT_REPOSITORY https://github.com/thexai/Pillow
    GIT_TAG dbde60e30ffc8bfe8c461b576d41343f46e9553e
    GIT_SHALLOW ON
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
        -DCMAKE_PREFIX_PATH:PATH=${PREFIX}%3B%3B${PREFIX}/freetype%3B%3B${PREFIX}/libjpeg-turbo%3B%3B${PREFIX}/python%3B%3B${PREFIX}/zlib
)
add_dependency_project_package(pillow 11.2.1)

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

ExternalProject_Add(dav1d
    DOWNLOAD_NAME dav1d-1.5.1.tar.gz
    DOWNLOAD_DIR ${CMAKE_SOURCE_DIR}/downloads
    URL https://code.videolan.org/videolan/dav1d/-/archive/1.5.1/dav1d-1.5.1.tar.gz
    URL_HASH SHA256=fa635e2bdb25147b1384007c83e15de44c589582bb3b9a53fc1579cb9d74b695
    PATCH_COMMAND ${PATCH} -p1 -i ${CMAKE_SOURCE_DIR}/patches/$(TargetName).diff
    CMAKE_ARGS
        ${ADDITIONAL_ARGS}
        -DCMAKE_INSTALL_PREFIX:PATH=${INSTALL_PREFIX}
)
add_dependency_project_package(dav1d 1.5.1)

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
        crossguid
        curl
        date
        dav1d
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
        libmicrohttpd
        libpng
        libudfread
        libwebp
        libxml2
        libxslt
        mariadb-connector-c
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
        uwp_compat
)

add_custom_target(DependenciesRequiredDebug
    DEPENDS
        crossguid
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
