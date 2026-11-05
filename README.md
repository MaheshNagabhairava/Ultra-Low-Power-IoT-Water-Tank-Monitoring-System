# Ultra-Low-Power Autonomous IoT Water Tank Monitoring System

An autonomous, battery-operated IoT liquid level monitoring system engineered with an ESP8266 (ESP-12F). The device operates on an ultra-low-power hardware architecture utilizing a custom capacitive differentiator circuit to convert static float-switch state changes into hardware reset pulses. Achieving a deep-sleep current draw of $<20\,\mu\text{A}$ and an active operational window of under $5\text{ seconds}$ per event, the system delivers multi-month deployment life on a single 18650 Li-ion cell.

---

## Key Highlights & Performance Metrics

* **Power Profile:** $<20\,\mu\text{A}$ sustained deep-sleep current; operates for multi-month intervals on a standard $2600\text{ mAh}$ 1S 18650 Li-ion battery.
* **Transient Execution Duration:** $\approx 3\text{ to }5\text{ seconds}$ total active run time from physical switch contact to deep sleep re-entry.
* **Hardware Pulse-Wake Architecture:** Dual-channel capacitive differentiator network ($1\,\mu\text{F}\text{ MLCC} + 100\text{ k}\Omega / 10\text{ k}\Omega\text{ pull-ups}$) converts sustained reed closures to a single $\approx 11\text{ ms}$ negative pulse, preventing continuous reset loops without requiring continuous MCU polling.
* **Overvoltage Protection:** Fast-switching 1N5819 Schottky diode clamping network shunts positive transient inductive/capacitive spikes ($>3.3\text{V}$) into the power rail upon switch release.
* **Accelerated Cloud Association:** Firmware employs RTC User Memory caching for Wi-Fi channel and BSSID parameters, reducing network connection latency from $\sim 3\text{--}5\text{ seconds}$ down to $250\text{--}350\text{ ms}$.
* **Secure Telemetry Pipeline:** Dispatches TLS-secured REST payloads directly to a Supabase PostgreSQL backend via HTTP POST (`Prefer: return=minimal`).

---

## System Architecture

```
+---------------------------------------------------------------------------------------+
|                                    POWER SUBSYSTEM                                    |
|  +--------------------+      +--------------------+      +-------------------------+  |
|  | 1S 18650 Li-ion    | ---> | Step-Down Buck     | ---> | Filter / Decoupling:    |  |
|  | (3.0V - 4.2V Rail) |      | Regulator (3.3V)   |      | 220uF Bulk + 100nF MLCC |  |
|  +--------------------+      +--------------------+      +-------------------------+  |
+---------------------------------------------------------------------------------------+
                                           |
                                  3.3V Power Bus
                                           |
+---------------------------------------------------------------------------------------+
|                                HARDWARE SENSING & WAKE                                |
|                                                                                       |
|  Float 1 (20% Threshold) ---+---> GPIO4 (Level Sense)                                 |
|                             |                                                         |
|                             +--- [ 1uF MLCC ] --- [ 1k R_iso1 ] ---+                  |
|                                                                    |                  |
|  Float 2 (90% Threshold) ---+---> GPIO5 (Level Sense)              |                  |
|                             |                                      |                  |
|                             +--- [ 1uF MLCC ] --- [ 1k R_iso2 ] ---+---> ESP8266 RST  |
|                                                                    |     (Pin 1)      |
|  Protection & Pull-Up Network:                                     |                  |
|  - 3.3V Rail --- [ 10k Pull-Up ] ----------------------------------+                  |
|  - RST Node  --->| [ 1N5819 Schottky Diode ] ---> 3.3V Rail (Clamp)                   |
+---------------------------------------------------------------------------------------+
                                           |
                                      Wake Trigger
                                           |
+---------------------------------------------------------------------------------------+
|                                 MICROCONTROLLER & CLOUD                               |
|  +---------------------------------------------------------------------------------+  |
|  | ESP8266 (ESP-12F) @ 160MHz                                                      |  |
|  | - ROM Bootloader execution                                                      |  |
|  | - Interrogate GPIO4 / GPIO5 state                                               |  |
|  | - Retrieve cached AP BSSID / Channel from RTC SRAM                              |  |
|  | - Establish 802.11 association (<350ms)                                         |  |
|  | - Execute HTTPS POST to Supabase REST Endpoint                                  |  |
|  | - Flush Wi-Fi radio & enter ESP.deepSleep(0)                                    |  |
|  +---------------------------------------------------------------------------------+  |
+---------------------------------------------------------------------------------------+
```

---

## Hardware Engineering Deep-Dive

