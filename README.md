# STM32F103C8T6 Battery Voltage Monitor

A software-based battery voltage monitoring system implemented on the STM32F103C8T6 (Blue Pill). The project measures battery voltage through an analog voltage divider and provides battery status through LEDs and UART output.

The firmware uses timer interrupts, ADC with Normal DMA, UART interrupts, and an Independent Watchdog (IWDG) to implement a reliable interrupt-driven battery monitoring system while maintaining the core voltage measurement and battery status functionality.

## Features

- Battery voltage measurement using ADC1 Channel 0 (PA0)
- 10 kΩ / 10 kΩ voltage divider
- 10 ADC samples collected using Normal DMA
- Automatic measurement every 1 second
- TIM2 periodic interrupt
- ADC/DMA completion interrupt
- Non-blocking UART transmission using interrupts
- Battery percentage estimation
- LOW / MID / FULL LED indication
- Voltage change (ΔV) between consecutive measurements
- Independent Watchdog (IWDG)
- USART communication at 9600 baud
- Tera Term compatible serial output

## Hardware

| Component | Connection |
|---|---|
| STM32F103C8T6 Blue Pill | Main MCU |
| Voltage Divider | 10 kΩ + 10 kΩ |
| Battery ADC Input | PA0 / ADC1_IN0 |
| LOW LED | PA1 |
| MID LED | PA2 |
| FULL LED | PA3 |
| USART1 TX | PA9 |
| USART1 RX | PA10 |
| External Crystal | 8 MHz HSE |

## Voltage Measurement

The battery voltage is reduced using a 10 kΩ / 10 kΩ voltage divider.

The ADC voltage is calculated as:

```text
Vadc = ADC_Value / 4095 × 3.3 V
```

Since the voltage divider divides the battery voltage by two:

```text
Vbattery = Vadc × 2
```

Therefore:

```text
Vbattery = (ADC_Value / 4095) × 3.3 × 2
```

The STM32F103 ADC provides 12-bit resolution, giving ADC values from 0 to 4095.

## Battery Percentage

The firmware uses a simple voltage-based percentage estimation:

```text
2.8 V → 0%
3.3 V → 100%
```

The calculated percentage is limited between 0% and 100%.

This is a voltage-based estimate and is not intended to represent a chemistry-specific battery State of Charge (SOC).

## LED Indication

| Battery Percentage | LED | Status |
|---:|---|---|
| 0–29% | PA1 | LOW |
| 30–79% | PA2 | MID |
| 80–100% | PA3 | FULL |

## Firmware Architecture

The measurement process is interrupt-driven:

```text
TIM2 Interrupt
      ↓
Start ADC + DMA
      ↓
Collect 10 ADC Samples
      ↓
DMA Complete Interrupt
      ↓
ADC Completion Callback
      ↓
Set adc_data_ready Flag
      ↓
Main Loop
      ↓
Average Samples
      ↓
Calculate Battery Voltage
      ↓
Calculate Percentage
      ↓
Calculate Voltage Change
      ↓
Update LEDs
      ↓
UART Interrupt Transmission
```

The interrupt callbacks only handle the required event and set a flag. The main loop performs the calculations and application processing.

## ADC DMA

ADC data is transferred using **DMA1 Channel 1**.

DMA operates in **Normal Mode**:

```text
10 samples
    ↓
DMA transfer complete
    ↓
Process samples
    ↓
Wait for next 1-second measurement
```

Circular DMA is not used.

## Watchdog

The **Independent Watchdog (IWDG)** is enabled to protect the system against firmware lockups.

The main loop periodically refreshes the watchdog:

```c
HAL_IWDG_Refresh(&hiwdg);
```

The watchdog is intentionally not refreshed inside interrupt callbacks. If the main application becomes stuck, the watchdog can expire and reset the MCU.

## UART Configuration

USART1 is configured as:

```text
Baud Rate : 9600
Data      : 8 bits
Parity    : None
Stop Bits : 1
Flow Ctrl : None
```

Example Tera Term output:

```text
V: 3.30V  P:100%  S:FULL  dV:+0.00V
V: 3.29V  P:98%   S:FULL  dV:-0.01V
V: 3.20V  P:80%   S:FULL  dV:-0.09V
```

Where:

- `V` = Battery voltage
- `P` = Estimated battery percentage
- `S` = Battery status
- `dV` = Voltage change from the previous measurement

## Clock Configuration

The system uses an external 8 MHz crystal:

```text
HSE       = 8 MHz
PLL       = ×9
SYSCLK    = 72 MHz
AHB       = 72 MHz
APB1      = 36 MHz
APB2      = 72 MHz
ADC Clock = 12 MHz
```

TIM2 is configured to generate an approximately 1-second periodic interrupt.

## Development Tools

- STM32CubeMX
- STM32CubeIDE
- STM32CubeProgrammer
- Tera Term
- GitHub

## Project Structure

```text
STM32-Battery-Monitor/
│
├── Core/
│   ├── Inc/
│   └── Src/
│       ├── main.c
│       ├── stm32f1xx_hal_msp.c
│       └── stm32f1xx_it.c
│
├── Drivers/
│
├── STM32-Battery-Monitor.ioc
├── .project
├── .cproject
└── README.md
```

## Limitations

This project measures **battery voltage only**.

No current-sensing hardware is included, so the system does not directly measure:

- Battery current
- Power consumption
- Capacity in mAh
- Energy in Wh
- Accurate battery State of Health (SOH)

The battery percentage is therefore a simple voltage-based estimation.

## Future Improvements

Possible future extensions include:

- Battery current sensing
- Coulomb counting
- More accurate battery SOC estimation
- Low-voltage protection
- External EEPROM/Flash data logging
- PC-based graphical monitoring

## Author

**Kunal Kailash Shinde**

Developed as an academic embedded-systems project using the **STM32F103C8T6**.

**Concepts demonstrated:**  
ADC • DMA • Interrupts • Timers • UART • Watchdog • Embedded C • STM32 HAL
