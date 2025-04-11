# ESP32 WiFi Connect

## 【修改 wifi_station.cc 中关闭wifi和重新连接wifi导致的各类异常和bug】

This component helps with WiFi connection for the device.

It first tries to connect to a WiFi network using the credentials stored in the flash. If this fails, it starts an access point and a web server to allow the user to connect to a WiFi network.

The URL to access the web server is `http://192.168.4.1`.

Here is a screenshot of the web server:

![Access Point Configuration](assets/ap_v2.png)

## Changelog: v2.3.0

- Add support for language request.

## Changelog: v2.2.0

- Add support for ESP32 SmartConfig(ESPTouch v2)

## Changelog: v2.1.0

- Improve WiFi connection logic.

## Changelog: v2.0.0

- Add support for multiple WiFi SSID management.
- Auto switch to the best WiFi network.
- Captive portal for WiFi configuration.
- Support for multiple languages (English, Chinese).

## Configuration

The WiFi credentials are stored in the flash under the "wifi" namespace.

The keys are "ssid", "ssid1", "ssid2" ... "ssid9", "password", "password1", "password2" ... "password9".

## Usage
【简易功能函数，引用虾哥的小智代码】
```cpp
#include "wifi_utils.hpp"

#include <freertos/FreeRTOS.h>
#include <freertos/task.h>
#include <esp_log.h>
#include "nvs_flash.h"
#include <wifi_station.h>
#include <wifi_configuration_ap.h>
#include <ssid_manager.h>

static const char *TAG = "WIFI_utils";

bool wifi_config_mode = false;

void EnterWifiConfigMode() {

    auto& wifi_ap = WifiConfigurationAp::GetInstance();
    wifi_ap.SetSsidPrefix("ESP32S3_dev");
    wifi_ap.Start();

    // Wait forever until reset after configuration
    while (true) {
        int free_sram = heap_caps_get_free_size(MALLOC_CAP_INTERNAL);
        int min_free_sram = heap_caps_get_minimum_free_size(MALLOC_CAP_INTERNAL);
        ESP_LOGI(TAG, "Free internal: %u minimal internal: %u", free_sram, min_free_sram);
        vTaskDelay(pdMS_TO_TICKS(10000));
    }
}

void StartNetwork() {
    esp_err_t ret;
    // Initialize the default event loop
    ret = esp_event_loop_create_default();
    ret = nvs_flash_init();
    if (ret == ESP_ERR_NVS_NO_FREE_PAGES || ret == ESP_ERR_NVS_NEW_VERSION_FOUND) {
        ESP_ERROR_CHECK(nvs_flash_erase());
        ret = nvs_flash_init();
    }

    // User can press BOOT button while starting to enter WiFi configuration mode
    if (wifi_config_mode) {
        EnterWifiConfigMode();
        return;
    }

    // If no WiFi SSID is configured, enter WiFi configuration mode
    auto& ssid_manager = SsidManager::GetInstance();
    auto ssid_list = ssid_manager.GetSsidList();
    if (ssid_list.empty()) {
        wifi_config_mode = true;
        EnterWifiConfigMode();
        return;
    }

    auto& wifi_station = WifiStation::GetInstance();
    wifi_station.Start();

    // Try to connect to WiFi, if failed, launch the WiFi configuration AP
    if (!wifi_station.WaitForConnected(60 * 1000)) {
        wifi_station.Stop();
        wifi_config_mode = true;
        ESP_LOGI(TAG,"连接失败，进入wifi配置模式");
        EnterWifiConfigMode();
        return;
    }else{
        ESP_LOGI(TAG,"WIFI连接成功");
    }
}

void stopNetwork() {
    auto& wifi_station = WifiStation::GetInstance();
    wifi_station.Stop();
}

std::string GetBoardJson() {
    const char* BOARD_NAME = "esp32s3_n16r8"; 
    auto& wifi_station = WifiStation::GetInstance();
    std::string board_json = std::string("{");
    board_json += "\"name\":\"" +std::string(BOARD_NAME)+ "\",";
    if (!wifi_config_mode) {
        board_json += "\"ssid\":\"" + wifi_station.GetSsid() + "\",";
        board_json += "\"rssi\":" + std::to_string(wifi_station.GetRssi()) + ",";
        board_json += "\"channel\":" + std::to_string(wifi_station.GetChannel()) + ",";
        board_json += "\"ip\":\"" + wifi_station.GetIpAddress() + "\",";
    }
    board_json += "\"}";
    return board_json;
}

void SetPowerSaveMode(bool enabled) {
    auto& wifi_station = WifiStation::GetInstance();
    wifi_station.SetPowerSaveMode(enabled);
}



```
以及头文件

```cpp
#pragma once

#ifndef WIFI_BOARD_HPP
#define WIFI_BOARD_HPP

#include <string>

extern bool wifi_config_mode; // 是否处于 WiFi 配置模式

std::string GetBoardJson();

// 启动网络
void StartNetwork();

// 停止网络
void stopNetwork();

// 设置电源节省模式
void SetPowerSaveMode(bool enabled);

// 进入 WiFi 配置
void EnterWifiConfigMode();


#endif // WIFI_BOARD_H

```

