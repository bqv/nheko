cmake_minimum_required(VERSION 3.13)

option(APPVEYOR_BUILD "Build on appveyor" OFF)
option(CI_BUILD "Set when building in CI. Enables -Werror where possible" OFF)
option(ASAN "Compile with address sanitizers" OFF)
option(QML_DEBUGGING "Enable qml debugging" OFF)

set(
	CMAKE_TOOLCHAIN_FILE "${CMAKE_CURRENT_LIST_DIR}/toolchain.cmake"
	CACHE
	FILEPATH "Default toolchain"
	)


option(HUNTER_ENABLED "Enable Hunter package manager" OFF)
include("cmake/HunterGate.cmake")
HunterGate(
	URL "https://github.com/cpp-pm/hunter/archive/v0.23.244.tar.gz"
	SHA1 "2c0f491fd0b80f7b09e3d21adb97237161ef9835"
	LOCAL
	)

option(USE_BUNDLED_BOOST "Use the bundled version of Boost." ${HUNTER_ENABLED})
option(USE_BUNDLED_SPDLOG "Use the bundled version of spdlog."
	${HUNTER_ENABLED})
option(USE_BUNDLED_OLM "Use the bundled version of libolm." ${HUNTER_ENABLED})
option(USE_BUNDLED_GTEST "Use the bundled version of Google Test."
	${HUNTER_ENABLED})
option(USE_BUNDLED_CMARK "Use the bundled version of cmark."
	${HUNTER_ENABLED})
option(USE_BUNDLED_JSON "Use the bundled version of nlohmann json."
	${HUNTER_ENABLED})
option(USE_BUNDLED_OPENSSL "Use the bundled version of OpenSSL."
	${HUNTER_ENABLED})
option(USE_BUNDLED_MTXCLIENT "Use the bundled version of the Matrix Client library." ${HUNTER_ENABLED})
option(USE_BUNDLED_SODIUM "Use the bundled version of libsodium."
	${HUNTER_ENABLED})
option(USE_BUNDLED_ZLIB "Use the bundled version of zlib."
	${HUNTER_ENABLED})
option(USE_BUNDLED_LMDB "Use the bundled version of lmdb."
	${HUNTER_ENABLED})
option(USE_BUNDLED_LMDBXX "Use the bundled version of lmdb++."
	${HUNTER_ENABLED})
option(USE_BUNDLED_TWEENY "Use the bundled version of tweeny."
	${HUNTER_ENABLED})

list(APPEND CMAKE_MODULE_PATH "${CMAKE_SOURCE_DIR}/cmake")

if(${CMAKE_VERSION} VERSION_LESS "3.14.0")
	message("Adding FetchContent_MakeAvailable")
	# from cmakes sources
	macro(FetchContent_MakeAvailable)

		foreach(contentName IN ITEMS ${ARGV})
			string(TOLOWER ${contentName} contentNameLower)
			FetchContent_GetProperties(${contentName})
			if(NOT ${contentNameLower}_POPULATED)
				FetchContent_Populate(${contentName})

				# Only try to call add_subdirectory() if the populated content
				# can be treated that way. Protecting the call with the check
				# allows this function to be used for projects that just want
				# to ensure the content exists, such as to provide content at
				# a known location.
				if(EXISTS ${${contentNameLower}_SOURCE_DIR}/CMakeLists.txt)
					add_subdirectory(${${contentNameLower}_SOURCE_DIR}
						${${contentNameLower}_BINARY_DIR})
				endif()
			endif()
		endforeach()

	endmacro()
endif()

include(GNUInstallDirs)

# Include Qt basic functions
include(QtCommon)

project(nheko LANGUAGES CXX C)
set(CPACK_PACKAGE_VERSION_MAJOR "0")
set(CPACK_PACKAGE_VERSION_MINOR "7")
set(CPACK_PACKAGE_VERSION_PATCH "1")
set(PROJECT_VERSION_MAJOR ${CPACK_PACKAGE_VERSION_MAJOR})
set(PROJECT_VERSION_MINOR ${CPACK_PACKAGE_VERSION_MINOR})
set(PROJECT_VERSION_PATCH ${CPACK_PACKAGE_VERSION_PATCH})

