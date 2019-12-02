# GNOME Shell integration for Chrome build script

cmake_minimum_required (VERSION 2.8)
project (chrome-gnome-shell NONE)

# Suppress warning
if(NOT DEFINED CMAKE_SIZEOF_VOID_P)
	set(CMAKE_SIZEOF_VOID_P 8)
endif(NOT DEFINED CMAKE_SIZEOF_VOID_P)

# Variables
set(PROJECT_VERSION "10.1")

set(ARCHIVE_NAME ${CMAKE_PROJECT_NAME}-${PROJECT_VERSION})
set(ARCHIVE_FULL_NAME ${ARCHIVE_NAME}.tar.xz)
set(ARCHIVE_DEB_NAME ${CMAKE_PROJECT_NAME}_${PROJECT_VERSION}.orig.tar.xz)
set(DEB_DIR ${CMAKE_CURRENT_BINARY_DIR}/deb)

set(EXTENSION_SOURCES ${CMAKE_CURRENT_SOURCE_DIR}/extension)
set(EXTENSION_BUILD_DIR ${CMAKE_BINARY_DIR}/extension)
set(CHROME_BUILD_DIR ${EXTENSION_BUILD_DIR}/chrome)
set(OPERA_BUILD_DIR ${EXTENSION_BUILD_DIR}/opera)
set(FIREFOX_BUILD_DIR ${EXTENSION_BUILD_DIR}/firefox)

# Options
option(BUILD_EXTENSION		"Build extension zip package"	TRUE)
option(BUILD_CONNECTOR		"Build native messaging host"	TRUE)
option(BUILD_SOURCE_PACKAGE	"Build source package"		FALSE)
option(BUILD_DEB		"Build debian package"		FALSE)
option(BUILD_MESSAGES		"Update translation strings"	FALSE)
option(USE_DEBIAN_LAYOUT	"Use --install-layout=deb distutils parameter"	FALSE)

# Default extension key for Chrome web store
if(NOT DEFINED CHROME_EXTENSION_KEY)
	set(CHROME_EXTENSION_KEY "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEAlig8TAPPQZinMkJnptC0ldizx6fG9jSjZDJ9c8GuLcXeGRH+NMlQuPC9bR5IQlT7+4VY/1tm1+IZ4xvITx1wXCNTR+KXzZv3VNc2D+logehK7oIRTRj0fLhixrx4NLSNK7L7HgV2xcIoW6QV0jOdFcTPL0mWXodXSzZePrvXuflF7qpwNxLzYVi04Vh3xu2oR2Pc9SwfZ4SNbyCaunH/p8n5AYmDuogI2Ah++RZw0ctnqn7mmHrGXteBu/vkpcHZu3B3eW9PFSrv69rRs8duybYR9C91hJm6yzRqZqIpindIU3k2HnNWeCFWkRVpZPhaNVoxcBUO7wWUUwdIflW2JwIDAQAB")
endif(NOT DEFINED CHROME_EXTENSION_KEY)

# Default extension key for Opera addons
if(NOT DEFINED OPERA_EXTENSION_KEY)
	set(OPERA_EXTENSION_KEY "MIIBIjANBgkqhkiG9w0BAQEFAAOCAQ8AMIIBCgKCAQEA1QBzvBFxSTBP1Z3u+B3GnxOhZiT/LLGJ9R3Mu3R5ap1YgvbHNqSj0CmrWE+MvJUYA+AxHSRP6ctKfPBZl/kdGYGLPgvagGBEEbutCerj9www7T2LAsFNc5gIgDXaU0P74Yik8MjLZIXOC3Q91Kien5Jbtit382HNVHl2nRbhgGZ9LZ+6UJnzSseW9NHw0/XRRnT0kv0Ih7lgC55xWfP7guam1uaT5DPxC7W5cy9V1z7mljBf50OxgnbEmf7Xvfz4QpQfDyQWE5hqKlBuPc8W/8bvYmWN+FDBqhls/FCml3icAElNMBg0YryFEDzu6xCTzgHqDKYu5SN49u+m6tyamQIDAQAB")
endif(NOT DEFINED OPERA_EXTENSION_KEY)

