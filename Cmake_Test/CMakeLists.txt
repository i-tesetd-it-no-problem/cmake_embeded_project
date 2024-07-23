cmake_minimum_required(VERSION 3.20)

cmake_policy(SET CMP0123 NEW)

set(CMAKE_SYSTEM_NAME Generic)
set(CPU_CORE cortex-m3)
set(CMAKE_SYSTEM_PROCESSOR ${CPU_CORE})

set(CMAKE_TOOLCHAIN_FILE ${CMAKE_CURRENT_LIST_DIR}/toolchain.cmake)

project(TEST C ASM)

string(TOUPPER ${CMAKE_BUILD_TYPE} BUILD_TYPE_UPPER)
if(${BUILD_TYPE_UPPER} STREQUAL "RELEASE")
    add_compile_options(-Os)
else()
    add_compile_options(-Og -g)
endif()

include_directories(
    ${CMAKE_CURRENT_LIST_DIR}/Firmware/Firmware/CMSIS/
    ${CMAKE_CURRENT_LIST_DIR}/Firmware/Firmware/CMSIS/GD/GD32F10x/Include/
    ${CMAKE_CURRENT_LIST_DIR}/Firmware/Firmware/GD32F10x_standard_peripheral/Include/
    ${CMAKE_CURRENT_LIST_DIR}/Firmware//inc/
)

file(GLOB SOURCE_FILE
    ${CMAKE_CURRENT_LIST_DIR}/Firmware/Firmware/CMSIS/GD/GD32F10x/Source/*.c
    ${CMAKE_CURRENT_LIST_DIR}/Firmware/Firmware/GD32F10x_standard_peripheral/Source/*.c
    ${CMAKE_CURRENT_LIST_DIR}/Firmware//src/*.c
)

set(EXECUTABLE_OUTPUT_PATH ${CMAKE_BINARY_DIR}/output)
set(HEX_FILE ${EXECUTABLE_OUTPUT_PATH}/${PROJECT_NAME}.hex)
set(BIN_FILE ${EXECUTABLE_OUTPUT_PATH}/${PROJECT_NAME}.bin)
set(MAP_FILE ${EXECUTABLE_OUTPUT_PATH}/${PROJECT_NAME}.map)

if(USE_ARMGCC)
    SET(STARTUP_FILE ${CMAKE_CURRENT_LIST_DIR}/Firmware/gcc_startup/startup_gd32f10x_hd.S)
    SET(LINKER_SCRIPT ${CMAKE_CURRENT_LIST_DIR}/Firmware/gd32f10x_flash.ld)

    add_compile_options(
        -mcpu=${CPU_CORE}
        -mthumb
        -mthumb-interwork
        -ffunction-sections
        -fdata-sections
        -fno-common
        -fmessage-length=0
    )

    add_link_options(
        -Wl,-Map=${MAP_FILE}
        -Wl,--print-memory-usage
        -Wl,--gc-sections
        -T ${LINKER_SCRIPT}
    )
else()
    SET(STARTUP_FILE ${CMAKE_CURRENT_LIST_DIR}/startup_gd32f10x_hd.s)
    SET(LINKER_SCRIPT ${CMAKE_CURRENT_LIST_DIR}/gd32f10x_flash.sct)

    add_compile_options(
        --target=arm-arm-none-eabi
        -mcpu=${CPU_CORE}
        -masm=armasm
        -Wa,armasm,--diag_suppress=A1950W
        -gdwarf-4
    )

    add_link_options(
        --strict
        --scatter ${LINKER_SCRIPT}
        --info=sizes,totals,unused,veneers
        --map
        --list=${MAP_FILE}
    )
endif()

add_executable(${PROJECT_NAME} ${SOURCE_FILE} ${STARTUP_FILE})
set_target_properties(${PROJECT_NAME} PROPERTIES
    RUNTIME_OUTPUT_DIRECTORY ${EXECUTABLE_OUTPUT_PATH}
)

if(USE_ARMGCC)
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -Oihex $<TARGET_FILE:${PROJECT_NAME}> ${HEX_FILE}
        COMMAND ${CMAKE_OBJCOPY} -Obinary $<TARGET_FILE:${PROJECT_NAME}> ${BIN_FILE}
        COMMAND ${CMAKE_SIEZE} $<TARGET_FILE:${PROJECT_NAME}>
    )
else()
    add_custom_command(TARGET ${PROJECT_NAME} POST_BUILD
        COMMAND ${CMAKE_OBJCOPY} -i32 --output=${HEX_FILE}  $<TARGET_FILE:${PROJECT_NAME}>
        COMMAND ${CMAKE_OBJCOPY} -bin --output=${BIN_FILE}  $<TARGET_FILE:${PROJECT_NAME}>
        COMMAND ${CMAKE_SIEZE} --info=sizes $<TARGET_FILE:${PROJECT_NAME}>
    )
endif()