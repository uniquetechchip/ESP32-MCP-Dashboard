=========================================
SD Card Firmware Update Guide (Offline OTA)
Developer: Developer Munna Kumar
You can flash new firmware directly to this device using a Micro SD card, without needing to connect it to a computer.
Steps for Flashing:
1. Take your compiled binary (⁠.bin⁠) firmware file.
2. Rename the file to ⁠upgrade.bin⁠ (if you are rolling back to an older version, rename it to ⁠downgrade.bin⁠).
3. Copy this file directly to the root directory of your Micro SD card (do not put it inside any folders).
4. Insert the SD card into the ESP32's SD module and power ON or reboot the device.
Visual Feedback & Safety:
 Upon booting, the device will automatically detect the file and begin the flashing process.
 You will see "UPGRADING" (or "DOWNGRADING") displayed on the TFT screen along with a live progress bar.
 Once the flashing is 100% successful, the system will automatically rename the file on the SD card to ⁠upgrade.done⁠ (or ⁠downgrade.done⁠). This ensures the device does not get stuck in a continuous boot loop trying to flash the same file on the next restart.
 Finally, the device will automatically reboot with the newly installed firmware.