# Debian variables
if(NOT DEFINED DEBIAN_VERSION)
	set(DEBIAN_VERSION "0ubuntu1")
endif(NOT DEFINED DEBIAN_VERSION)

if(NOT DEFINED DEBIAN_DISTRO)
	set(DEBIAN_DISTRO "trusty")
endif(NOT DEFINED DEBIAN_DISTRO)

macro(find_program_ex)
	string(TOUPPER ${ARGV0} _PROGRAM_UPPER)

	set(_NAMES ${ARGV})
	math(EXPR _NAMES_LAST_INDEX "${ARGC}-1")
	list(GET _NAMES ${_NAMES_LAST_INDEX} _MESSAGE_STATUS)
	list(REMOVE_AT _NAMES ${_NAMES_LAST_INDEX})

	find_program(${_PROGRAM_UPPER}_EXECUTABLE NAMES ${_NAMES})

	if(${_PROGRAM_UPPER}_EXECUTABLE)
		message(STATUS "Found ${ARGV0}: ${${_PROGRAM_UPPER}_EXECUTABLE}")
		set(${_PROGRAM_UPPER}_FOUND TRUE)
	else(${_PROGRAM_UPPER}_EXECUTABLE)
		message(${_MESSAGE_STATUS} "Could NOT find ${ARGV0}.")
	endif(${_PROGRAM_UPPER}_EXECUTABLE)
endmacro(find_program_ex)

include(GNUInstallDirs)

# Options validating
if(NOT BUILD_EXTENSION AND NOT BUILD_CONNECTOR AND NOT BUILD_MESSAGES)
	message(FATAL_ERROR "No build options selected")
endif(NOT BUILD_EXTENSION AND NOT BUILD_CONNECTOR AND NOT BUILD_MESSAGES)

if(BUILD_DEB AND NOT BUILD_SOURCE_PACKAGE)
	message(STATUS "Building of debian package enabled. Turning on building source package.")
	set(BUILD_SOURCE_PACKAGE TRUE)
endif(BUILD_DEB AND NOT BUILD_SOURCE_PACKAGE)

if(BUILD_SOURCE_PACKAGE AND NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/.git/)
	message(FATAL_ERROR "Unable to build source package outside of git repository.")
endif(BUILD_SOURCE_PACKAGE AND NOT EXISTS ${CMAKE_CURRENT_SOURCE_DIR}/.git/)

if(BUILD_MESSAGES AND NOT DEFINED GETTEXT_REPORT_EMAIL)
	message(FATAL_ERROR "GETTEXT_REPORT_EMAIL must be specified to generate po template.")
endif(BUILD_MESSAGES AND NOT DEFINED GETTEXT_REPORT_EMAIL)

if(BUILD_EXTENSION OR BUILD_CONNECTOR)
	find_program_ex(base64 gbase64 FATAL_ERROR)
	find_program_ex(sha256sum gsha256sum FATAL_ERROR)
	find_program_ex(ghead head FATAL_ERROR)
	find_program_ex(tr gtr FATAL_ERROR)
	find_program_ex(jq FATAL_ERROR)

	# https://github.com/adobe/chromium/blob/master/chrome/common/extensions/extension.cc#L696
	# http://stackoverflow.com/questions/23873623/obtaining-chrome-extension-id-for-development
	execute_process(COMMAND echo "${CHROME_EXTENSION_KEY}"
			COMMAND ${BASE64_EXECUTABLE} -d
			COMMAND ${SHA256SUM_EXECUTABLE}
			COMMAND ${GHEAD_EXECUTABLE} -c32
			COMMAND ${TR_EXECUTABLE} 0-9a-f a-p
			OUTPUT_VARIABLE CHROME_EXTENSION_ID)
	message(STATUS "Calculated Chrome extension id: ${CHROME_EXTENSION_ID}")

	execute_process(COMMAND echo "${OPERA_EXTENSION_KEY}"
			COMMAND ${BASE64_EXECUTABLE} -d
			COMMAND ${SHA256SUM_EXECUTABLE}
			COMMAND ${GHEAD_EXECUTABLE} -c32
			COMMAND ${TR_EXECUTABLE} 0-9a-f a-p
			OUTPUT_VARIABLE OPERA_EXTENSION_ID)
	message(STATUS "Calculated Opera extension id: ${OPERA_EXTENSION_ID}")
