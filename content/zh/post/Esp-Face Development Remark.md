---
date: 2019-07-09
title: Esp-Face Development Remark
tags: ["ESP32","ESP-EYE"]
categories: ["IoT"]
summary: A Remark document for ESP-Face human recognition solution
links:
  - icon_pack: fab
    icon: github
    name: ESP-Face on GitHub
    url: 'https://github.com/espressif/esp-face'
  - icon_pack: fab
    icon: github
    name: ESP-Who framework on GitHub
    url: 'https://github.com/espressif/esp-who'
  - icon_pack: fas
    icon: microchip
    name: ESP-EYE Overview
    url: 'https://www.espressif.com/en/products/hardware/esp-eye/overview'
#To show tableof contact, adding below tag.
#{{% toc %}}
---

> I found a interesting human (face) recognition solution framework for ESP-EYE which is a -development board for image recognition.

And here I will mark down some key steps about face recognition for future recalling.

The whole project is based on [[**app_facenet.c**](https://github.com/espressif/esp-who/blob/master/examples/single_chip/recognition_wechat/main/app_facenet.c)] in example [**recognition_wechat**].

First of all, please make sure below header added to to your project.

```c
/* 
Image size refrence
SXGA(1280x1024)
XGA(1024x768)
SVGA(800x600)
VGA(640x480)
CIF(400x296)
*QVGA(320x240)
HQVGA(240x176)
QQVGA(160x120)
*/

//define image size
#define img_width 240
#define img_high 176
#define main_face 160

#define ENROLL_CONFIRM_TIMES 3
#define FACE_ID_SAVE_NUMBER 3
static face_id_list id_list = {0};

face_id_name_list st_face_list = {0};
static dl_matrix3du_t *aligned_face = NULL;

static bool g_state_LOCK = FALSE;

extern QueueHandle_t gpst_input_image;
extern QueueHandle_t gpst_output;

static inline mtmn_config_t app_mtmn_config()
{
    mtmn_config_t mtmn_config = {0};
    mtmn_config.type = NORMAL;
    mtmn_config.min_face = main_face;
    mtmn_config.pyramid = 0.99;
    mtmn_config.pyramid_times = 4;
    mtmn_config.p_threshold.score = 0.6;
    mtmn_config.p_threshold.nms = 0.7;
    mtmn_config.p_threshold.candidate_number = 20;
    mtmn_config.r_threshold.score = 0.7;
    mtmn_config.r_threshold.nms = 0.7;
    mtmn_config.r_threshold.candidate_number = 10;
    mtmn_config.o_threshold.score = 0.7;
    mtmn_config.o_threshold.nms = 0.7;
    mtmn_config.o_threshold.candidate_number = 1;

    return mtmn_config;
}

static inline box_array_t *do_detection(camera_fb_t *fb, dl_matrix3du_t *image_mat, mtmn_config_t *mtmn_config)
{
    if(!fmt2rgb888(fb->buf, fb->len, fb->format, image_mat->item))
    {
        ESP_LOGW(TAG, "fmt2rgb888 failed");
        return NULL;
    }

    box_array_t *net_boxes = face_detect(image_mat, mtmn_config);
    return net_boxes;
}
```

Face ID initialization:

```c
face_id_init(&id_list, FACE_ID_SAVE_NUMBER, ENROLL_CONFIRM_TIMES);
aligned_face = dl_matrix3du_alloc(1, FACE_WIDTH, FACE_HEIGHT, 3);
```

Adding a task process for face handling task:

```c
xTaskCreatePinnedToCore(&task_process, "process", 4 * 1024, NULL, 5, NULL, 1);
```

In the **task_process(void *arg)**, please add blow codes first to getting ready for image capturing.

```c
    camera_fb_t * fb = NULL;
    dl_matrix3du_t *image_matrix = dl_matrix3du_alloc(1, img_width, img_high, 3);
    if (!image_matrix)
    { 
        ESP_LOGE(TAG, "dl_matrix3du_alloc failed");
        return;
    }

    //http_img_process_result out_res = {0};
    //out_res.lock = xSemaphoreCreateBinary();

    //out_res.image = image_matrix->item;
    //--------------------------------------------
    size_t frame_num = 0;
    
    /* 1. Load configuration for detection */
    mtmn_config_t mtmn_config = app_mtmn_config();

    /*flipping image*/
    sensor_t *s = esp_camera_sensor_get();
    //s->set_hmirror(s, 1);
    s->set_vflip(s,1);

    /* face id enroll index*/
    int8_t next_enroll_index = 0;
    int8_t left_sample_face;
```

Then following steps should  be run within the loop and keep going.

```c
        //enroll state checking
        if (g_state == START_ENROLL){
        if (id_list.count >= FACE_ID_SAVE_NUMBER){
        g_state = START_DETECT;
        ESP_LOGE(TAG, "Face ID(s) reached the maxinum %d. Please reset and enroll new faces.",FACE_ID_SAVE_NUMBER);
        }else
        ESP_LOGE(TAG, ">>>Enroll Face ID Mode. Face ID:%d<<<",id_list.count);
        }else  g_state = START_DETECT;
```



```c

        int64_t start_time = esp_timer_get_time();
        /* 2. Get one image with camera */
        //ESP_LOGI(TAG, "2.Start capture image");
        fb = esp_camera_fb_get();
        if (!fb)
        {
            ESP_LOGE(TAG, "Camera capture failed");
            continue;
        }
```



```c
       /* 3. Allocate image matrix to store RGB data */
        image_matrix = dl_matrix3du_alloc(1, fb->width, fb->height, 3);
```



```c
        /* 4. Transform image to RGB */
        //ESP_LOGI(TAG, "4.Start transform image to RGB");
        uint32_t res = fmt2rgb888(fb->buf, fb->len, fb->format, image_matrix->item);
        if (true != res)
        {
            ESP_LOGE(TAG, "fmt2rgb888 failed, fb: %d", fb->len);
            dl_matrix3du_free(image_matrix);
            continue;
        }
        esp_camera_fb_return(fb);
```



```c
        /* 5. Do face detection */
        //ESP_LOGI(TAG, "5.Start face detection");
        box_array_t *net_boxes = face_detect(image_matrix, &mtmn_config);
        ESP_LOGI(TAG, "Detection time consumption: %lldms", (esp_timer_get_time() - start_time) / 1000);
```



```c
		//face detected
		if (net_boxes)
        {
            g_state_LOCK = TRUE;
            frame_num++;
            if (align_face(net_boxes, image_matrix, aligned_face) == ESP_OK)
            {
                ESP_LOGE(TAG, "Detacted Count: %d\n", frame_num);
                detacted_process();

                // Enroll new face id
                if (g_state == START_ENROLL)
                {
                        left_sample_face = enroll_face(&id_list, aligned_face);
                        
                        ESP_LOGI(TAG, "Face ID Enrollment: Taking the %d sample",
                                ENROLL_CONFIRM_TIMES - left_sample_face
                                //number_suffix(ENROLL_CONFIRM_TIMES - left_sample_face)
                                );
                        ESP_LOGI(TAG, "Left sample face to enroll: %d", left_sample_face);
                        gpio_set_level(GPIO_LED_RED, 0);
                        gpio_set_level(GPIO_LED_WHITE, 0);

                        if (left_sample_face == 0)
                        {
                            next_enroll_index++;
                            ESP_LOGI(TAG, "\nEnrolled Face ID: %d", id_list.count-1);
                            vTaskDelay(2000 / portTICK_PERIOD_MS);

                            if (id_list.count == FACE_ID_SAVE_NUMBER)
                            {
                                g_state = START_DETECT;
                                ESP_LOGI(TAG, "\n>>> Face Recognition Starts <<<\n");
                                vTaskDelay(2000 / portTICK_PERIOD_MS);
                            }
                            else
                            {
                                g_state = START_DETECT;
                                ESP_LOGI(TAG, "Please press button to start enrolling another one.\n>>> Left faces count: %d",
                                FACE_ID_SAVE_NUMBER - id_list.count
                                );
                                vTaskDelay(2500 / portTICK_PERIOD_MS);
                            }
                        }
                }
                else{
                    /* 6. Do face recognition */
                    int64_t recog_match_time = esp_timer_get_time();

                    int matched_id = recognize_face(&id_list, aligned_face);

                        ESP_LOGI(TAG, "Recognition time consumption: %lldms",
                        (esp_timer_get_time() - recog_match_time) / 1000);

                    if (matched_id >= 0)
                    {
                        ESP_LOGI(TAG, "Matched Face ID: %d\n", matched_id);
                        recognized_process();
                    }
                    else
                    {
                        ESP_LOGI(TAG, "No Matched Face ID\n");
                        vTaskDelay(5000 / portTICK_PERIOD_MS);
                    }
                }
            }
            else{
                ESP_LOGI(TAG, "Detected face is not proper.");
            }

            //if (g_state == START_DETECT){
            //ESP_LOGE(TAG, "Wait for 5 sec to continue next detaction ...");
            //vTaskDelay(5000 / portTICK_RATE_MS);
            //}
            free(net_boxes->score);
            free(net_boxes->box);
            free(net_boxes->landmark);
            free(net_boxes);
            g_state_LOCK = FALSE;
        }
```

Remember to free up image memory before next image capturing:

```c
    dl_matrix3du_free(image_matrix);
    //g_state = WAIT_FOR_CONNECT;
    //ESP_LOGI(TAG, "Pending for 1 sec...");
    vTaskDelay(500 / portTICK_RATE_MS);
```

**detacted_process()** and **recognized_process()** like below:

```c
void detacted_process(){
    ESP_LOGE(TAG, "Hi :)");
    if (g_state != START_ENROLL) g_state = STA_DETECTED;
    //add something here
    
}

void recognized_process(){
    ESP_LOGE(TAG, "Hi. It's you! :)");
    g_state = STA_RECOGNIZED;
    vTaskDelay(10000 / portTICK_RATE_MS);
    //add somthing here
    
}
```

