# Many of this is based on the scripts in the Polycode project.  Many thanks
# to the respective authors for going through the work of figuring this out!

cmake_minimum_required(VERSION 2.8.3)

# Set defaults.  Must be before project().
if(APPLE)
  set(CMAKE_OSX_ARCHITECTURES "x86_64" CACHE STRING "")
  set(CMAKE_OSX_DEPLOYMENT_TARGET "10.9" CACHE STRING "")
  set(CMAKE_XCODE_ATTRIBUTE_CLANG_CXX_LIBRARY "libc++")
endif()

project(panda3d-thirdparty)

include(ExternalProject)

set(DEFAULT_ENABLE ON)
if(DISABLE_ALL)
  set(DEFAULT_ENABLE OFF)
endif()

if(APPLE)
  set(NOT_APPLE OFF)
else()
  set(NOT_APPLE ${DEFAULT_ENABLE})
endif()

option(BUILD_ZLIB "Build zlib" ${DEFAULT_ENABLE})
option(BUILD_PNG "Build zlib (requires zlib)" ${DEFAULT_ENABLE})
option(BUILD_ASSIMP "Build assimp (requires zlib)" ${NOT_APPLE})
option(BUILD_BULLET "Build bullet" ${DEFAULT_ENABLE})
option(BUILD_HARFBUZZ "Build harfbuzz (requires freetype)" ${DEFAULT_ENABLE})
option(BUILD_FREETYPE "Build freetype (requires harfbuzz, zlib and png)" ${DEFAULT_ENABLE})
option(BUILD_VORBIS "Build vorbis" ${DEFAULT_ENABLE})
option(BUILD_OPUS "Build opus" ${DEFAULT_ENABLE})
option(BUILD_OPENAL "Build OpenAL" ${NOT_APPLE})
option(BUILD_JPEG "Build libjpeg" ${DEFAULT_ENABLE})
option(BUILD_SQUISH "Build libsquish" ${DEFAULT_ENABLE})
option(BUILD_FCOLLADA "Build FCollada" ${DEFAULT_ENABLE})
option(BUILD_VRPN "Build VRPN" ${DEFAULT_ENABLE})
option(BUILD_TIFF "Build TIFF (requires zlib and jpeg)" ${DEFAULT_ENABLE})
option(BUILD_EIGEN "Copy Eigen headers" ${DEFAULT_ENABLE})
option(BUILD_ODE "Build ODE physics engine" ${DEFAULT_ENABLE})
option(BUILD_ARTOOLKIT "Build ARToolKit" ${DEFAULT_ENABLE})
option(BUILD_NVIDIACG "Copy NVIDIA Cg Toolkit binaries" ${DEFAULT_ENABLE})
option(BUILD_OPENSSL "Build OpenSSL" ${DEFAULT_ENABLE})
option(BUILD_OPENEXR "Build OpenEXR" ${DEFAULT_ENABLE})
option(BUILD_FFMPEG "Build FFMpeg (requires vorbis, freetype and zlib)" ${DEFAULT_ENABLE})


if(CMAKE_GENERATOR MATCHES "^Visual Studio [0-9]+ ")
  string(REPLACE " " ";" WORDS ${CMAKE_GENERATOR})
  list(GET WORDS 2 VCVER)
  set(THIRDPARTY_DIR ${CMAKE_SOURCE_DIR}/win-libs-vc${VCVER})

  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(THIRDPARTY_DIR "${THIRDPARTY_DIR}-x64")
  endif()