# Set PROJECT_VERSION_PATCH & PROJECT_VERSION_TWEAK to 0 if not present
# Needed by add_project_meta.
fix_project_version()

# Set additional project information
set(COMPANY "Nheko")
set(COPYRIGHT "Copyright (c) 2019 Nheko Contributors")
set(IDENTIFIER "com.github.mujx.nheko")

add_project_meta(META_FILES_TO_INCLUDE)

if(NOT MSVC AND NOT APPLE)
	set(THREADS_PREFER_PTHREAD_FLAG ON)
	find_package(Threads REQUIRED)
endif()

if (BUILD_DOCS)
	find_package(Doxygen)

	if (DOXYGEN_FOUND)
		set(DOXYGEN_IN ${CMAKE_CURRENT_SOURCE_DIR}/cmake/Doxyfile.in)
		set(DOXYGEN_OUT ${CMAKE_CURRENT_BINARY_DIR}/Doxyfile)

		configure_file(${DOXYGEN_IN} ${DOXYGEN_OUT})

		add_custom_target(docs ALL
			COMMAND ${DOXYGEN_EXECUTABLE} ${DOXYGEN_OUT}
			WORKING_DIRECTORY ${CMAKE_CURRENT_BINARY_DIR}
			COMMENT "Generating API documentation with Doxygen"
			VERBATIM )
	else (DOXYGEN_FOUND)
		message("Doxygen need to be installed to generate the doxygen documentation")
	endif (DOXYGEN_FOUND)
endif()


#
# LMDB
#
#include(LMDB)
if(USE_BUNDLED_LMDB)
	hunter_add_package(lmdb)
	find_package(liblmdb CONFIG REQUIRED)
else()
	find_package(LMDB)
endif()

#
# Discover Qt dependencies.
#
find_package(Qt5 COMPONENTS Core Widgets LinguistTools Concurrent Svg Multimedia Qml QuickControls2 QuickWidgets REQUIRED)
find_package(Qt5QuickCompiler)
find_package(Qt5DBus)

if (APPLE)
	find_package(Qt5MacExtras REQUIRED)
endif(APPLE)

if (Qt5Widgets_FOUND)
	if (Qt5Widgets_VERSION VERSION_LESS 5.9.0)
		message(STATUS "Qt version ${Qt5Widgets_VERSION}")
		message(WARNING "Minimum supported Qt5 version is 5.9!")
	endif()
endif(Qt5Widgets_FOUND)

set(CMAKE_INCLUDE_CURRENT_DIR ON)
if(NOT MSVC)
	set(
		CMAKE_CXX_FLAGS
		"${CMAKE_CXX_FLAGS} \
		-Wall \
		-Wextra \
		-pipe \
		-pedantic \
		-fsized-deallocation \
		-fdiagnostics-color=always \
		-Wunreachable-code \
		-std=c++17"
		)
	if (NOT CMAKE_COMPILER_IS_GNUCXX)
		# -Wshadow is buggy and broken in GCC, so do not enable it.
		# see https://gcc.gnu.org/bugzilla/show_bug.cgi?id=79328
		set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -Wshadow")
	endif()
endif()

if (MSVC)
	set(
		CMAKE_CXX_FLAGS
		"${CMAKE_CXX_FLAGS} /bigobj"
		)
endif()

if(NOT (CMAKE_BUILD_TYPE OR CMAKE_CONFIGURATION_TYPES))
	set(CMAKE_BUILD_TYPE "Debug" CACHE STRING
		"Choose the type of build, options are: None Debug Release RelWithDebInfo MinSizeRel."
		FORCE)
	message("Setting build type to '${CMAKE_BUILD_TYPE}'")
else(NOT (CMAKE_BUILD_TYPE OR CMAKE_CONFIGURATION_TYPES))
	message("Build type set to '${CMAKE_BUILD_TYPE}'")
endif(NOT (CMAKE_BUILD_TYPE OR CMAKE_CONFIGURATION_TYPES))

set(SPDLOG_DEBUG_ON false)

# Windows doesn't handle CMAKE_BUILD_TYPE.
if(NOT WIN32)
	if(${CMAKE_BUILD_TYPE} STREQUAL "Debug")
		set(SPDLOG_DEBUG_ON true)
	else()
		set(SPDLOG_DEBUG_ON false)
	endif()
