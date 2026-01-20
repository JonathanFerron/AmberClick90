# AmberClick90 (AVRDU.Enc.Mat)


## 1. Overview

**AmberClick90 (AVRDU.Enc.Mat)** is a custom 90‑key USB keyboard built around the **AVR64DU32** microcontroller.

Key characteristics:

* 90 keys (10 rows × 9 columns matrix, but really 5 rows x 20 columns physically)
* Matias Click switches
* One rotary encoder (ALPS EC11) with push switch (spst normally open)
* USB Full‑Speed device (keyboard + bootloader)
* Amber LEDs (1 power LED + 2 PWM indicator LEDs)
* Minimalist black PCB, minimal grey 3D‑printed case
* Firmware-driven bootloader entry (no dedicated boot pin)
* toolchain: avr-gcc compiLer, updi flashing software = avrdude with serialupdi, usb flashing software custom.

This document consolidates **the current decisions** made throughout the design discussion.

in general we want to borrow from qmk, dxcore, mcc, lufa the ideas that still apply well to the avrdu, and enhance or adapt where it makes sense.

---

## 2. MCU Selection

**Microcontroller:** AVR64DU32

Reasons:

* Native USB Full‑Speed (no external USB PHY)
* Adequate flash/RAM for keyboard firmware
* Sufficient GPIO count for 90‑key matrix + encoder + LEDs
* UPDI programming and recovery
Supports USB HID natively
No external USB-to-serial chip required
32 pins in total: 23 usable gpio

USB operation:

* USB Full‑Speed works **without an external oscillator**
* Internal oscillator + USB clock recovery are sufficient

AVR64DU32
24 MHz clock
64 KB Flash
8 KB SRAM

Power & voltage
Operates at 3.3 V (we will feed it 5V)
USB-compatible (internal USB regulator)

---

## 3. Power Configuration

* USB bus power (5 V from host)
* Datasheet **Power Configuration 5b**
* VBUS → VDD (5 V domain)
* Internal regulator provides VUSB (3.3 V USB core)

Decoupling:

* Standard bulk capacitor on VDD (near USB connector, mind the 10uF max from USB specs)
* Local decoupling near MCU pins
* VUSB pin decoupled to ground only (no external load)

---

## 4. USB Interface

* USB Full‑Speed device
* USB Mini‑B connector
* Short, straight D+ / D− routing
* No external oscillator required
* Bare-metal approach

Bootloader exposes USB interface (Vendor defined class). Keyboard app exposes USB HID interface.

---

## 5. Key Matrix

* **10 rows × 9 columns = 90 keys**
* Matias Click switches
* Diode per switch (standard keyboard practice)

### Matrix GPIO allocation

* **Columns (9):**

  * PA0–PA3
  * PD1–PD5

* **Rows (10):**

  * PC3
  * PD0, PD6, PD7
  * PF0–PF5

Grouping rationale:

* Ports grouped to reduce register access
* Faster scanning
* Cleaner routing
* Lower firmware complexity

---

## 6. Rotary Encoder

* **Model:** ALPS EC11

* **Signals:**

  * Encoder A → PA6
  * Encoder B → PA7

* **Push switch:**

  * Wired to **RESET pin**
  * Acts purely as a reset button

Encoder signals are handled in firmware (quadrature decoding).

---

## 7. LEDs

### 7.1 Power LED

* Amber, through‑hole (3 mm diffused)
* Connected to **VDD (5 V)**
* Fixed resistor (no GPIO, no PWM)
* Brightness chosen to be non‑annoying in a dark room

### 7.2 Indicator LEDs (2×)

* Amber, through‑hole, diffused (3 mm)
* Driven by MCU GPIO with **PWM**
* Firmware‑controlled brightness
* maximum brightness chosen with 100% duty cycle to be visible in a brightly lit room, indoor

GPIO assignment:

* Indicator LED A → PA4 (PWM)
* Indicator LED B → PA5 (PWM)

Target current:

* ~5 mA per LED for indoor visibility

Brightness control:

* Default brightness (PWM settings) stored in flash (aim for visible but not blinding in a room with regular indoor lighting on a cloudy day)
* Runtime brightness stored in SRAM

---

## 8. GPIO Usage Summary

Total usable GPIOs: **23**

