#
# Glewlwyd
#
# CMake file used to build program
#
# Copyright 2018 Nicolas Mora <mail@babelouest.org>
#
# This program is free software; you can redistribute it and/or
# modify it under the terms of the MIT License
#
# This library is distributed in the hope that it will be useful,
# but WITHOUT ANY WARRANTY; without even the implied warranty of
# MERCHANTABILITY or FITNESS FOR A PARTICULAR PURPOSE.
#

cmake_minimum_required(VERSION 3.5)

project(glewlwyd C)

set(CMAKE_C_STANDARD 99)
set(CMAKE_C_FLAGS "${CMAKE_C_FLAGS} -Wall -Werror")

# library info

set(PROGRAM_VERSION_MAJOR "1")
set(PROGRAM_VERSION_MINOR "3")
set(PROGRAM_VERSION_PATCH "2")
set(PROJECT_DESCRIPTION "OAuth2 authentication server providing Json Web Tokens")
set(PROJECT_BUGREPORT_PATH "https://github.com/babelouest/glewlwyd/issues")
set(ULFIUS_VERSION_DOWNLOAD "2.3.0-b.3")
set(HOEL_VERSION_DOWNLOAD "1.4.0-b.3")

include(GNUInstallDirs)
include(CheckSymbolExists)

# cmake modules

set(CMAKE_MODULE_PATH ${CMAKE_CURRENT_SOURCE_DIR}/cmake-modules)

# check if _GNU_SOURCE is available

if (NOT _GNU_SOURCE)
    check_symbol_exists(__GNU_LIBRARY__ "features.h" _GNU_SOURCE)

    if (NOT _GNU_SOURCE)
        unset(_GNU_SOURCE CACHE)
        check_symbol_exists(_GNU_SOURCE "features.h" _GNU_SOURCE)
    endif ()
endif ()

if (_GNU_SOURCE)
    add_definitions(-D_GNU_SOURCE)
endif ()

include(FindJansson)
set(JANSSON_MIN_VERSION 2.4)
find_package(Jansson ${JANSSON_MIN_VERSION} REQUIRED)
if (JANSSON_FOUND)
    set(LIBS ${JANSSON_LIBRARIES})
endif ()

include(FindLibJWT)
find_package(LibJWT REQUIRED)
if (LIBJWT_FOUND)
    set(LIBS ${LIBS} ${LIBJWT_LIBRARIES})
endif ()

include(FindGnuTLS)
find_package(GnuTLS REQUIRED)
if (GNUTLS_FOUND)
    set(LIBS ${LIBS} ${GNUTLS_LIBRARIES})
endif ()

include(FindLdap)
find_package(Ldap REQUIRED)
if (LDAP_FOUND)
    set(LIBS ${LIBS} ${LDAP_LIBRARIES})
    include_directories(${LDAP_INCLUDE_DIRS})
endif ()

include(FindCRYPT)
find_package(CRYPT REQUIRED)
if (CRYPT_FOUND)
    set(LIBS ${LIBS} ${CRYPT_LIBRARIES})
    include_directories(${CRYPT_INCLUDE_DIRS})
endif ()

include(FindLibconfig)
find_package(Libconfig REQUIRED)
if (LIBCONFIG_FOUND)
    set(LIBS ${LIBS} ${LIBCONFIG_LIBRARIES})
    include_directories(${LIBCONFIG_INCLUDE_DIRS})
endif ()

include(FindUUID)
find_package(UUID REQUIRED)
if (UUID_FOUND)
    set(LIBS ${LIBS} ${UUID_LIBRARIES})
    include_directories(${UUID_INCLUDE_DIRS})
endif ()

# build

