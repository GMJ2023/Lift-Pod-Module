# LiftPod — Headlight Pod Controller

**Digconn Systems Limited**

A modern, intelligent drop-in replacement for the obsolete Lotus Esprit headlight pod controller module.

Replaces: **A082M6363F** (Lotus) / **G4886** / **GM 16525685**

---

## Compatible Vehicles

| Vehicle               | Years     | OEM Part    |
| --------------------- | --------- | ----------- |
| Lotus Esprit          | All years | A082M6363F  |
| Lotus Elan M100       | 1989–1995 | A100M6027F  |
| Jaguar XJ220          | 1992–1994 | —           |
| Alpine A610           | 1991–1995 | —           |
| Chevrolet Corvette C4 | 1984–1996 | GM 16525685 |
| Venturi Atlantique    | 1991–2000 | —           |

> **Confirmed tested and working:** Lotus Esprit (multiple cars, multiple years including V8)

---

## The Problem — Why Original Modules Fail

The original A082M6363F and its GM equivalents are now 30–35 years old. They fail in predictable ways:

- Power transistors overheat and fail (no heatsinking on original PCB)
- Relay contacts corrode and pit
- Solder joints crack from thermal cycling
- PCB traces burn around the switching transistors

When they fail, the symptoms are:

- One or both pods refusing to raise
- Pods travelling half distance then stopping
- One pod raising while the other lowers (winking)
- Intermittent operation that worsens over time

**The original module is no longer available from Lotus.** GM cross-reference parts exist but introduce their own problems — covered below.

---

## Why GM Cross-References Work on Some Cars, Not Others

The GM cross-reference modules (16525685, 16509097, 16521278 etc.) have the same connector pinout and the same basic function as the Lotus part. However, they were manufactured for a range of GM vehicles with varying motor wiring conventions.

Fitting a module sourced from a different vehicle can result in:

- Both pods working correctly ✓
- Both pods running in reverse (both lower when you want raise)
- One pod raising, one lowering (winking)

This is not a fault with the module — it is a **motor polarity mismatch** between the donor vehicle and the target vehicle. More on this below.

---

## Root Cause Analysis — The 1kΩ Switch Resistor

During development of LiftPod, field failures were traced to a fundamental problem with the original input circuit design.

The Lotus Esprit headlight switch contains an internal **1kΩ resistor** between its terminals. When the switch is in either position, the inactive terminal is not a clean ground or open circuit — it sits at a residual voltage determined by the resistor divider formed between the 1kΩ switch resistor and the module's own input pull-down network.

Under motor load, the 12V rail sags. This causes the active terminal voltage to drop from 12V into an indeterminate zone (typically 1.5–3V) — precisely the undefined region for a digital input on an AVR microcontroller.

The consequences:

- The MCU cannot reliably determine which command (RAISE or LOWER) is active
- IS (current sense) baseline is captured on a sagging rail, producing artificially low values
- Stall detection fires prematurely at approximately 400ms — half travel
- Both pods stop halfway — the classic field failure symptom

This problem is **worsened by 35 years of switch wear, corroded contacts, and high-resistance connections** throughout the loom.

---

## The Solution — ADC Differential Input

LiftPod replaces the digital input approach entirely with an **ADC differential read**.

Both the RAISE and LOWER input pins are read simultaneously via the AVR's ADC (PC4/A4 and PC5/A5). Whichever pin reads the higher ADC count wins — and becomes the active command.

Validated readings from a real Lotus Esprit (1999 switch, cleaned contacts):

| State           | LOWER ADC | RAISE ADC | Margin      |
| --------------- | --------- | --------- | ----------- |
| LOWER commanded | 1023      | 84–87     | 936+ counts |
| RAISE commanded | 115–175   | 1023      | 848+ counts |

**850+ count margin** — completely unambiguous under all real-world conditions:

- 1kΩ switch resistor ✓
- Rail sag under motor load ✓
- Worn and corroded contacts ✓
- Load variation (sidelights, fans, AC) ✓
- 35 year old wiring resistance ✓
- Cold and damp conditions ✓