### 1. Pulse-Wake Capacitive Differentiator

When monitoring liquid levels with traditional reed float switches, the contacts remain closed for hours or days while the water level remains above or below a threshold. A direct connection to the ESP8266 `RST` pin would hold the microcontroller in a permanent hardware reset state, drawing continuous current and preventing firmware execution.

To solve this without adding an active secondary low-power timer (e.g., TPL5110), this circuit implements a passive **capacitive differentiator network**:

1. **Idle State (Switch Open):** 
   * Both sides of the differentiator capacitor ($C_1 = 1\,\mu\text{F}$) are held at $3.3\text{V}$ via pull-ups ($100\text{ k}\Omega$ on the switch side, $10\text{ k}\Omega$ on the RST side).
   * The potential difference across the dielectric is $0\text{V}$, resulting in zero static current leakage.
2. **Contact Event (Switch Closes to GND):**
   * The negative plate of $C_1$ is pulled to $0\text{V}$ instantly.
   * Because charge across a dielectric cannot change in zero time ($I = C \frac{dV}{dt}$), the positive plate is pulled down by an identical voltage step ($\Delta V = -3.3\text{V}$), bringing the `RST` node to $\approx 0\text{V}$.
   * The low voltage breaches the reset threshold ($V_{IL} < 0.8\text{V}$), initiating MCU bootloader initialization.
3. **Autonomous RC Reset Release:**
   * Current flows from the $3.3\text{V}$ rail through the $10\text{ k}\Omega$ pull-up and isolation resistor ($1\text{ k}\Omega$), charging the capacitor back to a stable differential.
   * The duration of the reset pulse is governed by:
     $$\tau = R_{\text{eq}} \times C_1 \approx (10\text{ k}\Omega + 1\text{ k}\Omega) \times 1\,\mu\text{F} \approx 11\text{ ms}$$
   * The RST node springs back above the minimum logic HIGH threshold ($>2.47\text{V}$) in $\sim 11\text{ ms}$, releasing reset and letting the chip boot normally even while the float switch remains submerged.

### 2. High-Voltage Clamping via Schottky Diode

When the water level recedes and the float switch snaps open:
* The $100\text{ k}\Omega$ pull-up pulls the negative plate from $0\text{V}$ back up to $3.3\text{V}$.
* Because the capacitor is charged ($3.3\text{V}$ differential), pushing the lower plate up by $+3.3\text{V}$ causes the upper plate to track upward toward:
  $$V_{\text{peak}} = V_{\text{neg}} + V_{\text{stored}} = 3.3\text{V} + 3.3\text{V} = 6.6\text{V}$$
* **Protection Mechanism:** A **1N5819 Schottky diode** is connected with its Anode on `RST` and Cathode on `3.3V`. The moment the voltage attempts to exceed $3.3\text{V} + V_F$ ($\approx 3.55\text{V}$), the diode conducts, dumping the small capacitive charge into the $220\,\mu\text{F}$ power-rail reservoir capacitor. This protects the internal gate oxides of the ESP8266 from overvoltage breakdown.

---

## Bill of Materials (BOM)

| Designator | Component | Package / Footprint | Function |
| :--- | :--- | :--- | :--- |
| **U1** | ESP8266 (ESP-12F) | Module / Adapter Header | Primary Microcontroller & Wi-Fi Engine |
| **U2** | Mini-360 / Step-Down Buck | 4-Pin Module | High-efficiency 3.3V DC-DC Step-Down |
| **BT1** | 1S 18650 Li-ion Cell | Battery Cradle | Main Power Supply ($3.0\text{V}\text{--}4.2\text{V}$) |
| **SW1, SW2** | Magnetic Reed Float Switches | External Wire Lead (J2, J3) | 20% (Low) and 90% (High) Level Sensors |
| **C1, C2** | $1\,\mu\text{F}$ 50V Ceramic (105) | Radial Through-Hole | Differentiator Edge-Coupling Capacitors |
| **C3** | $220\,\mu\text{F}$ 16V Electrolytic | Radial Through-Hole | Power Rail RF Current Surge Reservoir |
| **C4** | $100\text{ nF}$ Ceramic (104) | Radial Through-Hole | High-Frequency Decoupling across VCC/GND |
| **D1** | 1N5819 Schottky Diode | DO-41 Through-Hole | Positive Transient Overvoltage Clamp |
| **R1, R2** | $100\text{ k}\Omega$ 1/4W 5% | Axial Through-Hole | Float Node Pull-Up Resistors |
| **R3, R4** | $1\text{ k}\Omega$ 1/4W 5% | Axial Through-Hole | Pulse Current Isolation Resistors |
| **R5** | $10\text{ k}\Omega$ 1/4W 5% | Axial Through-Hole | RST Line Hardware Pull-Up Resistor |
| **R6, R7, R8, R9** | $10\text{ k}\Omega$ 1/4W 5% | Axial Through-Hole | Hardware Boot Strapping (EN, GPIO0, GPIO2, GPIO15) |
| **J1, J2, J3** | 2-Pin Screw Terminals | 5.08mm Pitch THT | Power In, Float 1, Float 2 Interfaces |

