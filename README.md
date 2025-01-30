# esp32c3-data-logger-tri
方案概述

ESP32 每 3 分钟通过 GitHub API 提交数据到 GitHub Issues，Issue 标题为时间戳，内容为随机整数数据。

GitHub Actions 每 5 分钟检查 Issues，将数据汇总到 data.json，并提交到仓库。



---

1️⃣ ESP32 代码修改（提交数据到 GitHub Issues）

ESP32 通过 GitHub API 直接创建 Issue。

📌 main.c（ESP32 代码）

#include <stdio.h>
#include <stdlib.h>
#include <string.h>
#include <time.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "esp_log.h"
#include "esp_system.h"
#include "esp_http_client.h"
#include "nvs_flash.h"
#include "esp_wifi.h"
#include "esp_event.h"
#include "protocol_examples_common.h"

#define WIFI_SSID "你的WiFi名称"
#define WIFI_PASS "你的WiFi密码"
#define GITHUB_OWNER "你的GitHub用户名"
#define GITHUB_REPO "esp32c3-data-logger"
#define GITHUB_TOKEN "你的GitHub Token"

static const char *TAG = "GitHub_Logger";

// 生成随机数
int get_random_value() {
    return rand() % 201 + 100; // 100-300 范围的随机整数
}

// 生成当前时间戳
void get_timestamp(char *buffer, size_t size) {
    time_t now = time(NULL);
    struct tm timeinfo;
    localtime_r(&now, &timeinfo);
    strftime(buffer, size, "%Y-%m-%d %H:%M:%S", &timeinfo);
}

// 发送 HTTP 请求到 GitHub Issues
esp_err_t http_event_handler(esp_http_client_event_t *evt) {
    return ESP_OK;
}

void send_data_to_github() {
    char issue_title[64];
    char post_data[256];
    int random_value = get_random_value();
    get_timestamp(issue_title, sizeof(issue_title));

    snprintf(post_data, sizeof(post_data),
             "{\"title\":\"%s\", \"body\":\"Random Value: %d\"}", issue_title, random_value);

    char url[256];
    snprintf(url, sizeof(url),
             "https://api.github.com/repos/%s/%s/issues", GITHUB_OWNER, GITHUB_REPO);

    esp_http_client_config_t config = {
        .url = url,
        .method = HTTP_METHOD_POST,
        .event_handler = http_event_handler,
        .buffer_size = 512,
        .user_agent = "ESP32",
        .headers = "Authorization: token " GITHUB_TOKEN "\r\n"
                   "Accept: application/vnd.github.v3+json\r\n"
                   "Content-Type: application/json\r\n"
    };

    esp_http_client_handle_t client = esp_http_client_init(&config);
    esp_http_client_set_post_field(client, post_data, strlen(post_data));

    esp_err_t err = esp_http_client_perform(client);
    if (err == ESP_OK) {
        ESP_LOGI(TAG, "Data sent: %s", post_data);
    } else {
        ESP_LOGE(TAG, "Failed to send data: %s", esp_err_to_name(err));
    }

    esp_http_client_cleanup(client);
}

// WiFi 连接
void wifi_init() {
    ESP_ERROR_CHECK(nvs_flash_init());
    ESP_ERROR_CHECK(esp_netif_init());
    ESP_ERROR_CHECK(esp_event_loop_create_default());
    ESP_ERROR_CHECK(example_connect()); // 连接 WiFi
}

// 主任务：每 3 分钟上传数据
void app_main() {
    wifi_init();
    while (1) {
        send_data_to_github();
        vTaskDelay(pdMS_TO_TICKS(180000)); // 3 分钟
    }
}


---

2️⃣ GitHub Actions（处理 Issues 并更新 data.json）

📌 .github/workflows/process_issues.yml

GitHub Actions 每 5 分钟运行：

获取所有未关闭的 Issues。

解析数据并存入 data.json。

将数据提交回 GitHub 仓库。


name: Process Issues and Update Data

on:
  schedule:
    - cron: '*/5 * * * *'  # 每 5 分钟运行一次
  workflow_dispatch:  # 允许手动触发

jobs:
  update-data:
    runs-on: ubuntu-latest
    steps:
      - name: 检出仓库代码
        uses: actions/checkout@v3

      - name: 获取 Issues 数据
        id: fetch_issues
        run: |
          curl -s -H "Authorization: token ${{ secrets.GITHUB_TOKEN }}" \
               "https://api.github.com/repos/${{ github.repository }}/issues?state=open" > issues.json

      - name: 解析 Issues 并更新 data.json
        run: |
          echo "[]" > data.json
          cat issues.json | jq '[.[] | {timestamp: .title, value: (.body | sub("Random Value: "; "")) | tonumber }]' > data.json
          cat data.json

      - name: 提交更新到 GitHub
        run: |
          git config --global user.name "github-actions"
          git config --global user.email "github-actions@github.com"
          git add data.json
          git commit -m "Updated data.json from issues"
          git push


---

3️⃣ 配置 GitHub

(1) 添加 GitHub Secrets

1. 进入 GitHub 仓库 esp32c3-data-logger


2. 打开 Settings → Secrets → Actions


3. 创建一个新 Secret：

Name：GITHUB_TOKEN

Value：填入你的 GitHub PAT 访问令牌





---

4️⃣ 数据可视化

你的 index.html 代码不变，但 data.json 会由 GitHub Actions 自动更新。

访问 GitHub Pages

1. 确保 GitHub Pages 启用（Settings → Pages）。


2. 访问 https://你的GitHub用户名.github.io/esp32c3-data-logger/，即可看到实时更新的图表。




---

5️⃣ 运行步骤

1. ESP32 代码编译 & 烧录



idf.py build flash monitor

2. GitHub Actions 自动更新 data.json


3. 在 GitHub Pages 上查看实时数据




---

总结

✅ ESP32 每 3 分钟发送数据到 GitHub Issues
✅ GitHub Actions 每 5 分钟合并数据到 data.json
✅ GitHub Pages 实时展示可视化数据

🔹 这样可以有效减少 GitHub API 请求频率限制，ESP32 端也更轻量！ 🚀
如果有问题，请告诉我！

