# **RD6xxx Series Power Supply Modbus Protocol Reference**

**Note:** This documentation is based on reverse-engineered protocol data for the RD6006. While widely applicable to the RD6xxx series (RD6006, RD6012, RD6018, etc.), some registers or features may vary by specific firmware version.

## **1\. Communication Settings**

The RD6xxx series uses the standard Modbus RTU protocol over a USB-to-TTL (or RS485) serial interface.

| Parameter | Value |
| :---- | :---- |
| **Interface** | USB / TTL Serial |
| **Baud Rate** | **115200** (Default) |
| **Data Bits** | 8 |
| **Stop Bits** | 1 |
| **Parity** | None |
| **Flow Control** | None |
| **Slave Address** | 1-247 (Default: 1\) |

### **Frame Timing**

* **Frame Interval:** \> 3.5 character times.  
* **Response Timeout:** Suggest implementation of \>200ms timeout due to processing time.

## **2\. Data Format & Scaling**

All data is transmitted as 16-bit Integers (Big Endian).  
To get the actual physical value, the raw integer must be divided by a Scaling Factor.

### **Scaling Examples**

| Unit | Decimals | Scaling Divisor | Raw Value | Actual Value |
| :---- | :---- | :---- | :---- | :---- |
| **Voltage (V)** | 2 | / 100 | 1250 | 12.50 V |
| **Current (A)** | 3 | / 1000 | 3000 | 3.000 A |
| **Power (W)** | 2 | / 100 | 5000 | 50.00 W |
| **Temp (°C)** | 0 | / 1 | 45 | 45 °C |
| **Capacity (Ah)** | 3 | / 1000 | 5000 | 5.000 Ah |

## **3\. Function Codes**

The device supports the following Modbus function codes:

| Code | Name | Description |
| :---- | :---- | :---- |
| **0x03** | Read Holding Registers | Read current values, settings, and status. |
| **0x06** | Write Single Register | Set a single parameter (e.g., Set Voltage). |
| **0x10** | Write Multiple Registers | Set multiple parameters (e.g., Set Volts & Amps). |

## **4\. System Register Map (Real-time Data)**

These registers control the active state of the power supply.

