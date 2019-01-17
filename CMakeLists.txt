cmake_minimum_required(VERSION 3.8.2)

include(cmake_arduino_local_install/arduino-local-install.cmake)

SET(ArduinoToolchainVersion 1.8.8)

set(DOWNLOADS ${CMAKE_SOURCE_DIR}/Downloads)

file(MAKE_DIRECTORY ${DOWNLOADS})

set(ARDUINO_SDK_PATH ${CMAKE_SOURCE_DIR}/arduino-${ArduinoToolchainVersion})

if(NOT EXISTS ${ARDUINO_SDK_PATH}/revisions.txt)
    install_arduino_localy(${ArduinoToolchainVersion} ${CMAKE_SOURCE_DIR} ${DOWNLOADS})
endif()

project(at16u2_cmsis_dap)

