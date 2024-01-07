# the Matrix-inator
NOTE: not manufactured yet.  Awaiting testing.
A simple? breakout board that provides 32 matrix pins (16 row and 16 column) over SPI - using only 5 MCU pins!
Designed for QMK and requires a custom matrix.  Expects you to be using ROW2COL diode orientation.

Inate your keyboard matrix with the Matrix-inator!
![Perry...Perry the platypus?!](/matrixinator.jpg)

# usage
Very important to understand how QMK's matrix scanning works - here is my simplification of that, for ROW2COL diode orientation.
1. start matrix scan
2. get list of rows
3. for each row: raise row pin HIGH, then scan all col pins for HIGH; if HIGH, key pressed
4. report keys pressed via matrix object
5. end matrix scan

- custom_matrix=lite
- more details to come...

# why ROW2COL and not COL2ROW?
I have been working on integrating the Cyboard flex-pcb system into my latest build, and Cyboard is ROW2COL.

# pins
- ROW_CS - your trigger for sending bits to the Row 74HC595 shift registers
- COL_CS - your trigger for reading bits from the Col 74HC589 shift registers
- SPI Clock, MOSI, MISO - standard SPI pins
- 3V3, GND - expects +3.3v and a ground

# pcb notes
- m3 mounts
- the shift registers are only available as extended parts from JLC
- 2.54mm spacing on all pin footprints
- row/col pins designed for 4x 8-pin JST-XH 2.54 headers
- normal 2.54mm pin headers also work fine
- Two-layer board w. two ground planes tied w. vias, most signal routed on bottom plane
