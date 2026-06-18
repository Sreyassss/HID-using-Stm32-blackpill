# STM32 Black Pill USB HID Gamepad

This repository contains the firmware and outline for a custom USB HID device that functions as a gamepad/joystick. 

I built this by editing a generated mouse report descriptor and modifying it for my needs. Fair warning: the USB protocol is incredibly stubborn, finicky, and strict. When it fails, it gives you almost no idea why. Because of this, I relied heavily on a couple of free debugging tools to set up the HID properly. 

## 🛠 Hardware Setup
* **Microcontroller:** STM32 Black Pill (STM32F401CCU6 chip).
* **Inputs:** Basic potentiometer thumbsticks and 5 push buttons. 
* *Note:* You are completely free to add as many buttons and hardware pieces as you like! Just remember to update the HID descriptor accordingly.
* **Pins:** GPIO pins for the buttons are internally pulled up.

## ⚙️ How the Code Works

### 1. Data Collection (ADC + DMA)
This was my first time really playing with the ADC channels for this kind of application. I used a hardware timer to cut down the sampling rate and DMA (Direct Memory Access) to store the data. 

**Why DMA?** If your controller does not provide data the exact moment Windows asks for it, Windows will immediately label the device as "malfunctioned" and disconnect it. Our device must *always* be ready to provide data. DMA combined with interrupts ensures we never miss a beat.

### 2. The USB HID Descriptor
Before sending anything over USB, we first take the data from the thumbsticks and buttons and verify them via a debug menu. Then, we construct the USB HID descriptor (located in `usb_hid.c`). 

**This is the step where your board is most likely to fail.**
The report descriptor has a very specific and strict way of being written. You can refer to online sources, but I highly recommend using the **HID Descriptor Tool** (free) to look into the byte information. 

Once you write your report, make sure to count the exact number of bits/bytes in them and update the report size in `usb_hid.h`.

### 3. Structuring and Padding Data
After the descriptor is done, we have to package our data into a single `struct` file. 
* The analog axes are sent as individual variables.
* The buttons only have 2 states, so they can be represented as individual bits and sent together as a single 5-bit piece of data.

**Note :**
If you are modifying the descriptor, u would have notice the report count being adde dthat is actual padding so the tht final data we sent is actual 8bit and not 5 bit just somthing random i noticed

### 4. Sending the Report
After constructing the struct, we use a send report function to shoot the data over to Windows. Logically, it's very simple. But again, if there is a single error in formatting or sizing, Windows won't even detect the USB.

## 🐛 Debugging & Testing
Because Windows fails silently, I used another free USB analyzer tool. This tool tells you the actual state of the USB connection. If the device is failing to be detected, it will create an error dump which you can view to debug the issue. 
* *Example:* If you give the wrong report description size in `usb_hid.h`, this tool will catch it and generate an error dump.

**Final Testing:**
To verify the controller is working, press `Win + R` and type `joy.cpl`. This opens the native Windows Gamepad checker so you can see if all axes and buttons are tracking properly.

## ⚠️ Limitations & The Xbox Controller Issue
Right now, this controller works perfectly for emulators and older games (which use DirectInput). 

However, most modern PC games look exclusively for an Xbox Controller descriptor (XInput). Frankly, it is not worth the hassle to code an XInput descriptor manually from scratch. There are many existing GitHub repositories that provide XInput firmware which can be flashed directly to the board if you need modern game support.
