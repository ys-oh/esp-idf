idf_build_get_property(target IDF_TARGET)

# On Linux, we only support a few features, hence this simple component registration
if(${target} STREQUAL "linux")
    idf_component_register(SRCS "port/linux/esp_random.c"
                                 "port/linux/chip_info.c"
                           INCLUDE_DIRS "include")
    return()
endif()

set(requires soc)
# only esp_hw_support/adc_share_hw_ctrl.c requires efuse component
set(priv_requires efuse spi_flash bootloader_support)

set(srcs "cpu.c" "esp_memory_utils.c")
if(NOT BOOTLOADER_BUILD)
    list(APPEND srcs "esp_clk.c"
                     "clk_ctrl_os.c"
                     "hw_random.c"
                     "intr_alloc.c"
                     "mac_addr.c"
                     "periph_ctrl.c"
                     "rtc_module.c"
                     "sleep_modes.c"
                     "sleep_gpio.c"
                     "sleep_mac_bb.c"
                     "regi2c_ctrl.c"
                     "adc_share_hw_ctrl.c"
                     "port/${target}/io_mux.c"
                     "port/${target}/clk_tree.c"
                     "port/clk_tree_common.c")

    if(NOT CONFIG_IDF_TARGET_ESP32 AND NOT CONFIG_IDF_TARGET_ESP32S2)
        list(APPEND srcs "sleep_retention.c")
    endif()

    # [refactor-todo]: requires "driver" for GPIO and RTC  (by sleep_gpio and sleep_modes)
    list(APPEND priv_requires driver esp_timer)

    if(CONFIG_IDF_TARGET_ESP32 OR CONFIG_IDF_TARGET_ESP32S2)
        list(APPEND srcs "rtc_wdt.c")
    endif()

    if(CONFIG_SOC_GDMA_SUPPORTED)
        list(APPEND srcs "dma/gdma.c" "dma/async_memcpy_impl_gdma.c")
    endif()

    if(CONFIG_SOC_CP_DMA_SUPPORTED)
        list(APPEND srcs "dma/async_memcpy_impl_cp_dma.c")
    endif()

    if(CONFIG_SOC_ASYNC_MEMCPY_SUPPORTED)
        list(APPEND srcs "dma/esp_async_memcpy.c")
    endif()

    if(CONFIG_SOC_GDMA_SUPPORT_ETM)
        list(APPEND srcs "dma/gdma_etm.c")
    endif()

    if(CONFIG_SOC_SYSTIMER_SUPPORTED)
        list(APPEND srcs "port/${target}/systimer.c")
    endif()

    if(CONFIG_SOC_HMAC_SUPPORTED)
        list(APPEND srcs "esp_hmac.c")
    endif()

    if(CONFIG_SOC_ETM_SUPPORTED)
        list(APPEND srcs "esp_etm.c")
    endif()

    if(CONFIG_SOC_DIG_SIGN_SUPPORTED)
        list(APPEND srcs "esp_ds.c")
    endif()

    if(CONFIG_SOC_MODEM_CLOCK_IS_INDEPENDENT)
        list(APPEND srcs "modem_clock.c")
    endif()

    if(CONFIG_SOC_MEMSPI_SRC_FREQ_120M)
        list(APPEND srcs "mspi_timing_tuning.c" "port/${target}/mspi_timing_config.c")
    endif()

    if(CONFIG_SOC_RTC_FAST_MEM_SUPPORTED)
        list(APPEND srcs "sleep_wake_stub.c")
    endif()

    if(CONFIG_IDF_TARGET_ESP32H2)
        list(REMOVE_ITEM srcs
                "adc_share_hw_ctrl.c" # TODO: IDF-6215
                "port/clk_tree_common.c" # TODO: IDF-6265
            )
    endif()
else()
    # Requires "_esp_error_check_failed()" function
    list(APPEND priv_requires "esp_system")
endif()

idf_component_register(SRCS ${srcs}
                       INCLUDE_DIRS include include/soc include/soc/${target}
                       PRIV_INCLUDE_DIRS port/include include/esp_private
                       REQUIRES ${requires}
                       PRIV_REQUIRES "${priv_requires}"
                       LDFRAGMENTS linker.lf)

idf_build_get_property(target IDF_TARGET)
add_subdirectory(port/${target})

if(NOT BOOTLOADER_BUILD)
    if(CONFIG_SPIRAM)
        idf_component_optional_requires(PRIVATE esp_psram)
    endif()
endif()

target_compile_options(${COMPONENT_LIB} PRIVATE "-Wno-format")
