# Product Requirements Document (PRD)

**Project:** Silent Guardian
**Version:** 0.1 | **Date:** 28 November 2026 | **Type:** Academic minor project (ECE)

---

## 1. Purpose

Build a physically demonstrable prototype that listens through a microphone, classifies the sound on a Raspberry Pi 4B using a small trained neural network, and shows the result on a screen and in a log file.

## 2. Problem statement

Forest staff cannot listen everywhere. A cheap device that recognises a few important sounds could help prioritise where a person should look. This project is **not** trying to solve forest protection. It proves that the sound-to-label pipeline works on low-cost hardware, and measures how well.

## 3. Goals and non-goals

### Goals
- G1. Capture real sound with an INMP441 microphone on a Raspberry Pi 4B.
- G2. Classify each 1-second window as `bird`, `chainsaw`, `gunshot` or `background`.
- G3. Run the classifier on the Pi without needing the laptop.
- G4. Log every detection with time and confidence, and show it on a simple dashboard.
- G5. Measure accuracy, false alarms, speed and power honestly, including on our own microphone recordings.
- G6. (Optional) Send an alert to an ESP32 beacon.

### Non-goals (do not build these now)
- Bird **species** identification.
- Proof of illegal logging, poaching or shooter identity.
- Locating where the sound came from.
- Year-round outdoor operation, waterproofing or tamper protection.
- Running the Pi from solar power (the panel is far too small; see Section 9).
- LoRaWAN, satellite or cellular connectivity.
- Selling or deploying the product.

## 4. Users and stakeholders

| Who | What they need |
|---|---|
| Student team | A clear path from data to a working demo. |
| Faculty examiner | A live demonstration, honest results and clean documentation. |
| (Future) forest staff | Not users now. Mentioned only as motivation. |

## 5. Hardware available

| Item | Notes |
|---|---|
| Raspberry Pi 4B, 4 GB RAM, 32 GB SD | Main computer. |
| INMP441 I2S MEMS microphone | 3.3 V digital microphone. |
| ESP32 | Optional alert beacon. Exact variant to confirm. |
| Solar module (model to confirm) | Charge controller for Li-ion. |
| Solar panel DS-A-1.3 | 1.3 W, 7.5 V at maximum power, 180 × 90 mm. |
| Li-ion 2500 mAh cell with holder | About 9.25 Wh nominal. |
| Breadboard, jumper wires | Prototyping. |

Missing items are listed in [`06_MISSING_COMPONENTS.md`](06_MISSING_COMPONENTS.md).

## 6. Functional requirements

| ID | Requirement | Priority |
|---|---|---|
| FR-1 | The system shall record mono audio at 16 kHz from the INMP441, or record at 48 kHz and resample to 16 kHz in software. | Must |
| FR-2 | The system shall process the latest 1.0 s of audio every 0.5 s. | Must |
| FR-3 | The system shall convert audio to a 64-mel × 63-frame log-Mel feature using the single shared `sg_features.py`. | Must |
| FR-4 | The system shall classify each window into 4 classes using a TFLite model. | Must |
| FR-5 | The system shall apply a per-class confidence threshold and a decision rule before declaring an event. | Must |
| FR-6 | The system shall write each event to a log with time, class, confidence and inference time. | Must |
| FR-7 | The system shall show the latest class, confidence and event count on a local dashboard. | Must |
| FR-8 | The system shall keep running for at least 30 minutes without crashing. | Must |
| FR-9 | The system shall report dropped or late audio windows. | Should |
| FR-10 | The system shall send an alert line to an ESP32 over USB serial. | Could |
| FR-11 | The ESP32 shall light an LED or buzzer when an alert is received. | Could |
| FR-12 | The system shall save a short audio clip around each event for later review. | Could |

## 7. Non-functional requirements

| ID | Requirement | Proposed target (not yet measured) |
|---|---|---|
| NFR-1 | Processing time per window (features + model) | Under 250 ms on the Pi |
| NFR-2 | Held-out test macro-F1 (dataset audio) | At least 0.85 |
| NFR-3 | Macro-F1 on our own microphone recordings | At least 0.70, reported separately |
| NFR-4 | False `chainsaw` or `gunshot` alerts in quiet or speech background | At most 1 per 30 minutes |
| NFR-5 | RAM used by the program | Under 1 GB |
| NFR-6 | Model file size | Under 1 MB |
| NFR-7 | Startup to first prediction | Under 60 s after Pi boot, when run as a service |
| NFR-8 | Reproducibility | Fixed random seed, saved manifests, pinned package versions |

These are proposals for the team to accept or change. Do not claim them until the measured column below is filled in.

## 8. Data requirements

