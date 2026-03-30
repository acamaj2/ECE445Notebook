# Alex Camaj Worklog

[[_TOC_]]

# 2026-02-17 - Worklog Entry

Joined Any-Surface-Stylus for Computer (team 86) late due to my original project getting unapproved last week. For previous project had data architecture designed as well as specific requirements for each sub-part determined. Talked with machine shop about Polycase for getting enclosurement for the main frame of the design. I read and understood the current proposal for the current team as well as added requirements for the project components. 

# 2026-02-23 - Determining Parts

Over the past few days, I have been looking into determining which sensor would be best for our main way of tracking movement with the stylus pen. Through research, I determined there was 2 main pathways. One was to use an optical flow sensor to see x and y movement. The other was to use a 3axis IMU to determine the relative change in position of the pen and then update the cursor based on wrist rotation over movement. The problem I found with the optical flow sensor was that there was a lot of issues with the range. The minimum range flow sensor I could find was 80mm which means the sensor has to be at least 80mm off the ground in order to work. This poses an issue with the pen since it will make us need to have a bulky design and have the pen have a sensor pop out the side without being obscured. The other option which I tested with was an IMU. I used a microcontroller board with an LSM6DSL IMU built in and I was able to use that to write an algorithm to move the cursor on my computer through the pitch and roll movement of the microcontroller. This shows us that an IMU is very practical for our application. The remaining problem was what sensor to use when the board is on the table since the IMU will be less effective. I believe going with a standard mouse optical sensor would do the job, and based on whether the pen is down or up we can decide which sensor controls the cursor movement. The LSM6DSL has 6 DOF, which is the perfect amount for this stylus. A standard PMW3360 would be fit for the on-surface tracking. <img width="750" height="500" alt="image" src="https://github.com/user-attachments/assets/3adf30f3-8ba0-4dfc-a8b9-dd5721e67255" />

# 2026-03-06 - Final Design Updates

Over the past week, we have finalized parts and are moving into the implementation phase. We will be implementing an optical sensor with a fiber optic cable in order to get the lighting to reach the lens. This will be used for the ground surface tracking, and the IMU will control air movement. Right now, we need to update our requirements and verification since a lot of them are very strict and time-consuming. The design document was the main focus of last week, and we were able to put all of our research into that. We begin testing this weekend with our optical sensor and HID interface to setup basic mouse functionality for our design. 

# 2026-03-30 - PCB Finalization

## PCB Design:
In developing the PCB for our final design, I was in charge of the control unit, which handled most of the sensor processing by housing the ESP32-S3 MCU. For the final design, we had to decide what was needed for each PCB, the control unit, and the actual pen PCB. Since we want the pen shell to be the smallest and lightest it could be to resemble a real pen, we decided to only keep sensors in there. We went with the optical sensor, right click, scroll wheel, and IMU. The control unit PCB was then left with the max3421e chip, a USB host IC that can communicate optical sensor data to the ESP32s3. The control unit also has an ESP32-S3 and connectors for communication with the host device (PC).

## Connectors Justifications:
We went with three designs: a USB-C, a mini USB, and a pin-header output. These are all functional designs, but each has its own strengths and weaknesses. For the USB-C, we have a more complex PCB design because there are 14 pins to configure. This also makes soldering harder. The mini USB also has a connector that is easier to plug in and program, but the soldering was slightly more difficult. The pin header is the simplest to set up, but it requires wires soldered to it at all times, which isn't very clean. 
These are the differences in our schematic designs. 
## USBC-C
<img width="325" height="285" alt="image" src="https://github.com/user-attachments/assets/523c5a0a-563a-4901-b0b2-5332597bb4c6" />

## Pin Header
<img width="325" height="285" alt="image" src="https://github.com/user-attachments/assets/9b8c83e7-5755-4be3-bad3-1cbe9951adbe" />

## Final design: 
<img width="2121" height="1216" alt="image" src="https://github.com/user-attachments/assets/5935a35e-dcb3-4dbf-8cf8-fc47bf8a8384" />

## Final schematic:
<img width="3507" height="2480" alt="image" src="https://github.com/user-attachments/assets/f4776837-1bb8-4e82-8346-c332a8ca9b68" />

