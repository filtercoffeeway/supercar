# VisionAutomation — Lexus ES350 Camera ADAS

A Raspberry Pi–based camera system for a Lexus ES350, built in four phases:

1. **Side view on turn signal:** show the left or right camera feed on a 10" display when that indicator is on.
2. **Blind spot detection:** detect vehicles in the immediately adjacent lane and show an alert on the display.
3. **Lead car departure alert:** chime when stopped and the car ahead drives off.
4. **Forward collision detection:** detect vehicles ahead in the ego lane and warn when a collision is imminent.

The Pi is in the car from Phase 1 so that cameras, display pipeline, and recorded footage carry directly into the ML phases.

---

## Architecture

```
 Left camera ──┐                                   ┌──> 10" display
               ├──> Frame capture ──> Compositor ──┘
Right camera ──┘         │   │             ▲   ▲
                         │   │             │   │
                         │   │   Blinker listener (left / right / off)
                         │   │             │
                         │   └──> Detector (Hailo) ──> Tracker + lane filter ──> alert
                         │
                         └──> Recorder ──> SSD (raw clips for training data)
```

- **Frame capture:** reads both side cameras continuously.
- **Blinker listener:** reads the turn-signal state and tells the compositor which side to show.
- **Compositor:** renders the selected camera feed full-screen and overlays alerts. Blank or idle when no indicator is on.
- **Detector (Phase 2+):** runs a vehicle detection model on the Hailo accelerator using the same camera frames.
- **Tracker + lane filter (Phase 2+):** tracks detections across frames and keeps only vehicles in the immediately adjacent lane, then sends an alert to the compositor.
- **Speed input (Phase 3+):** reads vehicle speed over OBD-II so alerts know when the car is stopped.
- **Audio (Phase 3+):** plays alert chimes.
- **Recorder:** saves raw clips to the SSD to build the training and evaluation dataset.

---

## Phase 1 — Side View on Turn Signal

**Goal:** turn on the left indicator and the left camera feed appears on the display (same for the right), so there is no need to turn and look over the shoulder.

- Install both side cameras (under or near the mirrors) and the display.
- Tap the turn-signal state into the Pi.
- Stream the selected side to the display with minimal latency.
- Start recording drives to the SSD from day one to build the Phase 2 dataset.
- Drive with the setup for a couple of weeks before committing to permanent mounting.

**Done when:** the correct side feed appears quickly and reliably whenever the indicator is on, in daylight and at night.

---

## Phase 2 — Blind Spot Detection

**Goal:** show a blind-spot alert (e.g. a red dot) on the display when a vehicle is in the adjacent lane.

- Scope: freeway driving, vehicles only in the **immediately adjacent lane** (ignore vehicles two lanes over and oncoming or parked objects).
- Run a vehicle detector on the Hailo HAT on the side camera frames.
- Track objects across frames and apply an adjacent-lane filter (region of interest, then learned lane position).
- Draw the alert overlay on the side view, and also show an alert when the indicator is on toward an occupied lane.
- Label recorded footage and fine-tune or train the model for this camera angle.
- Priorities: low latency, few false alerts, no missed vehicles in the lane.

**Done when:** alerts match real blind-spot occupancy on recorded and live freeway drives with acceptable latency and false-alert rate.

---

## Phase 3 — Lead Car Departure Alert

**Goal:** when stopped in traffic (e.g. at a red light), chime if the car ahead pulls away and you haven't moved.

- Add a forward-facing camera, mounted behind the rearview mirror.
- Detect vehicles in the ego lane with the Phase 2 detector and track the lead vehicle over time.
- Read vehicle speed from the OBD-II port to know when the car is stopped.
- Trigger when: own car stopped for at least a few seconds, a lead vehicle was present and close, and it then moves away (box shrinks / rises in frame) for about 1 second while own car is still stopped.
- Play a short chime through a speaker, plus a brief visual cue on the display. One chime per stop, no repeats.
- Record intersection clips from the forward camera to tune thresholds and to seed Phase 4.
- This is a nudge to look up, never a "go" signal.

**Done when:** the chime fires reliably when the lead car departs, and rarely fires for creeping, lane changes by the lead car, or cars in adjacent lanes.

**Later (optional):** green light chime for when there is no lead car. Detect traffic lights (COCO `traffic light` class) and classify red/green by color. Harder because of turn arrows, lights for other lanes, and LED flicker / exposure issues.

---

## Phase 4 — Forward Collision Detection

**Goal:** warn the driver when the vehicle ahead is closing too fast.

- Reuse the Phase 3 forward camera, lead-vehicle tracking, and OBD speed.
- Estimate distance and closing speed, and derive time-to-collision.
- Show a visual warning on the display when time-to-collision falls below a threshold.

**Done when:** warnings trigger reliably in closing scenarios without frequent false alarms in normal following traffic.

---

## Hardware

| Component | Choice | Notes |
|---|---|---|
| Compute | Raspberry Pi 5 | Mount under a seat or in the center console, where it stays cooler than the dash. |
| ML accelerator | Raspberry Pi AI HAT+ (Hailo-8L, 13 TOPS) | Needed from Phase 2. Uses the Pi 5's single PCIe lane. |
| Storage | USB 3 external SSD (256GB+) | Stores recorded clips for training. USB rather than NVMe, so it doesn't compete with the AI HAT for PCIe. |
| Side cameras | 2× digital cameras, weatherproof, mounted near the mirrors | Digital (not analog) so frames go straight to the Pi for ML. |
| Forward camera | 1× digital camera, behind the rearview mirror | Phase 3. Needs enough vertical field of view to keep the lead car in frame when stopped close behind it. |
| Vehicle speed | OBD-II adapter (USB, ELM327-compatible or CAN) | Phase 3. Reads vehicle speed (standard PID) so alerts know when the car is stopped. |
| Audio | Small USB speaker (or HDMI audio if the display has a speaker) | Phase 3. The Pi 5 has no headphone jack. |
| Display (bench) | Elecrow 10.1" IPS, 1280×800 | Cheap option for indoor development. Too dim for direct sun. |
| Display (car) | Xenarc 1022YH 10.1", 1280×800, 1200 nits | Sunlight-readable, anti-reflective, 9–36V DC input, -20°C to 70°C. No touch (not needed). |
| Video cable | Short micro-HDMI to HDMI | Keep it short and secured. Loose HDMI is a common in-car glitch. |
| Turn-signal input | Tap from the left and right indicator circuits into the Pi | Isolate the 12V signal before it reaches the Pi GPIO. |
| Power | 12V car power to 5V supply for the Pi | The display runs directly on 12V. |

### Display selection criteria
- 1000+ nits brightness for daylight readability.
- 1280×800 resolution.
- 12V input so it runs directly from car power.
- No logo or "No Signal" screen at boot.

### Mounting
- **Display location:** lower center, below the factory screen, on a RAM Mounts arm attached to a seat bolt or the side of the center console, angled toward the driver. It is adjustable and needs no drilling. Watch for it covering the climate controls.
- **Mount hardware:** bolt it down using the display's VESA 75 holes and a RAM VESA plate. No adhesive, vent, or suction mounts (the display weighs about 3 lb).
- **Keep clear of:** the windshield view and airbag deployment zones (passenger dash panel, knee area).
- **Cabling:** route under trim panels.
- **Alternatives considered for later:** on top of the dash beside the factory screen (needs a custom screwed bracket), or two small screens at the A-pillars.
