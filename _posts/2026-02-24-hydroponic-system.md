---
title: "The Project That Got Me Into Programming"
date: 2026-02-24 20:00:00 +0000
categories: [projects]
tags:
  [
    arduino,
    hydroponics,
    embedded,
    sensors,
    farmingdale,
    senior-project,
    hardware,
    control-systems,
  ]
description: "My Farmingdale State College senior project: an enclosed, automated hydroponic system built with Arduino, pH sensors, moisture sensors, and relays. Spring 2022. This is where the coding started."
---

## Where It Started

Spring 2022. Senior year at Farmingdale State College. EET 452W—the capstone design course for Computer Engineering Technologies.

Me and two of my classmates decided to build an automated hydroponic system from scratch. Enclosed, sensor-monitored, Arduino-controlled. The goal was to grow plants with as little human interaction as possible—like a fish tank, but for lettuce.

I was responsible for software. This was, genuinely, the project that made me realize I actually liked programming.

---

## What Hydroponics Is

Hydroponics replaces soil with a nutrient-rich water medium. Plants grow faster, use less water, and don't need pesticides. The global market was around $9.5 billion when we wrote the report in 2021, projected to hit $17.9 billion by 2026.

Consumer systems exist—AeroGarden Harvest was the benchmark at $149.95, six pods, 20W LED, automatic reminders. Our pitch: more grow space, lower cost, fully customizable parameters per plant.

The specific type we implemented was an **Ebb and Flow** system. A submersible pump in the reservoir periodically floods the upper plant tray with nutrient-rich water, then gravity drains it back. Simple, low-maintenance, and well-documented.

---

## How the System Worked

An Arduino Uno sat at the center of everything, running a feedback loop:

```
Sensors → Arduino → Relays → Pumps / LEDs / Nutrient Dispenser
```

**Sensors:**

- **Grove Temperature Sensor** — thermistor-based, -40°C to 125°C, ±1.5°C accuracy
- **Gravity Analog pH Sensor** — 0–14 range, ±0.1pH at 25°C
- **Capacitive Soil Moisture Sensor** — corrosion-resistant, 3.3–5.5V
- **TDS Sensor** — measures total dissolved solids in the reservoir water, 0–1000ppm

**Actuators:**

- DC 3–6V brushless water pump (submersible, in the reservoir)
- CHENBO channel relays (switching the pump and LED strips)
- Timed LED strips for the light cycle

The control logic was conditional: if pH dropped below threshold, trigger the nutrient dispenser. If moisture sensor read dry, run the water pump. If it's lights-on time, turn the LEDs on. The water pump ran on timed cycles independent of sensor readings.

Data streamed from each sensor to a laptop over USB, logging to Excel in real time. The temperature sensor held steady at 21°C (70°F ambient) in the proposal test—matching exactly what we expected.

---

## The Physical Build

The enclosure was a plastic bin with PVC pipe running through it. Plants sat in net pots along the PVC structure, roots hanging down into the reservoir as they matured. The lid had:

- Air vents for humidity control
- Handles for portability
- A hinge for plant access
- LED strips mounted inside facing down
- The Arduino mounted in the case wall

One teammate handled the physical construction—cutting PVC, applying waterproof adhesive (which needs hours to cure and test), wiring the power side. Another managed materials and assembly. I wrote the code and ran sensor testing.

---

## The Problems We Actually Hit

**Library hell.** The TDS sensor required `GravityTDS.h` and the temperature sensor required `<math.h>`. Both caused compiler errors until I tracked down the right library versions. Eventually got them both compiling together.

**The relay MOSFET question.** There was a genuine concern mid-project about whether the relays had an internal MOSFET capable of amplifying the Arduino's signal (the Arduino outputs ~20mA, the motors needed ~70mA). One teammate sourced PS170 MOSFETs and designed a discrete amplifier stage as a backup. Took over a week for parts to arrive. Turns out the relays did have the internal MOSFET and worked fine—but having the backup option forced us to actually understand what was happening inside the relay module.

**The missing ground.** Classic embedded debugging story. Everything tested fine at home. Brought the project into school for the lab presentation. Relays stopped responding. Spent time checking code, checking ports, checking relay wiring. The problem: a ground connection from the Arduino to the relay configuration had come loose in transit. One wire. Fixed in two minutes once we found it.

**The loose signal wire.** A separate incident—a digital HIGH signal from the Arduino wasn't triggering the relay. Same kind of debugging loop. Turned out the ground on that circuit wasn't completing. Fixed by finding and securing the loose ground wire.

Two separate grounding issues across two different sessions. Embedded development lesson learned.

---

## What We Actually Built vs. What We Proposed

The proposal was ambitious: pH-triggered nutrient dispensing, full humidity control, a plant parameter database with per-species settings stored and uploaded via Excel.

What shipped: timed water pumping, working pH and moisture monitoring, conditional logic to trigger pumping based on sensor readings, real-time data logging to Excel. The nutrient dispenser worked. The full per-plant database and UI did not—cut due to time.

The honest conclusion from the project report holds up: _"Although our current implementation met the goal of automating timed water pumping for plant growth, a longer timeline and more funding could facilitate a more comprehensive system test."_

Budget came in under $200. Commercial equivalent: $149.95 with six pods and a proprietary app. We had more grow space and full access to the control logic.

---

## System's Mindset

We got an A by the way.

---

_Project by Xhovani Mali_
