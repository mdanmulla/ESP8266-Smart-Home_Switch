SMART HOME CONTROLLER — LOCAL + BLYNK VERSION | VERSION V1

Purpose: local dashboard over home Wi-Fi plus existing Blynk integration. 
Open the dashboard at http://<ESP8266-IP> from a phone/computer on the same Wi-Fi. Blynk requires internet access; local dashboard and physical switches continue to work without internet while connected to the same LAN.


CHANTE THIS BEFORE FLASHING

#define BLYNK_TEMPLATE_ID "YOUR_BLYNK_TEMPLATE_ID"  // <-- CHANGE THESE LINE BEFORE FLASHING
#define BLYNK_TEMPLATE_NAME "YOUR_BLYNK_TEMPLATE_NAME" //<--- CHANGE              
#define BLYNK_AUTH_TOKEN "YOUR_BLYNK_AUTH_TOKEN"  // <--- CHANGE

Before uploading, replace BLYNK_AUTH_TOKEN, TEMPLATE_ID AND TEMPLATE_NAME in SmartHomeController.ino with your token, id and name. Preserve your existing board and wiring. Existing relay/switch GPIO arrays and active-low relay logic are unchanged.

Compile using your ESP8266 Arduino Core 3.1.2 and installed project libraries. Full toolchain compilation and physical hardware testing have not been performed here.


------------Check the readme.md for help-----------------------

GitHub: https://github.com/mdanmulla/ESP8266-Smart-Home_Switch

