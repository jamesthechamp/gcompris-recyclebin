if(SAILFISHOS)
  cmake_minimum_required(VERSION 2.8.11)
else(SAILfISHOS)
  cmake_minimum_required(VERSION 2.8.12)
endif()

project(gcompris-qt C CXX)

# Set c++11 support
include(CheckCXXCompilerFlag)
CHECK_CXX_COMPILER_FLAG("-std=c++11" COMPILER_SUPPORTS_CXX11)
CHECK_CXX_COMPILER_FLAG("-std=c++0x" COMPILER_SUPPORTS_CXX0X)
if(COMPILER_SUPPORTS_CXX11)
        set(my_cxx_flags "-std=c++11")
elseif(COMPILER_SUPPORTS_CXX0X)
        set(my_cxx_flags "-std=c++0x")
else()
        message(STATUS "The compiler ${CMAKE_CXX_COMPILER} has no C++11 support. Please use a different C++ compiler.")
endif()

set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} ${my_cxx_flags}")
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} ${my_cxx_flags}")

set(GCOMPRIS_MAJOR_VERSION 0)
set(GCOMPRIS_MINOR_VERSION 60)
set(GCOMPRIS_PATCH_VERSION 0)

# Set the BUILD_DATE
string(TIMESTAMP BUILD_DATE %Y%M)

# Set executable filename
if(ANDROID)
  set(GCOMPRIS_EXECUTABLE_NAME GCompris)
  if("${ANDROID_ABI}" STREQUAL "x86")
    # We always want x86 to be a release above to arm one because the play
    # store want x86 to be pushed after arm.
    MATH(EXPR GCOMPRIS_MINOR_VERSION "${GCOMPRIS_MINOR_VERSION}+1")
  endif("${ANDROID_ABI}" STREQUAL "x86")
elseif(SAILFISHOS)
  set(GCOMPRIS_EXECUTABLE_NAME harbour-gcompris-qt)
else()
  set(GCOMPRIS_EXECUTABLE_NAME gcompris-qt)
endif()

set(GCOMPRIS_VERSION ${GCOMPRIS_MAJOR_VERSION}.${GCOMPRIS_MINOR_VERSION})

# An integer value that represents the version of the application
# Increase it at each release
set(GCOMPRIS_VERSION_CODE ${GCOMPRIS_MINOR_VERSION})

# cmake modules setup
find_package(ECM 1.4.0 QUIET NO_MODULE)
set(CMAKE_MODULE_PATH ${ECM_MODULE_PATH} ${ECM_KDE_MODULE_DIR} ${CMAKE_SOURCE_DIR}/cmake/)
set(CMAKE_PREFIX_PATH "${Qt5_DIR}/lib/cmake/Qt5")

# KDE po to qm tools
if(ECM_FOUND)
  include(ECMPoQmTools)
  if (IS_DIRECTORY "${CMAKE_CURRENT_SOURCE_DIR}/po")
    ecm_install_po_files_as_qm(po)
  endif()
endif(ECM_FOUND)

# prevent build in source directory
if("${CMAKE_BINARY_DIR}" STREQUAL "${CMAKE_SOURCE_DIR}")
    message(SEND_ERROR "Building in the source directory is not supported.")
    message(FATAL_ERROR "Please remove the created \"CMakeCache.txt\" file, the \"CMakeFiles\"
            directory and create a build directory and call \"${CMAKE_COMMAND} <path to the sources>\".")
endif("${CMAKE_BINARY_DIR}" STREQUAL "${CMAKE_SOURCE_DIR}")

# Sailfish does not provide a recent Qt version
if(SAILFISHOS)
  set(QT_REQUIRED_VERSION 5.2.2)
else()
  set(QT_REQUIRED_VERSION 5.3.0)
endif()

find_package(Qt5 ${QT_REQUIRED_VERSION} REQUIRED
            Qml Quick Gui Multimedia Core Svg Xml XmlPatterns LinguistTools Sensors)

find_package (KF5 QUIET COMPONENTS
    DocTools
)
if(KF5_FOUND)
    include(KDEInstallDirs)
endif(KF5_FOUND)

FIND_PROGRAM(LCONVERT_EXECUTABLE lconvert lconvert-qt5
  PATHS
  ${Qt5_DIR}/../../../bin/
  /usr/local/bin
  /usr/bin/
  NO_DEFAULT_PATH
)
FIND_PROGRAM(LRELEASE_EXECUTABLE lrelease lrelease-qt5
  PATHS
  ${Qt5_DIR}/../../../bin/
  /usr/local/bin
  /usr/bin/
  NO_DEFAULT_PATH
)

