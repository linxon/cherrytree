cmake_minimum_required(VERSION 3.13.2)

project(cherrytree)
set(CT_VERSION "0.99.0")


if(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)
  set(CMAKE_INSTALL_PREFIX "/usr" CACHE PATH "..." FORCE)
endif(CMAKE_INSTALL_PREFIX_INITIALIZED_TO_DEFAULT)


# used to install locale, packagers might overwrite this
if(NOT PACKAGE_LOCALE_DIR)
  set(PACKAGE_LOCALE_DIR "share/locale")
endif(NOT PACKAGE_LOCALE_DIR)

# used to install data, packagers might overwrite this
if(NOT SHARE_INSTALL)
  set(SHARE_INSTALL "share" CACHE STRING "Data file install path. Must be a relative path (from CMAKE_INSTALL_PREFIX), with no trailing slash.")
endif(NOT SHARE_INSTALL)
set(CHERRYTREE_SHARE_INSTALL "${SHARE_INSTALL}/cherrytree")


set(CPACK_PACKAGE_NAME "cherrytree")
set(CPACK_PACKAGE_CONTACT "Giuseppe Penone <giuspen@gmail.com>")
set(CPACK_PACKAGE_VENDOR ${CPACK_PACKAGE_CONTACT})
set(CPACK_PACKAGE_VERSION ${CT_VERSION})
set(CPACK_PACKAGE_DESCRIPTION_FILE "${CMAKE_CURRENT_SOURCE_DIR}/README.md")
set(CPACK_PACKAGE_DESCRIPTION_SUMMARY "A Hierarchical Note Taking Application, featuring Rich Text and Syntax Highlighting.")
set(CPACK_RESOURCE_FILE_LICENSE "${CMAKE_CURRENT_SOURCE_DIR}/../license.txt")
set(CPACK_PACKAGING_INSTALL_PREFIX ${CMAKE_INSTALL_PREFIX})
set(CPACK_PACKAGE_CHECKSUM "SHA256")
set(CPACK_PACKAGE_EXECUTABLES ${CPACK_PACKAGE_CONTACT})
set(CPACK_CREATE_DESKTOP_LINKS ${CPACK_PACKAGE_CONTACT})
set(CPACK_PACKAGE_INSTALL_DIRECTORY ${CPACK_PACKAGE_CONTACT})
set(CPACK_STRIP_FILES TRUE)

set(CPACK_DEBIAN_PACKAGE_DEPENDS "libgtkmm-3.0-dev, libgtksourceviewmm-3.0-dev, libxml++2.6-dev, libsqlite3-dev, gettext, libgspell-1-dev")
set(CPACK_DEBIAN_PACKAGE_HOMEPAGE "https://www.giuspen.com/cherrytree/")
#set(CPACK_DEBIAN_PACKAGE_SECTION "graphics") which one?
set(CPACK_DEBIAN_PACKAGE_SHLIBDEPS ON)

include(CPack)


set(CMAKE_CXX_STANDARD 17)

# temporarily disable these warnings: Only <gtksourceview/gtksource.h> can be included directly.
if("${CMAKE_CXX_COMPILER_ID}" STREQUAL "GNU")
  add_compile_options(-Wno-cpp)
elseif(CMAKE_CXX_COMPILER_ID MATCHES "Clang")
  add_compile_options("-Wno-#warnings")
  add_compile_options(-Wthread-safety)
endif()


if(APPLE)
  link_directories(/usr/local/lib /usr/local/opt/gettext/lib)
  include_directories(/usr/local/opt/gettext/include)
  set(CMAKE_MACOSX_RPATH 1)
  set(ENV{PKG_CONFIG_PATH} "$ENV{PKG_CONFIG_PATH}:/usr/local/lib/pkgconfig:/opt/X11/lib/pkgconfig:/usr/local/opt/libxml2/lib/pkgconfig:/usr/local/opt/libxml++/lib/pkgconfig")
  # to fix libxml++2.6 compilation
  add_compile_options(-D_LIBCPP_ENABLE_CXX17_REMOVED_AUTO_PTR)
endif()

add_compile_options(-Wno-deprecated -Wno-deprecated-declarations)
if(CMAKE_BUILD_TYPE STREQUAL "")
  add_compile_options(-O3)
endif()


# Create the configuration files config.h in the root dir
configure_file(${CMAKE_SOURCE_DIR}/config.h.cmake ${CMAKE_SOURCE_DIR}/config.h)

option(BUILD_TESTING "Build tests" ON)
if (BUILD_TESTING)
  message("Tests are ON")