| Function       | Pins   |
| -------------- | ------ |
| Matrix rows    | 10     |
| Matrix columns | 9      |
| Encoder A/B    | 2      |
| PWM LEDs       | 2      |
| **Total**      | **23** |

RESET and UPDI pins are not counted as GPIO.

---

## 9. Bootloader Strategy

### 9.1 Entry Method

* **Firmware key combination only** (on layer 2, out of 3 layers (layer 0, 1 and 2))
* No dedicated boot GPIO
* Encoder push switch is **RESET only**

Key combo:

* **Layer 1 (key on layer 0 to get to layer 1)+ Layer 2 (key on layer 1)+ BootLoader (special keycode on layer 2 in keymap)**

---

### 9.2 Bootloader Request Mechanism

* Uses a **General Purpose Register (GPR)** to signal bootloader entry
* Magic value written to the chosen GPR register
* Firmware triggers a **watchdog reset** or a direçt software reset

Conditions:

* Does not work with Power On and BOD resets

---

### 9.3 Bootloader Flow

1. Keyboard firmware detects key combo
2. Magic value written to reserved GPR
3. reset triggered
4. Bootloader starts
5. Bootloader checks GPR very early
6. If magic matches → stay in bootloader
7. Otherwise → jump to application

Bootloader clears the register immediately after detection.

---

## 10. Programming & Recovery

* **Primary flashing:** USB bootloader
* **Recovery flashing:** UPDI

UPDI header:

* Placed at PCB edge
* Mandatory for unbrick / development
* 3 pins, see avr-du datasheet

---

## 11. LED Electrical Notes

* IO pin max current (50 mA): absolute maximum sourcing/sinking

Design practice:

* LED current limited with resistor
* GPIO LED currents can easily be kept well below limits (~5 mA) and be sufficiently bright

R = (VCC – Vf) / I

PWM control in firmware for dimming

---

## 12. Firmware Size Expectations

Estimated keyboard firmware (3 layers + macros + encoder):

* ~15–25 kB flash

Bootloader size:

* ~2–6 kB (≈3–10% of flash)

Plenty of margin on AVR64DU32. Likely to reserve 8kb for bootcode.

---

## 13. Physical Layout Notes

* USB connector on top edge, aligned with Fn key
* Encoder located top‑left
* Three LEDs arranged horizontally near top edge with power led in the middle
* Reset via encoder push switch
* UPDI header at one of the PCB edge: top edge may make 3D case design easier? or else use UPDI header location that makes routing nicer.


black 2 layer pcb
gray case

---

## 14. Branding

* Name: **AmberClick90**
* Trim: **(AVRDU.Enc.Mat)**
* Aesthetic:

  * Minimalist
  * Technical / instrument‑like
  * Monochrome silkscreen

Logo to be simple and silkscreen‑friendly.

---

## 15. Status

This document represents the **current design direction** and can be used as:

* A handoff artifact for future chats
* Reference during schematic and PCB layout
* Firmware architecture guide

Use 6kro only.

we want 2 layers with real keycodes plus a 3rd layer for led brightness adjustment, access to bootloader, etc.

we want to be able to implement small macros (eg typing one key will issue ctrl+c to the PC)

need to setup usb hid descriptor

use dxcore to pull info on pin mapping, clock and fuse settings used, startup code and linker scripts, and how it does updi flashing with avrdude

use bare-metal approach + usb stack. use avr-gcc, direct register access with light hal.  we will need to write startup + linker (can lookup the dxcore or microchip mcc templates), a small hardware abstraction layer (HAL) for gpio, timers, matrix scanning. lufa and mcc code show how to handle usb enumeration and hid keyboard reports.

typical small HAL:
/src
  main.c
  usb_descriptors.c
  keyboard.c
  matrix.c
  hal_gpio.c
  hal_timer.c
  config.h

use updi programmer to flash that at this point

component list:
mcu
Use internal oscillator (no crystal needed)
usb mini-b connector, through-hole (can do usb-fs)
updi connector (3 pins) / programming header
usb d+ and d- series resistors
bulk cap near usb connector (mind the 10uF total max under usb std)
decoupling caps at each vdd and vusb pin
switches with their diodes (eg 1n4148)
encoder (includes reset button)
3 leds with their current limiting resistors
fuse near usb connector (500 mA ?)
maybe ess protection for usb: low capacitance usb esd diode array


