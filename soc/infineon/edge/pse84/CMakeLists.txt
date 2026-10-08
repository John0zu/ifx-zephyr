# Copyright (c) 2025 Infineon Technologies AG,
# or an affiliate of Infineon Technologies AG.
#
# SPDX-License-Identifier: Apache-2.0

include(${ZEPHYR_BASE}/soc/infineon/edge/pse84/pse84_metadata.cmake)

if(CONFIG_BUILD_WITH_TFM)
  if(CONFIG_SOC_PSE846GPS4DBZC4A)
    set(IFX_EPC_VARIANT epc4)
  else()
    set(IFX_EPC_VARIANT epc2)
  endif()
  set_property(TARGET zephyr_property_target APPEND PROPERTY TFM_CMAKE_OPTIONS
    -DIFX_EPC=${IFX_EPC_VARIANT}
    -DIFX_UART_ENABLED=OFF
    # Use local hal_infineon assets instead of fetching from GitHub
    -DIFX_CORE_LIB_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/core-lib
    -DIFX_DEV_SUPPORT_LIB_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/mtb-dsl-pse8xxgp
    -DIFX_MTB_SRF_LIB_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/mtb-srf
    -DIFX_ABSTRACTION_RTOS_LIB_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/abstraction-rtos
    -DIFX_MTB_IPC_LIB_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/mtb-ipc
    -DIFX_DEVICE_DB_LIB_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/device-db
    -DIFX_BSP_LIB_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/mtb-template-pse8xxgp/files
    # Pregenerated sources are supplied via IFX_BSP_GENERATED_FILES_OUTPUT_PATH;
    # disable Device Configurator code generation.
    -DIFX_GENERATE_BSP_SOURCES=OFF
    -DIFX_BOARD_PATH=${ZEPHYR_BASE}/modules/trusted-firmware-m/infineon/pse84/board
    -DZEPHYR_BASE=${ZEPHYR_BASE}
    -DZEPHYR_BINARY_DIR=${ZEPHYR_BINARY_DIR}
  )

  if(CONFIG_PSOC_EDGE_M55_SRF_SUPPORT)
    set_property(TARGET zephyr_property_target APPEND PROPERTY TFM_CMAKE_OPTIONS
      -DIFX_MTB_MAILBOX=ON
      -DIFX_CM55_NS_PRESENT=ON
    )
  endif()

  if(CONFIG_BOARD_KIT_PSE84_AI)
    set_property(TARGET zephyr_property_target APPEND PROPERTY TFM_CMAKE_OPTIONS
      -DIFX_BSP_GENERATED_FILES_OUTPUT_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/zephyr-ifx-cycfg/pse84/kit_pse84_ai
      -DIFX_BSP_DESIGN_DIR_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/zephyr-ifx-cycfg/pse84/kit_pse84_ai
      -DIFX_SMIF_0_MEMORY_CONFIG=S25HS512T_SMIF0_SlaveSlot_1
    )
  elseif(CONFIG_BOARD_KIT_PSE84_EVAL)
    if(IFX_EPC_VARIANT STREQUAL "epc2")
      set_property(TARGET zephyr_property_target APPEND PROPERTY TFM_CMAKE_OPTIONS
        -DIFX_BSP_GENERATED_FILES_OUTPUT_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/zephyr-ifx-cycfg/pse84/kit_pse84_eval
        -DIFX_BSP_DESIGN_DIR_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/zephyr-ifx-cycfg/pse84/kit_pse84_eval
      )
    else()
      set_property(TARGET zephyr_property_target APPEND PROPERTY TFM_CMAKE_OPTIONS
        -DIFX_BSP_GENERATED_FILES_OUTPUT_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/zephyr-ifx-cycfg/pse84/kit_pse84_eval_epc4
        -DIFX_BSP_DESIGN_DIR_PATH=${ZEPHYR_HAL_INFINEON_MODULE_DIR}/zephyr-ifx-cycfg/pse84/kit_pse84_eval_epc4
      )
    endif()
    set_property(TARGET zephyr_property_target APPEND PROPERTY TFM_CMAKE_OPTIONS
      -DIFX_SMIF_0_MEMORY_CONFIG=S25FS128S_SMIF0_SlaveSlot_1
    )
  else()
    message(FATAL_ERROR "No supported board selected.")
  endif()

endif()

# All image signing and merging logic lives in one place. Keep this after
# the TF-M cmake-args block above so the TF-M options are set before the
# signing flow runs.
include(${CMAKE_CURRENT_LIST_DIR}/pse84_signing.cmake)

if(CONFIG_CPU_CORTEX_M33 AND CONFIG_TRUSTED_EXECUTION_SECURE)
  zephyr_sources(soc_pse84_m33_s.c)

  zephyr_sources(security_config/pse84_s_mpc.c)

if(CONFIG_CUSTOM_ZEPHYR_IFX_SECURITY_CONFIG_DIR)
  zephyr_sources(${CONFIG_CUSTOM_ZEPHYR_IFX_SECURITY_CONFIG_DIR}/pse84_s_protection.c)
else()
  zephyr_sources(security_config/pse84_s_protection.c)
endif()
  zephyr_sources(security_config/pse84_s_system.c)
  zephyr_sources(security_config/pse84_s_sau.c)
elseif(CONFIG_CPU_CORTEX_M33)
  zephyr_sources(soc_pse84_m33_ns.c)
elseif(CONFIG_CPU_CORTEX_M55)
  zephyr_sources(soc_pse84_m55.c)
endif()

zephyr_include_directories(security_config)
zephyr_sources(security_config/pse84_boot.c)

zephyr_sources(mpu_regions.c)

zephyr_sources_ifdef(CONFIG_PM power.c)
if(NOT CONFIG_TRUSTED_EXECUTION_NONSECURE)
  zephyr_sources_ifdef(CONFIG_REBOOT reboot.c)
endif()
zephyr_sources_ifdef(CONFIG_SOC_PSE84_LPM_TIMER lpm_timer.c)
zephyr_sources_ifdef(CONFIG_POWEROFF poweroff.c)

zephyr_include_directories(.)

if(${ZEPHYR_TOOLCHAIN_VARIANT} STREQUAL "armclang")
  zephyr_library_sources(cy_syslib_ext.S)
endif()

# Add sections
if(CONFIG_CPU_CORTEX_M55)
zephyr_linker_sources(SECTIONS shared_mem_m55.ld)
else()
  if(CONFIG_TRUSTED_EXECUTION_SECURE)
    zephyr_linker_sources(SECTIONS shared_mem_sec.ld)
  else()
    zephyr_linker_sources(SECTIONS shared_mem_m33.ld)
  endif()
endif()

set(SOC_LINKER_SCRIPT ${ZEPHYR_BASE}/soc/infineon/edge/pse84/linker_exclude_syslib.ld CACHE INTERNAL "")

zephyr_linker_sources_ifdef(CONFIG_CPU_CORTEX_M33 RWDATA rwdata.ld)
zephyr_linker_sources_ifdef(CONFIG_CPU_CORTEX_M55 RWDATA rwdata.ld)

zephyr_linker_sources_ifdef(CONFIG_CPU_CORTEX_M55 ITCM_SECTION itcm_mem.ld)
zephyr_linker_sources_ifdef(CONFIG_CPU_CORTEX_M33 RAMFUNC_SECTION ramfunc_mem.ld)
