---
date: 2019-07-30
title: Esp-Eye Hand On With Simple WiFi Configuration - Post 1
tags: ["ESP32","ESP-EYE","HTTP"]
categories: ["IoT"]
#summary: summary here
  - icon_pack: fab
    icon: github
    name: Originally published on
    url: 'https://github.com/cornelis-61/esp32_Captdns'
  - icon_pack: fas
    icon: blog
    name: WebServer reference notes
    url: 'https://blog.csdn.net/qq_27114397/article/details/89643232'
#To show table of contact, adding below tag.
#{{% toc %}}
---

In recent days, I was studying ESP-EYE development kit (from ESPRESSIF). Try to make some fun experiences with that, now post my first note about how to make a simple web based Wi-Fi captive portal (so much bugs to be fixed, but it works). To lean more about 'Esp-Face' please visit my early post here: [https://vitoho.ml/post/esp-face-development-notes/] ({{<ref "/post/Esp-Face Development Notes.md" >}})

Now let's start our journey!

### Wi-Fi/Httpd Header to be added to the program:

```c
#include "esp_wifi.h"
#include "esp_wifi_types.h"
#include "tcpip_adapter.h"
#include "app_httpd.h"
#include "esp_http_server.h"
```



### Captive DNS

Refer to 'cornelis-61/esp32_Captdns', you can easily make your esp kit captive uri request to default page.

Link: [https://github.com/cornelis-61/esp32_Captdns](https://github.com/cornelis-61/esp32_Captdns) 



### Adding HTTP handler

Add few HTTP handler like below, register handler with  httpd_register_uri_handler(httpd_handle_t, httpd_uri_t)

```c
httpd_handle_t camera_httpd = NULL;

	httpd_uri_t index_uri = {
        .uri       = "/",
        .method    = HTTP_GET,
        .handler   = index_handler,
        .user_ctx  = NULL
    };

   httpd_uri_t capture_uri = {
        .uri       = "/stream",
        .method    = HTTP_GET,
        .handler   = capture_handler,
        .user_ctx  = NULL
    };

   httpd_uri_t gen204_uri = {       //Android Gen204
        .uri       = "/generate_204",
        .method    = HTTP_GET,
        .handler   = redirect_handler,
        .user_ctx  = NULL
    };

       httpd_uri_t ios_uri = {       //ios hotspot detact
        .uri       = "/hotspot-detect.html",       
        .method    = HTTP_GET,
        .handler   = redirect_handler,
        .user_ctx  = NULL
    };

        httpd_uri_t status_uri = {
        .uri       = "/status",
        .method    = HTTP_GET,
        .handler   = status_handler,
        .user_ctx  = NULL
    };

        httpd_uri_t cmd_uri = {
        .uri       = "/cmd",
        .method    = HTTP_GET,
        .handler   = cmd_handler,
        .user_ctx  = NULL
    };

    ESP_LOGI(TAG, "Starting web server on port: '%d'", config.server_port);
    config.max_resp_headers = 1204;
    if (httpd_start(&camera_httpd, &config) == ESP_OK) {
        httpd_register_uri_handler(camera_httpd, &index_uri);
        httpd_register_uri_handler(camera_httpd, &gen204_uri);
        httpd_register_uri_handler(camera_httpd, &capture_uri);
        httpd_register_uri_handler(camera_httpd, &ios_uri);
        httpd_register_uri_handler(camera_httpd, &status_uri);
        httpd_register_uri_handler(camera_httpd, &cmd_uri);
        //err handler
        httpd_register_err_handler(camera_httpd,HTTPD_404_NOT_FOUND,http_404_error_handler);

    }

```

From now on, the ESP kit should can handling your HTTP request.

Know issues & to-do list: 

- [x] Captive portal with Android phone could not do redirect, not solved yet.



### Complete the index page  handler

Refer to [LINK](https://blog.csdn.net/qq_27114397/article/details/89643232) (Chinese), you can make your ESP kit to be a webserver. Just use gzip to pack your html file then flash into chip. 

Using gzip to convert your file into .gz file:

```shell
gzip index.html
```

Adding following into the **component.mk** of your project:

```
COMPONENT_EMBED_FILES := www/index.html.gz
COMPONENT_EMBED_FILES += www/something_else
```
Adding your file to CMakeLists:
```
set(COMPONENT_EMBED_FILES
        "www/index.html.gz"
	"www/redirect.html.gz"
   )
```

To invoke your html file, just adding some codes like:

```c
tern const unsigned char index_html_gz_start[] asm("_binary_index_html_gz_start");
extern const unsigned char index_html_gz_end[]   asm("_binary_index_html_gz_end");
size_t index_html_gz_len = index_html_gz_end - index_html_gz_start;

httpd_resp_set_type(req, "text/html");
httpd_resp_set_hdr(req, "Content-Encoding", "gzip");
```

So I can make the index handler as this:

```c
static esp_err_t index_handler(httpd_req_t *req){
    extern const unsigned char index_html_gz_start[] asm("_binary_index_html_gz_start");
    extern const unsigned char index_html_gz_end[]   asm("_binary_index_html_gz_end");
    size_t index_html_gz_len = index_html_gz_end - index_html_gz_start;

    httpd_resp_set_type(req, "text/html");
    httpd_resp_set_hdr(req, "Content-Encoding", "gzip");
    
    //httpd_resp_set_status(req, HTTPD_204);
    return httpd_resp_send(req, (const char *)index_html_gz_start, index_html_gz_len);
}
```



# [NOT-COMPLETE]

{{% alert note %}}

You can also find source code form [GitHub Gist](https://gist.github.com/vitoho/7526d18e3c8697aa4f3c575706828542).

{{% /alert %}}