elseif(CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  set(THIRDPARTY_DIR ${CMAKE_SOURCE_DIR}/darwin-libs-a)

elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(THIRDPARTY_DIR ${CMAKE_SOURCE_DIR}/linux-libs-x64)
  else()
    set(THIRDPARTY_DIR ${CMAKE_SOURCE_DIR}/linux-libs-a)
  endif()

elseif(CMAKE_SYSTEM_NAME STREQUAL "FreeBSD")
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(THIRDPARTY_DIR ${CMAKE_SOURCE_DIR}/freebsd-libs-x64)
  else()
    set(THIRDPARTY_DIR ${CMAKE_SOURCE_DIR}/freebsd-libs-a)
  endif()

else()
  message(FATAL_ERROR "Unknown platform.")
endif()

if(NOT MSVC)
  set(CMAKE_C_FLAGS "-fpic")
  set(CMAKE_CXX_FLAGS "-fpic")
endif()

message("thirdparty packages will be installed to ${THIRDPARTY_DIR}")

set(COMMON_CMAKE_ARGS
  -DCMAKE_C_FLAGS=${CMAKE_C_FLAGS}
  -DCMAKE_CXX_FLAGS=${CMAKE_CXX_FLAGS}
  -DCMAKE_INSTALL_PREFIX:PATH=<INSTALL_DIR>
  -DCMAKE_BUILD_TYPE=${CMAKE_BUILD_TYPE}
  -DCMAKE_C_COMPILER=${CMAKE_C_COMPILER}
  -DCMAKE_CXX_COMPILER=${CMAKE_CXX_COMPILER}
  -DCMAKE_LINKER=${CMAKE_LINKER}
  -DCMAKE_TOOLCHAIN_FILE=${CMAKE_TOOLCHAIN_FILE}
  -DCMAKE_INSTALL_LIBDIR=lib
  -DBUILD_SHARED_LIBS=OFF
  -DSKIP_INSTALL_EXECUTABLES=ON
  -DSKIP_INSTALL_FILES=ON
)

if(APPLE)
  # Prevent the semicolon from becoming a space in the command.
  string(REPLACE ";" "$<SEMICOLON>" archs "${CMAKE_OSX_ARCHITECTURES}")
  set(COMMON_CMAKE_ARGS ${COMMON_CMAKE_ARGS}
    -DCMAKE_OSX_ARCHITECTURES=${archs}
    -DCMAKE_OSX_DEPLOYMENT_TARGET=${CMAKE_OSX_DEPLOYMENT_TARGET}
    -DCMAKE_OSX_SYSROOT=${CMAKE_OSX_SYSROOT}
  )
endif(APPLE)

if(BUILD_ZLIB)
  ExternalProject_Add(
    zlib
    GIT_REPOSITORY https://github.com/madler/zlib.git
    GIT_TAG cacf7f1d4e3d44d871b605da3b647f07d718623f #v1.2.11
    #URL https://github.com/madler/zlib/archive/v1.2.11.tar.gz
    #URL_HASH SHA256=629380c90a77b964d896ed37163f5c3a34f6e6d897311f1df2a7016355c45eff

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DZLIB_TESTS=OFF
    INSTALL_DIR ${THIRDPARTY_DIR}/zlib
  )

  # Remove the shared libraries, which cannot be disabled in the CMake file.
  if(WIN32)
    ExternalProject_Add_Step(zlib remove_extras
      COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/bin"
      COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/share"
      COMMAND ${CMAKE_COMMAND} -E remove "<INSTALL_DIR>/lib/zlib.lib"
      DEPENDEES install
    )
  elseif(APPLE)
    ExternalProject_Add_Step(zlib remove_extras
      COMMAND ${CMAKE_COMMAND} -E remove "<INSTALL_DIR>/lib/libz.dylib"
                                         "<INSTALL_DIR>/lib/libz.1.dylib"
                                         "<INSTALL_DIR>/lib/libz.1.2.11.dylib"
      DEPENDEES install
    )
  else()
    ExternalProject_Add_Step(zlib remove_extras
      COMMAND ${CMAKE_COMMAND} -E remove "<INSTALL_DIR>/lib/libz.so"
                                         "<INSTALL_DIR>/lib/libz.so.1"
                                         "<INSTALL_DIR>/lib/libz.so.1.2.11"
      DEPENDEES install
    )
  endif()
endif()

if(BUILD_PNG)
  ExternalProject_Add(
    png
    DEPENDS zlib

    # Please note that libpng 1.6.32+ requires CMake 3.0.2
    GIT_REPOSITORY https://github.com/glennrp/libpng.git
    GIT_TAG 9fcb846402d8692d25cb5286777a832ca852fa44 #v1.6.31
    #URL https://github.com/glennrp/libpng/archive/v1.6.31.tar.gz
    #URL_HASH SHA256=79c0391804c83a6e0b722c20be7c3a27aeb2fe09afbc3028abad243444c98aa0

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DCMAKE_PREFIX_PATH=${THIRDPARTY_DIR}/zlib
      -DPNG_SHARED=OFF
      -DPNG_STATIC=ON
      -DPNG_TESTS=OFF
      -DPNG_DEBUG=OFF
      -DAWK= # Needed for multi-arch build on macOS
    INSTALL_DIR ${THIRDPARTY_DIR}/png
  )

  ExternalProject_Add_Step(png remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/libpng"
    DEPENDEES install
  )
endif()

if(BUILD_ASSIMP)
  ExternalProject_Add(
    assimp
    DEPENDS zlib
    GIT_REPOSITORY https://github.com/assimp/assimp.git
    GIT_TAG 837a0742c4c7b4ce470516d9b7586570afe741ea

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DCMAKE_PREFIX_PATH=${THIRDPARTY_DIR}/zlib
      -DASSIMP_BUILD_TESTS=OFF
      -DASSIMP_BUILD_ASSIMP_TOOLS=OFF
      -DASSIMP_NO_EXPORT=ON
      -DLIBRARY_SUFFIX=
    INSTALL_DIR ${THIRDPARTY_DIR}/assimp
  )

  ExternalProject_Add_Step(assimp remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/cmake"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/pkgconfig"
    DEPENDEES install
  )
endif()

if(BUILD_BULLET)
  ExternalProject_Add(
    bullet
    GIT_REPOSITORY https://github.com/bulletphysics/bullet3.git
    GIT_TAG 2fb92bc40c16e4da5a9018479d4a8e1899702ab8 #v2.83.7
    #URL https://github.com/bulletphysics/bullet3/archive/2.83.7.tar.gz
    #URL_HASH SHA256=919087c764d0e93555c6ef0830c8951a9caafea99f52ac8b366e96adbe0112ed

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DINSTALL_LIBS=ON
      -DUSE_MSVC_RUNTIME_LIBRARY_DLL=ON
      -DBUILD_DEMOS=OFF
      -DBUILD_CPU_DEMOS=OFF
      -DBUILD_OPENGL3_DEMOS=OFF
      -DBUILD_BULLET2_DEMOS=OFF
      -DBUILD_BULLET3=OFF
      -DBUILD_EXTRAS=OFF
      -DBUILD_UNIT_TESTS=OFF
      -DINCLUDE_INSTALL_DIR:STRING=include/
    INSTALL_DIR ${THIRDPARTY_DIR}/bullet
  )

  ExternalProject_Add_Step(bullet remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/cmake"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/pkgconfig"
    DEPENDEES install
  )
endif()

if(BUILD_HARFBUZZ)
  # Ugh, freetype and harfbuzz depend on each other?!  Madness.  Well, we'll
  # have to make a temporary build of freetype so we can build HarfBuzz.
  ExternalProject_Add(
    freetype-nohb
    DEPENDS zlib png
    URL https://download.savannah.gnu.org/releases/freetype/freetype-2.7.1.tar.gz
    URL_HASH SHA256=162ef25aa64480b1189cdb261228e6c5c44f212aac4b4621e28cf2157efb59f5

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DCMAKE_PREFIX_PATH=${THIRDPARTY_DIR}/zlib;${THIRDPARTY_DIR}/png
      -DWITH_HarfBuzz=OFF
      -DWITH_BZip2=OFF
      -DPNG_INCLUDE_DIRS=${THIRDPARTY_DIR}/png/include
    INSTALL_DIR ${CMAKE_CURRENT_BINARY_DIR}/freetype-nohb
  )

  # Use the tarball instead of git repo in order to avoid the need for ragel.
  ExternalProject_Add(
    harfbuzz
    DEPENDS freetype-nohb
    URL https://www.freedesktop.org/software/harfbuzz/release/harfbuzz-2.6.4.tar.xz
    URL_HASH SHA256=9413b8d96132d699687ef914ebb8c50440efc87b3f775d25856d7ec347c03c12

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DCMAKE_PREFIX_PATH=${CMAKE_CURRENT_BINARY_DIR}/freetype-nohb
      -DHB_HAVE_FREETYPE=ON
      -DHB_BUILD_TESTS=OFF
      -DHB_BUILD_UTILS=OFF
      -DHB_BUILD_SUBSET=OFF
      -DHB_HAVE_INTROSPECTION=OFF
      -DHB_HAVE_CORETEXT=OFF # macOS 10.8+
      -DFREETYPE_INCLUDE_DIR_freetype2=${CMAKE_CURRENT_BINARY_DIR}/freetype-nohb/include/freetype2
      -DFREETYPE_INCLUDE_DIR_ft2build=${CMAKE_CURRENT_BINARY_DIR}/freetype-nohb/include/freetype2
    INSTALL_DIR ${THIRDPARTY_DIR}/harfbuzz
  )

  ExternalProject_Add_Step(harfbuzz move_headers
    COMMAND ${CMAKE_COMMAND} -E rename "<INSTALL_DIR>/include/harfbuzz" "<INSTALL_DIR>/include-tmp"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/include"
    COMMAND ${CMAKE_COMMAND} -E rename "<INSTALL_DIR>/include-tmp" "<INSTALL_DIR>/include"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/cmake"
    DEPENDEES install
  )
endif()

if(BUILD_FREETYPE)
  ExternalProject_Add(
    freetype
    DEPENDS zlib png harfbuzz
    URL https://download.savannah.gnu.org/releases/freetype/freetype-2.7.1.tar.gz
    URL_HASH SHA256=162ef25aa64480b1189cdb261228e6c5c44f212aac4b4621e28cf2157efb59f5

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DCMAKE_PREFIX_PATH=${THIRDPARTY_DIR}/zlib;${THIRDPARTY_DIR}/png;${THIRDPARTY_DIR}/harfbuzz
      -DWITH_HarfBuzz=ON
      -DWITH_BZip2=OFF
      -DPNG_INCLUDE_DIRS=${THIRDPARTY_DIR}/png/include
      -DHARFBUZZ_INCLUDE_DIRS=${THIRDPARTY_DIR}/harfbuzz/include
      -DPC_HARFBUZZ_INCLUDEDIR=${THIRDPARTY_DIR}/harfbuzz/include
      -DPC_HARFBUZZ_LIBDIR=${THIRDPARTY_DIR}/harfbuzz/lib
    INSTALL_DIR ${THIRDPARTY_DIR}/freetype
  )

  ExternalProject_Add_Step(freetype remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/cmake"
    DEPENDEES install
  )
endif()

if(BUILD_VORBIS)
  ExternalProject_Add(
    ogg
    GIT_REPOSITORY https://github.com/xiph/ogg.git
    GIT_TAG dd85929dbe38be4b3876c9c0d6d5dcb7a128f388

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
    INSTALL_DIR ${THIRDPARTY_DIR}/vorbis
  )

  ExternalProject_Add(
    vorbis
    DEPENDS ogg
    GIT_REPOSITORY https://github.com/xiph/vorbis.git
    GIT_TAG 8ef0f8058854b2ef55d2d42bbe84487a9aadae12

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DOGG_ROOT=${THIRDPARTY_DIR}/vorbis
    INSTALL_DIR ${THIRDPARTY_DIR}/vorbis
  )

  ExternalProject_Add_Step(vorbis remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/pkgconfig"
    DEPENDEES install
  )
endif()

if(BUILD_OPUS)
  ExternalProject_Add(
    ogg-opus
    GIT_REPOSITORY https://github.com/xiph/ogg.git
    GIT_TAG dd85929dbe38be4b3876c9c0d6d5dcb7a128f388

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
    INSTALL_DIR ${THIRDPARTY_DIR}/opus
  )

  ExternalProject_Add(
    opus
    GIT_REPOSITORY https://github.com/xnorpx/opus.git
    GIT_TAG e58822ef814b9bd9baeabecd7024aa7a6d4b5adf # PR#100

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
    INSTALL_DIR ${THIRDPARTY_DIR}/opus
  )

  ExternalProject_Add(
    opusfile
    DEPENDS ogg-opus opus
    GIT_REPOSITORY https://github.com/xiph/opusfile.git
    GIT_TAG b7c2ed3bafd9b08cd5d276894ca18fb58bcbfbc9 #0.11

    PATCH_COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_LIST_DIR}/libopusfile.cmake <SOURCE_DIR>/CMakeLists.txt

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DOGG_ROOT=${THIRDPARTY_DIR}/opus
      -DOPUS_ROOT=${THIRDPARTY_DIR}/opus
    INSTALL_DIR ${THIRDPARTY_DIR}/opus
  )

  ExternalProject_Add_Step(ogg-opus remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/pkgconfig"
    DEPENDEES install
  )
