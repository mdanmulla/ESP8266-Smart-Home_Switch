# ESP8266-Smart-Home_Switch
This program is for esp8266 that uses 6 relay to control the home appliance via mobile phone a small project by mdanmulla
# Smart Home Controller v1 — Complete Setup & User Manual

**Platform:** ESP8266 NodeMCU (ESP-12E / ESP-12F)  
**Interface:** Local web dashboard over HTTP  
**Cloud integration:** Blynk IoT  
**Firmware target:** ESP8266 Arduino Core 3.1.2

This manual explains how to install the USB driver, install Arduino IDE and the ESP8266 board package, select the correct board and COM port, upload the firmware, connect the controller to Wi-Fi, configure Blynk, and use the dashboard.

> **Important — hardware is already wired:** Keep the existing GPIO assignments exactly as shown in this guide. Do not change pins or move soldered wires as part of setup. This guide describes the ESP8266 version, not an ESP32 port.

---

## 1. What the controller does

The ESP8266 runs a local web server and provides a browser-based smart-home dashboard. The project includes:

- Six logical device cards in the dashboard.
- Five physical relay outputs in this hardware configuration.
- Three physical wall-switch inputs.
- Device names and categories.
- Dashboard switch enable/disable controls.
- Momentary and toggle operation modes.
- Per-device schedules and a global schedule fallback.
- Auto-off timers.
- Optional confirmation before turning a device OFF.
- Dashboard users and per-device switch permissions.
- Wi-Fi setup, access-point fallback and network settings.
- Status information, system uptime and event log.
- Power/energy-related settings and usage estimates based on configured wattage.
- Weather display using Open-Meteo when a location and internet access are available.
- Blynk Cloud synchronization and a manual Connect/Disconnect control.
- OTA firmware update support after the device has joined the network.

The local dashboard and physical switches can operate on the local network without Blynk. Blynk remote control requires working internet access.

## 2. Hardware connections

### 2.1 Relay outputs

| Dashboard device | NodeMCU board pin | ESP8266 GPIO | Blynk virtual pin |
|---|---|---:|---|
| Device 1 / Relay 1 | D1 | GPIO5 | V1 |
| Device 2 / Relay 2 | D2 | GPIO4 | V2 |
| Device 3 / Relay 3 | D5 | GPIO14 | V3 |
| Device 4 / Relay 4 | D6 | GPIO12 | V4 |
| Device 5 / Relay 5 | D0 | GPIO16 | V5 |
| Device 6 | No relay connected | None (`255`) | V6 |

Device 6 is a software/Blynk channel in this hardware build. It does not operate a physical relay.

Connect the relay board's low-voltage input pins:

| Relay board terminal | Connect to NodeMCU |
|---|---|
| IN1 | D1 / GPIO5 |
| IN2 | D2 / GPIO4 |
| IN3 | D5 / GPIO14 |
| IN4 | D6 / GPIO12 |
| IN5 | D0 / GPIO16 |
| IN6 | Leave unconnected |
| Logic GND | GND, according to relay board requirements |
| Logic VCC | Correct regulated supply for the relay board; check its label |

**Relay power:** Do not power relay coils from the NodeMCU 3.3 V pin. Use a suitable regulated supply with enough current for the relay board. Non-isolated relay boards normally need a common ground reference with the NodeMCU. Opto-isolated boards may require different input/coil supply wiring; follow the relay board manufacturer's diagram.

### 2.2 Physical wall-switch inputs

The firmware uses three switch inputs. Each simple dry-contact wall switch connects its GPIO input to NodeMCU GND when closed.

| Physical switch | NodeMCU pin | ESP8266 GPIO | Controls |
|---|---|---:|---|
| Switch 1 | D7 | GPIO13 | Device 2 / V2 |
| Switch 2 | RX | GPIO3 | Device 3 / V3 |
| Switch 3 | TX | GPIO1 | Device 4 / V4 |

The firmware uses internal pull-ups. One side of each switch goes to the listed GPIO; the other side goes to GND. Do not apply 5 V to a GPIO.