add_executable(glewlwyd ${CMAKE_CURRENT_SOURCE_DIR}/src/glewlwyd.h
                ${CMAKE_CURRENT_SOURCE_DIR}/src/authorization.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/authorization.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/oauth.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/webservice.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/token.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/user.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/client.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/admin.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/scope.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/resource.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/password.c
                ${CMAKE_CURRENT_SOURCE_DIR}/src/glewlwyd.c
                ${CMAKE_CURRENT_SOURCE_DIR}/glewlwyd.conf.sample
                ${CMAKE_CURRENT_SOURCE_DIR}/glewlwyd.service
                ${CMAKE_CURRENT_SOURCE_DIR}/glewlwyd-init )
target_link_libraries(glewlwyd ${LIBS})

set(Ulfius_FIND_QUIETLY ON)
include(FindUlfius)
find_package(Ulfius 2.2 QUIET)
if (NOT ULFIUS_FOUND)
    include(DownloadProject)
    download_project(PROJ ulfius
            URL "https://github.com/babelouest/ulfius/archive/v${ULFIUS_VERSION_DOWNLOAD}.tar.gz"
            QUIET)
    add_subdirectory(${ulfius_SOURCE_DIR} ${ulfius_BINARY_DIR})
    include_directories(${yder_SOURCE_DIR}/include)
    include_directories(${orcania_SOURCE_DIR}/include)
    include_directories(${ulfius_SOURCE_DIR}/include)
    add_dependencies(ulfius yder)
    add_dependencies(ulfius orcania)
    add_dependencies(ulfius ulfius)
endif ()

set(ULFIUS_LIBRARIES ulfius yder orcania)
target_link_libraries(glewlwyd ${LIBS} ${ULFIUS_LIBRARIES})

set(Hoel_FIND_QUIETLY ON)
include(FindHoel)
find_package(Hoel 1.3 QUIET)
if (NOT HOEL_FOUND)
    include(DownloadProject)
    download_project(PROJ hoel
            URL "https://github.com/babelouest/hoel/archive/v${HOEL_VERSION_DOWNLOAD}.tar.gz"
            QUIET)
    add_subdirectory(${hoel_SOURCE_DIR} ${hoel_BINARY_DIR})
    include_directories(${hoel_SOURCE_DIR}/include)
    add_dependencies(glewlwyd hoel)
endif ()

set(HOEL_LIBRARIES hoel)
target_link_libraries(glewlwyd ${LIBS} ${HOEL_LIBRARIES})

# tests

option(BUILD_TESTING "Build the testing tree." OFF)
option(BUILD_TEST_PROFILE "Build test glewlwyd_update_user_profile" OFF)
option(BUILD_TEST_HTTP_AUTH "Build test glewlwyd_http_auth only" OFF)

if (BUILD_TESTING)
    include(FindCheck)
    find_package(Check REQUIRED)
    if (CHECK_FOUND)
        include(FindSubunit)
        find_package(Subunit REQUIRED)

        enable_testing()

        set(CMAKE_CTEST_COMMAND ctest -V)

        set(TST_DIR ${CMAKE_CURRENT_SOURCE_DIR}/test)
        set(LIBS ${LIBS} ${CHECK_LIBRARIES} ulfius yder orcania)
        find_package(Threads REQUIRED)
        set(LIBS ${LIBS} ${SUBUNIT_LIBRARIES} ${CMAKE_THREAD_LIBS_INIT} m rt)

        if (BUILD_TEST_HTTP_AUTH)
            set(TESTS glewlwyd_http_auth)
        else ()
            set(TESTS glewlwyd_admin_authorization glewlwyd_oauth2_code_client_confidential glewlwyd_user_session_admin glewlwyd_crud_client glewlwyd_oauth2_implicit glewlwyd_user_session_user glewlwyd_crud_resource glewlwyd_oauth2_refresh_token glewlwyd_crud_scope glewlwyd_oauth2_refresh_token_client_confidential glewlwyd_crud_user glewlwyd_oauth2_resource_owner_pwd_cred glewlwyd_delete_token glewlwyd_oauth2_resource_owner_pwd_cred_client_confidential glewlwyd_oauth2_auth_code glewlwyd_oauth2_client_cred glewlwyd_user_refresh_token_admin glewlwyd_oauth2_code glewlwyd_user_refresh_token_user)
            
            if (BUILD_TEST_PROFILE)
                set(TESTS ${TESTS} glewlwyd_update_user_profile)
            endif ()
        endif ()

        configure_file(
                "${CMAKE_MODULE_PATH}/CTestCustom.cmake.in"
                "${CMAKE_CURRENT_BINARY_DIR}/CTestCustom.cmake"
                @ONLY)

        foreach (t ${TESTS})
            add_executable(${t} EXCLUDE_FROM_ALL ${TST_DIR}/${t}.c ${TST_DIR}/unit-tests.c ${TST_DIR}/unit-tests.h)
            target_include_directories(${t} PUBLIC ${TST_DIR})
            target_link_libraries(${t} PUBLIC ${LIBS})
            add_test(NAME ${t}
                    WORKING_DIRECTORY ${TST_DIR}
                    COMMAND ${t})
        endforeach ()
    endif ()