endif(BUILD_EXTENSION OR BUILD_CONNECTOR)

# Options handling
if(BUILD_EXTENSION)
	find_program_ex(7z FATAL_ERROR)

	file(COPY "${EXTENSION_SOURCES}/" DESTINATION "${CHROME_BUILD_DIR}"
		PATTERN "manifest*.json" EXCLUDE)

	file(COPY "${EXTENSION_SOURCES}/" DESTINATION "${OPERA_BUILD_DIR}"
		PATTERN "manifest*.json" EXCLUDE)

	file(COPY "${EXTENSION_SOURCES}/" DESTINATION "${FIREFOX_BUILD_DIR}"
		PATTERN "manifest*.json" EXCLUDE)

	set(PUBLIC_KEY ${CHROME_EXTENSION_KEY})
	configure_file("${EXTENSION_SOURCES}/manifest.json" "${CHROME_BUILD_DIR}/")

	set(PUBLIC_KEY ${OPERA_EXTENSION_KEY})
	configure_file("${EXTENSION_SOURCES}/manifest.json" "${OPERA_BUILD_DIR}/")

	unset(PUBLIC_KEY)

	add_custom_target(chrome-extension ALL
		COMMAND "${7Z_EXECUTABLE}" a -tzip "${CMAKE_BINARY_DIR}/extension-chrome.zip" ./
		WORKING_DIRECTORY "${CHROME_BUILD_DIR}")
	add_custom_target(opera-extension ALL
		COMMAND "${7Z_EXECUTABLE}" a -tzip "${CMAKE_BINARY_DIR}/extension-opera.zip" ./
		WORKING_DIRECTORY "${OPERA_BUILD_DIR}")
	add_custom_target(firefox-extension ALL
		COMMAND "${JQ_EXECUTABLE}" -s "'add|del(.key)'"
				'${EXTENSION_SOURCES}/manifest.json' '${EXTENSION_SOURCES}/manifest.firefox.json'
				> "${FIREFOX_BUILD_DIR}/manifest.json"
		COMMAND "${7Z_EXECUTABLE}" a -tzip "${CMAKE_BINARY_DIR}/extension-firefox.zip" ./
		WORKING_DIRECTORY "${FIREFOX_BUILD_DIR}")
endif(BUILD_EXTENSION)

