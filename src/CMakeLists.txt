# Copyright (C) 2022 The Qt Company Ltd.
# SPDX-License-Identifier: BSD-3-Clause

qt_internal_get_filename_path_mode(path_mode)
get_filename_component(WEBENGINE_ROOT_SOURCE_DIR "${CMAKE_CURRENT_SOURCE_DIR}/.." ${path_mode})
get_filename_component(WEBENGINE_ROOT_BUILD_DIR "${PROJECT_BINARY_DIR}" ${path_mode})


##
#  RUN CONFIGURE
##

qt_webengine_configure_begin()
qt_webengine_run_configure(../configure.cmake)

if(NOT QT_FEATURE_qtwebengine_build AND NOT QT_FEATURE_qtpdf_build)
    qt_webengine_configure_end()
    return()
endif()

##
#  MODULES
##

if(QT_FEATURE_qtwebengine_build)
    # Use implicit PROJECT_NAME.
    qt_internal_sbom_begin_qt_repo_project()
    qt_webengine_sbom_project_begin(WebEngine WebEngineCore)
endif()
if(QT_FEATURE_qtwebengine_core_build)
    add_subdirectory(core)
    add_subdirectory(process)
    qt_webengine_generate_documentation(
        core/doc/src/qtwebengine-platform-notes.qdoc.in
        core/api/qtwebengine-platform-notes.qdoc
    )
endif()
if(QT_FEATURE_qtwebengine_widgets_build)
    add_subdirectory(webenginewidgets)
endif()
if(QT_FEATURE_qtwebengine_quick_build)
    add_subdirectory(webenginequick)
endif()
if(QT_FEATURE_qtwebengine_build)
    qt_webengine_sbom_project_end()
    qt_internal_sbom_end_qt_repo_project()
endif()

if(QT_FEATURE_qtpdf_build)
    qt_internal_sbom_begin_qt_repo_project(SBOM_PROJECT_NAME QtPdf)
    qt_webengine_sbom_project_begin(Pdf Pdf)
    add_subdirectory(pdf)
endif()
if(QT_FEATURE_qtpdf_widgets_build)
    add_subdirectory(pdfwidgets)
endif()
if(QT_FEATURE_qtpdf_quick_build)
    add_subdirectory(pdfquick)
endif()

# Keep log order, pdf build after webengine

if(QT_FEATURE_qtpdf_build)
    if(QT_FEATURE_qtwebengine_core_build)
        add_dependencies(run_pdf_NinjaReady WebEngineCore)
    endif()
    if(QT_FEATURE_qtwebengine_widgets_build)
        add_dependencies(run_pdf_NinjaReady WebEngineWidgets)
    endif()
    if(QT_FEATURE_qtwebengine_quick_build)
        add_dependencies(run_pdf_NinjaReady WebEngineQuick)
    endif()
endif()

# No SBOM information will be generated for any targets created past this point in the file
# or any add_subdirectory calls. For the external project GN, we'll have to come up with some
# additional API.
if(QT_FEATURE_qtpdf_build)
    qt_webengine_sbom_project_end()
    qt_internal_sbom_end_qt_repo_project()
endif()

set(installDir ${PROJECT_BINARY_DIR}/install)

##
#  GN PROJECT
##

if(CMAKE_CROSSCOMPILING AND NOT Gn_FOUND)
   message(FATAL_ERROR "No gn found for cross-compilation")
endif()

if(NOT Gn_FOUND)
    qt_webengine_externalproject_add(gn
        SOURCE_DIR  ${CMAKE_CURRENT_LIST_DIR}/gn
        BINARY_DIR  ${CMAKE_CURRENT_BINARY_DIR}/gn
        INSTALL_DIR ${installDir}
        EXCLUDE_FROM_ALL TRUE
    )
    if(QT_FEATURE_qtwebengine_core_build)
        add_dependencies(run_core_GnReady gn)
    endif()
    if(QT_FEATURE_qtpdf_build)
        add_dependencies(run_pdf_GnReady gn)
    endif()
endif()

string(REGEX REPLACE "(.)" "\\\\\\1" path_to_match "${installDir}")
if(NOT Gn_FOUND OR Gn_EXECUTABLE MATCHES "^${path_to_match}")
    set(INSTALL_GN 1 CACHE INTERNAL "")