A single function — `resolveCommand()` — is the sole input truth. Nothing else in the firmware reads the input pins directly.

---

## The M2_DIR_INVERT Discovery — Why Winking Happens

During real-car testing, LiftPod initially produced the classic winking symptom — one pod raising while the other lowered.

Investigation revealed the root cause:

The two headlight pod mechanisms are **physical mirror images** of each other. To raise both pods simultaneously, the left motor must rotate clockwise and the right motor must rotate anti-clockwise (or vice versa).

The original Lotus factory wiring compensates for this by swapping the motor wire polarity on one side within the car's loom. The module itself treats both motors identically — the loom asymmetry does the work.

LiftPod routes both motor channels identically through the PCB. A firmware define — `M2_DIR_INVERT` — inverts the direction of M2 relative to M1, compensating for the physical mirror asymmetry in software rather than relying on loom wiring.

```c
#define M2_DIR_INVERT  1   // 0=normal, 1=invert M2 direction only
```

**This also explains why fitting an Esprit module into an Elan M100 causes winking.** The two cars have different motor wiring conventions in their looms. The original module has no way to compensate — it is hardwired for one convention. LiftPod's `M2_DIR_INVERT` define makes it configurable for any vehicle.

---

## Stall Detection — How LiftPod Knows When to Stop

The original module uses current sensing to detect when a motor has reached its end stop (the pod is fully raised or lowered). When the motor stalls against the end stop, current draw spikes — and the module cuts power.

LiftPod uses the same principle but with a more sophisticated three-phase timing window:

```
0ms      150ms      700ms          1200ms  1400ms
|---------|----------|--------------|-------|
  BLIND    BASELINE   STALL DETECT  SOFT   HARD
  no IS    capture    active        STOP   CUTOFF
```

- **BLIND (0–150ms):** Motor spinning up, rail recovering. No stall decisions. Hard overcurrent backstop active (child/obstruction safety).
- **BASELINE (150–700ms):** Motor at speed, rail stable. Running current ceiling captured per motor. No stall decisions yet.
- **STALL DETECT (700ms+):** IS exceeds fixed threshold OR spikes above baseline — motor stopped cleanly.
- **HARD CUTOFF (1400ms):** Unconditional. Both motors off. Catches genuine mechanical failures.

Each motor (M1 and M2) has completely independent stall detection. If one pod mechanism is stiffer than the other, or one pod is already at its end stop, the firmware handles it cleanly — the other motor continues until it too stalls.

**Confirmed real-car results (Lotus Esprit):**

```
Travel time:        778–784ms  (well within 1200ms window)
Stall IS values:    113–135    (healthy margin above baseline)
Both motors:        flags=0x03 every move
Board temperature:  ambient (BTN8982TA H-bridges barely warm)
```

---

## Position Memory

LiftPod remembers pod position across power cycles using EEPROM. If power is lost mid-move, the system detects this on the next power-on and recommissions cleanly from the switch state.

A brownout detection circuit prevents the infinite retry loop that can occur if a motor stalls hard at its end stop — clearing the pending request and allowing the supply voltage to recover before retrying.

---

## Idle Current

```
Production firmware (DEBUG_MODE 0):  2.41mA
```

Well within the requirements for permanent connection to the vehicle battery.

---

## Purchase

LiftPod is available from the Digconn Systems shop:

**[digconn-shop.bigcartel.com](https://digconn-shop.bigcartel.com)**

Price: **£295.00** including UK delivery.

Each unit is hand-assembled and tested by Digconn Systems Limited, Ellesmere Port, UK.

---

## About Digconn Systems Limited

Digconn Systems Limited designs and hand-assembles embedded electronics products for specialist automotive and enthusiast applications.

**Contact:** digconn@gmail.com

---

*LiftPod is not affiliated with, endorsed by, or connected to Lotus Cars Limited, General Motors, or any other vehicle manufacturer. All part numbers referenced are for identification purposes only.*