else()
  message("Tests are OFF")
endif()

include(FindPkgConfig)
pkg_check_modules(GTKMM gtkmm-3.0 REQUIRED)
pkg_check_modules(GTKSVMM gtksourceviewmm-3.0 REQUIRED)
pkg_check_modules(GLIBMM glibmm-2.4>=2.64 REQUIRED)
pkg_check_modules(PANGOMM pangomm-1.4 REQUIRED)
pkg_check_modules(GSPELL gspell-1 REQUIRED)
pkg_check_modules(LIBXML libxml++-2.6 REQUIRED)
pkg_check_modules(SQLITE sqlite3 REQUIRED)
if(BUILD_TESTING)
  pkg_check_modules(CPPUTEST cpputest  REQUIRED)
endif()

add_subdirectory(src/7za)


include(FindPackageHandleStandardArgs)

# For both src and tests targets
include_directories(
  SYSTEM ${GTKMM_INCLUDE_DIRS}
  SYSTEM ${GTKSVMM_INCLUDE_DIRS}
  SYSTEM ${GLIBMM_INCLUDE_DIRS}
  SYSTEM ${PANGOMM_INCLUDE_DIRS}
  SYSTEM ${GSPELL_INCLUDE_DIRS}
  SYSTEM ${LIBXML_INCLUDE_DIRS}
  SYSTEM ${SQLITE_INCLUDE_DIRS}
  include_directories(.) # to find src/fmt
)

add_subdirectory(src/ct)

if(BUILD_TESTING)
  enable_testing()
  add_subdirectory(tests)
endif()


# rebuild man pages
set(MANFILE_FULL "${CMAKE_SOURCE_DIR}/data/cherrytree.1")
set(MANFILE_FULL_GZ "${MANFILE_FULL}.gz")
add_custom_command(
  OUTPUT ${MANFILE_FULL_GZ}
  COMMAND gzip -f -k --best -n "${MANFILE_FULL}"
)
add_custom_target(man_pages ALL DEPENDS ${MANFILE_FULL_GZ})


# rebuild languages binaries
execute_process(
  COMMAND python3 ${CMAKE_SOURCE_DIR}/po/i18n_po_to_mo.py
  WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
)


# install cherrytree
install(TARGETS cherrytree RUNTIME DESTINATION bin)

# set icon on macos
if(APPLE)
  install(CODE "execute_process(COMMAND /usr/bin/python ${CMAKE_SOURCE_DIR}/scripts/set_icon_macos.py ${CMAKE_SOURCE_DIR}/icons/cherrytree.ico ${CMAKE_INSTALL_PREFIX}/bin/cherrytree)")
endif(APPLE)

# install other files
if(${CMAKE_SYSTEM_NAME} MATCHES Linux|.*BSD|DragonFly|Darwin)
  if(NOT APPLE)
    # share data
    install(FILES "${CMAKE_SOURCE_DIR}/data/cherrytree.desktop"
            DESTINATION "${SHARE_INSTALL}/applications")
    install(FILES "${CMAKE_SOURCE_DIR}/icons/cherrytree.svg"
            DESTINATION "${SHARE_INSTALL}/icons/hicolor/scalable/apps")
    install(FILES "${MANFILE_FULL_GZ}"
            DESTINATION "${SHARE_INSTALL}/man/man1")
    install(FILES
            "${CMAKE_SOURCE_DIR}/data/cherrytree.mime"
            "${CMAKE_SOURCE_DIR}/data/cherrytree.keys"
            DESTINATION "${SHARE_INSTALL}/mime-info")
    install(FILES "${CMAKE_SOURCE_DIR}/data/com.giuspen.cherrytree.metainfo.xml"
            DESTINATION "${SHARE_INSTALL}/metainfo")
  endif(NOT APPLE)

  # share/cherrytree data
  install(DIRECTORY ${CMAKE_SOURCE_DIR}/language-specs
          DESTINATION ${CHERRYTREE_SHARE_INSTALL})
  install(FILES
          "${CMAKE_SOURCE_DIR}/data/script3.js"
          "${CMAKE_SOURCE_DIR}/data/styles3.css"
          DESTINATION "${CHERRYTREE_SHARE_INSTALL}/data")

  # locale data
  file(GLOB children "${CMAKE_SOURCE_DIR}/po/*")
  foreach(child ${children})
    if(IS_DIRECTORY ${child})
      install(DIRECTORY ${child} DESTINATION "${PACKAGE_LOCALE_DIR}")
    endif()
  endforeach()
endif()