if(BUILD_CONNECTOR)
	find_package(PythonInterp REQUIRED)

	set(CONNECTOR_SETUP "${CMAKE_CURRENT_SOURCE_DIR}/connector/setup.py")
	add_custom_target(build-connector ALL
                       COMMAND ${PYTHON_EXECUTABLE} ${CONNECTOR_SETUP} build)

	configure_file("${CMAKE_CURRENT_SOURCE_DIR}/connector/org.gnome.chrome_gnome_shell.json"
			"${CMAKE_BINARY_DIR}/")
	configure_file("${CMAKE_CURRENT_SOURCE_DIR}/connector/org.gnome.ChromeGnomeShell.service.in"
			"${CMAKE_BINARY_DIR}/org.gnome.ChromeGnomeShell.service")

	add_custom_target(firefox-native-manifest ALL
			COMMAND "${JQ_EXECUTABLE}" -s "'add|del(.allowed_origins)'"
				"${CMAKE_BINARY_DIR}/org.gnome.chrome_gnome_shell.json"
				"${CMAKE_CURRENT_SOURCE_DIR}/connector/org.gnome.chrome_gnome_shell.firefox.json"
				> "${CMAKE_BINARY_DIR}/org.gnome.chrome_gnome_shell.firefox.json")

	if(USE_DEBIAN_LAYOUT)
		set(DISTUTILS_EXTRA_ARGS "--install-layout=deb")
	else()
		set(DISTUTILS_EXTRA_ARGS "")
	endif(USE_DEBIAN_LAYOUT)

	install(CODE "
		if(DEFINED ENV{DESTDIR})
			set(DESTDIR \"\$ENV{DESTDIR}\")
		else(DEFINED ENV{DESTDIR})
			set(DESTDIR \"/\")
		endif(DEFINED ENV{DESTDIR})

		execute_process(COMMAND ${PYTHON_EXECUTABLE} ${CONNECTOR_SETUP}
					install ${DISTUTILS_EXTRA_ARGS}
						--root \"\${DESTDIR}\"
						--prefix \"${CMAKE_INSTALL_PREFIX}\")")

	if(NOT CMAKE_SYSTEM_NAME MATCHES "DragonFly.*|FreeBSD")
		install(FILES "${CMAKE_BINARY_DIR}/org.gnome.chrome_gnome_shell.json" DESTINATION "/etc/chromium/native-messaging-hosts/")
		install(FILES "${CMAKE_BINARY_DIR}/org.gnome.chrome_gnome_shell.json" DESTINATION "/etc/opt/chrome/native-messaging-hosts/")
		install(FILES "${CMAKE_BINARY_DIR}/org.gnome.chrome_gnome_shell.json" DESTINATION "/etc/opt/edge/native-messaging-hosts/")
	else(NOT CMAKE_SYSTEM_NAME MATCHES "DragonFly.*|FreeBSD")
		# FreeBSD uses patch that forces Chromium to look into undocumented
		# "/usr/local/etc/chrome/native-messaging-hosts" folder for native messaging host manifest.
		# https://github.com/freebsd/freebsd-ports/blob/master/www/chromium/files/patch-chrome_common_chrome__paths.cc
		install(FILES "${CMAKE_BINARY_DIR}/org.gnome.chrome_gnome_shell.json" DESTINATION "/usr/local/etc/chrome/native-messaging-hosts/")
	endif(NOT CMAKE_SYSTEM_NAME MATCHES "DragonFly.*|FreeBSD")

	install(FILES "${CMAKE_BINARY_DIR}/org.gnome.chrome_gnome_shell.firefox.json"
			DESTINATION "${CMAKE_INSTALL_LIBDIR}/mozilla/native-messaging-hosts/"
			RENAME "org.gnome.chrome_gnome_shell.json")

	install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/connector/org.gnome.ChromeGnomeShell.desktop" DESTINATION "${CMAKE_INSTALL_DATADIR}/applications/")
	install(FILES "${CMAKE_BINARY_DIR}/org.gnome.ChromeGnomeShell.service" DESTINATION "${CMAKE_INSTALL_DATADIR}/dbus-1/services/")

	install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/extension/icons/GnomeLogo-16.png"
		DESTINATION "${CMAKE_INSTALL_DATADIR}/icons/hicolor/16x16/apps/"
		RENAME org.gnome.ChromeGnomeShell.png)
	install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/extension/icons/GnomeLogo-48.png"
		DESTINATION "${CMAKE_INSTALL_DATADIR}/icons/hicolor/48x48/apps/"
		RENAME org.gnome.ChromeGnomeShell.png)
	install(FILES "${CMAKE_CURRENT_SOURCE_DIR}/extension/icons/GnomeLogo-128.png"
		DESTINATION "${CMAKE_INSTALL_DATADIR}/icons/hicolor/128x128/apps/"
		RENAME org.gnome.ChromeGnomeShell.png)
endif(BUILD_CONNECTOR)

if(BUILD_SOURCE_PACKAGE)
	set(MESSAGE_NO_DIST "Unable to generate dist target. Dependencies missing.")

	find_program_ex(xz WARNING)
	find_package(Git)

	if(GIT_FOUND AND XZ_FOUND)
		# http://agateau.com/2009/cmake-and-make-dist-the-simple-version/
		add_custom_target(dist
			COMMAND ${GIT_EXECUTABLE} archive --prefix=${ARCHIVE_NAME}/ HEAD | ${XZ_EXECUTABLE} -z > ${CMAKE_BINARY_DIR}/${ARCHIVE_FULL_NAME}
			WORKING_DIRECTORY ${CMAKE_SOURCE_DIR})
	else(GIT_FOUND AND XZ_FOUND)
		if(BUILD_DEB)
			message(FATAL_ERROR ${MESSAGE_NO_DIST})
		else(BUILD_DEB)
			message(WARNING ${MESSAGE_NO_DIST})
		endif(BUILD_DEB)
	endif(GIT_FOUND AND XZ_FOUND)

	if(BUILD_DEB)
		find_program_ex(debuild FATAL_ERROR)
		find_program_ex(tar FATAL_ERROR)

		if(GPG_KEY)
			message(STATUS "Using GPG key ${GPG_KEY}")
			set(DEBUILD_KEY "-k${GPG_KEY}")
		else(GPG_KEY)
			message(STATUS "GPG_KEY not set. Using default GPG key")
		endif(GPG_KEY)

		configure_file("${CMAKE_CURRENT_SOURCE_DIR}/contrib/ubuntu/changelog"
				"${CMAKE_BINARY_DIR}/debian_changelog")

		add_custom_target(deb_prepare
			DEPENDS dist
			COMMAND ${CMAKE_COMMAND} -E make_directory "${DEB_DIR}/${ARCHIVE_NAME}"
			COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_CURRENT_BINARY_DIR}/${ARCHIVE_FULL_NAME}" "${DEB_DIR}/${ARCHIVE_DEB_NAME}"
			COMMAND ${TAR_EXECUTABLE} -xvf "${DEB_DIR}/${ARCHIVE_DEB_NAME}" -C "${DEB_DIR}/${ARCHIVE_NAME}" --strip-components=1)

		add_custom_target(deb ALL
			DEPENDS deb_prepare
			COMMAND ${CMAKE_COMMAND} -E copy_directory "${CMAKE_CURRENT_SOURCE_DIR}/contrib/ubuntu" "${DEB_DIR}/${ARCHIVE_NAME}/debian"
			COMMAND ${CMAKE_COMMAND} -E copy "${CMAKE_BINARY_DIR}/debian_changelog" "${DEB_DIR}/${ARCHIVE_NAME}/debian/changelog"
			COMMAND ${DEBUILD_EXECUTABLE} -S ${DEBUILD_KEY}
			WORKING_DIRECTORY ${DEB_DIR}/${ARCHIVE_NAME})
	endif(BUILD_DEB)
