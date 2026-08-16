cmake_minimum_required(VERSION 3.16)
project(DISPatch)

set(CMAKE_CXX_STANDARD 17)
set(CMAKE_CXX_STANDARD_REQUIRED ON)
set(CMAKE_AUTOMOC ON)
set(CMAKE_AUTORCC ON)

include(GNUInstallDirs)

option(DISPATCH_WITH_TESTS "Build DISPatch tests" OFF)
set(DISPATCH_QT_MAJOR "6" CACHE STRING "Qt major version to use")
set_property(CACHE DISPATCH_QT_MAJOR PROPERTY STRINGS 5 6)

if(NOT DISPATCH_QT_MAJOR MATCHES "^(5|6)$")
    message(FATAL_ERROR "DISPATCH_QT_MAJOR must be 5 or 6")
endif()

set(DISPATCH_QT_PACKAGE Qt${DISPATCH_QT_MAJOR})
find_package(${DISPATCH_QT_PACKAGE} REQUIRED COMPONENTS Core Widgets Network)

add_library(DISPatch_core STATIC
        include/DISPatch/AppConfig.h
        include/DISPatch/Constants.h
        include/DISPatch/DisProtocol.h
        include/DISPatch/DisTypes.h
        include/DISPatch/Theme.h
        src/AppConfig.cpp
        src/DisProtocol.cpp
        src/Theme.cpp)
target_include_directories(DISPatch_core PUBLIC include)
target_link_libraries(DISPatch_core PUBLIC
        ${DISPATCH_QT_PACKAGE}::Core
        ${DISPATCH_QT_PACKAGE}::Network)
target_compile_definitions(DISPatch_core PRIVATE
        DISPATCH_SYSTEM_CONFIG_DIR="${CMAKE_INSTALL_FULL_SYSCONFDIR}")

add_executable(DISPatch
        main.cpp
        include/DISPatch/MainWindow.h
        src/MainWindow.cpp
        resources/DISPatch.qrc)
target_include_directories(DISPatch PRIVATE include)
target_link_libraries(DISPatch PRIVATE DISPatch_core ${DISPATCH_QT_PACKAGE}::Widgets)
set_target_properties(DISPatch PROPERTIES OUTPUT_NAME dispatch)
if(UNIX)
    add_custom_command(TARGET DISPatch POST_BUILD
            COMMAND ${CMAKE_COMMAND} -E create_symlink
                    $<TARGET_FILE_NAME:DISPatch>
                    $<TARGET_FILE_DIR:DISPatch>/DISPatch)
endif()

configure_file(etc/dispatch.json dispatch.json COPYONLY)
configure_file(etc/dispatch_debug.json dispatch_debug.json COPYONLY)

install(TARGETS DISPatch
        RUNTIME DESTINATION ${CMAKE_INSTALL_BINDIR})
install(FILES etc/dispatch.json etc/dispatch_debug.json
        DESTINATION ${CMAKE_INSTALL_SYSCONFDIR})

if(DISPATCH_WITH_TESTS)
    find_package(Catch2 REQUIRED)
    enable_testing()
    add_subdirectory(tests)
    add_custom_target(check
            COMMAND ${CMAKE_CTEST_COMMAND} --output-on-failure
            DEPENDS DISPatch_tests
            USES_TERMINAL)
endif()