| Address (Hex) | Address (Dec) | Name | R/W | Unit | Scale | Description |
| :---- | :---- | :---- | :---- | :---- | :---- | :---- |
| **0x0000** | 0 | **ID** | R | \- | 1 | Product Signature (e.g. 60062\) |
| **0x0002** | 2 | **SN** | R | \- | 1 | Serial Number |
| **0x0003** | 3 | **FW** | R | \- | 100 | Firmware Version (e.g. 128 \= 1.28) |
| **0x0004** | 4 | **T\_SYS\_C** | R | °C | 1 | System Temperature (°C) |
| **0x0005** | 5 | **T\_SYS\_F** | R | °F | 1 | System Temperature (°F) |
| **0x0008** | 8 | **V-SET** | R/W | V | 100 | Target Output Voltage |
| **0x0009** | 9 | **I-SET** | R/W | A | 1000 | Target Output Current Limit |
| **0x000A** | 10 | **V-OUT** | R | V | 100 | Actual Output Voltage (Readback) |
| **0x000B** | 11 | **I-OUT** | R | A | 1000 | Actual Output Current (Readback) |
| **0x000D** | 13 | **POWER** | R | W | 100 | Actual Output Power |
| **0x000E** | 14 | **V-IN** | R | V | 100 | Input Voltage |
| **0x000F** | 15 | **LOCK** | R/W | \- | 1 | Key Lock (0: Unlock, 1: Lock) |
| **0x0010** | 16 | **PROTECT** | R | \- | 1 | [Protection Status Code](https://www.google.com/search?q=%2361-protection-status-codes) |
| **0x0012** | 18 | **ONOFF** | R/W | \- | 1 | Output Enable (0: OFF, 1: ON) |
| **0x0013** | 19 | **LOAD\_M** | R/W | \- | 1 | Trigger Preset Load (Write 0-9 to load M0-M9) |
| **0x0020** | 32 | **BAT\_MODE** | R | \- | 1 | Battery Charging Mode (0: Off, 1: Active) |
| **0x0021** | 33 | **V-BATT** | R | V | 100 | Battery Voltage (if probe connected) |
| **0x0023** | 35 | **T\_EXT\_C** | R | °C | 1 | External Probe Temp (°C) |
| **0x0025** | 37 | **T\_EXT\_F** | R | °F | 1 | External Probe Temp (°F) |
| **0x0026** | 38 | **AH\_HI** | R | Ah | 1000 | Capacity High Word (Ah \* 1000\) |
| **0x0027** | 39 | **AH\_LO** | R | Ah | 1000 | Capacity Low Word |
| **0x0028** | 40 | **WH\_HI** | R | Wh | 1000 | Energy High Word (Wh \* 1000\) |
| **0x0029** | 41 | **WH\_LO** | R | Wh | 1000 | Energy Low Word |
| **0x0030** | 48 | **YEAR** | R/W | Y | 1 | System Clock: Year |
| **0x0031** | 49 | **MONTH** | R/W | M | 1 | System Clock: Month |
| **0x0032** | 50 | **DAY** | R/W | D | 1 | System Clock: Day |
| **0x0033** | 51 | **HOUR** | R/W | h | 1 | System Clock: Hour |
| **0x0034** | 52 | **MIN** | R/W | m | 1 | System Clock: Minute |
| **0x0035** | 53 | **SEC** | R/W | s | 1 | System Clock: Second |
| **0x0048** | 72 | **BACKLIGHT** | R/W | \- | 1 | Brightness Level (0-5) |

## **5\. Data Group Registers (Presets M0-M9)**

The device stores 10 groups of settings (M0-M9).  
Note that unlike the SK120X, the RD6xxx series only groups Voltage, Current, OVP, and OCP in these blocks.

* **Formula:** Address \= 0x0050 \+ (Group\_Number \* 4\)

The table below shows offsets for **Data Group N** (where N is 0 to 9).

| Offset | Function | R/W | Unit | Scale | Description |
| :---- | :---- | :---- | :---- | :---- | :---- |
| **\+0x00** | **V-SET** | R/W | V | 100 | Preset Voltage |
| **\+0x01** | **I-SET** | R/W | A | 1000 | Preset Current |
| **\+0x02** | **S-OVP** | R/W | V | 100 | Over Voltage Protection Setting |
| **\+0x03** | **S-OCP** | R/W | A | 1000 | Over Current Protection Setting |

**Example Addresses:**

* **M0 V-SET:** 0x0050 (Default power-on group)  
* **M1 V-SET:** 0x0054  
* **M2 V-SET:** 0x0058  
* **M9 OCP:** 0x0077 (Base 0x0074 \+ Offset 0x03)

## **6\. Enumerated Values**

### **6.1 Protection Status Codes**

Read register 0x0010.

| Value | Code | Meaning |
| :---- | :---- | :---- |
| **0** | OK | Normal Operation |
| **1** | OVP | Over Voltage Protection Triggered |
| **2** | OCP | Over Current Protection Triggered |

### **6.2 Configuration Flags**

Various configuration registers use 0/1 logic.

| Register | Name | 0 | 1 |
| :---- | :---- | :---- | :---- |
| **0x000F** | LOCK | Unlocked | Locked |
| **0x0012** | ONOFF | Output OFF | Output ON |
| **0x0020** | BAT\_MODE | Normal Mode | Battery Charging Mode |
| **0x0045** | BUZZER | Mute | Sound On |

## **7\. Communication Examples**

### **Example 1: Turn Output ON**

Write **1** to Address **0x0012**.

* **Tx:** 01 06 00 12 00 01 E8 0F  
  * 01: Slave ID  
  * 06: Write Single  
  * 00 12: Register Address (ONOFF)  
  * 00 01: Data (ON)  
  * E8 0F: CRC16

### **Example 2: Set Voltage to 12.00V**

Write **1200** (0x04B0) to Address **0x0008**.

* **Tx:** 01 06 00 08 04 B0 0A 92  
  * 01: Slave ID  
  * 06: Write Single  
  * 00 08: Register Address (V-SET)  
  * 04 B0: Data (1200 dec)  
  * 0A 92: CRC16

### **Example 3: Read Live Voltage, Current, and Power**

Read 4 registers starting at 0x000A (V-OUT).  
This reads 0x000A (V-OUT), 0x000B (I-OUT), 0x000C (Reserved), and 0x000D (POWER).

* **Tx:** 01 03 00 0A 00 04 64 C9  
* **Rx:** 01 03 08 04 B0 0B B8 00 00 05 DC \[CRC\]  
  * 04 B0: V-OUT (1200 \-\> 12.00 V)  
  * 0B B8: I-OUT (3000 \-\> 3.000 A)  
  * 00 00: Reserved  
  * 05 DC: POWER (1500 \-\> 15.00 W) (Note: Power is calculated as V\*I)