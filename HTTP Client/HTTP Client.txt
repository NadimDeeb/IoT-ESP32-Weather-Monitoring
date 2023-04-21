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
            .ssid = "Hala iPhone",
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

void app_main(void)
{
    nvs_flash_init();
    wifi_connection();

    vTaskDelay(2000 / portTICK_PERIOD_MS);
    printf("WIFI was initiated............\n\n");

    rest_get();
    char*ptr=buffer2;
    cJSON *json=cJSON_Parse(ptr);

    //Line 1 (Name and Country)
    char *location=cJSON_GetObjectItem(json,"name")->valuestring;
    cJSON *sys=cJSON_GetObjectItem(json,"sys");
    char *cntr=cJSON_GetObjectItem(sys,"country")->valuestring;
    
    printf("Weather condition in %s, %s \n",(char*)location, (char*)cntr);
    
    //Line 2 (Time)
    int dt=cJSON_GetObjectItem(json,"dt")->valueint;
    int timezone=cJSON_GetObjectItem(json,"timezone")->valueint;
    time_t epoch= dt+timezone; //Coversion start
    struct tm ts;
    char buff [80];
    ts = * localtime (& epoch );
    strftime (buff , sizeof ( buff ), "%A, %B %d,%Y %I:%M:%S %p", &ts );
    
    printf ("%s \n", buff );
    
    //Line 3 (Temp)
    cJSON *pmain=cJSON_GetObjectItem(json,"main");
    double temp=cJSON_GetObjectItem(pmain,"temp")->valuedouble;

    printf("Temprature: %g C \n",temp);
    
    //Line 4 (Hum)
    double hum=cJSON_GetObjectItem(pmain,"humidity")->valuedouble;
    
    printf("Humidity: %g %% \n", hum);
    
     //Line 5 (Conditions)
    cJSON *weather=cJSON_GetObjectItem(json,"weather");
    cJSON *Arrw=cJSON_GetArrayItem(weather,0);
    char *description=cJSON_GetObjectItem(Arrw,"description")->valuestring;
    
    printf("Condition: %s \n",(char*)description);
    
    //Line 6 (wind)
    cJSON *wind=cJSON_GetObjectItem(json,"wind");
    double speed=cJSON_GetObjectItem(wind,"speed")->valuedouble;
    int degrees=cJSON_GetObjectItem(wind,"deg")->valueint;
    
    cJSON *gust_extract=cJSON_GetObjectItem(wind,"gust");
    bool gust_bool=cJSON_IsFalse(gust_extract);

    if (gust_bool==true){
        double gust=cJSON_GetObjectItem(wind,"gust")->valuedouble;
        if (degrees>=345 || degrees<30){
            printf("Wind: %g km/h from the %s and a gust of %g \n",speed,"N",gust);
        }
        else if (degrees>=30 && degrees<75){
            printf("Wind: %g km/h from the %s and a gust of %g \n",speed,"NE",gust);
        }
        else if (degrees>=75 && degrees<120){
            printf("Wind: %g km/h from the %s and a gust of %g \n",speed,"E",gust);
        }
        else if (degrees>=120 && degrees<165){
            printf("Wind: %g km/h from the %s and a gust of %g \n",speed,"SE",gust);
        }
        else if (degrees>=165 && degrees<210){
            printf("Wind: %g km/h from the %s and a gust of %g \n",speed,"S",gust);
        }
        else if (degrees>=210 && degrees<255){
            printf("Wind: %g km/h from the %s and a gust of %g \n",speed,"SW",gust);
        }
        else if (degrees>=255 && degrees<300){
            printf("Wind: %g km/h from the %s and a gust of %g \n",speed,"W",gust);
            }
        else if (degrees>=300 && degrees<345){
            printf("Wind: %g km/h from the %s and a gust of %g \n",speed,"NW",gust);
        }
    }
    else{
        if (degrees>=345 || degrees<30){
            printf("Wind: %g km/h from the %s \n",speed,"N");
        }
        else if (degrees>=30 && degrees<75){
            printf("Wind: %g km/h from the %s \n",speed,"NE");
        }
        else if (degrees>=75 && degrees<120){
            printf("Wind: %g km/h from the %s \n",speed,"E");
        }
        else if (degrees>=120 && degrees<165){
            printf("Wind: %g km/h from the %s \n",speed,"SE");
        }
        else if (degrees>=165 && degrees<210){
            printf("Wind: %g km/h from the %s \n",speed,"S");
        }
        else if (degrees>=210 && degrees<255){
            printf("Wind: %g km/h from the %s \n",speed,"SW");
        }
        else if (degrees>=255 && degrees<300){
            printf("Wind: %g km/h from the %s \n",speed,"W");
            }
        else if (degrees>=300 && degrees<345){
            printf("Wind: %g km/h from the %s \n",speed,"NW");
        }
    }
}