endif(BUILD_SOURCE_PACKAGE)

if(BUILD_MESSAGES)
	find_package(Gettext)
	if(NOT GETTEXT_FOUND)
		message(FATAL_ERROR "Gettext not found")
	endif(NOT GETTEXT_FOUND)

	add_custom_target(generate-pot
		COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/contrib/chrome-messages2po.py
			--email ${GETTEXT_REPORT_EMAIL}
			--reference-lang en
			--write-pot
			${CMAKE_CURRENT_SOURCE_DIR}/contrib/chrome-web-store/
			${CMAKE_CURRENT_SOURCE_DIR}/extension/_locales/
			${CMAKE_CURRENT_SOURCE_DIR}/po/)

	add_custom_target(update-po
		DEPENDS generate-pot)

	file(GLOB PO_FILES
		LIST_DIRECTORIES false
		RELATIVE ${CMAKE_CURRENT_SOURCE_DIR}/po/
		${CMAKE_CURRENT_SOURCE_DIR}/po/*.po)

	foreach(PO ${PO_FILES})
		add_custom_command(TARGET update-po
			COMMENT "Updating ${PO}"
			WORKING_DIRECTORY ${CMAKE_CURRENT_SOURCE_DIR}/po
			COMMAND ${GETTEXT_MSGMERGE_EXECUTABLE} -q --update
				${PO}
				${CMAKE_CURRENT_SOURCE_DIR}/po/chrome-gnome-shell.pot)
	endforeach()

	add_custom_command(TARGET update-po
		COMMAND ${CMAKE_COMMAND} -E remove ${CMAKE_CURRENT_SOURCE_DIR}/po/*.po~)

	add_custom_target(build-messages ALL
		DEPENDS update-po
		COMMAND ${CMAKE_CURRENT_SOURCE_DIR}/contrib/po2chrome-messages.py
			--reference-lang en
			${CMAKE_CURRENT_SOURCE_DIR}/contrib/chrome-web-store/
			${CMAKE_CURRENT_SOURCE_DIR}/extension/_locales/
			${CMAKE_CURRENT_SOURCE_DIR}/po/)
endif(BUILD_MESSAGES)
