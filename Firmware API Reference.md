> Note: This is a speculative documentation sample for portfolio purposes.

# **BME688 – I2C Firmware Reference**  
*For direct register access. Compliance with IEC 60079‑11 (intrinsic safety) requires external current‑limiting resistors on SDA/SCL – see section 4.2 of the system safety manual.*

### **1. I2C Hardware Requirements**

| **Pin** | **Connection** | **Condition** |
| --- | --- | --- |
| SCL | MCU I2C clock | Pull‑up resistor 4.7 kΩ (max 10 kΩ) |
| SDA | MCU I2C data | Pull‑up resistor 4.7 kΩ (max 10 kΩ) |
| SDO | GND or VDDIO | Sets I2C address (see §2) |
| CSB | VDDIO | Disables SPI, enables I2C |

### **2. I2C Addressing & Clock**

- **7‑bit device address**  
  - `0x76` (SDO = GND)  
  - `0x77` (SDO = VDDIO)  
- **8‑bit write address** = `(addr << 1) | 0x00`  
- **8‑bit read address**  = `(addr << 1) | 0x01`  
- **Supported clock rates**  
  - 100 kHz (Standard)  
  - 400 kHz (Fast)  
  - 3.4 MHz (High‑Speed – verify master timing)

### **3. Critical Registers**

| **Address**|**Name**|**Access** | **Purpose** |
| --- | --- | --- | --- |
| `0xD0` | `CHIP_ID` | R | Expected value = `0x61` |
| `0xE0` | `RESET` | W | Write `0xB6` to soft‑reset |
| `0x72` | `CTRL_HUM` | R/W | Humidity oversampling (set before `0x74`) |
| `0x74` | `CTRL_MEAS` | R/W | Temp/pressure oversampling + power mode |
| `0x22` | `TEMP_MSB` | R | Raw temperature MSB |

### **4. Initialization & Forced Measurement Sequence**

1. **MCU I2C peripheral** – conﬁgure pins, clock, pull‑ups.  
2. **Verify device** – read `CHIP_ID`; abort if not `0x61`.  
3. **Soft reset** – write `0xB6` to `RESET`; wait 2 ms minimum.  
4. **Set oversampling** – write `CTRL_HUM` then `CTRL_MEAS`.  
5. **Trigger forced mode** – set bits [1:0] of `CTRL_MEAS` to `0b01` or `0b10`.  
6. **Wait for completion** – poll `meas_status` bit in `STATUS` (`0x73`) until cleared.  
7. **Read data** – burst read from `0x22` (temperature), `0x24` (pressure), `0x26` (humidity), `0x2A` (gas).

### **5. Code Example – Chip ID Verification**

```c
#include <stdint.h>

#define BME688_ADDR      0x76
#define BME688_REG_CHIP  0xD0
#define BME688_EXPECTED  0x61

/* Hardware‑abstraction functions (implement per MCU):
 *   int8_t i2c_read(uint8_t dev_addr, uint8_t reg_addr, uint8_t *data, uint16_t len)
 */

int8_t bme688_verify(void) {
    uint8_t id;
    if (i2c_read(BME688_ADDR, BME688_REG_CHIP, &id, 1) != 0)
        return -1;   // bus error
    if (id != BME688_EXPECTED)
        return -2;   // wrong chip
    return 0;        // OK
}
```

***Compliance note:** The above code assumes non‑hazardous area operation. For Zone 1/2 installations, add a 10‑ms delay after each I2C transaction to limit peak power – refer to document IIoT‑SAFE‑BME688‑01.***