Route D+ / D− as a short, matched pair, no stubs, no vias if possible.

do not ever disable updi
reasearch whether to use a sleep + wake up using interupts approach (to save power)

we will use a vendor class bootloader:
Device enumerates as usb vendor class
libusb required on pc
Firmware update via:
Custom tool

Memory layout (important for keyboards):

Typical pattern:
|----------------------|  Flash start
| Bootloader           |
| (8 KB)             |
|----------------------|
| Application firmware |
| (keyboard logic)     |
|----------------------|  Flash end

Bootloader:
Own USB stack
Minimal functionality

App firmware:
Full HID stack
Keyboard logic

store keymap strictly in flash

Typical keyboard firmware pattern (best practice)
Flash:
  - Default keymaps
  - Macro definitions
  - USB descriptors

SRAM:
  - Active layer state
  - Matrix state
  - Debounce counters
  - Macro execution state


On AVR Dx / DU:
Flash is memory-mapped
You can usually read it like normal const data

Put “what the keyboard is” in flash.
Put “what the keyboard is doing right now” in SRAM.
Keymaps define what the keyboard is.
State defines what it’s doing.

encoder could do repetitive alt tab. ec11 with little clicks.

Encoder decoding logic
Typical quadrature decoder:
State machine with 4 states
Table-driven or bitwise decode

Alt-Tab encoder
Typical macro sequence:
Press Alt
Tap Tab
Release Tab
Release Alt

Encoder sampling
Typically sampled at:
Matrix scan rate (1–2 kHz)
Or via pin-change interrupt
Both approaches are fine.

Scan encoders in the same loop as matrix scanning
Or:
Sample encoder pins each scan
Decode transitions
Encoders do not interfere with:
Debouncing
Layer logic

encoder design pattern:

Recommended approach:
// Flash
const encoder_action_t encoder_map[] = {
  [ENCODER_1] = { .cw = MACRO_VOL_UP, .ccw = MACRO_VOL_DOWN },
  [ENCODER_2] = { .cw = MACRO_ALT_TAB, .ccw = MACRO_ALT_SHIFT_TAB }
};

// Flash
const macro_t macros[] = {
  MACRO_VOL_UP,
  MACRO_VOL_DOWN,
  MACRO_ALT_TAB,
  MACRO_ALT_SHIFT_TAB
};

Runtime:
Just track encoder deltas
Dispatch macro IDs

Encoder bounce
Mechanical encoders bounce more than keys
Use:
State-table decoder (preferred)
Or light debouncing (~1–2 ms)
⚠ Alt-Tab behavior
Some OSes require:
Alt held across multiple tabs
You may want:
Encoder “session” logic (hold Alt until pause)
Still trivial to implement
⚠ Power-on state
Initialize encoder state early
Avoid phantom rotations on boot

ask llm to Sketch a robust quadrature decoder for encoders,
Design an Alt-Tab “smart encoder” behavior,
Suggest GPIO + pull-up configuration for encoders,
Help integrate encoders cleanly into your matrix scan loop,
Show firmware logic for decoding encoders in a matrix,

The AVR64DU32 does NOT require an external crystal for USB FS.
It uses:
Internal 24 MHz oscillator
USB Clock Recovery / trimming (SOF-based)

Typical safe design target:
→ ≤ 5–10 mA per pin

For the led resistors, target to be able to clearly see the led in a well lighted room (indoor). That way, when it's bright we would set the PWM duty cycle to 100%, and then we would have a special macro on the 3rd layer to adjust the duty cycle for cases when the light inside is more dim.

Store brightness level in Flash defaults + runtime variable in ram
Clamp brightness range:
Min: ~5–10%
Max: 100%

Default brightness → Flash
Compiled into firmware
Fixed at build time
Used on every reset / power-up
Example:
#define LED_BRIGHTNESS_DEFAULT 128   // 0–255

Runtime brightness → RAM
Modified by macros / key combos
Fast access
No wear concerns
Reset on power cycle
Example:
uint8_t led_brightness;   // in SRAM
At startup:
led_brightness = LED_BRIGHTNESS_DEFAULT;

Estimate average and maximum current draw from the host usb cable.