endif()

find_program(GIT git)
if(GIT)
	execute_process(
		WORKING_DIRECTORY ${CMAKE_SOURCE_DIR}
		COMMAND ${GIT} rev-parse --short HEAD
		OUTPUT_VARIABLE GIT_OUT OUTPUT_STRIP_TRAILING_WHITESPACE
		)
	if(GIT_OUT)
		set(CPACK_PACKAGE_VERSION_PATCH "${CPACK_PACKAGE_VERSION_PATCH}-${GIT_OUT}")
	else()
		set(CPACK_PACKAGE_VERSION_PATCH "${CPACK_PACKAGE_VERSION_PATCH}")
	endif()
endif(GIT)

set(CPACK_PACKAGE_VERSION ${CPACK_PACKAGE_VERSION_MAJOR}.${CPACK_PACKAGE_VERSION_MINOR}.${CPACK_PACKAGE_VERSION_PATCH})
set(PROJECT_VERSION ${CPACK_PACKAGE_VERSION})

message(STATUS "Version: ${PROJECT_VERSION}")

cmake_host_system_information(RESULT BUILD_HOST QUERY HOSTNAME)
set(BUILD_USER $ENV{USER})
configure_file(cmake/nheko.h config/nheko.h)


#
# Declare source and header files.
#
set(SRC_FILES
	# Dialogs
	src/dialogs/CreateRoom.cpp
	src/dialogs/FallbackAuth.cpp
	src/dialogs/ImageOverlay.cpp
	src/dialogs/InviteUsers.cpp
	src/dialogs/JoinRoom.cpp
	src/dialogs/LeaveRoom.cpp
	src/dialogs/Logout.cpp
	src/dialogs/MemberList.cpp
	src/dialogs/PreviewUploadOverlay.cpp
	src/dialogs/ReCaptcha.cpp
	src/dialogs/ReadReceipts.cpp
	src/dialogs/RoomSettings.cpp
	src/dialogs/UserProfile.cpp

	# Emoji
	src/emoji/Category.cpp
	src/emoji/ItemDelegate.cpp
	src/emoji/Panel.cpp
	src/emoji/PickButton.cpp
	src/emoji/Provider.cpp

	# Timeline
	src/timeline/TimelineViewManager.cpp
	src/timeline/TimelineModel.cpp
	src/timeline/DelegateChooser.cpp

	# UI components
	src/ui/Avatar.cpp
	src/ui/Badge.cpp
	src/ui/DropShadow.cpp
	src/ui/LoadingIndicator.cpp
	src/ui/InfoMessage.cpp
	src/ui/FlatButton.cpp
	src/ui/FloatingButton.cpp
	src/ui/Label.cpp
	src/ui/OverlayModal.cpp
	src/ui/SnackBar.cpp
	src/ui/RaisedButton.cpp
	src/ui/Ripple.cpp
	src/ui/RippleOverlay.cpp
	src/ui/OverlayWidget.cpp
	src/ui/TextField.cpp
	src/ui/TextLabel.cpp
	src/ui/ToggleButton.cpp
	src/ui/Theme.cpp
	src/ui/ThemeManager.cpp

	src/AvatarProvider.cpp
	src/BlurhashProvider.cpp
	src/Cache.cpp
	src/ChatPage.cpp
	src/ColorImageProvider.cpp
	src/CommunitiesList.cpp
	src/CommunitiesListItem.cpp
	src/EventAccessors.cpp
	src/InviteeItem.cpp
	src/Logging.cpp
	src/LoginPage.cpp
	src/MainWindow.cpp
	src/MatrixClient.cpp
	src/MxcImageProvider.cpp
	src/Olm.cpp
	src/QuickSwitcher.cpp
	src/RegisterPage.cpp
	src/RoomInfoListItem.cpp
	src/RoomList.cpp
	src/SideBarActions.cpp
	src/Splitter.cpp
	src/TextInputWidget.cpp
	src/TopRoomBar.cpp
	src/TrayIcon.cpp
	src/UserInfoWidget.cpp
	src/UserSettingsPage.cpp
	src/Utils.cpp
	src/WelcomePage.cpp
	src/popups/PopupItem.cpp
	src/popups/SuggestionsPopup.cpp
	src/popups/UserMentions.cpp
	src/main.cpp

	third_party/blurhash/blurhash.cpp
	)


