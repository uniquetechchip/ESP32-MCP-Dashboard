============================================================
ESP32 MCP DASHBOARD: COMPLETE SYSTEM DOCUMENTATION
Developer: Munna Kumar
============================================================

1. FIRST-TIME SETUP & WI-FI CONFIGURATION
------------------------------------------------------------
* Captive Portal (Auto-Popup): When powered on for the first time, the device has no saved network credentials. It automatically creates its own Wi-Fi Access Point (Hotspot) named "MCP-ESP32 Setup". On most smartphones and laptops, connecting to this network will automatically trigger a pop-up screen taking you directly to the setup page.
* Manual Web Page Setup: If the setup page does not open automatically, simply remain connected to the hotspot, open any web browser, and type 192.168.4.1 in the address bar.
* Saving Data: On the setup interface, enter your local Wi-Fi SSID, password, and the MCP server URL (e.g., ws://ip:port). Click 'Save', and the ESP32 will reboot, connect to your router, and go online.

2. MULTI-FUNCTION BUTTON OPERATIONS
------------------------------------------------------------
![ESP32 Dashboard MQ Gas Sensors Calibrating process.](images/IMG_4632.JPG)

* Single Tap: Instantly logs the current environmental data (temperature, humidity, and all gas sensor readings) to the Micro SD card. You can later insert this SD card into a PC to view the timestamped .txt log files.
* Double Tap: Toggles the TFT display between standard Color mode and a high-contrast Black & White (B&W) mode. Double-tap again to revert.
* 3-Second Press (While Device is On): Triggers the non-blocking "Calibration" process for all MQ gas sensors. This averages the current clean-air readings to set a new baseline for accurate future measurements.
* 3-Second Press (During Boot): Holding the button down while powering on the device initiates a Factory Reset. This completely erases the saved Wi-Fi credentials and MCP settings from memory, forcing the device back into Captive Portal mode.

3. SENSORS, DISPLAY & SD CARD FUNCTIONALITY
------------------------------------------------------------
* Display Visuals: The 1.8-inch TFT display communicates with the ESP32 via the SPI (Serial Peripheral Interface) bus. The system uses a graphics library to render real-time pixel data, updating text, icons, and dynamic status alerts (Good, Warning, Bad) based on sensor thresholds.
* DHT22 (Temperature & Humidity): This sensor uses a single digital pin. It measures the environment using an internal capacitive humidity sensor and a thermistor, transmitting the processed data to the ESP32 as a digital signal.
* MQ Gas Sensors (MQ-135, 6, 3, 9, 8): Each MQ sensor contains a tiny internal heater. When exposed to target gases (LPG, CO, Alcohol, Smoke, H2), the sensor's internal electrical resistance changes. This creates a fluctuating analog voltage that the ESP32 reads and maps to a value between 0 and 4095.
* Micro SD Card Module: Sharing the SPI bus with the display, the SD module handles file storage. When a single-tap log is triggered, the ESP32 syncs with an internet NTP server to get the exact real-world time, creates a new file named with that timestamp, and writes the sensor data to it.

4. FIRMWARE FLASHING VIA MICRO SD CARD (OFFLINE OTA)
------------------------------------------------------------

![ESP32 Dashboard Offline OTA Firmware Upgrading process via SD Card.](images/IMG_4635.jpg)

* How it Works: The system supports offline firmware updates directly from the Micro SD card without needing to connect the ESP32 to a computer.
* The Process: 
  1. Compile your updated Arduino code and export the compiled Binary (.bin) file.
  2. Rename the file to upgrade.bin (or downgrade.bin if rolling back) and copy it to the root of the Micro SD card.
  3. Insert the SD card into the module and power on/reboot the ESP32.
* Visual Feedback & Safety: On boot, the ESP32 will detect the file and automatically begin the flashing process. The TFT display will show "UPGRADING" (or "DOWNGRADING") along with a live progress bar.
* Auto-Rename: Once the flash is 100% successful, the system automatically renames the file to upgrade.done (or downgrade.done). This ensures the device doesn't get stuck in a boot loop by trying to flash the same file again on the next restart. The ESP32 will then automatically reboot with the new software.

5. MCP SERVER, ROBOT CONTROL & LED INDICATORS
------------------------------------------------------------
* MCP Server Connectivity: The ESP32 maintains a persistent WebSocket connection to your MCP server over Wi-Fi. This allows any AI agent, external robot, or remote software connected to that same server to read your sensor data globally in real time.
* RGB LED (Robot Command): If the connected AI or robot sends a JSON command (e.g., {"color":"RED"}), the ESP32 intercepts it and instantly changes the RGB LED color. This serves as a physical indicator that the external AI has received the data and executed an action.
* Flashing Blink LED: A standard LED on the board toggles on and off every 500 milliseconds. This acts as a system "heartbeat," visually confirming that the FreeRTOS main loop is running smoothly and hasn't crashed or frozen.

![Real-time ESP32 MCP Dashboard showing live sensor data like temperature, humidity, and gas levels.](images/IMG_4631.JPG)




