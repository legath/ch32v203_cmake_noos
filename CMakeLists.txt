cmake_minimum_required(VERSION 3.15)

set(RISCV_TOOLCHAIN_PATH "/opt/xpack-riscv-none-elf-gcc-15.2.0-1" CACHE PATH "Path to RISC-V toolchain root directory (e.g., /opt/riscv-none-embed)")
set(TOOLCHAIN_PREFIXES
        "riscv-none-embed"
        "riscv-none-elf"
        "riscv64-unknown-elf"
        "riscv32-unknown-elf"
)


if (RISCV_TOOLCHAIN_PATH)
    set(TOOLCHAIN_BIN "${RISCV_TOOLCHAIN_PATH}/bin")
    message(STATUS "Using toolchain from: ${TOOLCHAIN_BIN}")
    if (NOT EXISTS "${TOOLCHAIN_BIN}")
        message(FATAL_ERROR "Toolchain bin directory not found: ${TOOLCHAIN_BIN}")
    endif ()


    foreach (prefix ${TOOLCHAIN_PREFIXES})
        if (NOT CMAKE_C_COMPILER)
            find_program(CMAKE_C_COMPILER
                    NAMES "${prefix}-gcc"
                    HINTS "${TOOLCHAIN_BIN}"
                    NO_DEFAULT_PATH
            )
            if (CMAKE_C_COMPILER)
                set(FOUND_PREFIX "${prefix}")
                break()
            endif ()
        endif ()
    endforeach ()

    if (NOT CMAKE_C_COMPILER)
        message(FATAL_ERROR "No RISC-V GCC compiler found in ${TOOLCHAIN_BIN}")
    endif ()


    set(CMAKE_ASM_COMPILER "${TOOLCHAIN_BIN}/${FOUND_PREFIX}-gcc" CACHE FILEPATH "ASM compiler" FORCE)
    set(CMAKE_OBJCOPY "${TOOLCHAIN_BIN}/${FOUND_PREFIX}-objcopy" CACHE FILEPATH "objcopy tool" FORCE)
    set(CMAKE_SIZE "${TOOLCHAIN_BIN}/${FOUND_PREFIX}-size" CACHE FILEPATH "size tool" FORCE)
    set(CMAKE_AR "${TOOLCHAIN_BIN}/${FOUND_PREFIX}-ar" CACHE FILEPATH "archive tool" FORCE)
    set(CMAKE_RANLIB "${TOOLCHAIN_BIN}/${FOUND_PREFIX}-ranlib" CACHE FILEPATH "ranlib tool" FORCE)

else ()
    message(STATUS "RISCV_TOOLCHAIN_PATH not set, searching in system PATH...")
    foreach (prefix ${TOOLCHAIN_PREFIXES})
        if (NOT CMAKE_C_COMPILER)
            find_program(CMAKE_C_COMPILER NAMES "${prefix}-gcc")
            if (CMAKE_C_COMPILER)
                set(FOUND_PREFIX "${prefix}")
                get_filename_component(TOOLCHAIN_BIN "${CMAKE_C_COMPILER}" DIRECTORY)
                message(STATUS "Found ${prefix} toolchain in: ${TOOLCHAIN_BIN}")
                break()
            endif ()
        endif ()
    endforeach ()

    if (NOT CMAKE_C_COMPILER)
        message(WARNING "RISC-V GCC compiler not found in PATH. Please set RISCV_TOOLCHAIN_PATH.")
    else ()
        set(CMAKE_ASM_COMPILER "${TOOLCHAIN_BIN}/${FOUND_PREFIX}-gcc" CACHE FILEPATH "ASM compiler" FORCE)
        set(CMAKE_OBJCOPY "${TOOLCHAIN_BIN}/${FOUND_PREFIX}-objcopy" CACHE FILEPATH "objcopy tool" FORCE)
        set(CMAKE_SIZE "${TOOLCHAIN_BIN}/${FOUND_PREFIX}-size" CACHE FILEPATH "size tool" FORCE)
        set(CMAKE_AR "${TOOLCHAIN_BIN}/${FOUND_PREFIX}-ar" CACHE FILEPATH "archive tool" FORCE)
        set(CMAKE_RANLIB "${TOOLCHAIN_BIN}/${FOUND_PREFIX}-ranlib" CACHE FILEPATH "ranlib tool" FORCE)
    endif ()
endif ()

# Настройка кросс-компиляции
set(CMAKE_SYSTEM_NAME Generic)
set(CMAKE_SYSTEM_PROCESSOR riscv32)


set(CMAKE_FIND_ROOT_PATH_MODE_PROGRAM NEVER)
set(CMAKE_FIND_ROOT_PATH_MODE_LIBRARY ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_INCLUDE ONLY)
set(CMAKE_FIND_ROOT_PATH_MODE_PACKAGE ONLY)

