# Define GNU standard installation directories
include(GNUInstallDirs)

# Generate git-version.h at build time.
#include(${CMAKE_CURRENT_SOURCE_DIR}/git-version.cmake)

# Check for a sufficient compiler and set build options
include(ConfigureCompiler)
include(CheckFunctionExists)

set(CMAKE_CXX_STANDARD 20)

set(ADDITIONAL_LIBS "")
if(CMAKE_SYSTEM_NAME STREQUAL "Android")
	set(ADDITIONAL_LIBS ${ADDITIONAL_LIBS} "android")
elseif(CMAKE_SYSTEM_NAME STREQUAL "Linux")
    #on some Linux distros shm_unlink and similar functions are in librt only
    set(ADDITIONAL_LIBS ${ADDITIONAL_LIBS} "rt")
elseif(NOT WIN32 AND NOT CMAKE_CXX_FLAGS MATCHES "LIBICONV_PLUG")
    #it seems like glibc includes the iconv functions we use but other libc
    #implementations like the one on OSX don't seem implement them
    set(ADDITIONAL_LIBS ${ADDITIONAL_LIBS} "iconv")
endif()

if(ANDROID)
    add_library(iconv INTERFACE)
    if (BUILD_IN_WINDOWS)

        target_include_directories(iconv INTERFACE ${CMAKE_SOURCE_DIR}/iconv_out/include)
        target_link_libraries(iconv INTERFACE
                "${CMAKE_SOURCE_DIR}/iconv_out/lib/libcharset.a"
                "${CMAKE_SOURCE_DIR}/iconv_out/lib/libiconv.a")
    else ()
    target_include_directories(iconv INTERFACE /data/data/com.termux/files/home/iconv_out/include)
	target_link_libraries(iconv INTERFACE
	"/data/data/com.termux/files/home/iconv_out/lib/libcharset.a"
	"/data/data/com.termux/files/home/iconv_out/lib/libiconv.a")

    endif ()
endif()

if(ANDROID)
    #add_definitions(-DDATADIR="/storage/emulated/0/aps3e")
elseif(UNIX AND NOT APPLE)
    add_definitions(-DDATADIR="${CMAKE_INSTALL_FULL_DATADIR}/rpcs3")
    # Optionally enable X11 for window management
    find_package(X11)
    if(X11_FOUND)
        add_definitions(-DHAVE_X11)
    endif()
    find_package(Wayland)
    if(WAYLAND_FOUND)
        add_definitions(-DHAVE_WAYLAND)
    endif()
endif()

# Qt
# finds Qt libraries and setups custom commands for MOC and UIC
# Must be done here because generated MOC and UIC targets cant
# be found otherwise
if(NOT ANDROID)
include(${CMAKE_SOURCE_DIR}/3rdparty/qt6.cmake)
endif()

# subdirectories
add_subdirectory(Emu)

if(ANDROID)
    return()
endif ()

if(NOT ANDROID)
add_subdirectory(rpcs3qt)
endif()

if(WIN32)
    add_executable(rpcs3 WIN32)
    target_sources(rpcs3 PRIVATE rpcs3.rc)
    target_compile_definitions(rpcs3 PRIVATE UNICODE _UNICODE)
elseif(APPLE)
    add_executable(rpcs3 MACOSX_BUNDLE)
    target_sources(rpcs3 PRIVATE rpcs3.icns update_helper.sh)
    set_source_files_properties(update_helper.sh PROPERTIES MACOSX_PACKAGE_LOCATION Resources)
    set_target_properties(rpcs3
        PROPERTIES
            MACOSX_BUNDLE_INFO_PLIST "${CMAKE_CURRENT_SOURCE_DIR}/rpcs3.plist.in")

elseif(ANDROID)
    add_library(rpcs3 SHARED)
    set_target_properties(rpcs3 PROPERTIES OUTPUT_NAME "e")

    if (BUILD_IN_WINDOWS)
        target_link_libraries(rpcs3 PRIVATE "{CMAKE_SOURCE_DIR}/iso_t/libiso_t.a")
    else ()
        target_link_libraries(rpcs3 PRIVATE "/data/data/com.termux/files/home/iso_t/target/aarch64-linux-android/release/libiso_t.a")
    endif ()
else()
    add_executable(rpcs3)
endif()

if(ANDROID)
target_sources(rpcs3
    PRIVATE
    Input/raw_mouse_config.cpp
    Input/raw_mouse_handler.cpp
    Input/product_info.cpp
    rpcs3_version.cpp
    stb_image.cpp
    stdafx.cpp
    aps3e.cpp
)
else()
target_sources(rpcs3
    PRIVATE
    display_sleep_control.cpp
    headless_application.cpp
    main.cpp
    main_application.cpp
    module_verifier.cpp
    rpcs3_version.cpp
    stb_image.cpp
    stdafx.cpp

    Input/basic_keyboard_handler.cpp
    Input/basic_mouse_handler.cpp
    Input/ds3_pad_handler.cpp
    Input/ds4_pad_handler.cpp
    Input/dualsense_pad_handler.cpp
    Input/evdev_joystick_handler.cpp
    Input/evdev_gun_handler.cpp
    Input/gui_pad_thread.cpp
    Input/hid_pad_handler.cpp
    Input/keyboard_pad_handler.cpp
    Input/mm_joystick_handler.cpp
    Input/pad_thread.cpp
    Input/product_info.cpp
    Input/raw_mouse_config.cpp
    Input/raw_mouse_handler.cpp
    Input/sdl_pad_handler.cpp
    Input/skateboard_pad_handler.cpp
    Input/xinput_pad_handler.cpp
)
endif()
gen_git_version(${CMAKE_CURRENT_SOURCE_DIR})
set_target_properties(rpcs3
    PROPERTIES
        AUTOMOC ON
        AUTOUIC ON)