endif()

if(BUILD_OPENAL)
  ExternalProject_Add(
    openal
    GIT_REPOSITORY https://github.com/kcat/openal-soft.git
    GIT_TAG 6761218e51699f46bf25c377e65b3e9ea5e434b9 # 1.19.1

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DLIBTYPE=STATIC
      -DALSOFT_HRTF_DEFS=OFF
      -DALSOFT_AMBDEC_PRESETS=OFF
      -DALSOFT_CONFIG=OFF
      -DALSOFT_NO_CONFIG_UTIL=ON
      -DALSOFT_EXAMPLES=OFF
      -DALSOFT_TESTS=OFF
      -DALSOFT_UTILS=OFF
      -DALSOFT_ENABLE_SSE2_CODEGEN=OFF      # for only 32-bit
    INSTALL_DIR ${THIRDPARTY_DIR}/openal
  )

  ExternalProject_Add_Step(openal remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/cmake"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/pkgconfig"
    DEPENDEES install
  )
endif()

if(BUILD_JPEG)
  ExternalProject_Add(
    jpeg
    URL http://www.ijg.org/files/jpegsrc.v9b.tar.gz
    URL_HASH SHA256=240fd398da741669bf3c90366f58452ea59041cacc741a489b99f2f6a0bad052

    PATCH_COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_LIST_DIR}/libjpeg.cmake <SOURCE_DIR>/CMakeLists.txt
          COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_LIST_DIR}/libjpeg-jconfig.h.cmake <SOURCE_DIR>/jconfig.h.cmake

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DBUILD_STATIC=ON
      -DBUILD_EXECUTABLES=OFF
      -DBUILD_TESTS=OFF
    INSTALL_DIR ${THIRDPARTY_DIR}/jpeg
  )

  if(WIN32)
    set(JPEG_LIBRARY ${THIRDPARTY_DIR}/jpeg/lib/jpeg-static.lib)
  else()
    set(JPEG_LIBRARY ${THIRDPARTY_DIR}/jpeg/lib/libjpeg.a)
  endif()
