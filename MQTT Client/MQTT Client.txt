// HTTP Client - FreeRTOS ESP IDF - GET
#include <stdio.h>
#include <string.h>
#include <time.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "freertos/timers.h"
#include "freertos/event_groups.h"
#include "esp_wifi.h"
#include "esp_log.h"
#include "nvs_flash.h"
#include "esp_netif.h"
#include "esp_http_client.h"
#include  "cJSON.h"

#include <stdint.h>
#include <stddef.h>

#include "freertos/semphr.h"
#include "freertos/queue.h"

#include "lwip/sockets.h"
#include "lwip/dns.h"
#include "lwip/netdb.h"

#include "esp_system.h"
#include "nvs_flash.h"
#include "esp_event.h"
#include "esp_netif.h"

#include "mqtt_client.h"

//Output Data buffers
char loc_out[100];
char time_out[200];
char temp_out[100];
char hum_out[100];
char cond_out[100];
char wind_out[100];

static const char *TAG = "MQTT_TCP";

static esp_mqtt_client_handle_t client = NULL;


static void wifi_event_handler(void *event_handler_arg, esp_event_base_t event_base, int32_t event_id, void *event_data)
{
    switch (event_id)
    {
    case WIFI_EVENT_STA_START:
        printf("WiFi connecting ... \n");
        break;
    case WIFI_EVENT_STA_CONNECTED:
        printf("WiFi connected ... \n");
        break;
    case WIFI_EVENT_STA_DISCONNECTED:
        printf("WiFi lost connection ... \n");
        break;
    case IP_EVENT_STA_GOT_IP:
        printf("WiFi got IP ... \n\n");
        break;
    default:
        break;
    }
}

void wifi_connection()
{
    // 1 - Wi-Fi/LwIP Init Phase
    esp_netif_init();                    // TCP/IP initiation                   s1.1
    esp_event_loop_create_default();     // event loop                          s1.2
    esp_netif_create_default_wifi_sta(); // WiFi station                        s1.3
    wifi_init_config_t wifi_initiation = WIFI_INIT_CONFIG_DEFAULT();
    esp_wifi_init(&wifi_initiation); //                                         s1.4
    // 2 - Wi-Fi Configuration Phase
    esp_event_handler_register(WIFI_EVENT, ESP_EVENT_ANY_ID, wifi_event_handler, NULL);
    esp_event_handler_register(IP_EVENT, IP_EVENT_STA_GOT_IP, wifi_event_handler, NULL);
    wifi_config_t wifi_configuration = {
        .sta = {
            .ssid = "Nadim",
            .password = "12345679"}};
    esp_wifi_set_config(ESP_IF_WIFI_STA, &wifi_configuration);
    // 3 - Wi-Fi Start Phase
    esp_wifi_start();
    // 4- Wi-Fi Connect Phase
    esp_wifi_connect();
}

char    buffer2[1024]; //stores weather info sent from http server
esp_err_t client_event_get_handler(esp_http_client_event_handle_t evt)
{
    char    buffer1[512]; //temporary buffer used for concatination or char types

    switch (evt->event_id)
    {
    case HTTP_EVENT_ON_DATA:
        //concatination of each data packet to using buffer 1 to buffer 2
        memset(buffer1, 0, sizeof(buffer1));
        sprintf(buffer1,(char *)evt->data);
        strncat(buffer2, buffer1, evt->data_len);
        break;
         
    default:
        break;
    }
    return ESP_OK;
}

static void rest_get()
{
    esp_http_client_config_t config_get = {
        .url = "http://api.openweathermap.org/data/2.5/weather?q=Beirut,LB&units=metric&appid=ce98450d4f5255e04b5d0cb14942371c",
        .method = HTTP_METHOD_GET,
        .cert_pem = NULL,
        .event_handler = client_event_get_handler};
       
    esp_http_client_handle_t client = esp_http_client_init(&config_get);
    esp_http_client_perform(client);
    esp_http_client_cleanup(client);
}

static esp_err_t mqtt_event_handler_cb(esp_mqtt_event_handle_t event)
{
    switch (event->event_id)
    {
    case MQTT_EVENT_CONNECTED:
        ESP_LOGI(TAG, "MQTT_EVENT_CONNECTED");
        //Subscribing to topics when connected
        esp_mqtt_client_subscribe(event->client, "button", 0);
        break;
    case MQTT_EVENT_DISCONNECTED:
        ESP_LOGI(TAG, "MQTT_EVENT_DISCONNECTED");
        break;
    case MQTT_EVENT_SUBSCRIBED:
        ESP_LOGI(TAG, "MQTT_EVENT_SUBSCRIBED, msg_id=%d", event->msg_id);
        break;
    case MQTT_EVENT_UNSUBSCRIBED:
        ESP_LOGI(TAG, "MQTT_EVENT_UNSUBSCRIBED, msg_id=%d", event->msg_id);
        break;
    case MQTT_EVENT_PUBLISHED:
        ESP_LOGI(TAG, "MQTT_EVENT_PUBLISHED, msg_id=%d", event->msg_id);
        break;
    case MQTT_EVENT_DATA:
        ESP_LOGI(TAG, "MQTT_EVENT_DATA");
        printf("\nTOPIC=%.*s\r\n", event->topic_len, event->topic);
        printf("DATA=%.*s\r\n", event->data_len, event->data);
        esp_mqtt_client_publish(event->client,"Location/Name/Country",loc_out,sizeof(loc_out),0,0);
        esp_mqtt_client_publish(event->client,"Time",time_out,sizeof(time_out),0,0);
        esp_mqtt_client_publish(event->client,"Temp",temp_out,sizeof(temp_out),0,0);
        esp_mqtt_client_publish(event->client,"Hum",hum_out,sizeof(hum_out),0,0);
        esp_mqtt_client_publish(event->client,"Cond",cond_out,sizeof(cond_out),0,0);
        esp_mqtt_client_publish(event->client,"Wind/Speed/Direction",wind_out,sizeof(wind_out),0,0);
        break;
    case MQTT_EVENT_ERROR:
        ESP_LOGI(TAG, "MQTT_EVENT_ERROR");
        break;
    default:
        ESP_LOGI(TAG, "Other event id:%d", event->event_id);
        break;
    }
    return ESP_OK;
}