## Alternative USB Interfacing (MAX3421E Chip Discussions):
We found a minor configuration issue with the optical sensor due to its pre-integration. The optical sensor we are using already acts as an HID device outputting mouse x and y values. The issue we have is that we want to control the IMU and optical sensor for data input, and then use the ESP32-S3's USB lines to output HID data to a host computer. This means we cannot directly connect the optical sensor to the ESP32 because the wires are already in use. The two options we were left with are either to use a new chip that supports multiple USB lines, like the STM32F4 series, or to have a USB host controller IC intercept the optical sensor data and communicate it to the ESP32 over SPI, which is available on the MCU. We decided the ESP32 has great features, and the MAX3421E chip would be best for the job since it is widely used and documented for being a host controller. For this design, we are using a USB-powered bus, so the corresponding circuit setup will be as described in the data sheet. 
## Schematic:
<img width="710" height="509" alt="image" src="https://github.com/user-attachments/assets/33ebb4cf-3c49-401f-84ca-751c62d51e9d" />

## Work Session Summary:
We had meetings to discuss these design changes and to resolve issues with the PCB design on both parts. We had an original issue with the ESP32 design: we were using GPIOs 47 and 48 for the scroll wheel encoder values. After further investigation, we found that these were unreliable for our design and only supported 1.8 V, so we went with GPIOs 15 & 16 for the encoder values. For SPI, we used GPIOs 10-13 since they were relatively close together and allowed easy wiring to the max3421e chip. We experimented with the GPIOs on the devboard and the breadboard using LEDs to verify that they function properly. SPI over lines 10-13 was confirmed by testing the communication between 2 ESP32s. 

## Breadboard Setup
<img width="481" height="640" alt="image" src="https://github.com/user-attachments/assets/0748bc06-e1c5-4b36-b888-4c1495a3bbb8" />

## SPI COM Port Output
<img width="582" height="288" alt="spi_master_working" src="https://github.com/user-attachments/assets/80c5523d-9363-4ce7-b1ad-17ab59a5676e" />

## SPI Master Code
```C
 * SPI master (SPI2) on fixed GPIOs:
 *   GPIO10  MOSI
 *   GPIO11  MISO
 *   GPIO12  CS  (SS)
 *   GPIO13  SCLK
 */
#include <string.h>
#include "freertos/FreeRTOS.h"
#include "freertos/task.h"
#include "driver/spi_master.h"
#include "esp_err.h"
#include "esp_log.h"

#define SPI_HOST_ID     SPI2_HOST
#define PIN_MOSI        10
#define PIN_MISO        11
#define PIN_CS          12
#define PIN_SCLK        13

#define SPI_CLOCK_HZ    (1 * 1000 * 1000)
#define POLL_MS         500

static const char *TAG = "spi_master";

void app_main(void)
{
    spi_bus_config_t buscfg = {
        .mosi_io_num = PIN_MOSI,
        .miso_io_num = PIN_MISO,
        .sclk_io_num = PIN_SCLK,
        .quadwp_io_num = -1,
        .quadhd_io_num = -1,
        .max_transfer_sz = 32,
    };
    ESP_ERROR_CHECK(spi_bus_initialize(SPI_HOST_ID, &buscfg, SPI_DMA_CH_AUTO));

    spi_device_interface_config_t devcfg = {
        .clock_speed_hz = SPI_CLOCK_HZ,
        .mode = 0,
        .spics_io_num = PIN_CS,
        .queue_size = 4,
    };
    spi_device_handle_t spi = NULL;
    ESP_ERROR_CHECK(spi_bus_add_device(SPI_HOST_ID, &devcfg, &spi));

    ESP_LOGI(TAG, "SPI master ready: MOSI=%d MISO=%d CS=%d SCLK=%d @ %d Hz mode0",
             PIN_MOSI, PIN_MISO, PIN_CS, PIN_SCLK, SPI_CLOCK_HZ);

    uint8_t tx = 0;
    uint8_t rx = 0;
    spi_transaction_t t;
    memset(&t, 0, sizeof(t));

    while (1) {
        tx++;
        t.length = 8;
        t.tx_buffer = &tx;
        t.rx_buffer = &rx;
        esp_err_t err = spi_device_transmit(spi, &t);
        if (err != ESP_OK) {
            ESP_LOGE(TAG, "transfer failed: %s", esp_err_to_name(err));
        } else {
            ESP_LOGI(TAG, "tx=0x%02x rx=0x%02x", tx, rx);
        }
        vTaskDelay(pdMS_TO_TICKS(POLL_MS));
    }
}
```
Citations: \
[1] https://www.analog.com/media/en/technical-documentation/data-sheets/MAX3421E.pdf \
[2] https://documentation.espressif.com/esp32-s3_datasheet_en.pdf \
[3] USB 2.0 Specification. Universal Serial Bus Specification Revision 2.0. USB-IF, 2000.\
[4] KiCad EDA Documentation. PCB Layout and Schematic Capture. https://docs.kicad.org \
[5] https://docs.espressif.com/projects/esp-dev-kits/en/latest/esp32s3/esp32-s3-devkitc-1/index.html