**RX/TX caution:** RX (GPIO3) and TX (GPIO1) are also used for USB serial upload and Serial Monitor. If uploading or serial communication fails, turn the controller off, temporarily disconnect the RX and TX switch wires, upload the firmware, then power off and reconnect them.

### 2.3 Simplified low-voltage wiring diagram

```text
                    ESP8266 NodeMCU
                 +----------------------+
 USB 5V / VIN --->| VIN              GND |---------+------ Relay board GND*
                 |                      |         |
 Relay IN1 <-----| D1 / GPIO5            |         +------ Switch common side
 Relay IN2 <-----| D2 / GPIO4            |                 |      |      |
 Relay IN3 <-----| D5 / GPIO14           |                SW1    SW2    SW3
 Relay IN4 <-----| D6 / GPIO12           |                 |      |      |
 Relay IN5 <-----| D0 / GPIO16           |                 |      |      |
                 |                      |                D7     RX     TX
                 +----------------------+              GPIO13 GPIO3  GPIO1
                                                        |      |      |
                                                        +------+------+
                                                       each switch closes
                                                        its GPIO to GND

       Separate regulated relay-board supply (as required):
       Supply +V  ----------------------------> Relay board VCC / JD-VCC**
       Supply GND ----------------------------> Relay board GND*

       Typical relay contact connection for a normally-OFF load:
       Supply LIVE ---- fuse ---- COM [Relay] NO ---- Load LIVE
       Supply NEUTRAL ------------------------------ Load NEUTRAL
       Protective EARTH ---------------------------- Load EARTH
```

`*` Common-ground wiring is typical for a non-isolated relay board. Follow the exact relay board documentation.

`**` Use the voltage specified on the relay module (often 5 V). Never connect relay coil supply voltage to a NodeMCU GPIO.

**Mains safety:** Relay contacts may carry dangerous mains voltage. Disconnect all power before wiring. Use correctly rated relays, fuses, enclosures, strain relief and insulated terminals. Have mains wiring done or checked by a qualified electrician. Never connect mains voltage to the NodeMCU, GPIO pins, USB connector or low-voltage side of the relay module.

### 2.4 Normally-OFF versus normally-ON load wiring

A relay's trigger logic and its contact selection are two different things:

- **COM + NO (Normally Open):** When the relay coil is not energized, the circuit is open. This is the usual choice for a load that should be OFF by default.
- **COM + NC (Normally Closed):** When the relay coil is not energized, the circuit is closed. The load may be ON while the relay is idle.

Changing COM/NO to COM/NC changes the load's default contact behavior. It does not change the firmware's active-LOW/active-HIGH trigger setting. Only use NC if that behavior is appropriate and safe for the appliance.

---

## 3. Install Arduino IDE

1. Download and install Arduino IDE from the official Arduino website: <https://www.arduino.cc/en/software>.
2. Launch Arduino IDE once installation is complete.
3. If Windows asks for permission to install device drivers, allow the installer to proceed.

Arduino IDE 2.x is recommended. The menu names below refer to Arduino IDE 2.x; in Arduino IDE 1.8.x the same options are generally under File, Tools and Sketch.

## 4. Install the NodeMCU USB-to-serial driver

The NodeMCU USB connector communicates with the computer through a USB-to-serial chip. The driver depends on the chip fitted to your particular board.

### 4.1 Identify the USB chip

Look near the NodeMCU USB connector for a small chip marked with one of these names:

- **CH340 / CH341:** Install the WCH CH341SER driver.
- **CP2102 / CP210x:** Install the Silicon Labs CP210x VCP driver.
- **FT232 / FTDI:** Install the FTDI VCP driver.

Do not install random driver packages from third-party download sites. Use the manufacturer's driver page:

- WCH CH340/CH341: <https://www.wch-ic.com/downloads/CH341SER_EXE.html>
- Silicon Labs CP210x: <https://www.silabs.com/developers/usb-to-uart-bridge-vcp-drivers>
- FTDI VCP: <https://ftdichip.com/drivers/vcp-drivers/>

### 4.2 Windows driver installation and COM port check