else()
  set(JPEG_LIBRARY JPEG_LIBRARY-NOTFOUND)
endif()

if(BUILD_SQUISH)
  ExternalProject_Add(
    squish
    DEPENDS png
    URL https://storage.googleapis.com/google-code-archive-downloads/v2/code.google.com/libsquish/squish-1.10.tar.gz
    URL_HASH SHA256=aec3428badafdbb3b0ed1ea3a385a8029e65694282459f61b4a538963e8cada3

    PATCH_COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_LIST_DIR}/libsquish.cmake <SOURCE_DIR>/CMakeLists.txt

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DCMAKE_PREFIX_PATH=${THIRDPARTY_DIR}/png
    INSTALL_DIR ${THIRDPARTY_DIR}/squish
  )
endif()

if(BUILD_FCOLLADA)
  ExternalProject_Add(
    fcollada
    GIT_REPOSITORY https://github.com/rdb/fcollada.git

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DBUILD_UNICODE=OFF
      -DBUILD_TESTS=OFF
    INSTALL_DIR ${THIRDPARTY_DIR}/fcollada
  )
endif()

if(BUILD_VRPN)
  ExternalProject_Add(
    vrpn
    # Pull from git since 7.33 has some compile issues
    GIT_REPOSITORY https://github.com/vrpn/vrpn.git
    GIT_TAG 9cfc1a536f39e8815066c39a1cef0f8a515a2860

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DQUATLIB_BUILD_EXAMPLES=OFF
      -DVRPN_BUILD_CLIENTS=OFF
      -DVRPN_BUILD_CLIENT_LIBRARY=ON
      -DVRPN_BUILD_JAVA=OFF
      -DVRPN_BUILD_PYTHON=OFF
      -DVRPN_BUILD_PYTHON_HANDCODED_2X=OFF
      -DVRPN_BUILD_PYTHON_HANDCODED_3X=OFF
      -DVRPN_BUILD_SERVERS=OFF
      -DVRPN_BUILD_SERVER_LIBRARY=OFF
      -DBUILD_TESTING=FALSE
    INSTALL_DIR ${THIRDPARTY_DIR}/vrpn
  )