#get_cmake_property(_variableNames VARIABLES)
#foreach (_variableName ${_variableNames})
#    message("${_variableName}=${${_variableName}}")
#endforeach()

set(ACTIVATION_MODE "no" CACHE STRING "Policy for activation [no|inapp|internal]")
option(WITH_DEMO_ONLY "Include only demo activities" OFF)

option(WITH_DOWNLOAD "Internal download" ON)
if(WITH_DOWNLOAD)
  set(ANDROID_INTERNET_PERMISSION "<uses-permission android:name=\"android.permission.INTERNET\" />")
  set(ANDROID_ACCESS_NETWORK_STATE_PERMISSION "<uses-permission android:name=\"android.permission.ACCESS_NETWORK_STATE\" />")
endif(WITH_DOWNLOAD)

# Set output directory
if(CMAKE_HOST_APPLE)
  set(_bundle_bin gcompris-qt.app/Contents/MacOS)
  set(_data_dest_dir bin/${_bundle_bin}/../Resources)
elseif(ANDROID)
  set(_data_dest_dir android/assets)
else()
  set(_data_dest_dir share/${GCOMPRIS_EXECUTABLE_NAME})
endif()
if(ANDROID)
  # Android .so output
  set(CMAKE_LIBRARY_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/android/libs/${ARM_TARGET}/)
  set(GCOMPRIS_TRANSLATIONS_DIR ${CMAKE_BINARY_DIR}/${_data_dest_dir} CACHE INTERNAL "" FORCE)
  set(GCOMPRIS_RCC_DIR ${CMAKE_BINARY_DIR}/${_data_dest_dir} CACHE INTERNAL "" FORCE)
  if(ACTIVATION_MODE STREQUAL "inapp")
    set(ANDROID_BILLING_PERMISSION "<uses-permission android:name=\"com.android.vending.BILLING\"/>")
    set(ANDROID_PACKAGE "net.gcompris")
  else(ACTIVATION_MODE)
    set(ANDROID_PACKAGE "net.gcompris.full")
  endif()
  add_subdirectory(android)
elseif(CMAKE_HOST_APPLE)
  # MacOSX build
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
  set(GCOMPRIS_TRANSLATIONS_DIR ${CMAKE_BINARY_DIR}/${_data_dest_dir}/translations CACHE INTERNAL "" FORCE)
  set(GCOMPRIS_RCC_DIR ${CMAKE_BINARY_DIR}/${_data_dest_dir}/rcc CACHE INTERNAL "" FORCE)
else()
  # Desktop build
  set(CMAKE_RUNTIME_OUTPUT_DIRECTORY ${CMAKE_BINARY_DIR}/bin)
  set(GCOMPRIS_TRANSLATIONS_DIR ${CMAKE_BINARY_DIR}/${_data_dest_dir}/translations CACHE INTERNAL "" FORCE)
  set(GCOMPRIS_RCC_DIR ${CMAKE_BINARY_DIR}/${_data_dest_dir}/rcc CACHE INTERNAL "" FORCE)
endif(ANDROID)

# Always create these folders
add_custom_command(
  OUTPUT shareFolders
  COMMAND cmake -E make_directory ${GCOMPRIS_TRANSLATIONS_DIR}
  COMMAND cmake -E make_directory ${GCOMPRIS_RCC_DIR}
  )
add_custom_target(
  createShareFolders ALL
  DEPENDS shareFolders
  )

include(cmake/rcc.cmake)

# Translations handling

# Simple command calling the python script
  add_custom_command(
    OUTPUT retrievePoFilesFromSvn
    COMMAND python2 tools/l10n-fetch-po-files.py
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
    )
# Install translations
add_custom_target(getSvnTranslations
  DEPENDS retrievePoFilesFromSvn
  COMMENT "Re-run cmake after this to be able to run BuildTranslations with the latest files"
  )