endif ()

# install target

install(TARGETS glewlwyd
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
install(FILES glewlwyd.conf.sample
        DESTINATION ${CMAKE_INSTALL_SYSCONFDIR}/glewlwyd RENAME glewlwyd.conf COMPONENT config)
install(DIRECTORY webapp/
        DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/glewlwyd/webapp/ COMPONENT runtime)

set(TARGETS glewlwyd)

install(TARGETS ${TARGETS}
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR}
        CONFIGURATIONS Release)

# uninstall target

if (NOT TARGET uninstall)
    configure_file(
            "${CMAKE_MODULE_PATH}/CMakeUninstall.cmake.in"
            "${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
            IMMEDIATE @ONLY)
    add_custom_target(uninstall
            COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
endif ()

# packaging

set(CPACK_PACKAGE_VERSION_MAJOR ${PROGRAM_VERSION_MAJOR})
set(CPACK_PACKAGE_VERSION_MINOR ${PROGRAM_VERSION_MINOR})
set(CPACK_PACKAGE_VERSION_PATCH ${PROGRAM_VERSION_PATCH})

set(PACKAGE_FILE_NAME
        "${CMAKE_PROJECT_NAME}_${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH}")

set(PACKAGE_IGNORED_FILES
        "${CMAKE_CURRENT_BINARY_DIR}/;/.git/;.gitignore;~$;${CPACK_SOURCE_IGNORE_FILES}")

set(CPACK_GENERATOR "TGZ;DEB")

set(CPACK_DEBIAN_PACKAGE_MAINTAINER "mail@babelouest.org")
set(CPACK_DEBIAN_PACKAGE_DESCRIPTION ${PROJECT_DESCRIPTION})
set(CPACK_DEBIAN_PACKAGE_HOMEPAGE "https://github.com/babelouest/glewlwyd")
set(CPACK_DEBIAN_PACKAGE_DEPENDS "libc6 (>= 2.3.4), libconfig9, libgnutls30 (>= 3.5.0), libjansson4 (>= 2.7), libldap-2.4-2 (>= 2.4.7), liborcania (>= 1.2.0), libyder (>= 1.2.0), libulfius, libhoel, libuuid1 (>= 2.16), lsb-base (>= 3.0-6)")
set(CPACK_PACKAGE_FILE_NAME ${PACKAGE_FILE_NAME})

set(CPACK_SOURCE_GENERATOR "TGZ")
set(CPACK_SOURCE_PACKAGE_FILE_NAME ${PACKAGE_FILE_NAME})
set(CPACK_SOURCE_IGNORE_FILES ${PACKAGE_IGNORED_FILES})

include(CPack)

add_custom_target(dist
        COMMAND ${CMAKE_MAKE_PROGRAM} package_source)