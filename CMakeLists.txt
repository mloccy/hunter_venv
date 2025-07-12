# Python_FIND_REGISTRY and CMAKE_FIND_FRAMEWORK should be set to NEVER:
# - https://cmake.org/cmake/help/v3.13/module/FindPython.html#hints
# From 3.15 use Python_FIND_VIRTUALENV = ONLY as it has fewer side effects
# - https://cmake.org/cmake/help/v3.15/module/FindPython.html#hints
cmake_minimum_required(VERSION 3.13)
include (cmake/HunterGate.cmake)
HunterGate(
    URL "https://github.com/mloccy/hunter/archive/refs/tags/v0.25.40.tar.gz"
    SHA1 "27cc417a830340d7da4c38d5ba0792a6bfc0ad7d"
    LOCAL)

project(hunter_venv)
hunter_add_package(python)

if (NOT HUNTER_NOT_CI)

  if ("${HUNTER_VENV_PYTHON_VERSION}" STREQUAL "")
    message(FATAL_ERROR "HUNTER_VENV_PYTHON_VERSION is empty")
  else()
    set(DESIRED_PY_VER ${HUNTER_VENV_PYTHON_VERSION})
  endif()
else()
  find_package(Python)

  if (NOT Python_FOUND)
    hunter_add_package(python)
    find_package(Python REQUIRED)
  endif()
  set(DESIRED_PY_VER ${Python_VERSION})
endif()


set(HUNTER_VENV_USE_2 FALSE)
set(HUNTER_VENV_USE_3 FALSE)

if("${DESIRED_PY_VER}" STREQUAL "2")
  set(HUNTER_VENV_USE_2 TRUE)
endif()

if("${DESIRED_PY_VER}" MATCHES "^2\\.")
  set(HUNTER_VENV_USE_2 TRUE)
endif()

if("${DESIRED_PY_VER}" STREQUAL "3")
  set(HUNTER_VENV_USE_3 TRUE)
endif()

if("${DESIRED_PY_VER}" MATCHES "^3\\.")
  set(HUNTER_VENV_USE_3 TRUE)
endif()

if(HUNTER_VENV_USE_2 AND HUNTER_VENV_USE_3)
  message(FATAL_ERROR "Internal error")
endif()

if(NOT HUNTER_VENV_USE_2 AND NOT HUNTER_VENV_USE_3)
  message(
      FATAL_ERROR
      "Incorrect version: ${DESIRED_PY_VER}\n"
      "Only 2.* and 3.* supported."
  )
endif()

if (NOT HUNTER_NOT_CI)
  if (NOT Python_FOUND)
    hunter_add_package(python)
    find_package(Python REQUIRED)
  endif()

  if (NOT Python_VERSION STREQUAL "${DESIRED_PY_VER}")
    message(FATAL_ERROR "Python version: ${DESIRED_PY_VER} was requested but not found,\
     if this is not needed consider setting HUNTER_NOT_CI=On\
      in your hunter cmake args for this package.")

  endif()
endif()

message("Python_EXECUTABLE: ${Python_EXECUTABLE}")
message("Python_SITELIB: ${Python_SITELIB}")

message("Python_VERSION: ${Python_VERSION}")
message("Python_VERSION_MAJOR: ${Python_VERSION_MAJOR}")
message("Python_VERSION_MINOR: ${Python_VERSION_MINOR}")
message("Python_VERSION_PATCH: ${Python_VERSION_PATCH}")

if(HUNTER_VENV_USE_2)
  if(NOT "${Python_VERSION_MAJOR}" STREQUAL "2")
    message(FATAL_ERROR "Internal error")
  endif()
  if (HUNTER_NOT_CI)
    find_package(Python2 ${HUNTER_VENV_PYTHON_VERSION} REQUIRED)
  else()
    find_package(Python2 ${HUNTER_VENV_PYTHON_VERSION} EXACT REQUIRED)
  endif()
  message("Python2_EXECUTABLE: ${Python2_EXECUTABLE}")
  message("Python2_SITELIB: ${Python2_SITELIB}")
endif()

if(HUNTER_VENV_USE_3)
  if(NOT "${Python_VERSION_MAJOR}" STREQUAL "3")
    message(FATAL_ERROR "Internal error")
  endif()
  if (HUNTER_NOT_CI)
    find_package(Python3 ${HUNTER_VENV_PYTHON_VERSION} REQUIRED)
  else()
    find_package(Python3 ${HUNTER_VENV_PYTHON_VERSION} EXACT REQUIRED)
  endif()
  message("Python3_EXECUTABLE: ${Python3_EXECUTABLE}")
  message("Python3_SITELIB: ${Python3_SITELIB}")
endif()

set(venv_dir "${CMAKE_CURRENT_BINARY_DIR}/venv")

set(
    cmd
    ${Python_EXECUTABLE}
    -m
    ensurepip
)

set(cmd_human)
foreach(x ${cmd})
  set(cmd_human "${cmd_human} \"${x}\"")
endforeach()

message("Running command: ${cmd_human}")

execute_process(COMMAND ${cmd} RESULT_VARIABLE result)