static void mqtt_event_handler(void *handler_args, esp_event_base_t base, int32_t event_id, void *event_data)
{
    mqtt_event_handler_cb(event_data);
}

static void mqtt_app_start(void)
{
    esp_mqtt_client_config_t mqtt_cfg = {
        .uri = "MQTT://MQTT.eclipseprojects.io",
    };
    client = esp_mqtt_client_init(&mqtt_cfg);
    esp_mqtt_client_register_event(client, ESP_EVENT_ANY_ID, mqtt_event_handler, client);
    esp_mqtt_client_start(client);
}

void app_main(void)
{
    nvs_flash_init();
    wifi_connection();
    mqtt_app_start();

    vTaskDelay(2000 / portTICK_PERIOD_MS);
    printf("WIFI was initiated............\n\n");

    rest_get();

    char*ptr=buffer2;
    cJSON *json=cJSON_Parse(ptr);
    //Line 1 (Name and Country)
    char *location=cJSON_GetObjectItem(json,"name")->valuestring;
    cJSON *sys=cJSON_GetObjectItem(json,"sys");
    char *cntr=cJSON_GetObjectItem(sys,"country")->valuestring;

    sprintf(loc_out,"<i class= \"fa fa-location-arrow \" aria-hidden= \"true \"></i> %s, %s \n", (char*)location,(char*)cntr);

    //Line 2 (Time)
    int dt=cJSON_GetObjectItem(json,"dt")->valueint;
    int timezone=cJSON_GetObjectItem(json,"timezone")->valueint;
    time_t epoch= dt+timezone; //Coversion start
    struct tm ts;
    char buff [80];
    ts = * localtime (& epoch );
    strftime (buff , sizeof ( buff ), "%A, %B %d,%Y %I:%M:%S %p", &ts );

    sprintf(time_out,"<i class=\"fa fa-calendar\" aria-hidden=\"true\"></i> %s \n",buff); 
    
    //Line 3 (Temp)
    cJSON *pmain=cJSON_GetObjectItem(json,"main");
    double temp=cJSON_GetObjectItem(pmain,"temp")->valuedouble;

    sprintf(temp_out,"<i class=\"fa fa-thermometer-three-quarters\" aria-hidden=\"true\"></i> %.2f C \n",temp);

    //Line 4 (Hum)
    double hum=cJSON_GetObjectItem(pmain,"humidity")->valuedouble;

    sprintf(hum_out,"<i class=\"fa fa-tint\" aria-hidden=\"true\"></i> %.2f %% \n",hum);

    //Line 5 (Conditions)
    cJSON *weather=cJSON_GetObjectItem(json,"weather");
    cJSON *Arrw=cJSON_GetArrayItem(weather,0);
    char *description=cJSON_GetObjectItem(Arrw,"description")->valuestring;

    sprintf(cond_out,"<i class=\"fa fa-cloud\" aria-hidden=\"true\"></i> %s \n", description);

    //Line 6 (wind)
    cJSON *wind=cJSON_GetObjectItem(json,"wind");
    double speed=cJSON_GetObjectItem(wind,"speed")->valuedouble;
    int degrees=cJSON_GetObjectItem(wind,"deg")->valueint;
    
    cJSON *gust_extract=cJSON_GetObjectItem(wind,"gust");
    bool gust_bool=cJSON_IsFalse(gust_extract);//returns true if gust is in JSON file and false otherwise

    if (gust_bool==true){
        double gust=cJSON_GetObjectItem(wind,"gust")->valuedouble;
        if (degrees>=345 || degrees<30){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s and a gust of %.2f \n",speed,"N",gust);
        }
        else if (degrees>=30 && degrees<75){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s and a gust of %.2f \n",speed,"NE",gust);
        }
        else if (degrees>=75 && degrees<120){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s and a gust of %.2f \n",speed,"E",gust);
        }
        else if (degrees>=120 && degrees<165){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s and a gust of %.2f \n",speed,"SE",gust);
        }
        else if (degrees>=165 && degrees<210){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s and a gust of %.2f \n",speed,"S",gust);
        }
        else if (degrees>=210 && degrees<255){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s and a gust of %.2f \n",speed,"SW",gust);
        }
        else if (degrees>=255 && degrees<300){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s and a gust of %.2f \n",speed,"W",gust);
            }
        else if (degrees>=300 && degrees<345){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s and a gust of %.2f \n",speed,"NW",gust);
        }
    }
    else{
        if (degrees>=345 || degrees<30){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s \n",speed,"N");
        }
        else if (degrees>=30 && degrees<75){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s \n",speed,"NE");
        }
        else if (degrees>=75 && degrees<120){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s \n",speed,"E");
        }
        else if (degrees>=120 && degrees<165){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s \n",speed,"SE");
        }
        else if (degrees>=165 && degrees<210){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s \n",speed,"S");
        }
        else if (degrees>=210 && degrees<255){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s \n",speed,"SW");
        }
        else if (degrees>=255 && degrees<300){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s \n",speed,"W");
            }
        else if (degrees>=300 && degrees<345){
            sprintf(wind_out,"<i class=\"fa fa-paper-plane\" aria-hidden=\"true\"></i> %.2f km/h from the %s \n",speed,"NW");
        }
    }
}