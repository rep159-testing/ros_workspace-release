cmake_minimum_required(VERSION 3.5)
project(ros_workspace NONE)

find_package(ament_cmake_core REQUIRED)

# REP 159: the install prefixes of the distributions this one extends with
# binary_import, nearest first. bloom passes them when releasing into such a
# distribution; a root distribution leaves the list empty.
set(ROS_WORKSPACE_PARENT_PREFIX_PATH "" CACHE STRING
  "Install prefixes of the binary_import parent distributions, nearest first")

# Collect current python 3 version
execute_process(COMMAND python3 -c "from distutils import sysconfig; print(sysconfig.get_python_version())" OUTPUT_VARIABLE PYTHON_MAJOR_MINOR OUTPUT_STRIP_TRAILING_WHITESPACE)

# Locate ament_package template files.
if(WIN32)
  set(PYTHON_INSTALL_DIR "lib/site-packages")
  set(SHELL_EXT "bat")
else()
  set(PYTHON_INSTALL_DIR "lib/python${PYTHON_MAJOR_MINOR}/site-packages")
  set(SHELL_EXT "sh")
endif()
# Look in this prefix first, then, for a binary_import child, in the
# prefixes of its parents.
set(AMENT_PACKAGE_DIR "")
set(_searched "")
foreach(_prefix "${CMAKE_INSTALL_PREFIX}" ${ROS_WORKSPACE_PARENT_PREFIX_PATH})
  set(_dir "${_prefix}/${PYTHON_INSTALL_DIR}/ament_package")
  list(APPEND _searched "${_dir}")
  if(EXISTS "${_dir}")
    set(AMENT_PACKAGE_DIR "${_dir}")
    break()
  endif()
  # Check for an .egg-link file and use the listed directory if it exists
  get_filename_component(_egg_link "${_dir}" DIRECTORY)
  set(_egg_link "${_egg_link}/ament-package.egg-link")
  list(APPEND _searched "${_egg_link}")
  if(EXISTS "${_egg_link}")
    file(STRINGS "${_egg_link}" _egg_dir)
    list(GET _egg_dir 0 _egg_dir)
    set(AMENT_PACKAGE_DIR "${_egg_dir}/ament_package")
    if(NOT EXISTS "${AMENT_PACKAGE_DIR}")
      message(FATAL_ERROR "Could not find '${AMENT_PACKAGE_DIR}' listed in '${_egg_link}'")
    endif()
    break()
  endif()
endforeach()
if(NOT AMENT_PACKAGE_DIR)
  string(REPLACE ";" "', '" _searched "${_searched}")
  message(FATAL_ERROR "Could not find ament_package in any of: '${_searched}'")
endif()
set(AMENT_PACKAGE_TEMPLATE_DIR "${AMENT_PACKAGE_DIR}/template")
set(PYTHONPATH_HOOK "${AMENT_PACKAGE_TEMPLATE_DIR}/environment_hook/pythonpath.${SHELL_EXT}.in")

# register information for .dsv generation
set(
  AMENT_CMAKE_ENVIRONMENT_HOOKS_DESC_pythonpath
  "prepend-non-duplicate;PYTHONPATH;${PYTHON_INSTALL_DIR}")

# Set environment hooks for default environment.
if(WIN32)
  ament_environment_hooks("${PYTHONPATH_HOOK}")
else()
  set(LIBRARY_PATH_HOOK "${AMENT_PACKAGE_TEMPLATE_DIR}/environment_hook/library_path.${SHELL_EXT}")
  # enable C language so that a trycompile can determine what the
  # anticipated libdir will be.
  enable_language(C)
  include(GNUInstallDirs)
  if(NOT ${CMAKE_INSTALL_LIBDIR} STREQUAL "lib")
    # register multiarch information for .dsv generation
    if(APPLE)
      set(LIBRARY_PATH_ENV_VAR "DYLD_LIBRARY_PATH")
    else()
      set(LIBRARY_PATH_ENV_VAR "LD_LIBRARY_PATH")
    endif()
    set(
      AMENT_CMAKE_ENVIRONMENT_HOOKS_DESC_multiarch_library_paths
      "prepend-non-duplicate;${LIBRARY_PATH_ENV_VAR};${CMAKE_INSTALL_LIBDIR}")
    set(MULTIARCH_LIBRARY_PATH_HOOK "env-hooks/multiarch_library_paths.sh.in")
  endif()
  ament_environment_hooks("${LIBRARY_PATH_HOOK}" "${PYTHONPATH_HOOK}" ${MULTIARCH_LIBRARY_PATH_HOOK})
endif()

if(ROS_WORKSPACE_PARENT_PREFIX_PATH)
  # REP 159: record the parents as this package's parent_prefix_path
  # resource, which the prefix-level setup files read to source each
  # parent's local_setup first. ament_cmake's package hook writes
  # $ENV{AMENT_PREFIX_PATH} into that resource when ament_package() runs,
  # so set it to exactly the parents, whatever the build environment holds.
  string(REPLACE ";" ":" _parent_prefix_path "${ROS_WORKSPACE_PARENT_PREFIX_PATH}")
  set(ENV{AMENT_PREFIX_PATH} "${_parent_prefix_path}")
  set(SKIP_PARENT_PREFIX_PATH "")
else()
  # skip using ament_index/resource_index/parent_prefix_path
  # if for Debian packages it is known that there are no underlays
  # note the template only checks for an empty (use) / non-empty (skip) string
  set(SKIP_PARENT_PREFIX_PATH "SKIP_PARENT_PREFIX_PATH")
endif()

ament_package()
ament_generate_environment()