if(NOT result EQUAL "0")
  message(FATAL_ERROR "Failed: ${result}")
endif()

set(
    cmd
    ${Python_EXECUTABLE}
    -m
    pip
    install
    virtualenv
)

set(cmd_human)
foreach(x ${cmd})
  set(cmd_human "${cmd_human} \"${x}\"")
endforeach()

message("Running command: ${cmd_human}")

execute_process(COMMAND ${cmd} RESULT_VARIABLE result)

if(NOT result EQUAL "0")
  message(FATAL_ERROR "Failed: ${result}")
endif()

set(
    cmd
    ${Python_EXECUTABLE}
    -m
    virtualenv
    --python=${Python_EXECUTABLE}
    --always-copy # Always copy files rather than symlinking
    ${venv_dir}
)

set(cmd_human)
foreach(x ${cmd})
  set(cmd_human "${cmd_human} \"${x}\"")
endforeach()

message("Running command: ${cmd_human}")

execute_process(COMMAND ${cmd} RESULT_VARIABLE result)

if(NOT result EQUAL "0")
  message(FATAL_ERROR "Failed: ${result}")
endif()

# `virtualenv --relocatable` will not help, still not relocatable
set(
    unrelocatable_list
    activate
    activate.bat
    activate.csh
    activate.fish
    activate.ps1
    activate.xsh
    deactivate.bat
)

foreach(x ${unrelocatable_list})
  file(REMOVE "${venv_dir}/bin/${x}")
  file(REMOVE "${venv_dir}/Scripts/${x}")
endforeach()

# `virtualenv --relocatable` will fail if path to the script is too long.
# Create some files manually. Taken from results of running conda, e.g. for pip:
#   > conda create --name a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789
#   > conda activate a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789a0123456789b0123456789c0123456789
#   > cat `which pip`

function(create_tool path from)
  set(dst "${venv_dir}/bin/${path}")

  if(NOT EXISTS "${dst}")
    return()
  endif()

  set(temp "${dst}.TEMP")

  # Use:
  # - from
  configure_file("cmake/template/tool.cmake.in" "${temp}" @ONLY)

  # Trick to keep file permissions
  file(READ "${temp}" dst_content)
  file(WRITE "${dst}" "${dst_content}")
  file(REMOVE "${temp}")
endfunction()

create_tool(easy_install "setuptools.command.easy_install")
create_tool(easy_install-${Python_VERSION_MAJOR}.${Python_VERSION_MINOR} "setuptools.command.easy_install")
create_tool(pip "pip._internal")
create_tool(pip${Python_VERSION_MAJOR} "pip._internal")
create_tool(pip${Python_VERSION_MAJOR}.${Python_VERSION_MINOR} "pip._internal")
create_tool(wheel "wheel.cli")

# Make an EXISTING virtualenv environment relocatable.
# This fixes up scripts and makes all .pth files
# relative.

set(cmd ${Python_EXECUTABLE} -m virtualenv --version)

set(cmd_human)
foreach(x ${cmd})
  set(cmd_human "${cmd_human} \"${x}\"")
endforeach()

message("Running command: ${cmd_human}")
execute_process(COMMAND ${cmd} RESULT_VARIABLE result OUTPUT_VARIABLE VIRTUALENV_VERSION OUTPUT_STRIP_TRAILING_WHITESPACE)

if (NOT result EQUAL "0")
  message(FATAL_ERROR "Failed: ${result}")
endif()

string (REGEX MATCH "([0-9]+)\.([0-9])+\.([0-9]+)" matches ${VIRTUALENV_VERSION})

if (${CMAKE_MATCH_1} GREATER_EQUAL 20)
  set(
      cmd
      ${Python_EXECUTABLE}
      -m
      virtualenv
      ${venv_dir}
  )
else()
  set(
      cmd
      ${Python_EXECUTABLE}
      -m
      virtualenv
      --relocatable
      ${venv_dir}
  )

endif()


set(cmd_human)
foreach(x ${cmd})
  set(cmd_human "${cmd_human} \"${x}\"")
endforeach()

message("Running command: ${cmd_human}")

execute_process(COMMAND ${cmd} RESULT_VARIABLE result)

if(NOT result EQUAL "0")
  message(FATAL_ERROR "Failed: ${result}")
endif()

install(
    DIRECTORY ${venv_dir}/
    DESTINATION .
    USE_SOURCE_PERMISSIONS
)

set(config_install_dir "lib/cmake/${PROJECT_NAME}")
set(generated_dir "${CMAKE_CURRENT_BINARY_DIR}/generated")
set(project_config "${generated_dir}/${PROJECT_NAME}Config.cmake")

include(CMakePackageConfigHelpers)

# Use variables:
#   - PROJECT_NAME
#   - HUNTER_VENV_PYTHON_VERSION
#   - HUNTER_VENV_USE_2
#   - HUNTER_VENV_USE_3
configure_package_config_file(
    "cmake/template/Config.cmake.in"
    "${project_config}"
    INSTALL_DESTINATION "${config_install_dir}"
)

install(
    FILES "${project_config}"
    DESTINATION "${config_install_dir}"
)
