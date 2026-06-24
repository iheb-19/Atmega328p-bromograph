# ATmega328P Bromograph Controller

Embedded C application for the ATmega328P microcontroller that simulates the control logic of a UV bromograph for photolithography. Implements a 4-state FSM with configurable lamp intensity (PWM) and exposure duration, a warm-up phase with dual simultaneous PWM channels, and an emergency stop sequence with visual feedback.

---

## Features

- **4 UV lamp intensity levels** (20%, 40%, 60%, 80%) selectable before exposure
- **4 exposure durations** (2, 4, 8, 16 seconds) selectable before exposure
- **3-second warm-up phase** with simultaneous independent PWM on L4 and L5
- **Software PWM** driving lamp simulation (L4) at configurable duty cycles
- **Emergency stop** (B3) with 2-second L5 flash feedback before reset
- **Binary LED display** of intensity (L0–L1) and duration (L2–L3)
- **4-state FSM**: MODALITA → RISCALDAMENTO → ESPOSIZIONE → STOP

---

## Intensity & Duration Settings

| Intensity Level | Duty Cycle | L1  | L0  |
|----------------|------------|-----|-----|
| 20%            | 20%        | OFF | OFF |
| 40%            | 40%        | OFF | ON  |
| 60%            | 60%        | ON  | OFF |
| 80%            | 80%        | ON  | ON  |

| Duration | Seconds | L3  | L2  |
|----------|---------|-----|-----|
| 2s       | 2       | OFF | OFF |
| 4s       | 4       | OFF | ON  |
| 8s       | 8       | ON  | OFF |
| 16s      | 16      | ON  | ON  |

---

## Hardware Mapping

| Pin          | Function                              |
|--------------|---------------------------------------|
| PORTD2 (B2)  | Start exposure                        |
| PORTD3 (B3)  | Emergency stop                        |
| PORTD4 (B4)  | Increase exposure duration            |
| PORTD5 (B5)  | Decrease exposure duration            |
| PORTD6 (B6)  | Increase lamp intensity               |
| PORTD7 (B7)  | Decrease lamp intensity               |
| PORTC0 (L0)  | Intensity binary display (bit 0)      |
| PORTC1 (L1)  | Intensity binary display (bit 1)      |
| PORTC2 (L2)  | Duration binary display (bit 0)       |
| PORTC3 (L3)  | Duration binary display (bit 1)       |
| PORTC4 (L4)  | UV lamp simulation (PWM)              |
| PORTC5 (L5)  | Process status indicator              |

---

## Timer Configuration

| Timer  | Role                        | OCR Value | Interval |
|--------|-----------------------------|-----------|----------|
| Timer0 | PWM tick (lamp + status)    | 77        | 5 ms     |
| Timer1 | Phase duration counter      | 7812      | 500 ms   |

Both timers use CTC mode with prescaler 1024.

---

## State Machine

```
MODALITA ──[B2]──► RISCALDAMENTO ──[3s]──► ESPOSIZIONE ──[timer end]──► MODALITA
                                                │
                                             [B3]
                                                │
                                             STOP ──[2s]──► MODALITA
```

- **MODALITA**: configure intensity and duration, all buttons active
- **RISCALDAMENTO**: 3-second warm-up, L5 flashes at 500ms, L4 at 5% PWM
- **ESPOSIZIONE**: full exposure, L4 PWM at selected intensity, L5 steady ON
- **STOP**: emergency abort, L4 off immediately, L5 flashes at 250ms for 2s

---

## PWM Implementation

Two independent PWM channels run simultaneously inside `ISR(TIMER0_COMPA_vect)`:

**During warm-up (RISCALDAMENTO):**
| LED | Period  | Duty cycle |
|-----|---------|------------|
| L5  | 1000ms  | 50%        |
| L4  | 100ms   | 5%         |

**During exposure (ESPOSIZIONE):**
| Intensity | L4 Duty cycle |
|-----------|--------------|
| 20%       | 20%          |
| 40%       | 40%          |
| 60%       | 60%          |
| 80%       | 80%          |

**During stop (STOP):**
| LED | Period | Duty cycle |
|-----|--------|------------|
| L5  | 500ms  | 50%        |

---

## Technical Highlights

- Bare-metal C, no Arduino libraries
- Dual independent PWM channels in single ISR (lamp intensity + status LED)
- Intensity locked at start of exposure — mid-exposure changes ignored by design
- PCINT2 interrupt for real-time button detection with edge detection
- Volatile variables for safe ISR ↔ main loop communication
- Clean full variable reset on every state transition

---

## Build

Compiled with AVR-GCC for ATmega328P (16 MHz clock).

```bash
avr-gcc -mmcu=atmega328p -DF_CPU=16000000UL -O2 -o bromograph.elf main.c
avr-objcopy -O ihex bromograph.elf bromograph.hex
avrdude -c usbasp -p m328p -U flash:w:bromograph.hex
```

---

## Author

Ehab Guezmir — B.Sc. Computer Engineering and Automation, eCampus University
