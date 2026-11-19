# Industrial 3-Phase 15 kW Compressor Motor Protection & Dual-Core SCADA HMI

**Author:** Oktafian Bayu Prakosa, S.T.  
**Context:** Engineering Development Center (EDC) Intern Project, PT Industri Kereta Api (Persero) (Aug 2026 - Feb 2027)  
**Target Hardware:** ESP32 Dual-Core (38-Pin + G-V-S Shield), Schneider Electric EasyLogic PM2220/PM2230, 3x Schneider METSECT5MA030 CT (300/5A), MAX31865 + 3-Wire Platinum RTD PT100, Fuji Electric SC-N2 Contactor (50A AC-3)

---

## 1. Engineering Brief

* **Challenge:** High-power 3-phase 15 kW (20 HP, 380 VAC L-L, **nominal current 25.64 A**) shop-floor air compressors face winding burnout from phase unbalance, mechanical overload, and telemetry blocking when network tasks run on the same core as protection logic.
* **Architecture:** Strict **Dual-Core FreeRTOS task isolation** on ESP32 (`PROD_PM2230_V19`):
  * **Core 1 (Deterministic Protection & Field Bus):** 500 ms Modbus RTU RS485 polling, SPI PT100 acquisition, 5-parameter auto-trip evaluation with 7.0 s inrush debounce, NVS Flash fault latch, contactor relay control, and an 8-page rolling I2C LCD display.
  * **Core 0 (Zero-Delay HMI & Telemetry):** Asynchronous Localhost Web SCADA HMI (`< 10 ms` response) over Dual-Mode Wi-Fi (STA + autonomous SoftAP fallback at `192.168.4.1`) plus 2-second JSON uplink to ThingsBoard Cloud MQTT.
* **Verified Outcome:** Bench-validated commissioning-ready panel protection with zero false startup trips, persistent post-outage fault memory, and sub-10 ms local SLD telemetry.

---

## 2. Industrial Hardware Sizing & Datasheet Alignment

| Subsystem | Industrial Component | Verified Specification | Functional Role & Margin |
| :--- | :--- | :--- | :--- |
| **Controller** | ESP32 DevKit V1 (38-Pin) + G-V-S Shield | Dual-Core Xtensa (160 MHz), 5V DC rail | FreeRTOS task isolation, Web SCADA server, NVS fault latch |
| **Power Meter** | Schneider EasyLogic PM2220 / PM2230 | 35 - 480 VAC L-L, 5A/1A CT input, RS485 Modbus RTU, Class 1.0 (IEC 62053-21) | True-RMS measurement of `V_RS`, `V_ST`, `V_TR`, `I_R`, `I_S`, `I_T`, kW, PF, Hz, kWh |
| **Current Transformers** | 3x Schneider METSECT5MA030 | 300/5A ratio (60x scaling), Class 0.5 (8 VA), 3 kV insulation | Prevents magnetic core saturation during 150 - 180 A DOL startup inrush |
| **Main Contactor** | Fuji Electric SC-N2 | AC-3 (380 - 440V): 18.5 kW / 40A (`Ith = 60A`), 220 VAC coil (IEC 60947-4-1) | Primary 3-phase disconnect with **+35% safety margin** above 15 kW load |
| **Thermal Sensor** | MAX31865 SPI + 3-Wire RTD PT100 | IEC 60751 Platinum RTD, ±0.5 °C accuracy, 6-cycle (3.0 s) anti-glitch filter | Continuous stator winding temperature tracking with open/short detection |
| **RS485 Transceiver** | MAX485 TTL / Auto-Flow RS485 | UART2 (`RX2 GPIO 16`, `TX2 GPIO 17`, `DE/RE GPIO 4`), -7V to +12V CM range | Differential twisted-pair Modbus with EIA/TIA-485-A ground tied to PM2230 Term 25 (0V C) |
| **Isolation & Pilot Lamps** | EL817 Optocoupler Relays (5 kV) + 220V SPDT Lamps | Active-Low control (`GPIO 25/26`, `GPIO 32`), 10A 250 VAC contacts | Fail-safe SPDT wiring: Red lamp lights on Standby, Trip, or MCU power loss; Green on Run |
| **Panel Display** | LCD 16x2 I2C (PCF8574) | I2C Bus (`SDA GPIO 21`, `SCL GPIO 22`), auto-scan `0x27` / `0x3F` | Displays 8 rotating operational and diagnostic screens on enclosure door |

---

## 3. Autonomous 5-Parameter Auto-Trip Matrix

| Fault Category | Source Sensor | Trip Threshold | Debounce | Protection Action & Latch Code |
| :--- | :--- | :--- | :--- | :--- |
| **Phase Overcurrent** | PM2220 / PM2230 (`I_R, I_S, I_T`) | `>= 29.49 A` (115% of 25.64 A nominal) | 7.0 s | Open Contactor K1, Buzzer, Latch NVS (`OC-R`, `OC-S`, `OC-T`, `OC-RST`) |
| **Undervoltage** | PM2220 / PM2230 (`V_LL`) | `< 342.0 VAC` (380V - 10%) | 7.0 s | Open Contactor K1, Red Pilot Lamp, Latch NVS (`UNDERVOLTAGE`) |
| **Overvoltage** | PM2220 / PM2230 (`V_LL`) | `> 399.0 VAC` (380V + 5%) | 7.0 s | Open Contactor K1, Protect Winding Insulation, Latch NVS (`OVERVOLTAGE`) |
| **Active Power Overload** | PM2220 / PM2230 (`P_total`) | `>= 16.50 kW` | 7.0 s | Open Contactor K1, Mechanical Jam Stop, Latch NVS (`OVERLOAD`) |
| **Stator Overheat** | MAX31865 3-Wire RTD PT100 | `>= 100.0 °C` | 7.0 s | Open Contactor K1, Thermal Trip, Latch NVS (`OVERHEAT`) |
| **Bus / Sensor Timeout** | Modbus CRC / SPI Fault Register | No valid frame or PT100 open circuit | 3.0 s | Fail-Safe Stop (`METER_SENSOR_ERROR` / `PT100_SENSOR_ERROR`) |

---

## 4. Modbus RTU Register Map & EMC Pin Allocation

* **PM2220 / PM2230 Holding Registers (9600 bps, 8-N-1, Slave ID 1, Float32):** `3020/3022/3024` (`V_RS, V_ST, V_TR`), `3000/3002/3004` (`I_R, I_S, I_T`), `3060` (Total kW), `3084` (PF), `3110` (Hz), `3204` (Total kWh). Runtime Web HMI toggles support PDU Offset (`-1 / 0`) and Word-Swap (`ABCD / CDAB`).
* **EMC Panel Segregation:** Right-side Low-Voltage signal cluster (`RS485 GPIO 16/17/4`, `SPI GPIO 5/18/19/23`, `I2C GPIO 21/22`); Left-side High-Voltage actuator cluster (`Contactor Coil GPIO 25/26`, `Pilot Lamp GPIO 32`, `Buzzer GPIO 27`).