- Four classes: `bird`, `chainsaw`, `gunshot`, `background`.
- Source of `bird`, `chainsaw`, `gunshot`: the supplied CSV and its linked datasets.
- Source of `background`: ESC-50 non-target classes plus our own recordings.
- Birds are capped per species (start at 60 clips each) to avoid swamping other classes.
- Split by source group before cutting windows. No recording appears in two splits.
- A **separate real-microphone test set** (at least 100 windows per class) is never used for training or threshold tuning.
- Remove 38 bird rows where one Xeno-canto recording has two species labels.
- Keep only the original synthetic gunshot files, not the 19 codec re-encodings of each.

## 9. Power and solar requirements

| Question | Answer |
|---|---|
| What powers the Pi? | A genuine 5.1 V, 3 A USB-C adapter (or a suitable power bank). |
| What can the solar parts power? | The ESP32 beacon, or a battery-charging demonstration. |
| Why not power the Pi from solar? | Pi: about 3 W idle, about 72 Wh a day. Panel: 1.3 W peak, a few Wh a day. Battery: about 9.25 Wh. |
| Is solar required to pass the project? | No. |

Required safety checks before any solar connection:
- Identify the exact charge-module model.
- Measure the panel's open-circuit voltage in sunlight with a multimeter.
- Confirm that voltage is inside the module's allowed solar-input range.
- Confirm battery polarity with a multimeter.

## 10. Acceptance tests

| ID | Test | Pass condition | Result |
|---|---|---|---|
| AT-1 | Record 10 s on the Pi | File plays back, speech is understandable, no constant noise | ____ |
| AT-2 | Frontend parity | Features for the same audio on laptop and Pi match within a stated tolerance | ____ |
| AT-3 | Held-out dataset test | Macro-F1 recorded, with per-class table and confusion matrix | ____ |
| AT-4 | Own-microphone test | Separate table of the same metrics | ____ |
| AT-5 | Live demo, bird clip played from phone | Bird detected, logged and shown | ____ |
| AT-6 | Live demo, chainsaw clip | Chainsaw detected, logged and shown | ____ |
| AT-7 | Live demo, gunshot clip | Gunshot-like event detected, or limitation explained | ____ |
| AT-8 | Live demo, speech and fan | No chainsaw or gunshot alert | ____ |
| AT-9 | 30-minute continuous run | No crash; count of dropped windows reported | ____ |
| AT-10 | Timing and resources | Per-window time, CPU %, RAM measured and recorded | ____ |
| AT-11 | ESP32 alert (optional) | LED lights within 2 s of an event | ____ |
| AT-12 | Power measurement | Pi average power logged with a USB meter, if available | ____ |

## 11. Constraints and assumptions

- 4 GB RAM and a 32 GB SD card are enough for training features if the dataset is capped. Large downloads may need an external drive or laptop storage.
- Training happens on a laptop or Colab, not on the Pi.
- A Wi-Fi connection or Ethernet is available for installing packages.
- The team can borrow a multimeter and, if needed, soldering tools.
- Data licences allow non-commercial academic use.

## 12. Risks

| Risk | Impact | Response |
|---|---|---|
| Model learns the dataset's recording style | Great test score, poor live performance | Own-microphone test set; background class; noise augmentation. |
| Very few unique gunshots | Weak or unreliable gunshot class | Treat as experimental; report separately; collect more licensed real data. |
| INMP441 audio is very quiet or distorted | Poor features | Check wiring and gain; compare 48 kHz plus resampling. |
| Pi under-voltage | Random crashes | Use the proper 3 A supply; check `vcgencmd get_throttled`. |
| Solar module and panel mismatch | Damaged module or battery | Check model and voltages first; skip solar if unsure. |
| Time overrun | Incomplete demo | Freeze scope. Core first; ESP32 and solar last. |

## 13. Milestones (8–10 weeks, small team)

| Week | Milestone |
|---|---|
| 1 | Pi set up, microphone wired and recording. |
| 2 | Pilot data downloaded, manifest built, duplicates and conflicts removed. |
| 3 | Audio cleaned, windows and features extracted, background data added. |
| 4–5 | Baseline and CNN trained, evaluated, confusion matrix reviewed. |
| 6 | Model on the Pi, live predictions working. |
| 7 | Own-microphone test set collected, threshold tuning, retraining if needed. |
| 8 | Dashboard, logging, 30-minute stability test. |
| 9 | Optional: ESP32 alert, solar charging test. |
| 10 | Final measurements, report, slides, demo video. |

## 14. Definition of done

A faculty member can play a bird, a chainsaw and a gunshot-like recording near the microphone, watch the Pi classify them live, see the events logged, and read honest measurements that include the system's weaknesses.

## 15. Open questions to resolve

1. What exact model is the solar charging module? (Brand, name, input voltage range.)
2. Is the ESP32 a plain ESP32 (Wi-Fi + Bluetooth)? Which USB connector does it use?
3. Does the INMP441 board have its header pins soldered?
4. Do we have a reliable 5.1 V / 3 A USB-C supply for the Pi?
5. Will the demo be indoors only, or also outdoors (needs power and permission)?
