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

# 2026-04-10 - PCB Construction & Software Development

## Project Overview
- **Breadboard Demo 2:** Bring-up and demonstration of the Week 3 PCB with a simplified control unit and sensor set, including debugging of USB enumeration, programming consistency, and FSR signal integrity issues.
- **Week 4 PCB Bring-Up:** Receipt and full population of the Week 4 revised PCB using SMD components, integrating new connectors and status LEDs to improve testability, and initiating firmware validation of the external IMU.

## Design Progress
Since we did not have any SMD components like capacitors and resistors, we were forced to use breadboard components for the second demo. This resulted in a good number of technical issues in our design process for our demo. 
To start we desoldered a chip from an ESP32 dev board. We then soldered that back onto the PCB. This was then followed by the buttons and usb data lines. We ran into three main issues, usb enumeration, programming consistency, fsr programming. 

### 2.2 Enable Pin Pull-Up — Design Flaw & Fix
 
On the Week 3 PCB, the ESP32-S3 EN (Enable) pin was left floating as there was no pull-up resistor was placed to the 3.3 V rail. This caused the EN pin to oscillate, preventing reliable USB enumeration and stable operation.
The schematic for the Week 3 PCB did not include a pull-up on EN. The ESP32-S3 datasheet and Espressif hardware design guidelines [1] specify that EN must be pulled high through a 10 kΩ resistor for reliable reset behavior.
In order to fix this for the demo a 10 kΩ resistor was soldered between the EN pin and the 3.3 V rail on the PCB surface. This stabilized the enable line and allowed consistent boot behavior.
 
**Design Change in Week 4 PCB:** A dedicated 10 kΩ pull-up resistor (R_EN) was added to the schematic and layout, connecting EN to 3.3 V. A 100 nF decoupling capacitor to GND was also added per Espressif guidelines.


 ### 2.3 Boot Pin Solder Joint
 Programming was inconsistent and the ESP32-S3 would sometimes fail to enter download mode when the BOOT button was pressed.
 Close visual inspection revealed a solder joint on the GPIO0 (BOOT) line connecting the tactile button to the PCB pad. A multimeter continuity test confirmed an intermittent open.
 The joint was resoldered and fixed. Post-fix continuity test confirmed a solid connection. Programming via USB CDC was confirmed reliable across 10 consecutive flash attempts.
 
### 2.4 FSR Signal Path : Unsoldered Pin
 
FSR ADC readings were non-responsive on the PCB, despite the breadboard circuit functioning on the hardware device readings as we could clearly see the voltage rise when pressure was applied.
When probing with a multimeter at the MCU pin it showed no voltage variation when the FSR was pressed. Tracing the net back revealed that the FSR signal pad on the PCB was completely unsoldered.
Once we soldered the PIN 6 which is the FSR pin on the ESP32 we immediately saw the software work 

### 2.5 AMS1117 3.3V Capacitor Useage

For the latest configuration I have decided to go with a 10 µF tantalum on output, 10 µF ceramic on input. The AMS1117 requires a minimum output ESR to maintain stability so a tandalum capacitor was necessary for this latest design. The output capacitor has about 1 Ω of ESR so we can fall into the necessary requirement for stability. The ESR has to be between 0.3 and 22. 

## 3. Test Setup & Experimental Methodology
 
### 3.1 Breadboard Demo 2 Configuration
 
The demo hardware consisted of the following:
 
- Week 3 PCB with desoldered and resoldered ESP32-S3 module, direct USB D+/D- lines routed to PCB header.
- External breadboard: FSR and push button connected to MCU GPIO pins.
- USB-A cable connecting PCB to host PC running Windows 11.
 
### 3.2 Test Procedure : USB Enumeration
 
1. Connected PCB to host PC via USB-A.
2. Opened Device Manager to observe device appearance on the USB bus.
3. Observed device appearing as "COM Port" or "HID Mouse."
7. Re-plugged USB; device enumerated successfully as ESP32-S3 CDC device, then as HID Mouse following firmware load.
 
### 3.3 Software Test Routine
 
A dedicated software test case was written to cycle through the following HID actions automatically upon boot:
 