endif()

if(BUILD_TIFF)
  ExternalProject_Add(
    tiff
    DEPENDS zlib jpeg
    URL http://download.osgeo.org/libtiff/tiff-4.0.6.tar.gz
    URL_HASH SHA256=4d57a50907b510e3049a4bba0d7888930fdfc16ce49f1bf693e5b6247370d68c

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DCMAKE_PREFIX_PATH=${THIRDPARTY_DIR}/zlib;${THIRDPARTY_DIR}/jpeg
      -DJPEG_INCLUDE_DIR=${THIRDPARTY_DIR}/jpeg/include
      -DJPEG_LIBRARY=${JPEG_LIBRARY}
      -Dzlib=ON -Djpeg=ON -Dlzma=OFF -Dcxx=OFF
    INSTALL_DIR ${THIRDPARTY_DIR}/tiff
  )

  # Remove the extra bin and share directories it created.
  ExternalProject_Add_Step(tiff remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/bin"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/share"
    DEPENDEES install
  )
endif()

if(BUILD_EIGEN)
  ExternalProject_Add(
    eigen
    URL https://bitbucket.org/eigen/eigen/get/3.2.8.tar.gz
    URL_HASH SHA256=8d01d2d3a173c59065642a08d1dd661b399e4fed6edbda632143e7be937de7e0

    # No need to build anything; just copy over the Eigen directory.
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""

    INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/Eigen <INSTALL_DIR>/include/Eigen
            COMMAND ${CMAKE_COMMAND} -E copy_if_different <SOURCE_DIR>/signature_of_eigen3_matrix_library <INSTALL_DIR>/include/signature_of_eigen3_matrix_library
    INSTALL_DIR ${THIRDPARTY_DIR}/eigen
  )
endif()

if(BUILD_ODE)
  ExternalProject_Add(
    ode
    URL https://bitbucket.org/odedevs/ode/downloads/ode-0.13.1.tar.gz
    URL_HASH SHA256=35e55e05c6c6ebb813a546017285a7aceedb3e8e55c8ff102e80e26bd84c5658

    PATCH_COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_LIST_DIR}/libode.cmake <SOURCE_DIR>/CMakeLists.txt
          COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_LIST_DIR}/libode-config.h.cmake <SOURCE_DIR>/ode/src/config.h.cmake

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
    INSTALL_DIR ${THIRDPARTY_DIR}/ode
  )
endif()