target_link_libraries(rpcs3 PRIVATE rpcs3_emu)
target_link_libraries(rpcs3 PRIVATE 3rdparty::yaml-cpp 3rdparty::discordRPC 3rdparty::hidapi 3rdparty::libusb 3rdparty::wolfssl 3rdparty::libcurl 3rdparty::zlib 3rdparty::glslang)
target_link_libraries(rpcs3 PRIVATE ${ADDITIONAL_LIBS})

if(NOT ANDROID)
target_link_libraries(rpcs3 PRIVATE rpcs3_ui 3rdparty::qt6)
endif()

if(ANDROID)
    message(STATUS "display manager does not exist for Android")
# Unix display manager
elseif(X11_FOUND)
    target_link_libraries(rpcs3 PRIVATE X11::X11)
elseif(USE_VULKAN AND UNIX AND NOT WAYLAND_FOUND AND NOT APPLE)
    # Wayland has been checked in 3rdparty/CMakeLists.txt already.
    message(FATAL_ERROR "RPCS3 requires either X11 or Wayland (or both) for Vulkan.")
endif()

if(UNIX)
    set(CMAKE_THREAD_PREFER_PTHREAD TRUE)
    find_package(Threads REQUIRED)
    target_link_libraries(rpcs3 PRIVATE Threads::Threads)
endif()

if(WIN32)
    target_link_libraries(rpcs3 PRIVATE bcrypt ws2_32 Iphlpapi Winmm Psapi gdi32 setupapi pdh)
else()
    target_link_libraries(rpcs3 PRIVATE ${CMAKE_DL_LIBS})
endif()

if(USE_PRECOMPILED_HEADERS)
    target_precompile_headers(rpcs3 PRIVATE stdafx.h)
endif()

if(ANDROID)
    return()
endif ()

# Copy icons to executable directory
if(APPLE)
    if (CMAKE_BUILD_TYPE MATCHES "Debug" OR CMAKE_BUILD_TYPE MATCHES "RelWithDebInfo")
        set(QT_DEPLOY_FLAGS "-no-strip")
    else()
        set(QT_DEPLOY_FLAGS "")
    endif()
    qt_finalize_target(rpcs3)
    add_custom_command(TARGET rpcs3 POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy ${CMAKE_CURRENT_SOURCE_DIR}/rpcs3.icns $<TARGET_FILE_DIR:rpcs3>/../Resources/rpcs3.icns
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/bin/Icons $<TARGET_FILE_DIR:rpcs3>/../Resources/Icons
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/bin/GuiConfigs $<TARGET_FILE_DIR:rpcs3>/../Resources/GuiConfigs
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/bin/git $<TARGET_FILE_DIR:rpcs3>/../Resources/git
        COMMAND "${MACDEPLOYQT_EXECUTABLE}" "${PROJECT_BINARY_DIR}/bin/rpcs3.app" "${QT_DEPLOY_FLAGS}")
elseif(UNIX)
    add_custom_command(TARGET rpcs3 POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/bin/Icons $<TARGET_FILE_DIR:rpcs3>/Icons
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/bin/GuiConfigs $<TARGET_FILE_DIR:rpcs3>/GuiConfigs
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/bin/git $<TARGET_FILE_DIR:rpcs3>/git)
elseif(WIN32)
    if(MSVC)
        add_custom_command(TARGET rpcs3 POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E copy ${PROJECT_BINARY_DIR}/3rdparty/OpenAL/openal-soft/$<CONFIG>/OpenAL32.dll $<TARGET_FILE_DIR:rpcs3>)
    endif()
    add_custom_command(TARGET rpcs3 POST_BUILD
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/bin/Icons $<TARGET_FILE_DIR:rpcs3>/Icons
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/bin/GuiConfigs $<TARGET_FILE_DIR:rpcs3>/GuiConfigs
        COMMAND ${CMAKE_COMMAND} -E copy_directory ${CMAKE_SOURCE_DIR}/bin/git $<TARGET_FILE_DIR:rpcs3>/git
        COMMAND "${WINDEPLOYQT_EXECUTABLE}" --no-compiler-runtime --no-opengl-sw --no-patchqt
            --no-translations --no-quick --no-system-d3d-compiler --no-quick-import
            --plugindir "$<IF:$<CXX_COMPILER_ID:MSVC>,$<TARGET_FILE_DIR:rpcs3>/plugins,$<TARGET_FILE_DIR:rpcs3>/share/qt6/plugins>"
            --verbose 0 "$<TARGET_FILE:rpcs3>")
endif()

# Unix installation
if(UNIX AND NOT APPLE)
    # Install the binary
    install(TARGETS rpcs3 RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
    # Install the application icon and menu item
    install(FILES rpcs3.svg
        DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/scalable/apps)
    install(FILES rpcs3.png
        DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/icons/hicolor/48x48/apps)
    install(FILES rpcs3.desktop
        DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/applications)
    install(FILES rpcs3.metainfo.xml
        DESTINATION ${CMAKE_INSTALL_DATAROOTDIR}/metainfo)
    # Install other files
    install(DIRECTORY ../bin/Icons
        DESTINATION ${CMAKE_INSTALL_DATADIR}/rpcs3)
    install(DIRECTORY ../bin/GuiConfigs
        DESTINATION ${CMAKE_INSTALL_DATADIR}/rpcs3)
    install(DIRECTORY ../bin/git
        DESTINATION ${CMAKE_INSTALL_DATADIR}/rpcs3)
    install(DIRECTORY ../bin/test
        DESTINATION ${CMAKE_INSTALL_DATADIR}/rpcs3)
endif()