---

## Hardware Pin Mapping & Strapping Configuration

To guarantee clean, repeatable boot execution without an external programmer or adapter board:

| ESP-12F Pin | Function | Connection Details |
| :--- | :--- | :--- |
| **Pin 1 (RST)** | Reset Line | Tied to Differentiator outputs + 10k Pull-Up to 3.3V + Diode Anode |
| **Pin 3 (EN / CH_PD)** | Chip Enable | Pulled to **3.3V** via $10\text{ k}\Omega$ (Mandatory for Chip Execution) |
| **Pin 8 (VCC)** | Supply Rail | Direct 3.3V from Buck Output + $100\text{ nF} \parallel 220\,\mu\text{F}$ filtering |
| **Pin 10 (GPIO4)** | Sensor Input 1 | Float 1 (20% Level) Junction + $100\text{ k}\Omega$ Pull-up |
| **Pin 15 (GND)** | Ground Reference | Common System Ground Plane |
| **Pin 16 (GPIO15)** | Strapping Pin | Pulled to **GND** via $10\text{ k}\Omega$ (Required for Flash Boot) |
| **Pin 17 (GPIO2)** | Strapping Pin | Pulled to **3.3V** via $10\text{ k}\Omega$ (Boot diagnostics HIGH) |
| **Pin 18 (GPIO0)** | Strapping Pin | Pulled to **3.3V** via $10\text{ k}\Omega$ (SPI Flash Boot Mode) |
| **Pin 20 (GPIO5)** | Sensor Input 2 | Float 2 (90% Level) Junction + $100\text{ k}\Omega$ Pull-up |

---

## Firmware Optimization & Execution Lifecycle

```text
[ Physical Level Event ]
         |
  Hardware Pulse (~11ms on RST pin)
         |
[ ESP-12F Boots: setup() ]
         |
  +---> Set CPU Frequency to 160MHz (Halves TLS math duration)
  +---> Debounce delay (60ms) -> Interrogate GPIO4 / GPIO5 (LOW = Active)
  +---> Read RTC Memory: Fetch cached Wi-Fi channel & AP BSSID
  +---> WiFi.begin(SSID, PASS, channel, bssid) [Connects in 250-350ms]
  +---> Establish TLS 1.2 Session (WiFiClientSecure with minimal buffer size)
  +---> HTTPS POST /rest/v1/todos to Supabase (Prefer: return=minimal)
  +---> Radio Teardown: WiFi.disconnect(true) & WiFi.mode(WIFI_OFF)
  +---> Enter ESP.deepSleep(0)
         |
[ Microampere Deep-Sleep Idle ]
```

### RTC Wi-Fi Fast-Connect Mechanism
Standard 802.11 association performs active/passive scanning across 13–14 channels, consuming up to $4\text{ seconds}$ of peak RF current ($300\text{--}400\text{ mA}$). 

By persisting the network channel and Gateway BSSID inside **RTC User SRAM** (which stays powered during deep sleep), the ESP8266 locks its synthesizer directly onto the target frequency without scanning:
```cpp
struct {
  uint32_t crc32;
  uint8_t channel;
  uint8_t bssid[6];
} rtcData;

// Read from RTC memory block 64
ESP.rtcUserMemoryRead(64, (uint32_t*)&rtcData, sizeof(rtcData));
WiFi.begin(STASSID, STAPSK, rtcData.channel, rtcData.bssid, true);
```

---

## Repository Structure

```text
├── firmware/
│   └── tank_level_telemetry/
│       └── tank_level_telemetry.ino   # Optimized fast-boot C++ firmware
├── hardware/
│   ├── schematics/                    # KiCad schematic files & netlists
│   └── gerbers/                       # Production fabrication files (ZIP)
├── enclosure/
│   └── parametric_box.html            # Three.js 3D web preview & STL exporter
├── docs/
│   ├── circuit_analysis.md            # Mathematical model of differentiator
│   └── power_profiles.png             # Oscilloscope traces of active wake cycle
├── LICENSE
└── README.md
```

---

## License

This project is released under the **MIT License**.