# Get all po files in po/. You can get them doing : python2 tools/l10n-fetch-po-files.py
file(GLOB TRANSLATIONS_FILES RELATIVE ${CMAKE_CURRENT_SOURCE_DIR} "po/*.po")
# Set the output dir for the translation files to /bin
foreach(PoSource ${TRANSLATIONS_FILES})
  # Changes the .po extension to .ts
  string(REPLACE ".po" ".ts" TsSource ${PoSource})
  # Removes the po/ folder
  string(REPLACE "po/" "" TsSource ${TsSource})
  # qm filename
  string(REPLACE ".ts" ".qm" QmOutput ${TsSource})

  set(OutTsFile ${CMAKE_BINARY_DIR}/tmp/${TsSource})
  add_custom_command(
    OUTPUT ${QmOutput}
    COMMAND cmake -E make_directory ${GCOMPRIS_TRANSLATIONS_DIR}
    COMMAND cmake -E make_directory ${CMAKE_BINARY_DIR}/tmp
    # Remove the obsolete translations and set po in the ts output file
    COMMAND msgattrib --no-obsolete  ${CMAKE_CURRENT_SOURCE_DIR}/${PoSource} -o ${OutTsFile}
    # Convert the po into ts
    COMMAND ${LCONVERT_EXECUTABLE} -if po -of ts -i ${OutTsFile} -o ${OutTsFile}
    # Convert the ts in qm removing non finished translations
    COMMAND ${LRELEASE_EXECUTABLE} -compress -nounfinished ${OutTsFile} -qm ${GCOMPRIS_TRANSLATIONS_DIR}/${QmOutput}
    )
  list(APPEND QM_FILES ${QmOutput})
endforeach()

# Install translations
add_custom_target(BuildTranslations
  DEPENDS ${QM_FILES}
  COMMENT "If you don't have the .po, you need to run make getSvnTranslations first then re-run cmake"
  )

if(CMAKE_HOST_APPLE)
  install(DIRECTORY ${GCOMPRIS_TRANSLATIONS_DIR} DESTINATION ${_bundle_bin})
else()
  install(DIRECTORY ${GCOMPRIS_TRANSLATIONS_DIR} DESTINATION ${_data_dest_dir})
endif()

# Build standalone package option -> if ON, we will copy the required Qt files in the build package.
# If OFF, "make install" will not copy Qt files so only GCompris files will be packaged.
# By default, it is true on Windows (as we deliver NSIS package), macOS (bundled), android (apk) and false on linux (to do make install)
# If you want to create a STGZ package for linux (auto-extractible), override this variable by typing : cmake -DBUILD_STANDALONE=ON
if(UNIX AND NOT ANDROID AND NOT APPLE)
  option(BUILD_STANDALONE "Build a standalone package when typing 'make package'" OFF)
else()
  option(BUILD_STANDALONE "Build a standalone package when typing 'make package'" ON)
endif()

option(WITH_KIOSK_MODE "Set the kiosk mode by default" OFF)

if(WIN32)
  set(COMPRESSED_AUDIO "mp3" CACHE STRING "Compressed Audio format [ogg|aac|mp3]")
elseif(APPLE)
  set(COMPRESSED_AUDIO "aac" CACHE STRING "Compressed Audio format [ogg|aac|mp3]")
else()
  set(COMPRESSED_AUDIO "ogg" CACHE STRING "Compressed Audio format [ogg|aac|mp3]")
endif()

file(GLOB_RECURSE OGG_FILES ${CMAKE_CURRENT_SOURCE_DIR}/src/ "*.ogg")

foreach(OGG_FILE ${OGG_FILES})
  # This should only replace the extension
  string(REGEX REPLACE "ogg$" "aac" AAC_FILE ${OGG_FILE})
  add_custom_command(
    OUTPUT ${AAC_FILE}
    # Put the good line depending on your installation
    COMMAND avconv -v warning -i ${OGG_FILE} -acodec libvo_aacenc ${AAC_FILE}
    #COMMAND ffmpeg -v warning -i ${OGG_FILE} -acodec aac -strict -2 ${AAC_FILE}
    )
  list(APPEND AAC_FILES ${AAC_FILE})

  # This should only replace the extension
  string(REGEX REPLACE "ogg$" "mp3" MP3_FILE ${OGG_FILE})
  add_custom_command(
    OUTPUT ${MP3_FILE}
    # Put the good line depending on your installation
    #COMMAND avconv -v warning -i ${OGG_FILE} -acodec mp3 ${MP3_FILE}
    COMMAND ffmpeg -v warning -i ${OGG_FILE} -acodec mp3 -strict -2 ${AAC_FILE}
    )
  list(APPEND MP3_FILES ${MP3_FILE})
endforeach()

add_custom_target(
  createAacFromOgg
  DEPENDS ${AAC_FILES}
  )

add_custom_target(
  createMp3FromOgg
  DEPENDS ${MP3_FILES}
  )

