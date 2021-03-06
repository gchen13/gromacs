#
# This file is part of the GROMACS molecular simulation package.
#
# Copyright (c) 2014,2015, by the GROMACS development team, led by
# Mark Abraham, David van der Spoel, Berk Hess, and Erik Lindahl,
# and including many others, as listed in the AUTHORS file in the
# top-level source directory and at http://www.gromacs.org.
#
# GROMACS is free software; you can redistribute it and/or
# modify it under the terms of the GNU Lesser General Public License
# as published by the Free Software Foundation; either version 2.1
# of the License, or (at your option) any later version.
#
# GROMACS is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.  See the GNU
# Lesser General Public License for more details.
#
# You should have received a copy of the GNU Lesser General Public
# License along with GROMACS; if not, see
# http://www.gnu.org/licenses, or write to the Free Software Foundation,
# Inc., 51 Franklin Street, Fifth Floor, Boston, MA  02110-1301  USA.
#
# If you want to redistribute modifications to GROMACS, please
# consider that scientific software is very special. Version
# control is crucial - bugs must be traceable. We will be happy to
# consider code for inclusion in the official distribution, but
# derived work must not be called official GROMACS. Details are found
# in the README & COPYING files - if they are missing, get the
# official version at http://www.gromacs.org.
#
# To help us fund GROMACS development, we humbly ask that you cite
# the research papers on the package. Check out http://www.gromacs.org.

# This directory provides a unified place for building all kinds of
# GROMACS documentation. This includes some "static" content (Doxygen
# code documentation, reference manual, install guide, old online HTML
# images), and content generated from the gmx program for the various
# tools (man and HTML pages). It also provides the "webpage" target,
# that combines all of the above (except man pages in man format) into
# a form suitable for automated deployment to the GROMACS website. It
# also provides the INSTALL file for the tarball.
#
# The webpage is mostly built by Sphinx.  Variable values for Sphinx
# substitutions are configured by CMake (for things like version numbers),
# using gmx_configure_version_file().  This happens during build time instead
# of configure time, because 1) some of the version variables are only
# available during build time, and 2) we don't want to do all the Sphinx setup
# during configuration to save some time when not building the content.
# All the generated values get put into conf-vars.py (generated from
# conf-vars.py.cmakein), which in turn is included by the Sphinx configuration
# file conf.py.

option(GMX_BUILD_WEBPAGE "Whether to try to configure to build the GROMACS static webpages" OFF)
option(GMX_BUILD_MANUAL "Whether to try to configure to build the PDF manual" OFF)
mark_as_advanced(GMX_BUILD_WEBPAGE GMX_BUILD_MANUAL)

set(EXPECTED_DOXYGEN_VERSION 1.8.5)

find_package(PythonInterp)
find_package(Sphinx 1.2.3 COMPONENTS pygments)

# Even if we aren't going to make the full webpage, set up to put all
# the documentation output in the same place, for convenience
set(HTML_OUTPUT_DIR "${CMAKE_CURRENT_BINARY_DIR}/html")
file(MAKE_DIRECTORY ${HTML_OUTPUT_DIR})

# The directory from which man pages will be installed; if it remains
# empty, they will be silently skipped.
set(MAN_PAGE_DIR)
if (SOURCE_IS_SOURCE_DISTRIBUTION)
    # When building from the tarball, install the bundled man pages
    # (unless overridden).
    set(MAN_PAGE_DIR ${CMAKE_CURRENT_SOURCE_DIR})
endif()

add_subdirectory(doxygen)

set(SOURCE_MD5SUM "unknown" CACHE STRING "MD5 sum of the source tarball, normally used only for the pre-release webpage build")
# REGRESSIONTEST_MD5SUM is set in cmake/gmxVersionInfo.cmake because it is used also in tests/CMakeLists.txt
mark_as_advanced(SOURCE_MD5SUM)

if(GMX_BUILD_MANUAL)
    # Make sure we only do detection of manual-building dependencies
    # when the user opted in for that.
    add_subdirectory(manual)
