set(BOARD_TYPE "waveshare-s3-rlcd-4.2")
set(BOARD_NAME "${BOARD_TYPE}" CACHE STRING "Firmware build name")
set(BUILTIN_TEXT_FONT font_puhui_basic_30_4)
set(BUILTIN_ICON_FONT font_awesome_30_4)
set(DEFAULT_EMOJI_COLLECTION twemoji_32)
set(LANG_DIR "zh-CN")

set(SOURCES
    "audio/audio_codec.cc"
    "audio/audio_service.cc"
    "audio/codecs/box_audio_codec.cc"
    "audio/processors/afe_audio_processor.cc"
    "audio/processors/audio_debugger.cc"
    "audio/wake_words/afe_wake_word.cc"
    "boards/common/afsk_demod.cc"
    "boards/common/board.cc"
    "boards/common/button.cc"
    "boards/common/wifi_board.cc"
    "display/display.cc"
    "display/lcd_display.cc"
    "display/lvgl_display/emoji_collection.cc"
    "display/lvgl_display/gif/gifdec.c"
    "display/lvgl_display/gif/lvgl_gif.cc"
    "display/lvgl_display/lvgl_display.cc"
    "display/lvgl_display/lvgl_font.cc"
    "display/lvgl_display/lvgl_image.cc"
    "display/lvgl_display/lvgl_theme.cc"
    "application.cc"
    "assets.cc"
    "device_state_machine.cc"
    "main.cc"
    "mcp_server.cc"
    "ota.cc"
    "protocols/mqtt_protocol.cc"
    "protocols/protocol.cc"
    "protocols/websocket_protocol.cc"
    "settings.cc"
    "system_info.cc"
)

set(INCLUDE_DIRS
    "."
    "audio"
    "boards/common"
    "boards/${BOARD_TYPE}"
    "display"
    "display/lvgl_display"
    "protocols"
)

file(GLOB BOARD_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/boards/${BOARD_TYPE}/*.cc"
    "${CMAKE_CURRENT_SOURCE_DIR}/boards/${BOARD_TYPE}/*.c"
)
file(GLOB_RECURSE BOARD_SUB_SOURCES
    "${CMAKE_CURRENT_SOURCE_DIR}/boards/${BOARD_TYPE}/managers/*.cc"
    "${CMAKE_CURRENT_SOURCE_DIR}/boards/${BOARD_TYPE}/managers/*.c"
    "${CMAKE_CURRENT_SOURCE_DIR}/boards/${BOARD_TYPE}/assets/*.c"
)
list(APPEND SOURCES ${BOARD_SOURCES} ${BOARD_SUB_SOURCES})

file(GLOB LANG_SOUNDS "${CMAKE_CURRENT_SOURCE_DIR}/assets/locales/${LANG_DIR}/*.ogg")
file(GLOB COMMON_SOUNDS "${CMAKE_CURRENT_SOURCE_DIR}/assets/common/*.ogg")
set(LANG_JSON "${CMAKE_CURRENT_SOURCE_DIR}/assets/locales/${LANG_DIR}/language.json")
set(LANG_HEADER "${CMAKE_CURRENT_SOURCE_DIR}/assets/lang_config.h")

idf_component_register(
    SRCS ${SOURCES}
    EMBED_FILES ${LANG_SOUNDS} ${COMMON_SOUNDS}
    INCLUDE_DIRS ${INCLUDE_DIRS}
    WHOLE_ARCHIVE
    PRIV_REQUIRES
        esp_pm
        esp_psram
        esp_netif
        esp_driver_gpio
        esp_driver_uart
        esp_driver_spi
        esp_driver_i2c
        esp_driver_i2s
        esp_app_format
        app_update
        spi_flash
        console
        efuse
        esp_http_client
        esp_lcd
        json
        esp_adc
        fatfs
        sdmmc
)

target_compile_definitions(${COMPONENT_LIB}
    PRIVATE
        BOARD_TYPE=\"${BOARD_TYPE}\"
        BOARD_NAME=\"${BOARD_NAME}\"
        BUILTIN_TEXT_FONT=${BUILTIN_TEXT_FONT}
        BUILTIN_ICON_FONT=${BUILTIN_ICON_FONT}
)

add_custom_command(
    OUTPUT ${LANG_HEADER}
    COMMAND python ${PROJECT_DIR}/scripts/gen_lang.py
            --language "${LANG_DIR}"
            --output "${LANG_HEADER}"
    DEPENDS
        ${LANG_JSON}
        ${PROJECT_DIR}/scripts/gen_lang.py
    COMMENT "Generating ${LANG_DIR} language config"
)
add_custom_target(lang_header ALL DEPENDS ${LANG_HEADER})

idf_build_get_property(build_components BUILD_COMPONENTS)
function(find_component_by_pattern PATTERN COMPONENT_VAR PATH_VAR)
    foreach(COMPONENT ${build_components})
        if(COMPONENT MATCHES "${PATTERN}")
            set(${COMPONENT_VAR} ${COMPONENT} PARENT_SCOPE)
            idf_component_get_property(COMPONENT_PATH ${COMPONENT} COMPONENT_DIR)
            set(${PATH_VAR} "${COMPONENT_PATH}" PARENT_SCOPE)
            return()
        endif()
    endforeach()
endfunction()

find_component_by_pattern("espressif__esp-sr" ESP_SR_COMPONENT ESP_SR_COMPONENT_PATH)
if(ESP_SR_COMPONENT_PATH)
    set(ESP_SR_MODEL_PATH "${ESP_SR_COMPONENT_PATH}/model")
endif()

find_component_by_pattern("xiaozhi-fonts" XIAOZHI_FONTS_COMPONENT XIAOZHI_FONTS_COMPONENT_PATH)
if(XIAOZHI_FONTS_COMPONENT_PATH)
    set(XIAOZHI_FONTS_PATH "${XIAOZHI_FONTS_COMPONENT_PATH}")
endif()

set(GENERATED_ASSETS_BIN "${CMAKE_BINARY_DIR}/generated_assets.bin")
add_custom_command(
    OUTPUT ${GENERATED_ASSETS_BIN}
    COMMAND python ${PROJECT_DIR}/scripts/build_default_assets.py
            --sdkconfig "${SDKCONFIG}"
            --output "${GENERATED_ASSETS_BIN}"
            --builtin_text_font "${BUILTIN_TEXT_FONT}"
            --emoji_collection "${DEFAULT_EMOJI_COLLECTION}"
            --esp_sr_model_path "${ESP_SR_MODEL_PATH}"
            --xiaozhi_fonts_path "${XIAOZHI_FONTS_PATH}"
    DEPENDS
        ${SDKCONFIG}
        ${PROJECT_DIR}/scripts/build_default_assets.py
    COMMENT "Building default assets.bin"
    VERBATIM
)
add_custom_target(generated_default_assets ALL DEPENDS ${GENERATED_ASSETS_BIN})

partition_table_get_partition_info(size "--partition-name assets" "size")
partition_table_get_partition_info(offset "--partition-name assets" "offset")
if("${size}" AND "${offset}")
    esptool_py_flash_to_partition(flash "assets" "${GENERATED_ASSETS_BIN}")
else()
    message(FATAL_ERROR "The 16 MB partition table must define an assets partition.")
endif()
