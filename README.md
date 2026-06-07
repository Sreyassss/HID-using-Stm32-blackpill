# STM32 USB HID Gamepad

A gamepad built on the **STM32F401CCU6 (Black Pill)**. Plug it in and Windows picks it up as a gamepad instantly — no drivers to install, no software needed.

---

## What it does

- Works out of the box on Windows (shows up in `joy.cpl`)
- 2 analog joysticks (4 axes total)
- 4 buttons
- The STM32 reads joystick voltages in the background using DMA — the CPU doesn't have to do anything for that
- No integer overflow bug on joystick axis conversion (explained below)

---

## How it works

Here's the data flow from physical input to your PC:

```
[Joysticks / Buttons] → [ADC reads voltage] → [DMA stores values] → [Format into 5 bytes] → [Send over USB every 10ms]
```

1. **Joysticks** output a voltage between 0V and 3.3V depending on position.
2. **ADC** converts those voltages into numbers (0–255).
3. **DMA** automatically moves those numbers into RAM — the CPU doesn't need to be involved at all.
4. **Formatting** — the numbers are converted from the ADC range (0–255) to what USB expects (−127 to 127), and button states are packed into one byte.
5. **USB** sends a 5-byte packet to the PC every 10 milliseconds.

---

##  CubeMX Settings
![](WhatsApp%20Image%202026-06-07%20at%2022.11.52.jpeg)
| Setting | Value |
|---|---|
| Debug | SWD |
| Clock | HSE Crystal → PLL → 84 MHz |
| PA0–PA3 | GPIO Input with internal pull-up |
| ADC1 | 8-bit resolution, Scan mode on, DMA enabled |
| ADC trigger | TIM3 |
| DMA2 | Circular mode |

---

## Generate Your Custom HID Descriptor
![](WhatsApp%20Image%202026-06-07%20at%2022.15.41.jpeg)
The USB descriptor is a byte array that tells Windows what your device is — how many buttons, how many axes, what range the axes use, etc.

Download [DT.exe](https://www.usb.org/document-library/hid-descriptor-tool) from the USB-IF website. It's a Windows app that lets you visually build a descriptor and export it as a C array. Useful if you want to understand exactly what each byte means.

---

## Swap the Mouse Descriptor in usbd_hid.c

CubeMX generates a mouse HID descriptor by default. You need to replace it with your gamepad one.

### Replace the descriptor array

Open `USB_DEVICE/App/usbd_hid.c`. Find the array `HID_MOUSE_ReportDesc` — it looks like this:

```c
__ALIGN_BEGIN static uint8_t HID_MOUSE_ReportDesc[HID_MOUSE_REPORT_DESC_SIZE] __ALIGN_END =
{
  /* your default mouse descriptor bytes here */
};
```

Replace the entire contents of the array with the gamepad descriptor you generated in Step 2.

### Update the size macro

Open `USB_DEVICE/App/usbd_hid.h`. Find:

```c
#define HID_MOUSE_REPORT_DESC_SIZE    XX
```

Change `XX` to the exact byte count of your new descriptor. If this number is wrong, the device won't enumerate — Windows will reject it.

---

## Write the Code in main.c

### Define the report struct

Add this near the top of `main.c`, outside of any function:

```c
// 5-byte HID report: 1 byte buttons + 4 bytes for axes
typedef struct {
  uint8_t buttons;   // bits 0-3 = buttons X, Y, A, B
  int8_t  lx;        // left stick X
  int8_t  ly;        // left stick Y
  int8_t  rx;        // right stick X
  int8_t  ry;        // right stick Y
} __attribute__((packed)) GamepadReport;

GamepadReport report;

// DMA will fill this buffer with ADC readings
uint8_t adcbuf[4];
```

### Start ADC and DMA

In `main()`, after all the init calls, start the ADC in DMA mode:

```c
HAL_ADC_Start_DMA(&hadc1, (uint32_t*)adcbuf, 4);
```

### Convert ADC values in the DMA callback

Add this function anywhere in `main.c`. It runs automatically every time the DMA finishes a full cycle of ADC readings:

```c
void HAL_ADC_ConvCpltCallback(ADC_HandleTypeDef *hadc)
{
  // ADC gives 0-255. HID wants -127 to 127.
  // If ADC hits 255, subtracting 127 gives 128 which overflows int8_t.
  // So we cap at 254 before doing the math.
  report.lx = (adcbuf[0] > 254) ? 127 : (int8_t)(adcbuf[0] - 127);
  report.ly = (adcbuf[1] > 254) ? 127 : (int8_t)(adcbuf[1] - 127);
  report.rx = (adcbuf[2] > 254) ? 127 : (int8_t)(adcbuf[2] - 127);
  report.ry = (adcbuf[3] > 254) ? 127 : (int8_t)(adcbuf[3] - 127);
}
```

### Read buttons and send the report

In your main loop (`while(1)`), add this:

```c
// Read buttons (pins are pulled high, so pressed = 0)
report.buttons = 0;
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_0) == GPIO_PIN_RESET) report.buttons |= (1 << 0); // X
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_1) == GPIO_PIN_RESET) report.buttons |= (1 << 1); // Y
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_2) == GPIO_PIN_RESET) report.buttons |= (1 << 2); // A
if (HAL_GPIO_ReadPin(GPIOA, GPIO_PIN_3) == GPIO_PIN_RESET) report.buttons |= (1 << 3); // B

// Send the 5-byte report to the PC
USBD_HID_SendReport(&hUsbDeviceFS, (uint8_t*)&report, sizeof(report));

HAL_Delay(10); // send every 10ms
```

---

## Flash and Test
![](WhatsApp%20Image%202026-06-07%20at%2022.11.52%20(1).jpeg)
1. Flash the firmware and plug in the board over USB.
2. Press **Win + R**, type `joy.cpl`, hit Enter.
3. Your device should appear. Click **Properties** to see the axes and buttons respond in real time.

If it doesn't show up, the most common causes are:
- `HID_MOUSE_REPORT_DESC_SIZE` doesn't match the actual descriptor byte count
- Windows is using a cached old descriptor — bump the PID and replug
- The `__attribute__((packed))` is missing from the report struct, causing padding bytes to shift everything

---

## Notes

### The 5-byte packet

Every 10ms, the STM32 sends this to the PC:

```
Byte 0:  Button states (1 bit per button, bits 0–3)
Byte 1:  Left Stick X   (-127 to 127)
Byte 2:  Left Stick Y   (-127 to 127)
Byte 3:  Right Stick X  (-127 to 127)
Byte 4:  Right Stick Y  (-127 to 127)
```

### The overflow bug (and how it's fixed)

The ADC gives values from 0 to 255. The USB HID format expects −127 to 127. The obvious fix is to subtract 127.

The problem: if the ADC reads 255, then `255 - 127 = 128`. But a signed 8-bit integer (`int8_t`) can only go up to 127. So 128 wraps around to −128, and the joystick axis suddenly jumps to the opposite extreme.

The fix is to cap the ADC value at 254 before doing the math.

### Windows caches the descriptor

Windows saves a copy of the HID descriptor tied to your device's Product ID (PID). If you change the descriptor but keep the same PID, Windows will try to use the old cached version and the device won't work.

**Fix:** Every time you change the descriptor, bump `#define USBD_PID` by 1 in `usbd_desc.c`.