if(BUILD_ARTOOLKIT)
  ExternalProject_Add(
    artoolkit
    URL http://www.panda3d.org/download/noversion/ARToolKit-2.72.1.tgz
    URL_HASH SHA256=1a49a67c6e196b0e9990e90255ce840287ac47b5a53371a68a6dd93a5425d259

    PATCH_COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_LIST_DIR}/artoolkit.cmake <SOURCE_DIR>/CMakeLists.txt
          COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_LIST_DIR}/artoolkit-config.h.cmake <SOURCE_DIR>/include/AR/config.h.cmake

    CMAKE_ARGS ${COMMON_CMAKE_ARGS} -DCMAKE_OSX_ARCHITECTURES=x86_64
    INSTALL_DIR ${THIRDPARTY_DIR}/artoolkit
  )
endif()

if(BUILD_NVIDIACG AND CMAKE_SYSTEM_NAME STREQUAL "Windows")
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    ExternalProject_Add(
      nvidiacg
      URL http://www.panda3d.org/download/noversion/nvidiacg-win64.zip
      URL_HASH SHA256=955d6e38bcba7a48e0efc8f2f0f7d5de39901788806e411744c63b55c83edfe2
      CONFIGURE_COMMAND ""
      BUILD_COMMAND ""
      INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/lib <INSTALL_DIR>/lib
              COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/bin <INSTALL_DIR>/bin
              COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/include <INSTALL_DIR>/include
      INSTALL_DIR ${THIRDPARTY_DIR}/nvidiacg
    )
  else()
    ExternalProject_Add(
      nvidiacg
      URL http://www.panda3d.org/download/noversion/nvidiacg-win32.zip
      URL_HASH SHA256=602ae301f3b68953a56a81ba89db0f53c0e10e8350faeb27842c5b827d9783a3
      CONFIGURE_COMMAND ""
      BUILD_COMMAND ""
      INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/lib <INSTALL_DIR>/lib
              COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/bin <INSTALL_DIR>/bin
              COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/include <INSTALL_DIR>/include
      INSTALL_DIR ${THIRDPARTY_DIR}/nvidiacg
    )
  endif()
elseif(BUILD_NVIDIACG AND CMAKE_SYSTEM_NAME STREQUAL "Darwin")
  ExternalProject_Add(
    nvidiacg
    URL http://www.panda3d.org/download/noversion/NVIDIA_Cg.tgz
    URL_HASH SHA256=cd6b12cca647749101b0286353095c7435251097d1aed0171cfc8afdcededd29

    CONFIGURE_COMMAND chmod -R u+w <SOURCE_DIR>
    BUILD_COMMAND install_name_tool -id @loader_path/../Frameworks/Cg.framework/Cg <SOURCE_DIR>/Library/Frameworks/Cg.framework/Cg
    INSTALL_COMMAND rm -rf "<INSTALL_DIR>/Cg.framework"
            COMMAND cp -R <SOURCE_DIR>/Library/Frameworks/Cg.framework <INSTALL_DIR>/Cg.framework

    INSTALL_DIR ${THIRDPARTY_DIR}/nvidiacg
  )
elseif(BUILD_NVIDIACG AND CMAKE_SYSTEM_NAME STREQUAL "Linux")
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    ExternalProject_Add(
      nvidiacg
      URL http://developer.download.nvidia.com/cg/Cg_3.1/Cg-3.1_April2012_x86_64.tgz
      URL_HASH SHA256=e8ff01e6cc38d1b3fd56a083f5860737dbd2f319a39037528fb1a74a89ae9878
      CONFIGURE_COMMAND ""
      BUILD_COMMAND ""
      INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/lib64 <INSTALL_DIR>/lib
              COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/include <INSTALL_DIR>/include
      INSTALL_DIR ${THIRDPARTY_DIR}/nvidiacg
    )
  else()
    ExternalProject_Add(
      nvidiacg
      URL http://developer.download.nvidia.com/cg/Cg_3.1/Cg-3.1_April2012_x86.tgz
      URL_HASH SHA256=cef3591e436f528852db0e8c145d3842f920e0c89bcfb219c466797cb7b18879
      CONFIGURE_COMMAND ""
      BUILD_COMMAND ""
      INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/lib <INSTALL_DIR>/lib
              COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/include <INSTALL_DIR>/include
      INSTALL_DIR ${THIRDPARTY_DIR}/nvidiacg
    )
  endif()
endif()