1. Disconnect the NodeMCU from USB.
2. Install the driver matching the USB chip.
3. Restart Windows if the driver installer requests it.
4. Connect the NodeMCU using a **USB data cable**. Some USB cables provide power only and cannot upload firmware.
5. Open **Device Manager**.
6. Expand **Ports (COM & LPT)**.
7. Look for an entry such as `USB-SERIAL CH340 (COMx)` or `Silicon Labs CP210x USB to UART Bridge (COMx)`.
8. Note the COM number. You will select this in Arduino IDE under **Tools → Port**.

If no COM port appears, try another USB data cable and USB socket, check Device Manager for an unknown device, and install the correct driver. Avoid selecting a COM port while the board is disconnected.

### 4.3 macOS and Linux

- **macOS:** Install the manufacturer driver only if your macOS version and USB chip require it. Reconnect the board and check Arduino IDE's Port menu.
- **Linux:** Many distributions include CH340 and CP210x serial drivers in the kernel. Reconnect the board and check the available serial device (commonly `/dev/ttyUSB0` or `/dev/ttyACM0`). Your user may need permission to access the serial port (often membership in the `dialout` group); log out and back in after changing group membership.

---

## 5. Install the ESP8266 board package

1. Open Arduino IDE.
2. Open **File → Preferences** (on macOS, Arduino IDE → Settings/Preferences).
3. Find **Additional Boards Manager URLs**.
4. Add this ESP8266 package URL:

   `https://arduino.esp8266.com/stable/package_esp8266com_index.json`

5. If another URL is already present, separate the URLs with a comma or use the URL-list button rather than deleting the existing entries.
6. Save Preferences.
7. Open **Tools → Board → Boards Manager**.
8. Search for `esp8266`.
9. Install **esp8266 by ESP8266 Community**.
10. For this project, use ESP8266 Arduino Core **3.1.2** where possible. If Boards Manager offers a version selector, select 3.1.2.

The board package supplies the ESP8266 compiler, upload tools and core libraries such as Wi-Fi, web server, EEPROM, DNS, mDNS and OTA support.

## 6. Install the Blynk library

1. In Arduino IDE, open **Sketch → Include Library → Manage Libraries**.
2. Search for `Blynk`.
3. Install the official **Blynk** library published for Arduino.
4. Restart Arduino IDE if it does not detect the library immediately.

Do not install several unrelated Blynk libraries with similar names. The project uses the standard Blynk IoT Arduino library.

## 7. Open the project and select the board and port

1. Extract the project ZIP to a normal folder on your computer. Do not try to compile the sketch while it is still inside the ZIP.
2. Open the `SmartHomeController` folder.
3. Open `SmartHomeController.ino` in Arduino IDE.
4. Keep `webui.h` in the same folder as the `.ino` file. Do not rename it, move it to another folder, or open it as a separate sketch.
5. In Arduino IDE, select **Tools → Board → ESP8266 Boards → NodeMCU 1.0 (ESP-12E Module)**.
6. Connect the NodeMCU by USB.
7. Select **Tools → Port → COMx** (Windows) or the matching `/dev/tty...` port (macOS/Linux).
8. If the exact NodeMCU board profile is not listed, confirm that the ESP8266 board package installed successfully.

Use the normal NodeMCU board defaults initially. Do not change the GPIO definitions or select an ESP32 board profile.

## 8. Configure Blynk before uploading

Blynk is optional for local dashboard operation. If you want cloud control, configure Blynk and add the device token before compiling and uploading.

### 8.1 Create or open the Blynk template

