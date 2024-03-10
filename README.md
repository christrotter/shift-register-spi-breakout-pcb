# Overview
> **NOTE**: 20240309: Order placed, but awaiting final testing.

A simple? breakout board that provides 32 matrix pins (_16 row and 16 column_) over SPI - using only 6 MCU pins!
Designed for and tested in QMK - requires a custom matrix.  Expects you to be using ROW2COL diode orientation.

This document includes:
- Understanding shift registers (_rather extensive_)
- Matrix-inator PCB notes
- Matrix-inator schematic detail
- Sample code

There's also a STEP file for testing fitment on your models.

![Perry...Perry the platypus?!](/images/matrix-inator-rev2.jpg)
*-inate your keyboard matrix with the Matrix-inator!*

# Usage

#### Connection
You can use the included 2x4 2.54mm pin header arrangement (_fits up to JST-XH headers_), or a [VIK-format 12-pin@0.5mm pitch FPC cable](https://github.com/sadekbaroudi/vik)
- _Power_: GND, 3.3V
- _Any GPIO_: Row CS, Col CS, Latch
- _SPI_: Clock/MISO/MOSI from the same SPI bus
- _rows and cols_: 2.54mm headers, do what you want here.

#### PCBA (PCB assembly)
If you want to order this, the gerbers are included in the latest [Github release](https://github.com/christrotter/shift-register-spi-breakout-pcb/releases/latest).

I seem to recall there being a silly part showing up that shouldn't have been (like a logo or something), but everything important came through.

Headers not part of the BOM, only the complicated and fiddly stuff.

#### Additonal parts you'll need
See the latest [Github release](https://github.com/christrotter/shift-register-spi-breakout-pcb/releases/latest) if you need all the parts.
- (_optional_) some flavour of 2.54mm-spaced headers
- 12-pin @ 0.5mm pitch FPC cable of sufficient length and a keyboard with a VIK port
  - (_or_) 2x 4-pin 2.54mm headers
- M3 hardware or some way of mounting the pcb to your model

#### Latch header notes
There's a 3-pin header labeled **Enable Latch**.  *It will need to have pins 2 & 3 connected for the latch to work.*  This is a development leftover.  Future revisions will not have it.

# Understanding shift registers
In order to understand shift registers, it's very important to understand how QMK's matrix scanning works.  And, of course, to understand that, you need to read their docs:
- https://docs.qmk.fm/#/understanding_qmk
- https://docs.qmk.fm/#/how_keyboards_work
- https://docs.qmk.fm/#/how_a_matrix_works

This doc is my simplification of the topic.

> **NOTE**: for the sake of illustration, you can use the idea that HIGH = "pin has voltage" and LOW = "pin is grounded". 

## Matrix scanning, tl;dr: 
1. Start matrix scan
2. Loop over each row pin.  on each loop, set this row pin HIGH (_all others LOW_) and then check **ALL** col pins for a reciprocal HIGH.
3. Build a `new_matrix_maybe` object with the result of each row iteration
4. Compare `live_matrix` with `new_matrix_maybe` - if differences, replace `live_matrix` with `new_matrix_maybe`
5. End matrix scan loop
(_and now QMK does everything else_)

![Matrix basic](/images/qmk-matrix-sketch.jpg)

> Have you ever looked at the QMK console messages for 'scan rate'?  Well that number is how many times per second the above matrix scan runs (_simplifying_).

> **NOTE**: I use the term 'row pin' in this document only to keep things simple - QMK and shift registers don't work the same way as if you had GPIOs configured as row pins.

### why ROW2COL and not COL2ROW?
I have been working on integrating the Cyboard flex-pcb system into my latest build, and Cyboard is ROW2COL.  I'm sure the design could be refactored to handle any config method.  Indeed, if you use `direct pin` (_where the switches connect to ground_) you can have an even simpler shift register and matrix design.

If you're used to COL2ROW, it's, for all intents and purposes, identical, just flip rows & cols around.

## Pseudo matrix scan code
This is (mostly) not real code.
```c
bool matrix_scan_custom(matrix_row_t current_matrix[]) {
    static matrix_row_t temp_matrix[ROWS_COUNT] = {0};
    for (uint8_t row = 0; row < (ROWS_COUNT); row++) {
        set_row_high(row_pin);              // this row pin is set HIGH; any keypresses will show as HIGH on col pins
        cycle_latch_pin(latch_pin);         // prepares the col shift registers for data shipping; shift register magic
        col_state = spi_receive(col_pin);   // ship col state from shift register into the MCU as col_state
        new_matrix_maybe[row] = col_state;  // update this row of the new_matrix_maybe object with the returned col state
    }
    bool matrix_has_changed = compare_matrix(new_matrix_maybe, live_matrix);
    if (matrix_has_changed) {
        update_active_matrix(new_matrix_maybe);
    }
    return matrix_has_changed;
}
```

## Shift register magic explained
The shift registers each have 8 "data" pins and a bunch of SPI/functionality pins.

There are two types of registers being used here:
- 74HC589: get the HIGH/LOW state of the data pins and return via SPI ([74HC589 datasheet](https://www.onsemi.com/pdf/datasheet/mc74hc589a-d.pdf))
- 74HC595: set the data pins to a HIGH/LOW state as per instructions sent via SPI ([74HC595 datasheet](https://www.onsemi.com/pdf/datasheet/mc74hc595a-d.pdf))

One accepts data from our code (595), the other provides data for our code (589).

- "Hay, row_shift_register, set a row pin to HIGH, please"
- "Okay, while that row is HIGH; col_shift_register, check all your pins - any HIGH?"
(repeat for every row pin)

Consider the matrix sketch above...instead of using GPIOs to send the signals to rows/cols, we use magical shift registers.

![Matrix using shift registers](/images/qmk-matrix-shift-register.jpg)

While there are some nuances for handling resets and the like - for keyboard applications you endlessly loop over rows, so no need for resets or anything fancy on the row_shift_registers.  But, the col_shift_registers handle ordered data...and if you handle the data incorrectly, your matrix results are sporadic at best.

So you need to understand a tiny bit about the 589 shift registers.

![Logic of 589](/images/74HC589-logic-diagram.jpg)

Simply put - the data we need is stored in the `data latch`.  To retrieve it via SPI, we need to tell the register to transfer it from the `data latch` into the `shift register`, which has a small amount of memory for serial operations.  Once in the next 'serial transfer' part of the clock cycling, the data in the `shift register` will be pulled out over SPI.

### Deep magicks
Each shift register has 8 bits (_1 byte_) - the data pins are digital, 0 or 1 - so you can understand what is going on in your digital matrix (HIGH/LOW) by polling your shift registers with sets/gets (_layman's terms_).

![Shift register magic](/images/qmk-set-row-high.jpg)

### for each row pin
> **NOTE**: CS pins on registers are normally HIGH.  You set them LOW to activate them.

1. Set `row_cs` LOW; the rows register is now the active SPI device
2. Send the data packet telling the register which pin to set HIGH (`spi_transmit`)
3. Set `row_cs` HIGH; we are done with the rows register, so de-select it
4. Set the `latch` HIGH; the 589 register transfers current pin data from latch to register
5. Set the `latch` LOW; register cannot do data transfer while latch is HIGH
6. Set the `col_cs` LOW - the register will start transferring data over SPI
7. Get the data packet that will tell us which col pin is HIGH (`spi_receive`)
8. Set the `col_cs` HIGH - we are done with the cols register, so de-select it

![Shift register flow](/images/qmk-register-flow.jpg)

## even more depth
...is something I won't get into.  Suffice to say that the registers function with a lot of clock timing and conditions available based on what is HIGH and what is LOW.

The matrix code, Kicad files, and part datasheets provide the rest of the puzzle.

A very important feature of the design are the 10k resistor arrays/networks that pull down the col pins to ground.  If you fail to implement this you will get super random/repeating responses.  How do I know this?

### it's complicated
The order of operations is determined by the timing diagram.  That sentence makes me sound like an elite who knows more than you.  _lies.  deception._

![Timing of 589](/images/74HC589-timing-diagram.jpg)

# Matrix-inator notes
![PCB overview](/images/pcb-overview.jpg)
![3D render - front](/images/pcb-3d-front.jpg)
![3D render - back](/images/pcb-3d-back.jpg)

## For the DIY-er
If you want to incorporate shift registers into your PCB, highly advise you do the following...
- Read the datasheets, carefully, twice, then thrice.
- Remember that pcb manufacturers have tolerances.  Abide by them and err on the generous side.
- The pull-down effect of the 10k resistor network is non-negotiable.  Floating signal is a guarantee, not a maybe.  Floating signal means your board does not work.
- Remember that decoupling caps need to be as close as possible to their device, and the current should flow through the cap trace into the device.

## Performance
It's worth noting that if you use the full 16 rows you will not have a very performant keyboard - scan rate drops to **1100**.  Still usable as a keyboard, but you won't have a ton of CPU overhead for other features.

## PCB details
- M3 mounts; 65mm center-to-center
- The shift registers are only available as extended parts from JLC (_extra fees_)
- 2.54mm spacing on all pin footprints
- Row/col pins designed for 4x 8-pin JST-XH 2.54 headers
- Normal 2.54mm pin headers also work fine
- Two-layer board; ground pour on both sides tied together with vias

## Pins
- ROW_CS - trigger to enable sending bits to the Row 74HC595 shift registers
- COL_CS - trigger enable reading bits from the Col 74HC589 shift registers
- LATCH_CS - trigger to prep all Col 74HC589 shift registers for data retrieval
- SPI: crucial that you use SPI pins on the same SPI bus
  - Clock: all SPI devices need this
  - MOSI: ROW shift registers are sent data from the MCU
  - MISO: COL shift registers return data to the MCU
- 3V3, GND - expects +3.3v and a ground

## Schematics
![Schematic overview](/images/sch-overview.jpg)
![Schematic - row registers detail](/images/sch-row-registers.jpg)
![Schematic - col registers detail](/images/sch-col-registers.jpg)

# Sample code
This will undoubtedly get improved, but this is enough to get you off the ground and using the Matrix-inator.

## info.json
Bits you'll need in `keyboard/info.json`:
```json
    "diode_direction": "ROW2COL",
    "matrix_pins": {
        "custom_lite": true,
        "rows": ["NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN"],
        "cols": ["NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN", "NO_PIN"]
      },
    "layouts": {
        "LAYOUT": {
            "layout": [
                {"matrix": [5, 0], "x": 0, "y": 0},
                {"matrix": [5, 1], "x": 1, "y": 0},
                etc
        }
    },
```

## config.h
Bits you'll need in `keyboard/config.h`:

For figuring out which SPI GPIO to use, section 1.4.3 here: https://datasheets.raspberrypi.com/rp2040/rp2040-datasheet.pdf
```c
// SPI configuration
#define SPI_MATRIX_DIVISOR 16
#define SPI_MODE 0
#define SPI_DRIVER SPID1 // you might change this
// GPIO config for main SPI config needs to match up with the SPI bus you are using
#define SPI_SCK_PIN  GPxx // e.g. SPI0 SCK
#define SPI_MOSI_PIN GPxx // e.g. SPI0 TX (Master Out, Slave In)
#define SPI_MISO_PIN GPxx // e.g. SPI0 RX (Master In, Slave Out)
// GPIO config for CS/latch pins can be any GPIO
#define SPI_MATRIX_LATCH_PIN GPxx
#define SPI_MATRIX_CHIP_SELECT_PIN_ROWS GPxx
#define SPI_MATRIX_CHIP_SELECT_PIN_COLS GPxx
// custom matrix config
#define MATRIX_COLS_SHIFT_REGISTER_COUNT 2
#define MATRIX_ROWS_SHIFT_REGISTER_COUNT 2
#define ROWS_COUNT 12 // this can be replaced w. array_size or something?
#define ROWS { \
    0b0000000000000001, \
    0b0000000000000010, \
    0b0000000000000100, \
    0b0000000000001000, \
    0b0000000000010000, \
    0b0000000000100000, \
    0b0000000001000000, \
    0b0000000010000000, \
    0b0000000100000000, \
    0b0000001000000000, \
    0b0000010000000000, \
    0b0000100000000000, \
} // this makes up the magic of the 'message' we send to the 595 register
```

## rules.mk
For `keyboard/rules.mk`:
```
SRC += matrix.c
```

## halconf.h
Bits you'll need in `keyboard/halconf.h`:

```c
#define HAL_USE_SPI TRUE
#define SPI_USE_WAIT TRUE
#define SPI_SELECT_MODE SPI_SELECT_MODE_PAD
```

## mcuconf.h
Bits you'll need in `keyboard/mcuconf.h`:

You might use SPI0?
```c
#undef RP_SPI_USE_SPI1
#define RP_SPI_USE_SPI1 TRUE
```

## matrix.c
This file goes in: `keyboard/keymap/map/` (_I think it might actually be meant to go in the keymap dir?_)

Actual working code.  (_mostly tested as of this writing_)
```c
// Copyright 2018-2022 Nick Brassel (@tzarc)
// Copyright 2020-2023 alin m elena (@alinelena, @drFaustroll)
// Copyright 2023 Stefan Kerkmann (@karlk90)
// Copyright 2023 (@burkfers)
// SPDX-License-Identifier: GPL-3.0-or-later

#include "quantum.h"
#include "spi_master.h"

static const uint16_t row_values[ROWS_COUNT] = ROWS;
static const pin_t latch_pin = SPI_MATRIX_LATCH_PIN;

void matrix_init_custom(void) {
    setPinOutput(SPI_MATRIX_CHIP_SELECT_PIN_COLS);
    writePinHigh(SPI_MATRIX_CHIP_SELECT_PIN_COLS);
    setPinOutput(SPI_MATRIX_CHIP_SELECT_PIN_ROWS);
    writePinHigh(SPI_MATRIX_CHIP_SELECT_PIN_ROWS);
    setPinOutput(latch_pin);
    writePinLow(latch_pin);
    spi_init();
}

static inline void write_to_rows(uint16_t value) {
    uint8_t message[2] = {(uint8_t)(value & 0xFF), (value >> 8) & 0xFF}; // cut 0xABCD into {0xAB, 0xCD}
    spi_start(SPI_MATRIX_CHIP_SELECT_PIN_ROWS, true, SPI_MODE, SPI_MATRIX_DIVISOR);
    spi_transmit(message, 2);
    spi_stop();
}

bool matrix_scan_custom(matrix_row_t current_matrix[]) {
    static matrix_row_t temp_matrix[ROWS_COUNT] = {0};
    
    for (uint8_t row = 0; row < (ROWS_COUNT); row++) {
        uint8_t temp_col_receive[MATRIX_COLS_SHIFT_REGISTER_COUNT] = {0};
        uint16_t temp_col_state;

        write_to_rows(row_values[row]);

        // get the shift registers to move data from latch to register
        write_and_wait_for_pin(latch_pin, 1);
        write_and_wait_for_pin(latch_pin, 0);

        // read the cols shift register contents over serial
        spi_start(SPI_MATRIX_CHIP_SELECT_PIN_COLS, true, SPI_MODE, SPI_MATRIX_DIVISOR);
        spi_receive((uint8_t*)temp_col_receive, MATRIX_COLS_SHIFT_REGISTER_COUNT);
        spi_stop();

        temp_col_state = temp_col_receive[0] | (temp_col_receive[1] << 8);
        temp_matrix[row] = temp_col_state;
    }
    bool matrix_has_changed = memcmp(current_matrix, temp_matrix, sizeof(temp_matrix)) != 0;
    if (matrix_has_changed) {
        memcpy(current_matrix, temp_matrix, sizeof(temp_matrix));
    }
    return matrix_has_changed;
}
```