endif()

if (SPHINX_FOUND)
    # We need to have all the Sphinx input files in a single directory, and
    # since some of them are generated, we copy everything into the build tree,
    # to this directory.
    set(SPHINX_INPUT_DIR ${CMAKE_CURRENT_BINARY_DIR}/sphinx-input)
    set(SPHINX_CONFIG_FILE ${CMAKE_CURRENT_SOURCE_DIR}/conf.py)
    set(SPHINX_CONFIG_VARS_FILE ${SPHINX_INPUT_DIR}/conf-vars.py)
    set(SPHINX_EXTENSION_PATH ${CMAKE_CURRENT_SOURCE_DIR})
    if (SOURCE_IS_SOURCE_DISTRIBUTION OR GMX_BUILD_TARBALL)
        # The real build of the webpage happens from the tarball, and
        # this should be set to the matching MD5 sum.
        set(REGRESSIONTEST_MD5SUM_STRING "${REGRESSIONTEST_MD5SUM}")
    else()
        # But for testing the webpage build (e.g. from the repo) we
        # need a default value.
        set(REGRESSIONTEST_MD5SUM_STRING "unknown")
    endif()
    set(SPHINX_SOURCE_FILES
        index.rst
        download.rst
        dev-manual/index.rst
        dev-manual/codelayout.rst
        dev-manual/doxygen.rst
        dev-manual/doxygen-links.rst
        dev-manual/formatting.rst
        dev-manual/gmxtree.rst
        dev-manual/includestyle.rst
        dev-manual/naming.rst
        dev-manual/overview.rst
        dev-manual/style.rst
        dev-manual/testutils.rst
        dev-manual/tools.rst
        dev-manual/uncrustify.rst
        install-guide/index.rst
        user-guide/index.rst
        user-guide/getting-started.rst
        user-guide/flow.rst
        user-guide/cutoff-schemes.rst
        user-guide/mdrun-features.rst
        user-guide/mdrun-performance.rst
        user-guide/mdp-options.rst
        user-guide/file-formats.rst
        user-guide/tools.rst
        user-guide/environment-variables.rst
        user-guide/plotje.gif
        user-guide/xvgr.gif
        conf.py
        links.dat
        )

    # Remove other rst files from the build tree, since they confuse Sphinx.
    file(GLOB_RECURSE _obsolete_sources RELATIVE ${SPHINX_INPUT_DIR}
         ${SPHINX_INPUT_DIR}/*.rst)
    list(REMOVE_ITEM _obsolete_sources ${SPHINX_SOURCE_FILES})
    foreach(_file ${_obsolete_sources})
        # Skip generated files.
        if (NOT _file MATCHES "^(programs/.*|man/bytopic)\\.rst$")
            message(STATUS "Removing obsolete Sphinx input ${_file}")
            file(REMOVE ${SPHINX_INPUT_DIR}/${_file})
        endif()
    endforeach()

    gmx_configure_version_file(conf-vars.py.cmakein ${SPHINX_CONFIG_VARS_FILE}
        EXTRA_VARS
            SPHINX_EXTENSION_PATH
            EXPECTED_DOXYGEN_VERSION
            GMX_CMAKE_MINIMUM_REQUIRED_VERSION REQUIRED_CUDA_VERSION
            REQUIRED_CUDA_COMPUTE_CAPABILITY REGRESSIONTEST_VERSION
            SOURCE_MD5SUM REGRESSIONTEST_MD5SUM_STRING
        COMMENT "Configuring Sphinx configuration file")
    set(SPHINX_INPUT_FILES ${SPHINX_CONFIG_VARS_FILE})
    foreach(_file ${SPHINX_SOURCE_FILES})
        add_custom_command(
            OUTPUT ${SPHINX_INPUT_DIR}/${_file}
            COMMAND ${CMAKE_COMMAND} -E copy
                ${CMAKE_CURRENT_SOURCE_DIR}/${_file}
                ${SPHINX_INPUT_DIR}/${_file}
            DEPENDS
                ${CMAKE_CURRENT_SOURCE_DIR}/${_file}
            COMMENT "Copying Sphinx input file ${_file}"
            VERBATIM)
        list(APPEND SPHINX_INPUT_FILES ${SPHINX_INPUT_DIR}/${_file})
    endforeach()
    gmx_add_custom_output_target(sphinx-input OUTPUT STAMP
        DEPENDS ${SPHINX_INPUT_FILES})
    # TODO: Make this remove obsolete .rst files.
    gmx_add_custom_output_target(sphinx-programs OUTPUT STAMP
        COMMAND ${CMAKE_COMMAND} -E make_directory programs
        COMMAND ${CMAKE_COMMAND} -E make_directory man
        COMMAND gmx -quiet help -export rst
        DEPENDS gmx
        WORKING_DIRECTORY ${SPHINX_INPUT_DIR}
        COMMENT "Generating reStructuredText help")
    # This dependency ensures that the directories exist before the
    # executable tries to write things there.
    add_dependencies(sphinx-programs sphinx-input)

    # Make the INSTALL file for CPack for the tarball. This gets put
    # into the tarball via the CPack rules below, which requires that
    # the INSTALL file is in a separate directory by itself.
    set(TEXT_INSTALL_GUIDE_OUTPUT_DIR "install-guide/text")
    add_custom_target(install-guide
        COMMAND
            ${SPHINX_EXECUTABLE}
            -q -b text
            -w sphinx-install.log
            -d ${CMAKE_CURRENT_BINARY_DIR}/install-guide/_doctrees
            -c ${SPHINX_INPUT_DIR}
            "${SPHINX_INPUT_DIR}/install-guide"
            "${TEXT_INSTALL_GUIDE_OUTPUT_DIR}"
        COMMAND
            ${CMAKE_COMMAND} -E rename
            ${TEXT_INSTALL_GUIDE_OUTPUT_DIR}/index.txt
            ${TEXT_INSTALL_GUIDE_OUTPUT_DIR}/INSTALL
        WORKING_DIRECTORY
            ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Building INSTALL with Sphinx"
        VERBATIM
        )
    add_dependencies(install-guide sphinx-input)
    gmx_cpack_add_generated_source_directory(install-guide/text DESTINATION /)

    # Sphinx cache with pickled ReST documents
    set(SPHINX_CACHE_DIR "${CMAKE_CURRENT_BINARY_DIR}/_doctrees")

    add_custom_target(webpage-sphinx
        COMMAND
            ${CMAKE_COMMAND} -E make_directory ${SPHINX_INPUT_DIR}/_static
        COMMAND
            ${SPHINX_EXECUTABLE}
            -q -b html
            -w sphinx-html.log
            -d "${SPHINX_CACHE_DIR}"
            "${SPHINX_INPUT_DIR}"
            "${HTML_OUTPUT_DIR}"
        WORKING_DIRECTORY
            ${CMAKE_CURRENT_BINARY_DIR}
        COMMENT "Building HTML documentation with Sphinx"
        VERBATIM
        )
    add_dependencies(webpage-sphinx sphinx-input sphinx-programs)

    add_custom_target(man
        COMMAND
            ${SPHINX_EXECUTABLE}
            -q -b man
            -w sphinx-man.log
            -d ${SPHINX_CACHE_DIR}
            -t do_man
            ${SPHINX_INPUT_DIR}
            ${CMAKE_CURRENT_BINARY_DIR}/man
        COMMENT "Building man pages with Sphinx"
        VERBATIM)
    add_dependencies(man sphinx-input sphinx-programs)
    if (GMX_BUILD_HELP)
        # If requested, install the man pages built by the 'man' target
        # created above.  Nothing will be installed if the user did not
        # manually build the target.
        set(MAN_PAGE_DIR ${CMAKE_CURRENT_BINARY_DIR})
    endif()
endif()

if (MAN_PAGE_DIR)
    set(MAN_PAGE_DIR ${MAN_PAGE_DIR}/man)
    # Trailing slash on directory is significant for
    # install(DIRECTORY). See CMake docs.
    install(DIRECTORY ${MAN_PAGE_DIR}/
        DESTINATION ${MAN_INSTALL_DIR}/man1
        COMPONENT man OPTIONAL
        FILES_MATCHING PATTERN "*.1")
    install(DIRECTORY ${MAN_PAGE_DIR}/
        DESTINATION ${MAN_INSTALL_DIR}/man7
        COMPONENT man OPTIONAL
        FILES_MATCHING PATTERN "*.7")
endif()
gmx_cpack_add_generated_source_directory(man)

set(HTML_BUILD_IS_POSSIBLE OFF)
# We can only configure to build the webpage if the user asked for it,
# the build is outside of the source dir, and all the components can
# be built. There's no intrinsic need to be talkative if we fail -
# most people never need to know, unless they've asked for the webpage
# build.
if(GMX_BUILD_WEBPAGE)
    set(HTML_BUILD_IS_POSSIBLE ON)
    # Next, turn it off in any of the preconditions are unsatisified

    if(NOT MANUAL_BUILD_IS_POSSIBLE)
        set(HTML_BUILD_IS_POSSIBLE OFF)
        message(STATUS "Cannot build webpage without being able to build the reference PDF manual")
    endif()

    if(NOT PYTHON_EXECUTABLE)
        set(HTML_BUILD_IS_POSSIBLE OFF)
        message(STATUS "Cannot build webpage without python")
    endif()

    if(NOT SPHINX_FOUND)
        set(HTML_BUILD_IS_POSSIBLE OFF)
        message(STATUS "Cannot build webpage without sphinx")
    endif()

    if(NOT DOXYGEN_EXECUTABLE)
        set(HTML_BUILD_IS_POSSIBLE OFF)
        message(STATUS "Cannot build webpage without doxygen")
    endif()

    if(NOT DOXYGEN_MSCGEN_EXECUTABLE)
        set(HTML_BUILD_IS_POSSIBLE OFF)
        message(STATUS "Cannot build webpage without mscgen")
    endif()
endif()

if(HTML_BUILD_IS_POSSIBLE)
    # Make the PDF reference guide
    # TODO Try to make the PDF arrive directly in ${HTML_OUTPUT_DIR}
    add_custom_command(
        OUTPUT ${HTML_OUTPUT_DIR}/manual-${GMX_VERSION_STRING}.pdf
        COMMAND ${CMAKE_COMMAND}
            -E remove -f ${HTML_OUTPUT_DIR}/manual-${GMX_VERSION_STRING}.pdf
        COMMAND ${CMAKE_COMMAND}
            -E copy ${CMAKE_CURRENT_BINARY_DIR}/manual/gromacs.pdf ${HTML_OUTPUT_DIR}/manual-${GMX_VERSION_STRING}.pdf
        # UseLATEX.cmake makes a target called pdf, not ${CMAKE_CURRENT_BINARY_DIR}/manual/gromacs.pdf
        DEPENDS pdf
        VERBATIM
        )

    # The Doxygen configuration in doxygen/Doxyfile-common.cmakein
    # makes all the Doxygen output directly in
    # ${HTML_OUTPUT_DIR}/doxygen (and makes the directory if it needs
    # to).

    # Add a top-level target for the others to hook onto
    add_custom_target(webpage
        DEPENDS ${HTML_OUTPUT_DIR}/manual-${GMX_VERSION_STRING}.pdf)
    add_dependencies(webpage webpage-sphinx doxygen-all)
else()
    add_custom_target(webpage
        COMMAND ${CMAKE_COMMAND} -E echo "Cannot build webpage because of missing requirements. Check cmake status output for reasons"
        COMMENT "Webpage build disabled"
        )
endif()
