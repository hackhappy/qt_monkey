#tested with such version, if works with previous also, inform me
cmake_minimum_required(VERSION 2.8)
#qtmain policy
cmake_policy(SET CMP0020 NEW)
project(qt_monkey)

include(compiler.cmake)

set(USE_TESTS False CACHE BOOL "enable testing")
set(QT_VARIANT "qt5" CACHE STRING "variant of qt: qt4 or qt5")

if ((NOT ("${QT_VARIANT}" STREQUAL "qt4")) AND
    (NOT ("${QT_VARIANT}" STREQUAL "qt5")))
    message(FATAL_ERROR "only qt4 and qt5 supported")
endif ()

if ("${QT_VARIANT}" STREQUAL "qt4")
  set(QT_USE_QTSCRIPT True)
  set(QT_USE_QTNETWORK True)
  set(QT_USE_QTTEST True)
  find_package(Qt4 REQUIRED)
  include(${QT_USE_FILE})
  include_directories(${QT_INCLUDE_DIR})
  add_definitions(${QT_DEFINITIONS})
elseif ("${QT_VARIANT}" STREQUAL "qt5")
  find_package(Qt5Widgets REQUIRED)
  find_package(Qt5Network REQUIRED)
  find_package(Qt5Test REQUIRED)
  find_package(Qt5Script REQUIRED)
  include_directories(${Qt5Widgets_INCLUDE_DIRS})
  set(QT_LIBRARIES Qt5::Widgets Qt5::Network Qt5::Test Qt5::Script)
endif ()
message(STATUS "Build with ${QT_VARIANT} support")
include_directories(${CMAKE_CURRENT_SOURCE_DIR})
include_directories(${CMAKE_CURRENT_BINARY_DIR})

set(qtmonkey_agent_MOC_HDRS
  user_events_analyzer.hpp
  agent_qtmonkey_communication.hpp
  agent.hpp
  script_api.hpp
  )

set(qtmonkey_app_MOC_HDRS
  agent_qtmonkey_communication.hpp
  qtmonkey.hpp
  )

set(qtmonkey_gui_MOC_HDRS qtmonkey_gui.hpp jsedit.h)
set(qtmonkey_gui_UIS qtmonkey_gui.ui)

if ("${QT_VARIANT}" STREQUAL "qt4")
  qt4_wrap_cpp(qtmonkey_agent_MOC_SRCS ${qtmonkey_agent_MOC_HDRS})
  qt4_wrap_cpp(qtmonkey_app_MOC_SRCS ${qtmonkey_app_MOC_HDRS})
  qt4_wrap_cpp(qtmonkey_gui_MOC_SRCS ${qtmonkey_gui_MOC_HDRS})
  qt4_wrap_ui(qtmonkey_gui_UIS_H ${qtmonkey_gui_UIS})
elseif ("${QT_VARIANT}" STREQUAL "qt5")
  qt5_wrap_cpp(qtmonkey_agent_MOC_SRCS ${qtmonkey_agent_MOC_HDRS})
  qt5_wrap_cpp(qtmonkey_app_MOC_SRCS ${qtmonkey_app_MOC_HDRS})
  qt5_wrap_cpp(qtmonkey_gui_MOC_SRCS ${qtmonkey_gui_MOC_HDRS})
  qt5_wrap_ui(qtmonkey_gui_UIS_H ${qtmonkey_gui_UIS})
endif ()

add_library(qtmonkey_agent STATIC
  ${qtmonkey_agent_MOC_SRCS}
  ${qtmonkey_agent_MOC_HDRS}
  user_events_analyzer.cpp
  custom_event_analyzer.hpp
  custom_script_extension.hpp
  agent_qtmonkey_communication.cpp
  agent.cpp
  script_runner.cpp
  script_runner.hpp
  script.hpp
  script.cpp
  script_api.cpp
  common.hpp
  common.cpp
  )
target_link_libraries(qtmonkey_agent ${QT_LIBRARIES})

add_library(common_app_lib STATIC
  contrib/json11/json11.cpp
  contrib/json11/json11.hpp
  common.cpp
  common.hpp
  qtmonkey_app_api.hpp
  qtmonkey_app_api.cpp
  shared_resource.hpp
  semaphore.hpp
  )
target_include_directories(common_app_lib PRIVATE contrib/json11)
target_link_libraries(common_app_lib ${QT_LIBRARIES})

add_executable(qtmonkey_app
  ${qtmonkey_app_MOC_SRCS}
  ${qtmonkey_app_MOC_HDRS}
  agent_qtmonkey_communication.cpp
  qtmonkey.cpp
  qtmonkey_app.cpp
  script.hpp
  script.cpp
  )
target_include_directories(qtmonkey_app PRIVATE contrib/json11)
target_link_libraries(qtmonkey_app ${QT_LIBRARIES} common_app_lib)

add_executable(qtmonkey_gui
  ${qtmonkey_gui_MOC_SRCS}
  ${qtmonkey_gui_MOC_HDRS}
  ${qtmonkey_gui_UIS_H}
  jsedit.cpp
  qtmonkey_gui.cpp
  )
target_compile_definitions(qtmonkey_gui PRIVATE -DQTMONKEY_APP_NAME="$<TARGET_FILE_NAME:qtmonkey_app>")
target_include_directories(qtmonkey_gui PRIVATE contrib/json11)
target_link_libraries(qtmonkey_gui ${QT_LIBRARIES} common_app_lib)

if (USE_TESTS)
  enable_testing()
  add_subdirectory(contrib/gtest/googletest)
  include_directories(contrib/gtest/googletest/include)
  add_executable(run_unit_tests tests/unit_tests.cpp)
  target_include_directories(run_unit_tests PRIVATE contrib/json11)
  target_link_libraries(run_unit_tests qtmonkey_agent gtest gtest_main ${QT_LIBRARIES} common_app_lib)
  add_test(unit_tests run_unit_tests)

  add_executable(json11_test contrib/json11/test.cpp)
  target_link_libraries(json11_test common_app_lib)
  add_test(json11_test json11_test)
  add_subdirectory(tests/test_app)
  if (PYTHONINTERP_FOUND)
    message(FATAL_ERROR "we need python for tests")
  endif ()
  find_package(PythonInterp REQUIRED)
  add_test(NAME gui_test_general COMMAND "${PYTHON_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/run_gui_tests.py" $<TARGET_FILE:qtmonkey_app> $<TARGET_FILE:test_app> "${CMAKE_CURRENT_SOURCE_DIR}/tests/test1.js")
  add_test(NAME gui_test_restart COMMAND "${PYTHON_EXECUTABLE}" "${CMAKE_CURRENT_SOURCE_DIR}/tests/gui_restart_test.py" $<TARGET_FILE:qtmonkey_app> $<TARGET_FILE:test_app> "${CMAKE_CURRENT_SOURCE_DIR}/tests/test_restart.js")
endif ()