include(FeatureSummary)

if(USE_BUNDLED_BOOST)
	hunter_add_package(Boost COMPONENTS iostreams system thread)
endif()
find_package(Boost 1.70 REQUIRED
	COMPONENTS iostreams
	system
	thread)
if(USE_BUNDLED_ZLIB)
	hunter_add_package(ZLIB)
endif()
find_package(ZLIB REQUIRED)
if(USE_BUNDLED_OPENSSL)
	hunter_add_package(OpenSSL)
endif()
find_package(OpenSSL REQUIRED)
if(USE_BUNDLED_MTXCLIENT)
	include(FetchContent)
	set(BUILD_LIB_EXAMPLES OFF CACHE INTERNAL "")
	set(BUILD_LIB_TESTS OFF CACHE INTERNAL "")
	FetchContent_Declare(
		MatrixClient
		GIT_REPOSITORY https://github.com/Nheko-Reborn/mtxclient.git
		GIT_TAG        v0.3.0
		)
	FetchContent_MakeAvailable(MatrixClient)
else()
	find_package(MatrixClient 0.3.0 REQUIRED)
endif()
if(USE_BUNDLED_OLM)
	include(FetchContent)
	set(OLM_TESTS OFF CACHE INTERNAL "")
	FetchContent_Declare(
		Olm
		GIT_REPOSITORY https://gitlab.matrix.org/matrix-org/olm.git
		GIT_TAG        3.1.4
		)
	FetchContent_MakeAvailable(Olm)
else()
	find_package(Olm 3)
	set_package_properties(Olm PROPERTIES
		DESCRIPTION "An implementation of the Double Ratchet cryptographic ratchet"
		URL "https://git.matrix.org/git/olm/about/"
		TYPE REQUIRED
		)
endif()
if(USE_BUNDLED_SPDLOG)
	hunter_add_package(spdlog)
endif()
find_package(spdlog 1.0.0 CONFIG REQUIRED)

if(USE_BUNDLED_CMARK)
	include(FetchContent)
	FetchContent_Declare(
		cmark
		GIT_REPOSITORY https://github.com/commonmark/cmark.git
		GIT_TAG        242e277a661ec7e51f34dcaf86c1925d550b1498 #0.29.0 << doesn't work with fetch content yet
		CMAKE_ARGS     "CMARK_STATIC=ON CMARK_SHARED=OFF CMARK_TESTS=OFF CMARK_TESTS=OFF"
		)
	FetchContent_MakeAvailable(cmark)
	if (MSVC)
		add_library(cmark::cmark ALIAS libcmark)
	else()
		add_library(cmark::cmark ALIAS libcmark_static)
	endif()
else()
	find_package(cmark REQUIRED 0.29.0)
endif()

if(USE_BUNDLED_JSON)
	hunter_add_package(nlohmann_json)
endif()
find_package(nlohmann_json 3.2.0)
set_package_properties(nlohmann_json PROPERTIES
	DESCRIPTION "JSON for Modern C++, a C++11 header-only JSON class"
	URL "https://nlohmann.github.io/json/"
	TYPE REQUIRED
	)

if(USE_BUNDLED_LMDBXX)
	hunter_add_package(lmdbxx)
	find_package(lmdbxx CONFIG REQUIRED)
else()
	if(NOT LMDBXX_INCLUDE_DIR)
		find_path(LMDBXX_INCLUDE_DIR
			NAMES lmdb++.h
			PATHS /usr/include
			/usr/local/include
			$ENV{LIB_DIR}/include
			$ENV{LIB_DIR}/include/lmdbxx)

	endif()
	add_library(lmdbxx INTERFACE)
	target_include_directories(lmdbxx INTERFACE ${LMDBXX_INCLUDE_DIR})
	add_library(lmdbxx::lmdbxx ALIAS lmdbxx)
endif()