if(BUILD_OPENSSL AND MSVC)
  if(CMAKE_SIZEOF_VOID_P EQUAL 8)
    set(ssl_config VC-WIN64A)
  else()
    set(ssl_config VC-WIN32)
  endif()

  ExternalProject_Add(
    openssl
    URL https://www.openssl.org/source/openssl-1.0.2u.tar.gz
    URL_HASH SHA256=ecd0c6ffb493dd06707d38b14bb4d8c2288bb7033735606569d8f90f89669d16

    CONFIGURE_COMMAND perl Configure ${ssl_config} no-asm no-shared --prefix=<INSTALL_DIR>
              COMMAND ${CMAKE_CURRENT_LIST_DIR}/openssl-ms.bat ${ssl_config}
    BUILD_COMMAND nmake -f ms\\nt.mak
    BUILD_IN_SOURCE 1

    INSTALL_COMMAND nmake -f ms\\nt.mak install
    INSTALL_DIR ${THIRDPARTY_DIR}/openssl
  )

  ExternalProject_Add_Step(openssl remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/bin"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/ssl"
    DEPENDEES install
  )
elseif(BUILD_OPENSSL AND NOT APPLE)
  ExternalProject_Add(
    openssl
    URL https://www.openssl.org/source/openssl-1.0.2r.tar.gz
    URL_HASH SHA256=ae51d08bba8a83958e894946f15303ff894d75c2b8bbd44a852b64e3fe11d0d6

    CONFIGURE_COMMAND ./config no-asm no-shared --prefix=<INSTALL_DIR> ${CMAKE_C_FLAGS}
    BUILD_COMMAND ${MAKE}
    BUILD_IN_SOURCE 1

    INSTALL_DIR ${THIRDPARTY_DIR}/openssl
  )

  ExternalProject_Add_Step(openssl remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/bin"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/ssl"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/pkgconfig"
    DEPENDEES install
  )
elseif(BUILD_OPENSSL AND APPLE)
  set(ssl_flags ${CMAKE_C_FLAGS}
                no-asm no-shared no-dso
                -mmacosx-version-min=${CMAKE_OSX_DEPLOYMENT_TARGET}
                -Wno-nullability-completeness -Wno-expansion-to-defined)

  if(CMAKE_OSX_SYSROOT)
    set(ssl_flags ${ssl_flags} -isysroot${CMAKE_OSX_SYSROOT} -I/usr/include -Wl,-syslibroot,${CMAKE_OSX_SYSROOT})
  endif()

  ExternalProject_Add(
    openssl
    URL https://www.openssl.org/source/openssl-1.0.2r.tar.gz
    URL_HASH SHA256=ae51d08bba8a83958e894946f15303ff894d75c2b8bbd44a852b64e3fe11d0d6

    CONFIGURE_COMMAND /usr/bin/perl ./Configure darwin64-x86_64-cc --prefix=<INSTALL_DIR> --openssldir=<INSTALL_DIR> ${ssl_flags}
    BUILD_COMMAND ${MAKE}
    BUILD_IN_SOURCE 1

    INSTALL_DIR ${THIRDPARTY_DIR}/openssl
  )

  ExternalProject_Add_Step(openssl remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/bin"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/ssl"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/pkgconfig"
    DEPENDEES install
  )
endif()

if(BUILD_OPENEXR)
  ExternalProject_Add(
    openexr
    DEPENDS zlib
    GIT_REPOSITORY https://github.com/openexr/openexr.git
    GIT_TAG dcaf5fdb4d1244d8e60a58832cfe9c54734a2257 #v2.2.1
    URL https://github.com/openexr/openexr/archive/v2.2.1.tar.gz
    URL_HASH SHA256=63ebc7677d63ad59f1ec205d149dff1da31e7e3c8e78e944458b1d371c7e2f3c

    PATCH_COMMAND ${CMAKE_COMMAND} -E copy_if_different ${CMAKE_CURRENT_LIST_DIR}/openexr.cmake <SOURCE_DIR>/CMakeLists.txt

    CMAKE_ARGS ${COMMON_CMAKE_ARGS}
      -DCMAKE_PREFIX_PATH=${THIRDPARTY_DIR}/zlib
      -DNAMESPACE_VERSIONING=OFF
    INSTALL_DIR ${THIRDPARTY_DIR}/openexr
  )

  # Remove the extra bin and share directories it created.
  ExternalProject_Add_Step(openexr remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/bin"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/share"
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/pkgconfig"
    DEPENDEES install
  )
endif()

