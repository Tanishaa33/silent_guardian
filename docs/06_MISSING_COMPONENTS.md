# Missing Components and Compatibility Check

What you have, what is missing, and why. No prices are listed because current prices were not checked. Get quotes from the shop.

---

## 1. What you have: status check

| Item | Status for this project | Notes |
|---|---|---|
| Raspberry Pi 4B, 4 GB, 32 GB SD | **Good** | 4 GB is plenty for inference. Train on a laptop or Colab, not on the Pi. |
| INMP441 I2S microphone | **Good** | 3.3 V digital microphone. Check whether header pins are soldered. |
| ESP32 | **Optional** | Alert beacon only. Confirm board type and USB connector. |
| Solar module | **Unknown: must be identified** | Might not accept your panel. See Section 3. |
| Solar panel DS-A-1.3 (1.3 W, 7.5 V at Pmax, 180 × 90 mm) | **Too small for the Pi** | Fine for a charging experiment or a very small load. |
| Li-ion 2500 mAh cell with holder | **OK for small loads** | About 9.25 Wh nominal. Check whether it has protection. |
| Breadboard, jumper wires | **Partly sufficient** | Wire type matters (Section 2). |

## 2. Components you must have

| # | Component | Exact specification | Quantity | Why it is needed |
|---|---|---|---|---|
| 1 | **Pi power supply** | 5.1 V, 3 A, USB-C, genuine Raspberry Pi 15 W adapter or equivalent good-quality one | 1 | The Pi 4 is designed around 5.1 V / 3 A. A phone charger or weak cable can cause under-voltage, crashes and corrupted audio. [Source](https://forums.raspberrypi.com/viewtopic.php?t=351068) |
| 2 | **Microphone-to-Pi wires** | Dupont jumper wires, about 20 cm. If the mic is plugged straight onto the Pi: **female-to-female**. If mounted on a breadboard: **male-to-female**. | At least 6 (a spare set is helpful) | The six microphone signals must reach the Pi's header. Your listed "jumper wires" may be the wrong type. |
| 3 | **Soldered header on the INMP441** | 6-pin 2.54 mm header soldered on the board, or a pre-soldered board | 1 | Loose pins give crackly noise or no sound. |
| 4 | **microSD card reader** | USB reader (if the laptop has no slot) | 1 | To flash the Pi's SD card. |
| 5 | **Data USB cable for the ESP32** | Matching connector (micro-USB or USB-C, check your board), data-capable, not charge-only | 1 | Programming and serial alerts. |
| 6 | **Digital multimeter** (borrow) | DC voltage up to 20 V | 1 | To measure panel and battery voltage and check polarity. Without it the solar part is guesswork. |
| 7 | **Phone or speaker** | Any | 1 | To play test sounds. |
| 8 | **Laptop** | Any, ideally 8 GB RAM; or use Google Colab | 1 | Dataset work and training. |
| 9 | **Storage for the dataset** | Laptop or an external drive; not the 32 GB Pi card | — | The audio files and features can take many gigabytes. Check free space before downloading. |

## 3. The solar module and panel check

### Why this matters
Your panel is rated at 7.5 V at maximum power, and an unloaded panel gives a higher voltage. A charge module has a maximum solar-input voltage. Over-voltage can destroy it, and a damaged charger can harm the battery.

### Check
1. Read the module's name from the board.
2. Look up its **solar input voltage range**.
3. Measure the panel's open-circuit voltage in sunlight with a multimeter.

### Known examples from DFRobot
| Module | Solar input range | Fit with your panel |
|---|---|---|
| Solar Power Manager 5V (DFR0559) | Nominal 5 V panel, about 4.4–6 V; warns against more than about 6.5 V [DFRobot](https://wiki.dfrobot.com/dfr0559/docs/21586) | **Not a match.** |
| Micro Solar Power Manager (DFR0535) | 1–3 V [DFRobot](https://www.dfrobot.com/product-1714.html) | **Not a match.** |
| Solar Power Manager for 9V/12V/18V panels | 7–30 V, MPPT setting options OFF / 9 V / 12 V / 18 V, 5 V output up to 1.5 A [DFRobot](https://www.dfrobot.com/product-1714.html) | **Possible, but marginal** at 7.5 V; it may stop charging in weak light. Check the manual for the right MPPT setting. |

### If your module is not suitable, buy one of these (choose one)
| Option | What to buy | Why |
|---|---|---|
| A | A **~6 V (5–6 V), 2–5 W solar panel** to match a 5 V-class module | Keeps your module and replaces the panel |
| B | A **solar charge module with a 7–30 V input** for a single 3.7 V Li-ion cell, with battery protection | Keeps your 7.5 V panel |

Do not connect your panel until you have verified the match.

## 4. Why the Pi cannot run from your solar parts

Estimates (assumptions shown):

| Quantity | Number |
|---|---|
| Pi 4B idle power, about 600 mA at 5 V | about 3 W [Wikipedia summary of measured idle figures](https://en.wikipedia.org/wiki/Raspberry_Pi) |
| Pi energy per day at 3 W | 72 Wh |
| Panel: 1.3 W × 4 sun-hours × 0.6 real-world factor | about 3.1 Wh per day |
| Panel share of the Pi's daily need | about 4% |
| Battery: 2.5 Ah × 3.7 V | about 9.25 Wh nominal |
| Battery at 3 W, 85% usable | about 2.6 hours |

So the battery is a few hours for the Pi, and the panel cannot refill it.

Also, the DFRobot 9/12/18 V module's 5 V output is up to 1.5 A, while the Pi 4 is designed around 3 A. [DFRobot](https://www.dfrobot.com/product-1714.html)

### What would a genuinely off-grid Pi need? (not required for the minor project)
Rough design for about 4 W average, using the same sun-hour assumption:

| Part | Rough size | Reason |
|---|---|---|
| Solar panel | About 40 W nominal | 96 Wh per day ÷ (4 h × 0.6) |
| Battery | About 12 V, 30–40 Ah for about two days without sun | Autonomy and depth-of-discharge margin |
| Charge controller | Matching the panel and battery chemistry | Safe charging |
| DC-DC converter | 12 V to 5.1 V, at least 3 A, USB-C output | Clean Pi power |
| Enclosure, fuses, waterproofing | Outdoor rated | Safety and weather |

These are planning estimates only. This is too big and costly for the minor project. Present it as "future work" in the report.

## 5. Battery safety check

- Is the cell **protected** (small circuit under the wrap) or a bare cell? A bare cell needs charge and discharge protection, which some solar modules provide.
- Use the right charger for a **single 3.7 V Li-ion** cell (4.2 V when full).
- Never use damaged or swollen cells.
- Add a fuse near the battery for any permanent build.

## 6. Recommended extras (not mandatory)

| Item | Why |
|---|---|
| Heatsink or case with cooling for the Pi | The Pi warms up when running continuously. |
| USB power meter | Measure real power for the report. |
| Small project box | Protect the Pi and wiring; leave the microphone hole open. |
| Zip ties, tape | Stop wires pulling loose. |
| Shielded or short wires | Less noise on the microphone lines. |
| Foam windscreen (small) | Reduces wind puffs on the microphone. |

## 7. Only if you want long-range wireless (LoRa)

Your parts have no LoRa radio. Wi-Fi, Bluetooth and ESP-NOW from the ESP32 are short-range. For a LoRa link you would need:

| Part | Quantity | Why |
|---|---|---|
| LoRa module with SX1276 for 865–868 MHz, for example Ai-Thinker Ra-01H | 2 | One at each end |
| Matched 865–868 MHz antennas | 2 | Never transmit without an antenna |
| A second ESP32 (or a second Pi) | 1 | Receiver side |
| Radio breakout or carrier board | 2 | The module has tiny solder pads |
| Faculty/lab approval of the radio settings | — | Indian 865–868 MHz use has conditions on power, duty cycle and approval |

This is optional and not needed to pass the core demonstration.

## 8. Things you do NOT need to buy

| Item | Why not |
|---|---|
| A second microphone | One INMP441 is enough for classification. |
| An external USB sound card | The INMP441 connects directly to the Pi's I2S pins. |
| A GPU | Training on a laptop or free Colab is enough for this small model. |
| A camera | Not part of this project. |
| Another Pi | Not needed unless you want a separate receiver. |

## 9. Final shopping decision table

| Situation | Buy or borrow |
|---|---|
| Minimum to finish the core demo | 5.1 V 3 A USB-C supply; correct jumper wires; SD card reader if needed; soldered microphone header; multimeter (borrow) |
| Want the ESP32 beacon | Add a data USB cable (and an LED or buzzer if desired) |
| Want the solar experiment | Identify the module; if mismatched, buy a matching panel or a 7–30 V-input module (Section 3) |
| Want long range | Add the LoRa list in Section 7 |
