# STM32 USB Microphone

This repository demonstrates how to build a DIY USB microphone using an **STM32F411CEU (Black Pill)** and an **INMP441 I2S MEMS microphone**.

Although this project is based on the STM32F411CEU, it can be adapted to any STM32 microcontroller that supports both **I2S** and **USB Audio Class (UAC)**.

If your MCU does not have an I2S peripheral, you can use an analog microphone module such as the **MAX9814** with an ADC instead. However, USB Audio Class support is still required because the device will enumerate as a USB microphone.

---

# CubeMX Configuration

## 1. Create a new project

Create a new STM32CubeMX project and select a microcontroller that supports both **I2S** and **USB Device**.

---

## 2. Configure the system clock

- Set **HSE** to **Crystal/Ceramic Resonator**.
- Configure the clock tree so that the **USB clock is exactly 48 MHz**.

> USB Full-Speed requires an accurate 48 MHz clock. If the frequency is incorrect, the USB device may not enumerate correctly.

*Image here*

---

## 3. Configure I2S

Configure the I2S peripheral as:

- Half-Duplex Master
- Master Receive
- Data Format: **24-bit data on 32-bit frame**
- Audio Frequency: **48 kHz**

The actual sampling frequency depends on the generated I2S clock. After configuring the PLL, verify the real sampling rate and minimize the frequency error.

For this project, an I2S clock of **64.5 MHz** results in approximately **-0.01% sampling error**, which is sufficiently accurate.

*Image here*

---

## 4. Configure DMA

Add a DMA request for **SPI_RX** with the following settings:

- Mode: Circular
- Peripheral Data Width: Half Word (16 bits)
- Memory Data Width: Word (32 bits)

*Image here*

---

## 5. Configure GPIO

Set all GPIO pins used by I2S to **High** or **Very High** speed.

*Image here*

---

## 6. Configure USB Device

Navigate to

```
Connectivity
    → USB_OTG_FS
        → Device Only
```

Then open

```
Middleware
    → USB_Device
```

Select **Audio Device Class**.

Finally, set

```
USBD_AUDIO_FREQ = 48000
```

to match the I2S sampling rate.

*Image here*

---

After completing these steps, **do not regenerate the project unless necessary**.

This project requires modifications to several CubeMX-generated files. Regenerating the project may overwrite those changes.

---

# USB Audio Descriptor Configuration

By default, STM32CubeMX generates descriptors for a **USB speaker (Audio OUT)**.

To make the board appear as a **USB microphone (Audio IN)**, several descriptors must be modified.

The descriptors used in this project can be found in:

```
...
```

## 1.

Follow the directory below:

```
Middlewares/
    ST/
        STM32_USB_Device_Library/
            Class/
                AUDIO/
```

Open:

```
usbd_audio.c
usbd_audio.h
```

---

## 2.

Modify the following macros and descriptor definitions.

*Code here*

---

## 3.

Modify the USB audio callback in

```
main.c
```

*Code here*

---

## 4.

Make sure the initialization order is correct.

DMA must be initialized before any peripheral that depends on it, and all peripherals required by USB audio should be initialized before the USB device starts.

Example:

*Code here*

---

## 5.

Build the project and connect the board to your PC.

Open **Device Manager** and verify that a USB audio recording device appears.

Then open any recording application (Windows Voice Recorder, Audacity, etc.) and confirm that audio is being captured correctly.

---

# How It Works

The INMP441 continuously outputs 24-bit PCM audio samples through the I2S interface.

DMA transfers these samples directly into memory without CPU intervention. Whenever half or the entire DMA buffer is filled, an interrupt is generated.

Inside the DMA callback:

- Extract the valid 24-bit microphone sample.
- Convert it into a signed 16-bit PCM sample.
- Store the converted samples into the USB transmission buffer.

Whenever the host requests new audio data, the USB Audio Class driver simply transmits the prepared 16-bit PCM buffer.

The complete data flow is:

```
INMP441
    ↓
I2S Peripheral
    ↓
DMA (Circular Mode)
    ↓
DMA Interrupt
    ↓
24-bit → 16-bit PCM Conversion
    ↓
USB Audio Buffer
    ↓
USB Audio Class
    ↓
Computer
```

Because DMA handles all data transfers, the CPU only performs sample conversion, making the implementation efficient and lightweight.

---

# Notes

- 48 kHz / 16-bit PCM is the most common format for USB microphones and provides excellent compatibility across operating systems.

- Higher sampling rates (96 kHz, 192 kHz) or higher bit depths require significantly more RAM and USB bandwidth.

- The INMP441 internally produces 24-bit samples, but USB microphones commonly transmit 16-bit PCM to reduce bandwidth and improve compatibility.

- If you use another STM32 MCU, only the clock configuration, pin assignments, and USB descriptors may need to be adjusted. The overall architecture remains the same.
