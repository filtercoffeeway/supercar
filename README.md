# supercar

A Raspberry Pi–based camera ADAS for a Lexus ES350: side cameras, a forward camera, and on-device vehicle detection on a Hailo accelerator, shown on a 10" in-car display.

> **Status:** planning. No code yet. See [plan.md](plan.md) for the full plan and [proposal.html](proposal.html) for the written proposal.

## Phases

1. **Side view on turn signal:** show the left or right camera feed on the display when that indicator is on.
2. **Blind spot detection:** detect vehicles in the immediately adjacent lane and show an alert.
3. **Lead car departure alert:** chime when stopped and the car ahead drives off. Optional later add-on: green light chime when there is no lead car.
4. **Forward collision detection:** estimate time-to-collision with the vehicle ahead and warn when it gets too low.

The Pi is in the car from Phase 1 and records drives to an SSD, so footage from early phases becomes training and evaluation data for the ML phases.

## Hardware

- Raspberry Pi 5 + AI HAT+ (Hailo-8L, 13 TOPS)
- USB 3 SSD for recorded clips
- 2× side cameras near the mirrors, 1× forward camera behind the rearview mirror
- 10.1" 1280×800 sunlight-readable display (Xenarc 1022YH)
- Turn-signal tap into GPIO (isolated from 12V), OBD-II adapter for vehicle speed, USB speaker for chimes

Full parts list, display criteria, and mounting notes are in [plan.md](plan.md#hardware).

## Safety

This is a personal driver-aid project, not a certified safety system. Alerts are nudges to look, never a signal to act, and the setup must not block the windshield view or airbag deployment zones.