if(USE_BUNDLED_TWEENY)
	include(FetchContent)
	FetchContent_Declare(
		Tweeny
		GIT_REPOSITORY https://github.com/mobius3/tweeny.git
		GIT_TAG        6a5033372fe53c4c731c66c8a2d56261746cd85c #v3 <- v3 has unfixed warnings
		)
	FetchContent_MakeAvailable(Tweeny)
else()
	find_package(Tweeny REQUIRED)
endif()

# single instance functionality
set(QAPPLICATION_CLASS QApplication CACHE STRING "Inheritance class for SingleApplication")
add_subdirectory(third_party/SingleApplication-3.0.19/)

feature_summary(WHAT ALL INCLUDE_QUIET_PACKAGES FATAL_ON_MISSING_REQUIRED_PACKAGES)

qt5_wrap_cpp(MOC_HEADERS
	# Dialogs
	src/dialogs/CreateRoom.h
	src/dialogs/FallbackAuth.h
	src/dialogs/ImageOverlay.h
	src/dialogs/InviteUsers.h
	src/dialogs/JoinRoom.h
	src/dialogs/LeaveRoom.h
	src/dialogs/Logout.h
	src/dialogs/MemberList.h
	src/dialogs/PreviewUploadOverlay.h
	src/dialogs/RawMessage.h
	src/dialogs/ReCaptcha.h
	src/dialogs/ReadReceipts.h
	src/dialogs/RoomSettings.h
	src/dialogs/UserProfile.h

	# Emoji
	src/emoji/Category.h
	src/emoji/ItemDelegate.h
	src/emoji/Panel.h
	src/emoji/PickButton.h

	# Timeline
	src/timeline/TimelineViewManager.h
	src/timeline/TimelineModel.h
	src/timeline/DelegateChooser.h

	# UI components
	src/ui/Avatar.h
	src/ui/Badge.h
	src/ui/LoadingIndicator.h
	src/ui/InfoMessage.h
	src/ui/FlatButton.h
	src/ui/Label.h
	src/ui/FloatingButton.h
	src/ui/Menu.h
	src/ui/OverlayWidget.h
	src/ui/SnackBar.h
	src/ui/RaisedButton.h
	src/ui/Ripple.h
	src/ui/RippleOverlay.h
	src/ui/TextField.h
	src/ui/TextLabel.h
	src/ui/ToggleButton.h
	src/ui/Theme.h
	src/ui/ThemeManager.h

	src/notifications/Manager.h

	src/AvatarProvider.h
	src/BlurhashProvider.h
	src/Cache_p.h
	src/ChatPage.h
	src/CommunitiesList.h
	src/CommunitiesListItem.h
	src/InviteeItem.h
	src/LoginPage.h
	src/MainWindow.h
	src/MxcImageProvider.h
	src/QuickSwitcher.h
	src/RegisterPage.h
	src/RoomInfoListItem.h
	src/RoomList.h
	src/SideBarActions.h
	src/Splitter.h
	src/TextInputWidget.h
	src/TopRoomBar.h
	src/TrayIcon.h
	src/UserInfoWidget.h
	src/UserSettingsPage.h
	src/WelcomePage.h
	src/popups/PopupItem.h
	src/popups/SuggestionsPopup.h
	src/popups/UserMentions.h
	)

#
# Bundle translations.
#
include(Translations)
set(TRANSLATION_DEPS ${LANG_QRC} ${QRC} ${QM_SRC})

if (APPLE)
	set(CMAKE_EXE_LINKER_FLAGS "${CMAKE_EXE_LINKER_FLAGS} -framework Foundation -framework Cocoa")
	set(SRC_FILES ${SRC_FILES} src/notifications/ManagerMac.mm src/emoji/MacHelper.mm)
elseif (WIN32)
	file(DOWNLOAD
		"https://raw.githubusercontent.com/mohabouje/WinToast/41ed1c58d5dce0ee9c01dbdeac05be45358d4f57/src/wintoastlib.cpp"
		${PROJECT_SOURCE_DIR}/src/wintoastlib.cpp
		EXPECTED_HASH SHA256=1A1A7CE41C1052B12946798F4A6C67CE1FAD209C967F5ED4D720B173527E2073)

	file(DOWNLOAD
		"https://raw.githubusercontent.com/mohabouje/WinToast/41ed1c58d5dce0ee9c01dbdeac05be45358d4f57/src/wintoastlib.h"
		${PROJECT_SOURCE_DIR}/src/wintoastlib.h
		EXPECTED_HASH SHA256=b4481023c5782733795838be22bf1a75f45d87458cd4d9a5a75f664a146eea11)

	set(SRC_FILES ${SRC_FILES} src/notifications/ManagerWin.cpp src/wintoastlib.cpp)