if(BUILD_FFMPEG AND WIN32)
  set(FFMPEG_VERSION 3.2.4)

  if(CMAKE_GENERATOR MATCHES "Win64$")
    set(MACHINE_FLAG "/machine:x64")
    set(FFMPEG_ARCH "win64")
    set(FFMPEG_DEV_HASH 3486b694cf92b1480c53609cc3695b5f0b53cbc7c1894170fe60575b707fd300)
    set(FFMPEG_SHARED_HASH 31dab0e45aa2808fc422927eca5d49761985d1471b7122aadc6f85deb7bf6a7f)
  else()
    set(MACHINE_FLAG "/machine:x86")
    set(FFMPEG_ARCH "win32")
    set(FFMPEG_DEV_HASH 30e47528f8d16b4365ccbe76dc8ccfa145930064e7a496495a3e805d225b5801)
    set(FFMPEG_SHARED_HASH c62a6e2237097cedbfd6b2e122f6650baf468d1bb4e9f73105126bf2a44b2329)
  endif()
  ExternalProject_Add(
    ffmpeg-dev
    URL http://ffmpeg.zeranoe.com/builds/${FFMPEG_ARCH}/dev/ffmpeg-${FFMPEG_VERSION}-${FFMPEG_ARCH}-dev.zip
    URL_HASH SHA256=${FFMPEG_DEV_HASH}
    CONFIGURE_COMMAND ${CMAKE_COMMAND} -E make_directory <SOURCE_DIR>/outlib
    BUILD_COMMAND lib.exe /def:lib\\avcodec-57.def /out:outlib\\avcodec.lib ${MACHINE_FLAG}
          COMMAND lib.exe /def:lib\\avdevice-57.def /out:outlib\\avdevice.lib ${MACHINE_FLAG}
          COMMAND lib.exe /def:lib\\avfilter-6.def /out:outlib\\avfilter.lib ${MACHINE_FLAG}
          COMMAND lib.exe /def:lib\\avformat-57.def /out:outlib\\avformat.lib ${MACHINE_FLAG}
          COMMAND lib.exe /def:lib\\avutil-55.def /out:outlib\\avutil.lib ${MACHINE_FLAG}
          COMMAND lib.exe /def:lib\\postproc-54.def /out:outlib\\postproc.lib ${MACHINE_FLAG}
          COMMAND lib.exe /def:lib\\swresample-2.def /out:outlib\\swresample.lib ${MACHINE_FLAG}
          COMMAND lib.exe /def:lib\\swscale-4.def /out:outlib\\swscale.lib ${MACHINE_FLAG}
    BUILD_IN_SOURCE 1
    INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/outlib <INSTALL_DIR>/lib
            COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/include <INSTALL_DIR>/include
            COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/licenses <INSTALL_DIR>/licenses
    INSTALL_DIR ${THIRDPARTY_DIR}/ffmpeg
  )
  ExternalProject_Add(
    ffmpeg-shared
    URL http://ffmpeg.zeranoe.com/builds/${FFMPEG_ARCH}/shared/ffmpeg-${FFMPEG_VERSION}-${FFMPEG_ARCH}-shared.zip
    URL_HASH SHA256=${FFMPEG_SHARED_HASH}
    CONFIGURE_COMMAND ""
    BUILD_COMMAND ""
    INSTALL_COMMAND ${CMAKE_COMMAND} -E copy_directory <SOURCE_DIR>/bin <INSTALL_DIR>/bin
    INSTALL_DIR ${THIRDPARTY_DIR}/ffmpeg
  )
  ExternalProject_Add_Step(ffmpeg-shared remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove "<INSTALL_DIR>/bin/ffmpeg.exe"
                                       "<INSTALL_DIR>/bin/ffplay.exe"
                                       "<INSTALL_DIR>/bin/ffprobe.exe"
    DEPENDEES install
  )

  if (CMAKE_GENERATOR MATCHES "^Visual Studio 10 ")
    # Copy over the custom inttypes.h file.
    ExternalProject_Add_Step(ffmpeg-dev install_inttypes
      COMMAND ${CMAKE_COMMAND} -E copy_if_different "${CMAKE_CURRENT_LIST_DIR}/inttypes.h" "<INSTALL_DIR>/include/inttypes.h"
      DEPENDEES install
    )
  endif()
elseif(BUILD_FFMPEG)
  EXTERNALPROJECT_ADD(
    ffmpeg
    DEPENDS ogg vorbis freetype zlib
    URL https://ffmpeg.org/releases/ffmpeg-3.2.4.tar.gz
    URL_HASH SHA256=2838c37bd35dba6e9b08ebe4e6c11dedbe2d9909583e3c2b9e7a1ad3ce2a9607

    CONFIGURE_COMMAND ./configure --prefix=<INSTALL_DIR> --datadir=${CMAKE_BINARY_DIR}/etc --disable-shared --enable-static --enable-pic --disable-doc --disable-debug --disable-programs --disable-outdevs --enable-runtime-cpudetect
    BUILD_COMMAND ${MAKE}
    BUILD_IN_SOURCE 1

    INSTALL_DIR ${THIRDPARTY_DIR}/ffmpeg
  )

  ExternalProject_Add_Step(ffmpeg remove_extras
    COMMAND ${CMAKE_COMMAND} -E remove_directory "<INSTALL_DIR>/lib/pkgconfig"
    DEPENDEES install
  )
endif()