1. Open Blynk Console: <https://blynk.cloud/>.
2. Sign in or create a Blynk account.
3. Open **Developer Zone → Templates** (the menu wording may vary with Blynk's interface).
4. The firmware is configured with:
   - Template ID: `TMPL69FjLdUIm`
   - Template name: `Home Control Switch`
5. If this template is already available in your Blynk account, open it and check its datastreams.
6. If you create a new template instead, update the Template ID and Template name in the firmware so they exactly match the new template.

### 8.2 Create the six datastreams

In the template, create six **Virtual Pin** datastreams:

| Datastream | Type | Minimum | Maximum | Meaning |
|---|---|---:|---:|---|
| V1 | Integer | 0 | 1 | Device 1 |
| V2 | Integer | 0 | 1 | Device 2 |
| V3 | Integer | 0 | 1 | Device 3 |
| V4 | Integer | 0 | 1 | Device 4 |
| V5 | Integer | 0 | 1 | Device 5 |
| V6 | Integer | 0 | 1 | Device 6 (software channel) |

Use `0` for OFF and `1` for ON.

### 8.3 Create a device and copy its Auth Token

1. Create a device from the `Home Control Switch` template.
2. Open the new device in Blynk Console.
3. Find its **Device Info** or **Auth Token** section.
4. Copy the device's Auth Token.
5. Keep the token private. Anyone who gets it may be able to connect to or control the device.

### 8.4 Add the token to the firmware

In the sketch, find the Blynk Auth Token setting near the top. Replace the placeholder `PASTE_YOUR_BLYNK_AUTH_TOKEN_HERE` with the token copied from your Blynk device. Keep the quotation marks around the token.

If you use a different Blynk template, make sure the Template ID and Template name in the sketch match that template exactly.

### 8.5 Build the Blynk dashboard

In the Blynk template's Web Dashboard or Mobile Dashboard:

1. Add six Switch widgets.
2. Assign them to V1, V2, V3, V4, V5 and V6 respectively.
3. Configure each switch to use integer values `0` and `1`.
4. Save the dashboard/template changes.

The firmware reports device states to Blynk when connected. V1–V5 correspond to physical relay outputs; V6 is a software-only channel in this wiring.

---

## 9. Compile and upload the firmware

1. Make sure the correct NodeMCU board and COM port are selected.
2. Confirm the Blynk token has been entered if you want Blynk.
3. Click **Verify** (the checkmark) to compile.
4. If compilation succeeds, click **Upload** (the right arrow).
5. Wait for the upload process to finish. The board will restart.
6. If upload fails, see the troubleshooting section below.

**RX/TX switch warning during upload:** Because physical switches use RX and TX, they can interfere with serial programming. Turn off power, temporarily disconnect the switch wires from RX and TX, upload the firmware, then turn power off before reconnecting them.

### Upload troubleshooting

- Check that the USB cable supports data, not just charging.
- Check the correct COM port in Device Manager and Arduino IDE.
- Close Serial Monitor and other programs using the COM port.
- Install the correct CH340, CP210x or FTDI driver.
- Try another USB cable or USB port.
- If the RX/TX switches are wired, disconnect those two switch leads with power off while uploading.
- Confirm the selected board is **NodeMCU 1.0 (ESP-12E Module)** and the ESP8266 board package is installed.

---

## 10. First Wi-Fi setup and opening the dashboard

### 10.1 Connect to the controller's setup hotspot

1. Power the NodeMCU and relay board from their correct supplies.
2. If the controller has not joined a router, it provides a setup access point.
3. On your phone or computer, open Wi-Fi settings.
4. Connect to:
   - **Wi-Fi name:** `SmartHome_AP`
   - **Default password:** `12345678`
5. Open a browser and visit: `http://192.168.4.1`

Use `http://`, not `https://`. This firmware serves the local dashboard over HTTP.

### 10.2 Connect the controller to your home router

1. Open the Wi-Fi connection/setup section in the dashboard.
2. Scan for nearby networks or enter your router's Wi-Fi name (SSID).
3. Enter the Wi-Fi password.
4. Save/connect.
5. Use a 2.4 GHz Wi-Fi network. ESP8266 does not support a 5 GHz-only network.
6. After the controller joins your router, connect your phone/computer to the same home network.
7. Find the controller's IP address from the dashboard Status/Network section or your router's connected-device/DHCP list.
8. Open `http://<controller-ip>` in your browser, replacing `<controller-ip>` with the actual address.
9. If your network supports mDNS, `http://smart-home.local` may also work; otherwise use the IP address.

If the controller cannot connect to the router, reconnect to `SmartHome_AP` and correct the Wi-Fi settings.

### 10.3 Dashboard login

The firmware's administrator username is `anmulla`. Complete the dashboard's initial password setup/login process and keep the password safe. Do not assume that there is a universal default administrator password.

---

## 11. How to use the dashboard

The dashboard is designed to be opened in a modern browser on a phone, tablet or computer connected to the same local network as the controller.

### 11.1 Dashboard page

- Each device is shown as a card with its configured name and category.
- Tap/click a device card or its switch control to change the device state.
- The card's active appearance indicates an ON state; an inactive card indicates OFF.
- Use the device name/category settings to make the cards easy to identify.
- The six cards represent six logical channels. Remember that Device 6 has no physical relay output in this build.

### 11.2 Status page

Use the Status page to inspect the controller's current condition. Depending on network availability and firmware state, it includes:

- Relay/device states.
- Wi-Fi/network information such as connection state and IP details.
- Signal strength and system uptime.
- System information such as free heap.
- Hotspot/AP status and connected clients when applicable.
- Blynk connection state and the Blynk control button.
- Recent events recorded during the current boot.

**Blynk Connect/Disconnect button:**

1. Open **Status**.
2. Find **Blynk control**.
3. Press **Connect to Blynk** to request a cloud connection.
4. Wait for the status to update.
5. Press **Disconnect Blynk** to intentionally disconnect and stop automatic reconnection.
6. Press Connect again when you want to reconnect.

Automatic reconnection is enabled again after a controller reboot. The controller needs router internet access for Blynk; a Wi-Fi connection to a router without internet is not enough.

### 11.3 Settings page

The Settings page contains configuration groups. Available controls include:

- **Wi-Fi connection:** Scan for nearby networks and configure the router SSID/password.
- **Network / IP:** Choose DHCP or configure a fixed IP, gateway, subnet mask and DNS values where supported.
- **Hotspot (Auto-AP):** Change the setup hotspot name, password and related AP settings.
- **Device Name / Category:** Rename device cards and assign categories such as lighting, security, motor, server, entertainment, cooling, heating, water pump or other.
- **Enable this switch / Dashboard Switch:** Enable or disable a device's dashboard switch.
- **Physical Switches:** Enable or disable physical wall-switch input handling.
- **Momentary / Toggle:** Choose whether a device behaves as a momentary output or a normal latched ON/OFF output, where the mode is available.
- **Confirm before turning OFF:** Ask for confirmation before switching a device off, if enabled.
- **Auto-Off Timer:** Set a duration after which a device is automatically turned off.
- **Schedule:** Set per-device ON and OFF times.
- **Global Schedule (Fallback):** Configure a common schedule to use as a fallback for devices without an individual schedule.
- **Time & Timezone:** Use automatic network time synchronization or set time manually; select the correct timezone. Nepal uses UTC+5:45.
- **Weather Location:** Enter the location information needed for the weather display. Weather requires internet access.
- **Power Config:** Enter the known power rating in watts and current in amperes for energy-related estimates. These are estimates based on configured values, not readings from a physical energy meter.
- **Dashboard Access / Dashboard users:** Manage users and login requirements where available.
- **Switch access by user:** Configure which device switches an allowed user can control.
- **Automatic Restart:** Enable/disable a scheduled restart and set the restart interval in days. The interval is the number of days between scheduled automatic restarts; it is not a relay timer.
- **Factory Reset All Settings:** Restore the controller's saved configuration to its defaults. Use this carefully because Wi-Fi and dashboard settings may need to be configured again.

Some settings require pressing **Save**, **Apply**, or **Connect & Save** before they take effect. Follow the confirmation/status message shown by the dashboard.

### 11.4 Physical wall switches

The three wired physical switches control:

- D7 / GPIO13 → Device 2.
- RX / GPIO3 → Device 3.
- TX / GPIO1 → Device 4.

The firmware uses pull-up inputs and expects the switch to connect the GPIO to GND. Physical switch input handling can be enabled/disabled in Settings. If a switch is not responding, check its wiring and the physical-switch enable setting.

### 11.5 Timers and schedules

- **Auto-Off Timer:** Use this when a device should turn itself OFF after a chosen duration (for example, a pump or light).
- **Per-device Schedule:** Use this when a device should turn ON and OFF at its own daily times.
- **Global Schedule (Fallback):** Use this to apply a shared ON/OFF schedule as a fallback for devices without their own schedule.
- Ensure the controller has the correct time and timezone. Schedules may not run at the expected time if the clock has not synchronized or was set incorrectly.

### 11.6 Energy and power information

The power configuration lets you enter a device's known wattage/current so the firmware can estimate energy usage. These values are user-entered estimates. This project does not measure actual voltage, current or watt-hours unless an external metering sensor is separately installed and integrated.

### 11.7 Weather

Weather information is obtained from Open-Meteo and needs internet connectivity. Configure the weather location in Settings. If the device is offline, location is missing or the service cannot be reached, weather information may not appear. Weather is informational and is not required for relay operation.

### 11.8 Event log

The Status page's event log shows events captured during the current boot. Use **Refresh Log** to reload the displayed events. The log is useful for checking recent switching and system activity; it is not a permanent audit history.

---

## 12. Blynk operation and troubleshooting

Blynk and the local dashboard are separate control paths:

- Local dashboard and physical switches work on the local network without Blynk.
- Blynk Cloud control needs the ESP8266 connected to a router with internet access.
- A controller connected to Wi-Fi may still be offline from Blynk if the router has no internet or blocks outbound cloud connections.

If Blynk does not connect:

1. Verify that the Auth Token belongs to the device created from the correct template.
2. Check the Template ID and Template name in the firmware.
3. Check that V1–V6 are Virtual Pin Integer datastreams with min 0 and max 1.
4. Check the router's internet connection.
5. Open the Status page and press **Connect to Blynk**.
6. Check the Blynk Console device status.
7. Confirm the router/firewall allows outbound connections to Blynk Cloud.

---

## 13. Reverse relay trigger logic (active-LOW / active-HIGH)

The supplied firmware uses active-LOW relay logic:

- Relay ON signal = LOW.
- Relay OFF signal = HIGH.

Only if the relay board is confirmed to be active-HIGH should the firmware trigger polarity be reversed. This is a firmware setting and is separate from choosing COM/NO or COM/NC contacts. Test the relay board with the load disconnected and low voltage before reconnecting appliances. Do not change the GPIO mapping.

---

## 14. Factory reset

The firmware supports a physical double-press of the NodeMCU **RST** button:

1. Press RST once and allow the board to finish booting.
2. Press RST a second time within approximately 20 seconds of the first reset.
3. The firmware detects the double external reset and restores factory settings.

A normal power-on does not count as this double-press gesture. A factory reset clears saved configuration, so you may need to reconnect to `SmartHome_AP` and set up Wi-Fi and dashboard access again. The Blynk token is part of the compiled firmware and must be changed in the sketch if you need to replace it.

The dashboard also provides a **Factory Reset All Settings** control. Use it only when you intend to clear saved settings.

---

## 15. OTA firmware updates

After the controller is connected to the network, the firmware includes ArduinoOTA support. When the board is reachable on the same network, it may appear as a network port in Arduino IDE under **Tools → Port**. Select the network port and upload the sketch.

For the first installation, use USB. If OTA is unavailable, use USB upload. The firmware currently sets an OTA password in the source; change it to a strong private value before using OTA on a network you do not fully trust. Keep the OTA password private.

---

## 16. Troubleshooting

| Problem | What to check |
|---|---|
| `webui.h: No such file` | Extract the ZIP; keep `webui.h` beside `SmartHomeController.ino` in the same folder. |
| ESP8266 board or header missing | Install/select ESP8266 core 3.1.2 in Boards Manager. |
| Library not found | Install the version-specific library listed below and restart Arduino IDE. |
| Dashboard does not open | Confirm phone and ESP are on the same Wi-Fi, check the IP, use `http://` and port 80, or connect to the ESP hotspot and use `http://192.168.4.1`. |
| `https://192.168.4.1` fails | Expected. The ESP8266 web server is HTTP-only. Use `http://` for local access. |
| Home Wi-Fi not connecting | Check SSID/password, 2.4 GHz support, signal, DHCP availability, and router client isolation. Reconnect through the hotspot and correct credentials. |
| Browser says no internet on hotspot | This is normal; the ESP hotspot is for local control. Stay connected and open `http://192.168.4.1`. |
| Relay appears reversed | The firmware uses active-LOW relay logic. Verify that the relay module is compatible; do not invert the code or change pins casually. |
| Physical switch not responding / upload trouble | RX and TX are used as switch inputs in this hardware. USB serial programming/debugging may conflict. Do not change pin assignments; use a safe programming method for the assembled board. |
| Lost dashboard password | Use the full factory reset process; this erases saved configuration and requires setup again. |
| Events disappeared | The event log is RAM-only and clears on every restart. |
| No COM port in Arduino IDE | Install the correct CH340/CP210x/FTDI driver, use a USB data cable, reconnect the board and check Device Manager. |
| Upload fails | Select NodeMCU 1.0 (ESP-12E Module), select the correct COM port, close Serial Monitor, try another data cable and temporarily disconnect RX/TX switch wires. |
| Board package not visible | Recheck the ESP8266 Boards Manager URL and install `esp8266 by ESP8266 Community`. |
| `BlynkSimpleEsp8266.h` not found | Install the official Blynk library through Library Manager and restart Arduino IDE. |
| Dashboard hotspot not visible | Check power and allow the board time to boot; power-cycle once. |
| Cannot open `192.168.4.1` | Connect to `SmartHome_AP`, disable mobile-data auto-switching/VPN temporarily, and use `http://` rather than `https://`. |
| Controller will not join router | Use 2.4 GHz Wi-Fi, verify SSID/password, signal strength and DHCP availability. |
| Dashboard opens but Blynk is offline | Check internet access, Auth Token, template/device association and V1–V6 datastreams; press Connect to Blynk on Status. |
| Relay operates backwards | Verify relay-board trigger polarity and contact wiring. Do not guess; test with the load disconnected. |
| Relay does not click or switch | Check IN pin, relay supply voltage/current, ground/reference and board specifications. Never power relay coils from 3.3 V. |
| Physical switch does not work | Check GPIO-to-switch-to-GND wiring and enable physical switches in Settings. |
| Browser shows old dashboard | Reload the page; if needed clear the browser cache for the controller address or open a private window. |
| Weather does not show | Check internet, configured location and weather service availability. |
| Schedule runs at the wrong time | Check NTP/manual time and timezone. Nepal is UTC+5:45. |
| Physical switch not working | Turn on the physical switch option from setting in the dashboard |



---

## 17. Important security and operating notes

- Keep the Blynk Auth Token, dashboard password and OTA password private.
- Change the OTA password from the example/default value before enabling OTA on an untrusted network.
- Do not expose the local dashboard directly to the public internet with router port forwarding.
- This build uses HTTP, not HTTPS. A browser may allow a home-screen shortcut, but a local HTTP IP is not a secure origin for full PWA service-worker installation in many browsers.
- ESP8266 supports 2.4 GHz Wi-Fi, not 5 GHz-only Wi-Fi.
- ESP8266 GPIO pins use 3.3 V logic and are not 5 V tolerant.
- Keep the relay module and mains wiring safely enclosed.
- Test relay operation with the mains/load disconnected after firmware or wiring changes.

## 18. Quick reference

| Item | Value |
|---|---|
| Board | NodeMCU ESP8266 / ESP-12E |
| ESP8266 Arduino Core | 3.1.2 |
| Setup AP SSID | `SmartHome_AP` |
| Setup AP password | `12345678` |
| Setup page | `http://192.168.4.1` |
| Admin username | `anmulla` |
| Relay outputs | D1/GPIO5, D2/GPIO4, D5/GPIO14, D6/GPIO12, D0/GPIO16 |
| Physical switches | D7/GPIO13, RX/GPIO3, TX/GPIO1 |
| Blynk pins | V1 through V6 |
| Relay trigger logic | Active-LOW |
| Nepal timezone | UTC+5:45 |

## 19. Project files

- `SmartHomeController.ino` — ESP8266 firmware sketch.
- `webui.h` — dashboard HTML, CSS and JavaScript embedded in firmware; keep it beside the `.ino` file.
- `web-preview.html` — browser preview of the dashboard; it is not the firmware sketch and does not connect to the controller by itself.
- `README.md` — this setup and operating manual.

**Preserve the existing wiring.** The GPIO mapping in this manual matches the supplied ESP8266 firmware build.
