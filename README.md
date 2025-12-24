# smart_weather_clock

#include <stdbool.h>    /* 用尖括号<>引用标准库用 */
#include <stdint.h>
#include <stdio.h>
#include <string.h>
#include "stm32f10x.h"  /* 用双引号""引用自定义库 */
#include "main.h"
#include "led.h"
#include "rtc.h"
#include "timer.h"
#include "esp_at.h"
#include "mpu6050.h"
#include "st7735.h"
#include "stfonts.h"
#include "stimage.h"
#include "weather.h"

/* WIFI名称 */
static const char *wifi_ssid = "SJY";
/* WIFI密码 */
static const char *wifi_password = "SJY20040507";
/* 心知天气获取本地天气的URL链接 */
static const char *weather_uri = "https://api.seniverse.com/v3/weather/now.json?key=SEJICVxMVvVm92Yp3&location=hefei&language=en&unit=c";

/* 运行计数器，每1s自增1 */
static uint32_t runms;
/* 显示屏待显示内容的Y坐标 */
static uint32_t disp_height;

/**
 * @brief 1ms定时器回调，提供毫秒计数值给主程序使用
 * 计数器超过1天就重置
 *
 */
static void timer_elapsed_callback(void)
{
    runms++;
    /* 24小时 x 60分钟 x 60秒 x 1000毫秒 */
    if (runms > 24 * 60 * 60 * 1000)
    {
        /* 计数器归零 */
        runms = 0;
    }
}

/**
 * @brief 初始化wifi模块，并显示初始化信息在屏幕上
 *
 */
static void wifi_init(void)
{
    /* ST7735 LCD屏幕打印Init ESP32...日志 */
    st7735_write_string(0, disp_height, "Init ESP32...", &font_ascii_8x16, ST7735_WHITE, ST7735_BLACK);
    /* 更新下一条显示日志的Y坐标 */
    disp_height += font_ascii_8x16.height;
    /* 1、初始化ESP32_AT功能，主要是初始化主控和ESP32C3模块之间的串口 */
    if (!esp_at_init())
    {
        /* 初始化失败，屏幕用红色打印Failed!!!提示 */
        st7735_write_string(0, disp_height, "Failed!!!", &font_ascii_8x16, ST7735_RED, ST7735_BLACK);
        disp_height += font_ascii_8x16.height;
        /* 初始化失败，程序停留在while(1)处不允许继续运行后续代码，用户需要主动重启设备 */
        while (1);
    }

    st7735_write_string(0, disp_height, "Init WIFI...", &font_ascii_8x16, ST7735_WHITE, ST7735_BLACK);
    disp_height += font_ascii_8x16.height;
    /* 2、初始化esp32的wifi模块，设置wifi为station模式 */
    if (!esp_at_wifi_init())
    {
        st7735_write_string(0, disp_height, "Failed!!!", &font_ascii_8x16, ST7735_RED, ST7735_BLACK);
        disp_height += font_ascii_8x16.height;
        while (1);
    }

    st7735_write_string(0, disp_height, "Connect WIFI...", &font_ascii_8x16, ST7735_WHITE, ST7735_BLACK);
    disp_height += font_ascii_8x16.height;
    /* 3、配置esp32模块的wifi ssid和passwd，让esp32连接到wifi */
    if (!esp_at_wifi_connect(wifi_ssid, wifi_password))
    {
        st7735_write_string(0, disp_height, "Failed!!!", &font_ascii_8x16, ST7735_RED, ST7735_BLACK);
        disp_height += font_ascii_8x16.height;
        while (1);
    }

    st7735_write_string(0, disp_height, "Sync Time...", &font_ascii_8x16, ST7735_WHITE, ST7735_BLACK);
    disp_height += font_ascii_8x16.height;
    /* 4、设置esp32的sntp模块，sntp是ntp（Network Time Protocol）的简化版，用来使进行时钟同步，从互联网上获取当前的年月日时分秒 */
    if (!esp_at_sntp_init())
    {
        st7735_write_string(0, disp_height, "Failed!!!", &font_ascii_8x16, ST7735_RED, ST7735_BLACK);
        disp_height += font_ascii_8x16.height;
        while (1);
    }
}

/**
 * @brief 主程序入口
 *
 * @return main程序不应返回
 */
