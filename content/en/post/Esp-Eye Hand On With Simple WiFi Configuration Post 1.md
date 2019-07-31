---
date: 2019-07-30
title: Esp-Eye Hand On With Simple WiFi Configuration - Post 1
tags: ["ESP32","ESP-EYE","HTTP"]
categories: ["IoT"]
#summary: summary here
links:

  - icon_pack: fab
    icon: github
    name: Captdns
    url: 'https://github.com/cornelis-61/esp32_Captdns'
  - icon_pack: fas
    icon: blog
    name: WebServer note
    url: 'https://blog.csdn.net/qq_27114397/article/details/89643232'
  - icon_pack: fab
    icon: github
    name: Captive Portal Web UI
    url: 'https://github.com/tripflex/captive-portal-wifi-web'
#To show table of contact, adding below tag.
#{{% toc %}}
---
In recent days, I was studying ESP-EYE development kit (from ESPRESSIF). Try to make some fun experiences with that, now post my first note about how to make a simple web based Wi-Fi captive portal (so much bugs to be fixed, but it works). To lean more about 'Esp-Face' please visit my early post here: [https://vitoho.ml/post/esp-face-development-notes/] ({{<ref "/post/Esp-Face Development Notes.md" >}})

Now let's start our journey!

### Wi-Fi/Httpd headers to be added to the program:

```c
#include "esp_wifi.h"
#include "esp_wifi_types.h"
#include "tcpip_adapter.h"
#include "app_httpd.h"
#include "esp_http_server.h"
```

### Captive DNS

Refer to '**cornelis-61/esp32_Captdns**', you can easily make your esp kit captive URL request to your custom default page.

Link: [https://github.com/cornelis-61/esp32_Captdns](https://github.com/cornelis-61/esp32_Captdns) 

### Adding HTTP handler

Add few HTTP handlers like below, register handler with httpd_register_uri_handler(httpd_handle_t, httpd_uri_t):

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

From now on, the ESP kit can handling the HTTP request.

:beetle:Known issues & to-do list: 

:white_check_mark: Captive portal with Android phone could not do redirect, not solved yet.



### Complete the index page  handler

Refer to [LINK](https://blog.csdn.net/qq_27114397/article/details/89643232) (Chinese), you can make your ESP kit to be a webserver. Just use gzip to pack your html file then flash into chip. 

Using gzip to convert your file into .gz file:

```shell
gzip index.html
```

Adding following codes into the **component.mk** of your project:

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

To invoke your html file, just adding codes like:

```c
tern const unsigned char index_html_gz_start[] asm("_binary_index_html_gz_start");
extern const unsigned char index_html_gz_end[]   asm("_binary_index_html_gz_end");
size_t index_html_gz_len = index_html_gz_end - index_html_gz_start;

httpd_resp_set_type(req, "text/html");
httpd_resp_set_hdr(req, "Content-Encoding", "gzip");
```

So I can make the index handler at the end:

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

### Fetching AP info and showing them in web

In the **index.html**, I will use **fetch** to get wireless AP info from HTTP server then fill JSON format date into checklist.

```javascript
          fetch:function(){
            WiFiPortal.disable_all();
            var userField = document.getElementById("peapuserwrap");
            userField.style.display = "none";
            WiFiPortal.Info.show('Scanning for WiFi AP around...');
            var baseHost = document.location.origin;
            var selector = document.getElementById("networks");
            fetch(`${baseHost}/status?do=fetchap`)
              .then(function (response) {
                return response.json()
              })
              .then(function (state) {
                var ap_num = state["ap_num"];
                //console.log("ap number:"+state["ap_num"]);
                //console.log(state);

                if (ap_num>0){
                  selector.options.length=0;
                  for(var i=0;i<ap_num;i++){
                    var opt = document.createElement("option");
                    var key_ssid = "ssid.ap."+i;
                    //var key_bssid = "bssid.ap."+i;
                    var key_chan = "chan.ap."+i;
                    var key_rssi = "rssi.ap."+i;
                    opt.value = state[key_ssid];
                    if (i<9) var no = '0'+(i+1); else var no=i+1;
                    opt.innerText = no+' | '+state[key_ssid]+'('+state[key_ssid]+') | chl:'+state[key_chan]+' | rssi:'+state[key_rssi];
                    selector.appendChild(opt);
                    WiFiPortal.fetch_complete();
                  }
                  selector.disabled = false;
                  selector.value="ssid.ap.0"
                  //---custormize ssid
                  var optlast = document.createElement("option");
                  optlast.value = '-2';
                  optlast.innerText = 'Or type your SSID...';
                  selector.appendChild(optlast);
                  //---
                  WiFiPortal.Info.show(''+ap_num+' AP(s) have been found.');
                  WiFiPortal.fetch_complete();
                }
                else{
                
                WiFiPortal.Error.show('Sorry, can not find any APs arround.');
                }
            })

          }
```

The status handler in **app_httpd.c**, it will use 'esp_wifi_scan_start(wifi_scan_config_t,true)' to scan AP nearby then respond to HTTP client in JSON. 

```c
static esp_err_t status_handler(httpd_req_t *req){
    static char json_response[1024];
    char*  buf;
    size_t buf_len;
    char operation[16] = {0,};

    buf_len = httpd_req_get_url_query_len(req) + 1;
    if (buf_len > 1) {
        buf = (char*)malloc(buf_len);
        if(!buf){
            httpd_resp_send_500(req);
            return ESP_FAIL;
        }
        if (httpd_req_get_url_query_str(req, buf, buf_len) == ESP_OK) {
            if (httpd_query_key_value(buf, "do", operation, sizeof(operation)) == ESP_OK) {
            } else {
                free(buf);
                httpd_resp_send_404(req);
                return ESP_FAIL;
            }
        } else {
            free(buf);
            httpd_resp_send_404(req);
            return ESP_FAIL;
        }
        free(buf);
    } else {
        httpd_resp_send_404(req);
        return ESP_FAIL;
    }
    //---------------------------------------
    if(!strcmp(operation, "fetchap")){
        //sensor_t * s = esp_camera_sensor_get();
        wifi_scan_config_t scan_conf;
        uint16_t ap_num = 0;
        uint16_t maxAp = 10;
        wifi_ap_record_t apList[10];

        memset(&scan_conf, 0, sizeof(scan_conf));
        //ESP_ERROR_CHECK(esp_wifi_disconnect());
        if(esp_wifi_scan_start(&scan_conf,true)==ESP_OK){
            esp_wifi_scan_get_ap_num(&ap_num);
            if(ap_num>maxAp)    ap_num=maxAp;
            esp_wifi_scan_get_ap_records(&maxAp, apList);
        }
        else{
            esp_wifi_disconnect();
            if(esp_wifi_scan_start(&scan_conf,true)==ESP_OK){
            esp_wifi_scan_get_ap_num(&ap_num);
            if(ap_num>maxAp)    ap_num=maxAp;
            esp_wifi_scan_get_ap_records(&maxAp, apList);
            }
        }

        char * p = json_response;
        *p++ = '{';

        //p+=sprintf(p, "\"framesize\":%u,", s->status.framesize);
        p+=sprintf(p, "\"ap_num\":%u,", ap_num);
        
        for(uint8_t i = 0; i<ap_num; i++){
            wifi_ap_record_t *ap = apList + i;
            p+=sprintf(p, "\"ssid.ap.%d\":\"%s\",\"bssid.ap.%d\":\"%s\",\"rssi.ap.%d\":%d,\"chan.ap.%d\":%d,"
            , i, ap->ssid, i, ap->ssid, i, ap->rssi, i, ap->primary);
            
        }
        p+=sprintf(p,"\"end\":true");
        *p++ = '}';
        *p++ = 0;
        httpd_resp_set_type(req, "application/json");
        httpd_resp_set_hdr(req, "Access-Control-Allow-Origin", "*");
        return httpd_resp_send(req, json_response, strlen(json_response));
    }
    else {
        httpd_resp_send_404(req);
        return ESP_FAIL;
    }
}
```

### Connecting with Wi-Fi

Once user hit [Save&Test] button, will use **fetch** again to post the AP SSID and password to HEEP server. The difference from the pervious step  is that AP info will be putted into the headers.

```javascript
          function connect(){
            var network = document.getElementById('networks').value
            const password = document.getElementById('password').value
            if(network==='-2'){
              if(document.getElementById('puser').value!==null)
              network = document.getElementById('puser').value;
              else network = null;
            }
          
            WiFiPortal.disable_all();
            var baseHost = document.location.origin;
            //const query = `${baseHost}/cmd?do=connect&ssid=${network}&key=${password}`
            const query = `${baseHost}/cmd?do=connect`
            fetch(query,{
              headers: {
                'key' : password,
                'ssid': network
              }
            })
            .then(function (response) {
                return response.json()
                console.log(`request to ${query} finished, status: ${response.status}`)
              })
            .then(function (state) {
              if (state["ERR"]!==''){
                WiFiPortal.Error.show(state["ERR"]);
              }

            })
            WiFiPortal.Info.show('Connecting to ['+network+']...');
            state_fb();
          }
```

So command handler will receive the HTTP request from web UI, then get AP info from headers and try to connect with. One your esp kit is connected with the Wi-Fi AP, you can see the ip from esp-idf monitor.

```c
static esp_err_t cmd_handler(httpd_req_t *req){
    char*  buf;
    size_t buf_len;
    char operation[16] = {0,};

    buf_len = httpd_req_get_url_query_len(req) + 1;
    if (buf_len > 1) {
        buf = (char*)malloc(buf_len);
        if(!buf){
            httpd_resp_send_500(req);
            return ESP_FAIL;
        }
        if (httpd_req_get_url_query_str(req, buf, buf_len) == ESP_OK) {
            if (httpd_query_key_value(buf, "do", operation, sizeof(operation)) == ESP_OK) {
                ESP_LOGI(TAG, "Got query:%s",buf);
            } else {
                free(buf);
                httpd_resp_send_404(req);
                return ESP_FAIL;
            }
        } else {
            free(buf);
            httpd_resp_send_404(req);
            return ESP_FAIL;
        }
        free(buf);
    } else {
        httpd_resp_send_404(req);
        return ESP_FAIL;
    }
    if(!strcmp(operation, "connect")) {
        //char ssid[33];
        //char key[65];
        wifi_config_t wifi_config;

        /* Get header value string length and allocate memory for length + 1,
         * extra byte for null termination */
        buf_len = httpd_req_get_hdr_value_len(req, "ssid") + 1;
        buf = malloc(buf_len);
        /* Copy null terminated value string into buffer */
        if (httpd_req_get_hdr_value_str(req, "ssid", buf, buf_len) == ESP_OK) {
            if(buf!=NULL){
                ESP_LOGI(TAG, "Found ssid:%s, len:%d", buf,buf_len);
                strcpy((char *)wifi_config.sta.ssid, buf);
                buf_len = httpd_req_get_hdr_value_len(req, "key") + 1;
                if(buf_len>1){
                    httpd_req_get_hdr_value_str(req, "key", buf, buf_len);
                }   else buf="";
                strcpy((char *)wifi_config.sta.password, buf);
            }else{
                ESP_LOGE(TAG, "Wrong WiFi AP info!");
                httpd_resp_sendstr_chunk(req, "{\"ERR\":\"Not valid SSID\"}");
                httpd_resp_sendstr_chunk(req, NULL);
                return ESP_FAIL;

            }
        }
            free(buf);
            ESP_LOGI(TAG, "connect to ssid:%s, pwd:%s", wifi_config.sta.ssid, wifi_config.sta.password);
            ESP_ERROR_CHECK(esp_wifi_disconnect());
            char res[512];
            char* p=res;
            switch(esp_wifi_set_config(WIFI_IF_STA,&wifi_config)){
                case ESP_ERR_WIFI_PASSWORD:
                    *p++ = '{';
                    p+=sprintf(p, "\"error\":\"Password Wrong\"");
                    *p++ = '}';
                    *p++ = 0;
                    httpd_resp_set_type(req, "application/json");
                    httpd_resp_set_hdr(req, "Access-Control-Allow-Origin", "*");
                    ESP_LOGI(TAG, "Connect to AP:%s,PWD:%s",wifi_config.sta.ssid,wifi_config.sta.password);
                    return httpd_resp_send(req, res, strlen(res));
                break;
                case ESP_OK:
                    esp_wifi_connect();
                    ESP_LOGI(TAG, "Connected:%s",wifi_config.sta.ssid);
                    httpd_resp_sendstr_chunk(req, "OK");
                    httpd_resp_sendstr_chunk(req, NULL);
                    return ESP_OK;
                break;
                
            }

    }

    return ESP_OK;
}
```

:beetle:Known issues & to-do list:

:white_check_mark: ​Now web UI will show a fake connect successfully message, the next step might be to complete  the cmd_handler, make it return back connection success/failure status.

:white_check_mark: ​When input the wrong password the program will keep try to connect with AP, so need to set up a retry countdown to prevent this. 

:lollipop: One last thing: 

To stream the camera image from esp kit, I use a simple stupid to do with. Just set a 500ms interval to refresh the image. But this is not a quite good way to do so, I am looking for a better way to stream the video from camera. 

```javascript
          function reflash(){
            //console.log('123');
            var img_src = document.getElementById('cam_img');
            img_src.src = "./stream?"+Math.random();

          }
          function f_cam(){
            //console.log('123');
            var s = document.getElementById("cam_img");
            if(s.style.display == "none"){
              s.style.display="block";
              reflashVar = setInterval(reflash,500);
            }
            else{
              s.style.display = "none";
              clearInterval(reflashVar);
              delete reflashVar;
            }
          }
```



{{% alert note %}}

You could also find source code form this :octocat:[GitHub Gist](https://gist.github.com/vitoho/7526d18e3c8697aa4f3c575706828542).

{{% /alert %}}