PWM resolution
Avoid interaction with matrix scan (typically ~1 kHz)
On AVR DU:
Use TCA 
One timer can drive both leds

power led (connected to vbus=vdd):
to dim a non-GPIO power LED
Fixed resistor only
Choose a large resistor
LED is always dim but visible

For a power LED, typical design choice is:
Make it intentionally dim and fixed.

for the power led brightness then, use a fixed resistor, and decide of the resistor value by placing it in a very dim environment and making sure the led is not annoyingly bright in that environment, which should still make it visible (though dim) in a brighly lit room.

Using this method, you’ll almost certainly land in:
0.2–0.8 mA range
Often closer to 0.3–0.5 mA for modern LEDs

Put the resistor close to the LED, not the MCU
Reasons:
Reduces EMI pickup
Makes rework easier
Prevents accidental shorting during probing

use 3mm TH amber leds diffused

3 mm LEDs have:
Smaller emitting area
Lower perceived glare
Much easier to make them:
Subtle in the dark
Non-distracting in peripheral vision

Diffused amber:
Looks softer
Hides PWM artifacts
Better in low light

Viewing angle
Indicators with ~60–100° viewing angle are ideal; they don’t form sharp bright spots but give a nice gentle glow.

Part packages
Standard T-1 (3 mm) leads make it easy to mount LEDs through PCB with radial holes.

Quick resistor reminder
Targeting ~5 mA on 3.3 V with amber LED (≈2.0 V forward drop):
R ≈ (3.3 − 2.0) / 0.005 ≈ 260 Ω → use nearby standard: 270 Ω
This gives good visible brightness indoors and plenty of headroom for PWM dimming.

see pinout diagram for detailed mapping for each row and column

also design a typographic logo

Firmware identity alignment
Small but important:
USB device name: AmberClick90 or KeyDUBL / KeyDUApp
USB manufacturer: your name or handle
HID strings match PCB name
This avoids “generic HID keyboard” feel.

next step is to work further on logo and create kb schematic in kicad:
MCU
USB
Matrix
Encoder
LEDs
UPDI + Reset


Name nets early
ROW_0 … ROW_8
COL_0 … COL_9
ENC_A / ENC_B
LED_STATUS / LED_LAYER

A keyboard scan loop typically does this:
Set one column active (output low)
Read all row pins as inputs (pull up enabled)
Store results
Move to next column
Repeat
So for each scan step, the MCU:
Writes to exactly one GPIO pin
Reads a group of GPIO pins nearly simultaneously


On AVR (including AVR DU series), GPIO is controlled per-port, not per-pin.
Example:
Port A has a single PORTA.OUT register
Port A inputs are read via PORTA.IN
If rows are grouped on one port
You can read all rows at once:
uint8_t rows = PORTA.IN & ROW_MASK;

If rows are scattered across ports
You must do:
rowsA = PORTA.IN & MASKA;
rowsB = PORTB.IN & MASKB;
rowsC = PORTC.IN & MASKC;

If you ever:
Use pin-change interrupts
Or want to optimize wake-from-sleep
Grouping rows (or encoder pins) on one port means:
One interrupt source
One ISR path
One mask
This makes firmware simpler and more robust.

ask llm to:
Show a minimal scan loop for your exact pin map
Compare row-driven vs column-driven scanning for your layout
Help you decide pull-ups vs pull-downs for the matrix
Walk through diode orientation trade-offs


We will use power config 5b (5v coming from bus, ie from usb host computer), see datasheet section 6.4. We’ll go with 10 rows and 9 columns.

Bootloader / firmware branding (optional but nice)
Examples:
USB device name: AmberClick90
Bootloader string: KeyDUBL
Version tag: AC90-r1

How to get the mcu to enter a bootloader mode upon reading a special key combination on the Fn layer when in matrix scanning mode? you do it entirely in firmware, by deliberately resetting into the bootloader after setting a persistent “boot request” flag.

Detect a special key combination while scanning the matrix
Store a bootloader request flag somewhere that survives reset
Perform a software reset
Early boot code checks that flag
If set → stay in bootloader
If not → run normal keyboard firmware

Where to store the “bootloader request” flag
You have three viable options on AVR-DU:
Option A — GP register (best if supported)
Some AVR families preserve a general-purpose register across reset.
If available:
Fast
No flash wear
No EEPROM access