project(ch32v203_project C ASM)


set(MCU_ARCH "rv32imac_zicsr_zifencei")
set(MCU_COMMON_FLAGS
        -march=${MCU_ARCH}
        -mabi=ilp32
        -mcmodel=medany
        -mno-save-restore
        -msmall-data-limit=8
)


set(SOURCES
        Core/core_riscv.c
        Debug/debug.c
        User/main.c
        User/ch32v20x_it.c
        User/system_ch32v20x.c
)

file(GLOB PERIPHERAL_SOURCES "Peripheral/src/*.c")
list(APPEND SOURCES ${PERIPHERAL_SOURCES})


set(STARTUP_FILE "Startup/startup_ch32v20x_D6.S")
list(APPEND SOURCES ${STARTUP_FILE})

# ==================== INCLUDE ДИРЕКТОРИИ ====================
include_directories(
        Peripheral/inc
        Core
        Debug
        User
)


add_executable(${PROJECT_NAME}.elf ${SOURCES})

target_compile_options(${PROJECT_NAME}.elf PRIVATE
        ${MCU_COMMON_FLAGS}
        -Os
        -ffunction-sections
        -fdata-sections
        -fno-common
        -fno-builtin
        -Wall
        -Wextra
        -Wno-unused-parameter
        -Wno-missing-field-initializers
        -Wno-attributes            # Подавляет предупреждения о "WCH-Interrupt-fast"
        -Wno-format
        -g
)


target_compile_definitions(${PROJECT_NAME}.elf PRIVATE
        CH32V30X
        CH32V30X_D8C
        __riscv_cmodel_medany
)


set(LINKER_SCRIPT "${CMAKE_SOURCE_DIR}/Ld/Link.ld")
if (NOT EXISTS "${LINKER_SCRIPT}")
    message(FATAL_ERROR "Linker script not found: ${LINKER_SCRIPT}")
endif ()

#target_link_options(${PROJECT_NAME}.elf PRIVATE
#        ${MCU_COMMON_FLAGS}
#        -T${LINKER_SCRIPT}
#        -Wl,--gc-sections
#        #-Wl,--print-gc-sections
#        -Wl,-Map=${PROJECT_NAME}.map
#        -Wl,--cref
#        -nostartfiles
#        #-nodefaultlibs
#        -nostdlib
#        -lgcc
#        -lm
#        #--specs=nano.specs
#        --specs=nosys.specs
#)
target_link_options(${PROJECT_NAME}.elf PRIVATE
        ${MCU_COMMON_FLAGS}
        -T${LINKER_SCRIPT}
        -flto
        -Wl,--gc-sections
        -Wl,-Map=${PROJECT_NAME}.map
        -Wl,--cref
        -nostartfiles
        #-nostdlib
        --specs=nosys.specs
        --specs=nano.specs
)

target_link_libraries(${PROJECT_NAME}.elf PRIVATE
        -lgcc
        -lm
)


if (CMAKE_OBJCOPY)
    add_custom_command(TARGET ${PROJECT_NAME}.elf POST_BUILD
            COMMAND ${CMAKE_OBJCOPY} -O binary ${PROJECT_NAME}.elf ${PROJECT_NAME}.bin
            COMMAND ${CMAKE_OBJCOPY} -O ihex ${PROJECT_NAME}.elf ${PROJECT_NAME}.hex
            COMMAND ${CMAKE_COMMAND} -E echo "=== Firmware Size ==="
            COMMAND ${CMAKE_SIZE} ${PROJECT_NAME}.elf
    )
else ()
    message(WARNING "objcopy not found — skipping .bin/.hex generation")
endif ()


if (DEFINED ENV{WCH_LINK} OR DEFINED WCH_LINK_PATH)
    set(WCH_LINK_CMD "$ENV{WCH_LINK}")
    if (NOT WCH_LINK_CMD)
        set(WCH_LINK_CMD "${WCH_LINK_PATH}")
    endif ()

    add_custom_target(flash
            COMMAND ${WCH_LINK_CMD} -c -p ${PROJECT_NAME}.hex
            DEPENDS ${PROJECT_NAME}.elf
            COMMENT "Flashing firmware via WCH-Link"
    )
endif ()


message(STATUS "")
message(STATUS "============================================")
message(STATUS "CH32V203 Project Configuration")
message(STATUS "============================================")
message(STATUS "Toolchain path: ${TOOLCHAIN_BIN}")
message(STATUS "C compiler:     ${CMAKE_C_COMPILER}")
message(STATUS "Target:         ${MCU_ARCH} ")
message(STATUS "Linker script:  ${LINKER_SCRIPT}")
message(STATUS "Output files:   ${PROJECT_NAME}.elf/.bin/.hex")
message(STATUS "============================================")