set(ARCHIVE_NAME ${CMAKE_PROJECT_NAME}-${GCOMPRIS_VERSION})
add_custom_target(dist
    COMMAND git archive --prefix=${ARCHIVE_NAME}/ HEAD
        | xz > ${CMAKE_BINARY_DIR}/${ARCHIVE_NAME}.tar.xz
    WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})

if(KF5_FOUND)
    add_subdirectory(docs/docbook)
endif(KF5_FOUND)

if(SAILFISHOS)
  # Need to be done at the end, after src
  add_subdirectory(platforms/sailfishOS)
endif()

#
# qml-box2d
#

set(QML_BOX2D_MODULE "auto" CACHE STRING "Policy for qml-box2d module [auto|submodule|system|disabled]")

if (${QML_BOX2D_MODULE} STREQUAL "disabled")
  # disable all activities depending on qml-box2d
  set(_disabled_activities "balancebox")
  message(STATUS "Disabling qml-box2d module and depending activities: ${_disabled_activities}")
else()
  include(qt_helper)

  getQtQmlPath(_qt_qml_system_path)
  set (_box2d_system_dir "${_qt_qml_system_path}/Box2D.2.0")

  if (${QML_BOX2D_MODULE} STREQUAL "submodule")
    message(STATUS "Building qml-box2d module from submodule")
    set(_need_box2d_submodule "TRUE")
  else()
    # try to find module in system scope
    find_library(QML_BOX2D_LIBRARY NAMES Box2D libBox2D PATHS ${_box2d_system_dir} NO_DEFAULT_PATH)
    if (QML_BOX2D_LIBRARY)
      message(STATUS "Using system qml-box2d plugin at ${QML_BOX2D_LIBRARY}")
      # for packaging builds, copy the module manually to the correct location
      if(SAILFISHOS)
        file(COPY ${_box2d_system_dir}/qmldir ${QML_BOX2D_LIBRARY} DESTINATION share/harbour-gcompris-qt/lib/qml/Box2D.2.0)
      elseif(ANDROID)
        file(COPY ${_box2d_system_dir}/qmldir ${QML_BOX2D_LIBRARY} DESTINATION lib/qml/Box2D.2.0)
      endif()
      # FIXME: add others as needed
    else()
      if (${QML_BOX2D_MODULE} STREQUAL "auto")
        message(STATUS "Did not find the qml-box2d module in system scope, falling back to submodule build ...")
        set (_need_box2d_submodule "TRUE")
      else()
        message(FATAL_ERROR "Did not find the qml-box2d module in system scope and submodule build was not requested. Can't continue!")
      endif()
    endif()
  endif()

  if (_need_box2d_submodule)
    # build qml-box2d ourselves from submodule
    include(ExternalProject)
    get_property(_qmake_program TARGET ${Qt5Core_QMAKE_EXECUTABLE} PROPERTY IMPORT_LOCATION)
    set (_box2d_source_dir ${CMAKE_CURRENT_SOURCE_DIR}/external/qml-box2d)
    if(WIN32)
      set (_box2d_library_dir "release/")
      set (_box2d_library_file "Box2D.dll")
    else()
      set (_box2d_library_dir "")
      set (_box2d_library_file "libBox2D.so")
    endif()
    set (_box2d_install_dir ${CMAKE_CURRENT_BINARY_DIR}/lib/qml/Box2D.2.0)
    # make sure submodule is up2date
    find_package(Git)
    if(GIT_FOUND)
      execute_process(COMMAND ${GIT_EXECUTABLE} submodule update --init)
    endif()

    ExternalProject_Add(qml_box2d
      DOWNLOAD_COMMAND ""
      SOURCE_DIR ${_box2d_source_dir}
      CONFIGURE_COMMAND ${_qmake_program} ${_box2d_source_dir}/box2d.pro
      BUILD_COMMAND ${CMAKE_MAKE_PROGRAM}
      INSTALL_DIR ${_box2d_install_dir}
      INSTALL_COMMAND cp ${_box2d_library_dir}${_box2d_library_file} ${_box2d_source_dir}/qmldir ${_box2d_install_dir}
      )

    add_library(qml-box2d SHARED IMPORTED)
    set_target_properties(qml-box2d PROPERTIES IMPORTED_LOCATION ${_box2d_install_dir}/${_box2d_library_file})

    if(SAILFISHOS)
      install(DIRECTORY ${_box2d_install_dir} DESTINATION share/harbour-gcompris-qt/lib/qml)
    else()
      install(DIRECTORY ${_box2d_install_dir} DESTINATION lib/qml)
    endif()
  endif()
endif()

add_subdirectory(src)