4. Firmware flow (step by step)
4.1 Detect the key combination
In your matrix scanning logic / after applying active layer logic and keymap lookup:
if (key_is_pressed(KC_BOOT)) {
    request_bootloader();
}
Debounce normally — you don’t want accidental entry.

4.2 Set the bootloader request flag
see datasheet, set magic number in a chosen gpr, the below example uses eeprom: chg to gpr approach

#define BOOT_FLAG_ADDR  0x00
#define BOOT_MAGIC     0x42

void request_bootloader(void) {
    eeprom_write_byte((uint8_t*)BOOT_FLAG_ADDR, BOOT_MAGIC);
    _delay_ms(10);
    software_reset();
}

4.3 Trigger a reset
Use either:
Watchdog reset (preferred):
void software_reset(void) {
    wdt_enable(WDTO_15MS);
    while (1) { }
}
This behaves like a real reset and is bootloader-friendly.

5. Boot-time logic (critical)
5.1 In your main firmware (early)
Very early in main() or before USB init:
check for magic number
if present, clear the magic number and jump_to_bootloader();


5.2 How “jump to bootloader” actually works
This depends on how you install the bootloader:

Bootloader lives in BOOT section
You explicitly jump:
#define BOOTLOADER_ADDR  0x7000  // example only

void jump_to_bootloader(void) {
    cli();
    ((void (*)(void))BOOTLOADER_ADDR)();
}
This assumes:
Interrupts off
USB not initialized yet
Stack in a clean state
⚠ This is fragile if done late — early is key

USB behavior after entering bootloader
From the host’s point of view:
Keyboard disconnects
New USB device appears (Vendor class)
You flash firmware
Device resets
Keyboard reappears

The key combo is detected in normal keyboard mode

Firmware flow with your design

Normal operation
Power on / Reset
→ Bootloader starts
→ No GPR flag
→ Jumps immediately to keyboard firmware

Special keycode bootloader entry
KC_BOOT detected
→ Firmware writes GPR boot flag
→ Firmware triggers watchdog reset
→ Bootloader sees flag
→ Bootloader stays active (USB Vendor)

Manual reset (encoder switch)
Encoder switch pressed
→ RESET asserted
→ Bootloader starts
→ No flag
→ Jumps to keyboard firmware

look up correct reset button wiring in datasheet

Using a General Purpose Register (GPR): when it works
The idea is:
Firmware detects KC_BOOT
Firmware writes a magic value into a CPU register
Firmware triggers a reset
Early startup / bootloader checks that register
If magic present → stay in bootloader
This only works reliably if:
The reset is a watchdog reset (or regular software reset ?)
The register you choose is not cleared by reset
The compiler/startup code does not clobber it before you check


Historically, AVR projects use r2 or r3 for this purpose because:
r0, r1 are used by ABI / zero register conventions
r1 is often forced to zero very early
r2–r17 are “general” but may be touched by startup code
Best practice:
Pick one register
Treat it as reserved
Check it before C runtime init
Example magic:
#define BOOT_MAGIC  0x42


Reset type matters (this is critical)
✔ Watchdog reset — OK
GPRs are preserved
This is what you must use
wdt_enable(WDTO_15MS);
while (1) {}

Where the check must happen
The check must happen before:
.data initialization
.bss clearing
USB init
Stack setup (ideally)
That usually means:
In bootloader startup code, or
In a custom reset handler, not main()
If you check too late, the compiler may already have reused the register.


Example flow (conceptual)
In keyboard firmware (normal mode)
void enter_bootloader_via_keys(void) {
    asm volatile (
        "ldi r2, %0\n"
        :
        : "M" (BOOT_MAGIC)
    );
    wdt_enable(WDTO_15MS);
    while (1) {}
}

In bootloader early init (pseudocode)
if (r2 == BOOT_MAGIC) {
    r2 = 0;
    stay_in_bootloader();
} else {
    jump_to_application();
}


Guarding against false positives
Because power-on reset leaves registers undefined:
Your bootloader must:
Compare against a specific magic value
Clear the register immediately after detecting it
Probability of random match is extremely low (1/256), but still handled cleanly. If we want to reduce this probability to 1 in 64000, use r3 register as well and use ~0x42 there, checking both (r2 << 1 | r3 == magicword, or something similar)