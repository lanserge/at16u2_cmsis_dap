cmake_minimum_required(VERSION 3.6)

include(cmake_arduino_local_install/arduino-local-install.cmake)

SET(ArduinoToolchainVersion 1.8.8)

set(DOWNLOADS ${CMAKE_SOURCE_DIR}/Downloads)

file(MAKE_DIRECTORY ${DOWNLOADS})

set(ARDUINO_SDK_PATH ${CMAKE_SOURCE_DIR}/arduino-${ArduinoToolchainVersion})

if(NOT EXISTS ${ARDUINO_SDK_PATH}/revisions.txt)
    install_arduino_localy(${ArduinoToolchainVersion} ${CMAKE_SOURCE_DIR} ${DOWNLOADS})
endif()

SET(MCU atmega16u2)

SET(CMAKE_SYSTEM_NAME Generic)
SET(CMAKE_CROSSCOMPILING 1)

SET(AVR_TOOLS_PATH ${ARDUINO_SDK_PATH}/hardware/tools/avr)

SET(CMAKE_C_COMPILER ${AVR_TOOLS_PATH}/bin/avr-gcc)
SET(CMAKE_CXX_COMPILER ${AVR_TOOLS_PATH}/bin/avr-g++)

SET(C_FLAGS "-mmcu=${MCU} -g0 -Os -fno-exceptions -ffunction-sections -fdata-sections -Wno-error=narrowing -funsigned-char -funsigned-bitfields -fpack-struct -fshort-enums")

SET(CMAKE_C_FLAGS "${C_FLAGS}")
SET(CMAKE_CXX_FLAGS "-std=gnu++11 -fpermissive -fno-threadsafe-statics ${C_FLAGS}")
SET(CMAKE_EXE_LINKER_FLAGS "-Wl,--gc-sections,--relax -mmcu=atmega16u2")
SET(CMAKE_SHARED_LIBRARY_LINK_C_FLAGS)
SET(CMAKE_SHARED_LIBRARY_LINK_CXX_FLAGS)

include_directories(
    ${CMAKE_SOURCE_DIR}
    ${AVR_TOOLS_PATH}/avr/include
    ${ARDUINO_SDK_PATH}/hardware/arduino/avr/cores/arduino
    ${CMAKE_SOURCE_DIR}/HID/src
    ${ARDUINO_SDK_PATH}/hardware/arduino/avr/libraries/HID/src/
)

add_definitions(
    -D__AVR_ATmega16U2__
    -DF_CPU=16000000

    -DUSB_VID=0x2341
    -DUSB_PID=0x003D
    -DUSB_PRODUCT="Arduino CMSIS-DAP"

    -DARDUINO=10808
    -DARDUINO_ARCH_AVR

    -DDAP_SERIAL_LOG=0
    -DDAP_JTAG=0
    -DDAP_SWD=1
    -DAT16U2_DUE

    -DBOARD_ID="\\x1\\x1\\x1\\x1"
    -DBOARD_SECRET="\\x1\\x1\\x1\\x1\\x1\\x1\\x1\\x1"

)

function(get_folder_sources var folder)
    set(SRC_FILES ${${var}})
    foreach(pattern IN LISTS ARGN)
        file(GLOB_RECURSE FILES ${folder}/${pattern})
        list(APPEND SRC_FILES ${FILES})
    endforeach()
    set(${var} ${SRC_FILES} PARENT_SCOPE)
endfunction()

project(at16u2_cmsis_dap)

get_folder_sources(ARDUINO_CMSIS_DAP_SRC_FILES ${CMAKE_SOURCE_DIR}/arduino-cmsis-dap *.c *.cpp)
add_library(arduino_cmsis_dap STATIC ${ARDUINO_CMSIS_DAP_SRC_FILES})

get_folder_sources(AVR_CORE_SRC_FILES ${ARDUINO_SDK_PATH}/hardware/arduino/avr/cores/arduino *.c *.cpp)
add_library(avr_core STATIC ${AVR_CORE_SRC_FILES})

get_folder_sources(HID_SRC_FILES ${CMAKE_SOURCE_DIR}/HID/src/SingleReport *.c *.cpp)
add_library(HID STATIC ${HID_SRC_FILES})


add_executable(at16u2_cmsis_dap sketch.cpp)
set_target_properties(at16u2_cmsis_dap PROPERTIES SUFFIX ".elf")
target_link_libraries(at16u2_cmsis_dap arduino_cmsis_dap HID avr_core)

set(ELF_FILE $<TARGET_FILE:at16u2_cmsis_dap>)

add_custom_target(hex 
    COMMAND ${AVR_TOOLS_PATH}/bin/avr-size -A ${ELF_FILE}
    COMMAND ${AVR_TOOLS_PATH}/bin/avr-objcopy -O ihex -R .eeprom ${ELF_FILE} ${ELF_FILE}.hex
    BYPRODUCTS ${ELF_FILE}.hex
    DEPENDS at16u2_cmsis_dap
)

add_custom_target(flash_avrisp2
    COMMAND ${AVR_TOOLS_PATH}/bin/avrdude -c avrisp2 -p m16u2 -v -e -U flash:w:${ELF_FILE}.hex:i -C ${AVR_TOOLS_PATH}/etc/avrdude.conf
    DEPENDS ${ELF_FILE}.hex
)

add_dependencies(flash_avrisp2 hex)