int main(void)
{
    /* 初始化board，配置中断组，启动外设时钟，启动LSE晶振 */
    board_lowlevel_init();

    led_init();
    rtc_init();     /* rtc本地时钟用于本地获取当前年月日时分秒，不用每次都从sntp获取 */

    /* 定时器2配置为1ms周期触发，即每1ms执行一次timer_elapsed_callback回调函数 */
    timer_init(1000);
    timer_elapsed_register(timer_elapsed_callback);
    timer_start();

    mpu6050_init();

    st7735_init();
    st7735_fill_screen(ST7735_BLACK);

    /* 显示开机提示语，并更新显示屏显示下一行内容的Y坐标 */
    st7735_write_string(0, 0, "Initializing...", &font_ascii_8x16, ST7735_WHITE, ST7735_BLACK);
    disp_height += font_ascii_8x16.height;
    delay_ms(500);  /* 延迟500ms，LCD显示每一条语句慢一点，用户体验更好 */

    st7735_write_string(0, disp_height, "Wait ESP32...", &font_ascii_8x16, ST7735_WHITE, ST7735_BLACK);
    disp_height += font_ascii_8x16.height;
    delay_ms(1500);

    wifi_init();

    /* 开机准备流程完毕，显示一个绿色的Ready后，稍微等500ms让用户看清Ready文字，进入主页面 */
    st7735_write_string(0, disp_height, "Ready", &font_ascii_8x16, ST7735_GREEN, ST7735_BLACK);
    disp_height += font_ascii_8x16.height;
    delay_ms(500);

    /* 清屏，开机日志内容不用显示了 */
    st7735_fill_screen(ST7735_BLACK);

    runms = 0;  /* 复位runms，第一次开机应当所有内容都显示出来 */
    uint32_t last_runms = runms;    /* 当前的runms的值，避免在显示第一个内容的时候，runms更新导致后面的内容无法显示 */
    bool weather_ok = false;    /* 天气获取成功标志位，成功读取天气后，10分钟再次更新，否则下一个循环再次尝试获取天气 */
    bool sntp_ok = false;   /* 与天气同理 */
    char str[64];   /* 字符串打印缓存 */
    while (1)
    {
        /* 每1ms才执行一次后面的代码，当定时器回调执行之后，runms++，与last_runms不匹配，continue就不执行 */
        if (runms == last_runms)
        {
            continue;
        }
        /* 更新last_runms值 */
        last_runms = runms;

        /* 更新LCD时间信息，每100ms更新一次lcd上时间内容 */
        if (last_runms % 100 == 0)
        {
            /* 从STM32F4的RTC模块读取年月日时分秒 */
            rtc_date_t date;
            rtc_get_date(&date);
            /* 样例：2024年10月18日则显示：24-10-18 */
            snprintf(str, sizeof(str), "%02d-%02d-%02d", date.year % 100, date.month, date.day);
            st7735_write_string(0, 0, str, &font_ascii_8x16, ST7735_WHITE, ST7735_BLACK);
            /* 样例：12:34，并且中间的":"会1s闪烁一次 */
            snprintf(str, sizeof(str), "%02d%s%02d", date.hour, date.second % 2 ? " " : ":", date.minute);
            st7735_write_string(0, 78, str, &font_time_24x48, ST7735_CYAN, ST7735_BLACK);
        }

        /* 联网同步时间 */
        if (!sntp_ok || last_runms % (60 * 60 * 1000) == 0)
        {
            /* esp32从sntp服务器读取当前时间，并校准stm32f4的rtc时间 */
            uint32_t ts;
            sntp_ok = esp_at_get_time(&ts);
            rtc_set_timestamp(ts + 8 * 60 * 60);
        }

        /* 更新天气信息 */
        if (!weather_ok || last_runms % (10 * 60 * 1000) == 0)
        {
            /* esp32从心知天气获取本地天气数据，返回json结果 */
            const char *rsp;
            weather_ok = esp_at_get_http(weather_uri, &rsp, NULL, 10000);
            /* 解析json，获取天气数据 */
            weather_t weather;
            weather_parse(rsp, &weather);

            /* 根据天气内容，选择不同的图片显示 */
            const st_image_t *img = NULL;
            if (strcmp(weather.weather, "Cloudy") == 0) {
                img = &icon_weather_duoyun;
            } else if (strcmp(weather.weather, "Wind") == 0) {
                img = &icon_weather_feng;
            } else if (strcmp(weather.weather, "Clear") == 0) {
                img = &icon_weather_qing;
            } else if (strcmp(weather.weather, "Snow") == 0) {
                img = &icon_weather_xue;
            } else if (strcmp(weather.weather, "Overcast") == 0) {
                img = &icon_weather_yin;
            } else if (strcmp(weather.weather, "Rain") == 0) {
                img = &icon_weather_yu;
            }

            if (img != NULL) { /* 如果有匹配的天气，则显示天气图标 */
                st7735_draw_image(0, 16, img->width, img->height, img->data);
            } else { /* 否则直接显示天气文字 */
                snprintf(str, sizeof(str), "%s", weather.weather);
                st7735_write_string(0, 16, str, &font_ascii_8x16, ST7735_YELLOW, ST7735_BLACK);
            }

            /* 显示天气温度 */
            snprintf(str, sizeof(str), "%sC", weather.temperature);
            st7735_write_string(78, 0, str, &font_temper_16x32, ST7735_BLUE, ST7735_BLACK);
        }

        /* 显示环境温度 */
        if (last_runms % (1 * 1000) == 0)
        {
            /* 环境温度从mpu6050读取 ^_^ */
            float temper = mpu6050_read_temper();
            snprintf(str, sizeof(str), "%5.1fC", temper);
            st7735_write_string(78, 32, str, &font_ascii_8x16, ST7735_GREEN, ST7735_BLACK);
        }

        // 更新网络信息
        // if (last_runms % (30 * 1000) == 0)
        // {
        //     st7735_write_string(0, 127-48, wifi_ssid, &font_ascii_8x16, ST7735_WHITE, ST7735_BLACK);
        //     char ip[16];
        //     esp_at_wifi_get_ip(ip);
        //     st7735_write_string(0, 127-32, ip, &font_ascii_8x16, ST7735_WHITE, ST7735_BLACK);
        // }
    }
}