endif()

##
#  HOST PROJECT
##

if(CMAKE_CROSSCOMPILING AND NOT IOS AND NOT MACOS AND NOT WIN32)

    if(NOT Gn_FOUND)
        message(FATAL_ERROR "\nHost gn not found - cross compilation not possible")
    endif()

    externalproject_add(hostBuild
        SOURCE_DIR  ${CMAKE_CURRENT_LIST_DIR}/host
        BINARY_DIR  ${CMAKE_CURRENT_BINARY_DIR}/host
        PREFIX      host
        USES_TERMINAL_BUILD TRUE
        EXCLUDE_FROM_ALL TRUE
        CMAKE_ARGS -DCMAKE_TOOLCHAIN_FILE=${QT_HOST_PATH_CMAKE_DIR}/Qt6/qt.toolchain.cmake
                   -DQT_USE_ORIGINAL_COMPILER=ON
                   -DWEBENGINE_ROOT_BUILD_DIR=${PROJECT_BINARY_DIR}
                   -DWEBENGINE_ROOT_SOURCE_DIR=${WEBENGINE_ROOT_SOURCE_DIR}
                   -DGN_TARGET_CPU=${TEST_architecture_arch}
                   -DCMAKE_C_FLAGS=
                   -DCMAKE_CXX_FLAGS=
                   -DQT_FEATURE_qtwebengine_build=${QT_FEATURE_qtwebengine_build}
                   -DQT_FEATURE_qtpdf_build=${QT_FEATURE_qtpdf_build}
                   -DGCC_LEGACY_SUPPORT=${QT_FEATURE_webengine_gcc_legacy_support}
    )
    if(QT_FEATURE_qtwebengine_build)
        add_dependencies(run_core_GnReady hostBuild)
    endif()
    if(QT_FEATURE_qtpdf_build)
        add_dependencies(run_pdf_GnReady hostBuild)
    endif()

    # PATCHED: Fix hostBuild-generated BUILD.gn files to use native compilers
    # The hostBuild ExternalProject generates these files with cross-compilers,
    # but host tools need to be compiled with native compilers to run on the build machine.

    # Detect host architecture for GN
    execute_process(COMMAND uname -m OUTPUT_VARIABLE HOST_UNAME_ARCH OUTPUT_STRIP_TRAILING_WHITESPACE)
    if(HOST_UNAME_ARCH STREQUAL "x86_64")
        set(FIX_HOST_GN_ARCH "x64")
    elseif(HOST_UNAME_ARCH STREQUAL "aarch64")
        set(FIX_HOST_GN_ARCH "arm64")
    else()
        set(FIX_HOST_GN_ARCH "${HOST_UNAME_ARCH}")
    endif()

    # Find native compilers
    find_program(FIX_NATIVE_CC NAMES gcc clang cc PATHS /usr/bin /usr/local/bin NO_CMAKE_FIND_ROOT_PATH)
    find_program(FIX_NATIVE_CXX NAMES g++ clang++ c++ PATHS /usr/bin /usr/local/bin NO_CMAKE_FIND_ROOT_PATH)
    find_program(FIX_NATIVE_AR NAMES ar PATHS /usr/bin /usr/local/bin NO_CMAKE_FIND_ROOT_PATH)
    find_program(FIX_NATIVE_NM NAMES nm PATHS /usr/bin /usr/local/bin NO_CMAKE_FIND_ROOT_PATH)

    message(STATUS "Will fix host toolchain after hostBuild: CC=${FIX_NATIVE_CC} CXX=${FIX_NATIVE_CXX} ARCH=${FIX_HOST_GN_ARCH}")

    # Create a script that fixes the BUILD.gn files
    set(FIX_SCRIPT "${CMAKE_CURRENT_BINARY_DIR}/fix_host_toolchain.cmake")
    file(WRITE "${FIX_SCRIPT}" "
# Fix host_toolchain BUILD.gn for core
set(HOST_TOOLCHAIN_FILE \"${PROJECT_BINARY_DIR}/src/core/host_toolchain/BUILD.gn\")
if(EXISTS \"\${HOST_TOOLCHAIN_FILE}\")
    file(WRITE \"\${HOST_TOOLCHAIN_FILE}\"
\"import(\\\"//build/config/sysroot.gni\\\")
import(\\\"//build/toolchain/gcc_toolchain.gni\\\")
gcc_toolchain(\\\"host\\\") {
  cc = \\\"${FIX_NATIVE_CC}\\\"
  cxx = \\\"${FIX_NATIVE_CXX}\\\"
  ld = \\\"${FIX_NATIVE_CXX}\\\"
  ar = \\\"${FIX_NATIVE_AR}\\\"
  nm = \\\"${FIX_NATIVE_NM}\\\"
  extra_cppflags = \\\"\\\"
  toolchain_args = {
    current_os = \\\"linux\\\"
    current_cpu = \\\"${FIX_HOST_GN_ARCH}\\\"
    v8_current_cpu = \\\"${FIX_HOST_GN_ARCH}\\\"
    is_clang = false
    is_mingw = false
    use_gold = false
  }
}
\")
    message(STATUS \"Fixed host_toolchain BUILD.gn with native compilers\")
endif()

# Fix v8_toolchain BUILD.gn for core
set(V8_TOOLCHAIN_FILE \"${PROJECT_BINARY_DIR}/src/core/v8_toolchain/BUILD.gn\")
if(EXISTS \"\${V8_TOOLCHAIN_FILE}\")
    file(WRITE \"\${V8_TOOLCHAIN_FILE}\"
\"import(\\\"//build/config/sysroot.gni\\\")
import(\\\"//build/toolchain/gcc_toolchain.gni\\\")
gcc_toolchain(\\\"v8\\\") {
  cc = \\\"${FIX_NATIVE_CC}\\\"
  cxx = \\\"${FIX_NATIVE_CXX}\\\"
  ld = \\\"${FIX_NATIVE_CXX}\\\"
  ar = \\\"${FIX_NATIVE_AR}\\\"
  nm = \\\"${FIX_NATIVE_NM}\\\"
  extra_cppflags = \\\"\\\"
  toolchain_args = {
    current_os = \\\"linux\\\"
    current_cpu = \\\"${FIX_HOST_GN_ARCH}\\\"
    v8_current_cpu = \\\"${FIX_HOST_GN_ARCH}\\\"
    is_clang = false
    is_mingw = false
    use_gold = false
  }
}
\")
    message(STATUS \"Fixed v8_toolchain BUILD.gn with native compilers\")
endif()
")

    # Add custom target that depends on hostBuild and runs before GN
    add_custom_target(fix_host_toolchain
        COMMAND ${CMAKE_COMMAND} -P "${FIX_SCRIPT}"
        COMMENT "Fixing host toolchain BUILD.gn files with native compilers"
    )
    add_dependencies(fix_host_toolchain hostBuild)
    if(QT_FEATURE_qtwebengine_build)
        add_dependencies(run_core_GnReady fix_host_toolchain)
    endif()
    # End PATCHED block
endif()

# install gn for cross build
if((LINUX OR MACOS OR WIN32) AND INSTALL_GN)
    if(NOT QT_WILL_INSTALL)
        set(copyOutput
            ${QT_BUILD_DIR}/${INSTALL_LIBEXECDIR}/gn${CMAKE_EXECUTABLE_SUFFIX}
        )
        if(Gn_FOUND)
            set(copyInput ${Gn_EXECUTABLE})
            set(copyDep ${Gn_EXECUTABLE})
        else()
            set(copyInput ${installDir}/bin/gn${CMAKE_EXECUTABLE_SUFFIX})
            set(copyDep gn)
        endif()
        add_custom_target(copy-gn ALL DEPENDS ${copyOutput})
        add_custom_command(
            OUTPUT ${copyOutput}
            COMMAND ${CMAKE_COMMAND} -E copy ${copyInput} ${QT_BUILD_DIR}/${INSTALL_LIBEXECDIR}
            DEPENDS ${copyDep}
            USES_TERMINAL
        )
    else()
        get_install_config(installConfig)
        install(
            PROGRAMS "${installDir}/bin/gn${CMAKE_EXECUTABLE_SUFFIX}"
            CONFIGURATIONS ${installConfig}
            RUNTIME DESTINATION "${INSTALL_LIBEXECDIR}"
        )
    endif()
endif()

qt_webengine_configure_end()