else ()
	set(SRC_FILES ${SRC_FILES} src/notifications/ManagerLinux.cpp)
endif ()

set(NHEKO_DEPS
	${SRC_FILES}
	${UI_HEADERS}
	${MOC_HEADERS}
	${TRANSLATION_DEPS}
	${META_FILES_TO_INCLUDE})

if(ASAN)
	set(CMAKE_CXX_FLAGS "${CMAKE_CXX_FLAGS} -fsanitize=address,undefined")
endif()

if(WIN32)
	add_executable (nheko WIN32 ${OS_BUNDLE} ${NHEKO_DEPS})
else()
	add_executable (nheko ${OS_BUNDLE} ${NHEKO_DEPS})
endif()

if(APPLE)
	target_link_libraries (nheko PRIVATE Qt5::MacExtras)
elseif(WIN32)
	target_compile_definitions(nheko PRIVATE WIN32_LEAN_AND_MEAN)
	target_link_libraries (nheko PRIVATE ${NTDLIB} Qt5::WinMain)
else()
	target_link_libraries (nheko PRIVATE Qt5::DBus)
endif()
target_include_directories(nheko PRIVATE src includes third_party/blurhash)

target_link_libraries(nheko PRIVATE
	MatrixClient::MatrixClient
	Boost::iostreams
	Boost::system
	Boost::thread
	cmark::cmark
	spdlog::spdlog
	Qt5::Widgets
	Qt5::Svg
	Qt5::Concurrent
	Qt5::Multimedia
	Qt5::Qml
	Qt5::QuickControls2
	Qt5::QuickWidgets
	nlohmann_json::nlohmann_json
	lmdbxx::lmdbxx
	liblmdb::lmdb
	tweeny
	SingleApplication::SingleApplication)

if(MSVC)
	target_link_libraries(nheko PRIVATE ntdll)
endif()


if(QML_DEBUGGING)
	target_compile_definitions(nheko PRIVATE QML_DEBUGGING)
endif()


if(NOT MSVC)
	if("${CMAKE_BUILD_TYPE}" STREQUAL "Debug" OR CI_BUILD)
		target_compile_options(nheko PRIVATE "-Werror")
	endif()
endif()

set_target_properties(nheko PROPERTIES SKIP_BUILD_RPATH TRUE)

if(UNIX AND NOT APPLE)
	install (TARGETS nheko RUNTIME DESTINATION "${CMAKE_INSTALL_BINDIR}")
	install (FILES "resources/nheko-16.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/16x16/apps" RENAME "nheko.png")
	install (FILES "resources/nheko-32.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/32x32/apps" RENAME "nheko.png")
	install (FILES "resources/nheko-48.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/48x48/apps" RENAME "nheko.png")
	install (FILES "resources/nheko-64.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/64x64/apps" RENAME "nheko.png")
	install (FILES "resources/nheko-128.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/128x128/apps" RENAME "nheko.png")
	install (FILES "resources/nheko-256.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/256x256/apps" RENAME "nheko.png")
	install (FILES "resources/nheko-512.png" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/512x512/apps" RENAME "nheko.png")
	install (FILES "resources/nheko.svg" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/scalable/apps" RENAME "nheko.svg")
	install (FILES "resources/nheko.desktop" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/applications")
	install (FILES "resources/nheko.appdata.xml" DESTINATION "${CMAKE_INSTALL_DATAROOTDIR}/metainfo")

	if(NOT TARGET uninstall)
		configure_file(
			"${CMAKE_CURRENT_SOURCE_DIR}/cmake/cmake_uninstall.cmake.in"
			"${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake"
			IMMEDIATE @ONLY)
		add_custom_target(uninstall
			COMMAND ${CMAKE_COMMAND} -P ${CMAKE_CURRENT_BINARY_DIR}/cmake_uninstall.cmake)
	endif()
endif()