- **Left button click**  HID report: `buttons = 0x01`
- **Right button click**  HID report: `buttons = 0x02`
- **Circle drawing routine**  360-step sinusoidal X/Y movement at 1° increments
 
This test verified USB HID report delivery and host-side cursor control simultaneously, without requiring manual hardware interaction during the demo.
 
---
 
## 4. Debugging Log
 
| Issue # | Description | Status |
|---|---|---|
| 1 | EN pin floating, USB not enumerating | ✅ Resolved |
| 2 | BOOT pin incorrect solder joint, inconsistent download mode | ✅ Resolved |
| 3 | FSR ADC unresponsive, unsoldered signal pad | ✅ Resolved |
 
### 4.1 EN Pin Oscillation
 
| Field | Detail |
|---|---|
| Symptom | Device not appearing consistently on USB bus; Device Manager showed repeated disconnect/reconnect events |
| Test Equipment | multimeter on EN pin to GND |
| Measurement | EN toggling between 0 V and ~0.8 V at irregular intervals; no stable HIGH |
| Hypothesis | Floating EN pin driven by internal leakage/noise,no pull-up to maintain HIGH state |
| Fix | 10 kΩ resistor soldered from EN to 3.3 V rail |
| Verification | EN stable at 3.3 V. USB enumeration successful on 5 consecutive plug/unplug cycles |
 
### 4.2 Boot Pin Inconsistency
 
| Field | Detail |
|---|---|
| Symptom | flash failed with "Failed to connect to ESP32-S3: No serial data received" majority of attempts |
| Test | Continuity check on BOOT button PCB pad,open confirmed |
| Fix | Resoldered |
| Verification | 10 consecutive flash operations completed without failure |
 
### 4.3 FSR ADC Unresponsive
 
| Field | Detail |
|---|---|
| Symptom | Serial monitor showed ADC channel reading 0 regardless of FSR compression |
| Test 1 | Swapped PCB connection for breadboard, ADC readings correct so the fault isolated to PCB |
| Test 2 | Probed ESP32-S3 ADC pin with multimeter while compressing FSR reading 0 V (expected 1–3 V) |
| Test 3 | Probed FSR signal pad on PCB,correct voltage present. Continuity from pad to MCU pin: OPEN |
| Fix | Pin soldered to pad |
| Verification | ADC readings verified across full compression range |
 
---
 
## 5. Week 4 PCB — Population & Bring-Up
 
### 5.1 Changes from Week 3 to Week 4
 
- All bypass capacitors (100 nF, 10 µF) placed as SMD 0805 components directly adjacent to power pins, eliminates long traces from breadboard wiring.
- EN pull-up resistor (10 kΩ, R_EN) added to schematic and placed on PCB.
- Status LEDs added: green (3.3 V power good) and red (5V power good), each with 330 Ω and 150 Ω current-limiting resistors.
- Tandalum Capacitors for the LDO 3.3V output
- Board area reduced by approximately 40% versus Week 3 design.

  
---
 <img width="481" height="640" alt="image" src="https://github.com/user-attachments/assets/798f3d66-1f3e-4fd9-92b0-fedad0f2376d" />

### 5.2 IMU Integration Status
  
Firmware for SPI initialization and register readout is written and under test. Configuration follows the  datasheet register map [2]:
  
### Remaining Work Before Final PCB
 
- Complete IMU SPI driver verification: reliable accelerometer and gyroscope reads
- Validate MAX3421E USB host bring-up (SPI init sequence, USB connect interrupt). Requires oven reflow using solder paste and PCB stencil — **stencils on order.**
- Finalize  optical sensor SPI integration: delta X/Y accumulation and configuration via SPI write.
 
---
 
## References
 
[1] https://www.analog.com/media/en/technical-documentation/data-sheets/MAX3421E.pdf \
[2] KiCad EDA Documentation. PCB Layout and Schematic Capture. https://docs.kicad.org \
[3] https://docs.espressif.com/projects/esp-dev-kits/en/latest/esp32s3/esp32-s3-devkitm-1/index.html \
[4] https://www.alldatasheet.com/datasheet-pdf/pdf/205691/ADMOS/AMS1117-3.3.html




