# Hardware Roadmap, explained like you are 5

No wiring knowledge needed. Read slowly. Do one small step, check it, then do the next one.

Your parts: Raspberry Pi 4B, INMP441 microphone, ESP32, solar module, solar panel (1.3 W, 7.5 V), Li-ion battery with holder, breadboard, jumper wires.

---

## Part 0. Meet the parts

Think of the project as a tiny **listening robot**.

| Part | Kid-friendly picture | What it really does |
|---|---|---|
| Raspberry Pi 4B | The **brain**. It thinks. | A small computer. It runs the sound-recognising program. |
| INMP441 microphone | The **ear**. | Turns sound into numbers (1s and 0s) and sends them to the brain. |
| ESP32 | A **little helper with a light** (optional). | A small chip with Wi-Fi. It can blink a light when the brain finds something. |
| Solar panel | A **sun-drinker**. | Turns sunlight into electricity. Yours is small: 1.3 W at best. |
| Solar module | A **careful babysitter**. | Makes the battery fill up safely. It must match your panel. |
| Li-ion battery + holder | The **juice box**. | Stores electricity. |
| Breadboard | A **toy-brick board**. | You push wires into holes. No glue, no solder. |
| Jumper wires | **Strings** that carry electricity. | Connect one place to another. |

Three words you must know:
- **Voltage (V):** how hard the electricity pushes. Like water pressure.
- **Current (A or mA):** how much electricity flows. Like how much water flows.
- **Ground (GND):** the "way home" for electricity. Every part needs one.

---

## Part 1. The 8 golden rules (read these before every session)

1. **Power OFF before you touch wires.** Unplug the Pi first.
2. **The microphone gets 3.3 V, never 5 V.** The Pi has both. Choosing the wrong one can break the microphone.
3. **Never touch two bare metal wire ends together** on a powered battery. That is a short circuit. It can make heat, smoke or fire.
4. **Red = positive (+), Black = negative (−/ground).** Use red and black for power and ground only. This helps your eyes.
5. **Count pins twice.** Then have a friend count them again.
6. **Do not connect the solar panel to anything until you know the module's name and its allowed voltage** (Part 6).
7. **Never short, puncture, bend or heat the Li-ion battery.** Keep it away from metal tools and bags of keys.
8. **If something smells hot or looks weird, unplug it immediately.**

Gentle tools to borrow: a **multimeter** (a little box that measures voltage). You will use it for the solar part.

---

## Part 2. Set up the Pi brain (no wires except the power)

You need: Raspberry Pi 4B, microSD card, **5.1 V 3 A USB-C power adapter**, a laptop, and a microSD card reader.

Why the 3 A adapter? The Pi 4 is designed around a 5.1 V, 3 A USB-C supply, and weak chargers cause random crashes. [Source](https://forums.raspberrypi.com/viewtopic.php?t=351068)

### Steps

1. On the laptop, install **Raspberry Pi Imager** (from raspberrypi.com).
2. Put the microSD card in the reader, plug it into the laptop.
3. In Imager choose: *Raspberry Pi 4*, *Raspberry Pi OS (64-bit)*, and your SD card.
4. Click the gear or "edit settings" button. Set:
   - hostname: `silentguardian`
   - username and password (write them down)
   - your Wi-Fi name and password
   - turn on **SSH**
5. Write the card. Wait until it says finished.
6. Put the card in the Pi (the slot is under the board). Plug in the 3 A power.
7. Wait 2 minutes. On your laptop terminal type:

```bash
ssh <your-username>@silentguardian.local
```

**You did it when:** you see a command prompt that starts with your username on the Pi.

If `.local` does not work, find the Pi's IP address in your Wi-Fi router's page, and use `ssh username@192.168.x.x`.

Now update it:

```bash
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```

---

## Part 3. Plug the ear (microphone) into the brain

### 3A. Look at your microphone

The INMP441 board has **6 holes or pins** with tiny letters:

| Label on microphone | Meaning for a kid |
|---|---|
| **VDD** | "Food" for the microphone (3.3 V power) |
| **GND** | The way home (ground) |
| **SCK** | A ticking clock so the ear and brain stay in step |
| **WS** | Says "left ear" or "right ear" |
| **SD** | The line carrying the sound numbers to the brain |
| **L/R** | A switch: touch ground = left ear |

**Does your board have metal pins already stuck into it?**
- Yes, pins are soldered: great.
- No, the pins are in a little plastic bag: someone must solder them on (a lab technician can do this in 5 minutes). Do not try to hold loose pins in the holes and hope; the sound will be crackly.

### 3B. Look at the Pi's pin fence

The Pi has a **fence of 40 metal pins** on one edge. It is called the **GPIO header**.

Hold the Pi like this: **the USB and Ethernet ports on the right**, the fence of pins along the **top edge**.

```text
                     (top edge of the board)
 left end                                                      right end
 pin 2  pin 4  pin 6  pin 8  pin 10 pin 12 ........ pin 38 pin 40    <- TOP row (even numbers)
 pin 1  pin 3  pin 5  pin 7  pin 9  pin 11 ........ pin 37 pin 39    <- BOTTOM row (odd numbers)
 (3.3V)                                                  
```

- Pin 1 is the **bottom-left** end pin.
- Pins 2 and 4 (top-left) are **5 V. Do not use them for the microphone.**

### 3C. The six wires

Connect exactly this. **Pi unplugged first.**

| # | Wire from microphone | Goes to Pi physical pin | Pi pin's name | Suggested wire colour |
|---|---|---|---|---|
| 1 | VDD | **Pin 1** | 3.3 V power | Red |
| 2 | GND | **Pin 6** | Ground | Black |
| 3 | SCK | **Pin 12** | GPIO 18 (clock) | Yellow |
| 4 | WS | **Pin 35** | GPIO 19 (left/right) | Green |
| 5 | SD | **Pin 38** | GPIO 20 (data in) | Blue |
| 6 | L/R | **Pin 39** | Ground | Black or white |

This wiring (VDD to 3.3 V, GND to ground, SCK to GPIO18, WS to GPIO19, SD to GPIO20, L/R to ground for the left channel) follows Adafruit's and Maker Portal's published I2S microphone guides for the Raspberry Pi. [Adafruit](https://learn.adafruit.com/adafruit-i2s-mems-microphone-breakout/raspberry-pi-wiring-test) · [Maker Portal](https://makersportal.com/blog/recording-stereo-audio-on-a-raspberry-pi)

How to find each pin:
- **Pin 1:** bottom row, the very first pin on the left.
- **Pin 6:** top row, the 3rd pin from the left.
- **Pin 12:** top row, the 6th pin from the left.
- **Pin 35:** bottom row, the 18th pin from the left.
- **Pin 38:** top row, the 19th pin from the left.
- **Pin 39:** bottom row, the 20th pin from the left (the last one).

Tip: the label "GPIO 18" is not the same as "pin 18". Use the **pin number** column in the table.

### 3D. Which jumper wires?

- If the microphone's pins stick out and you plug **wires directly** onto them, you need **female-to-female** wires (a hole on each end) for the Pi side.
- If you put the microphone on the breadboard, you need **male-to-female** wires from the breadboard to the Pi.

Your kit lists only "jumper wires" and a breadboard. Check which kind you have (see the missing-components file).

### 3E. Double-check before power

Tick each:
- [ ] Pin 1 has the VDD wire (not pin 2!).
- [ ] No wire is on pins 2 or 4 (5 V).
- [ ] Pin 6 and pin 39 are both ground wires.
- [ ] No bare wire ends touch each other.
- [ ] Wires are short and neat (under about 20 cm).

Now plug in the Pi power.

### 3F. Teach the Pi how to hear

Log in with SSH. Edit the settings file:

```bash
sudo nano /boot/firmware/config.txt
```

(On older Raspberry Pi OS the file is `/boot/config.txt`.)

At the bottom add these lines:

```text
dtparam=i2s=on
dtoverlay=googlevoicehat-soundcard
```

Adafruit's guide uses `dtoverlay=googlevoicehat-soundcard` for I2S microphones such as this. [Adafruit](https://learn.adafruit.com/adafruit-i2s-mems-microphone-breakout/raspberry-pi-wiring-test) Save with `Ctrl+O`, `Enter`, `Ctrl+X`. Then:

```bash
sudo reboot
```

### 3G. Test the ear

```bash
arecord -l
```

You should see a new "card" listed (name may include `googlevoicehat`). Note its number.

Record 10 seconds while you talk. Replace `1` with your card number if different:

```bash
arecord -D plughw:1 -c1 -r 48000 -f S32_LE -t wav -d 10 test.wav
```

Play it on your laptop (copy it with `scp`) or on the Pi if a speaker is attached:

```bash
scp <username>@silentguardian.local:test.wav .
```

**You did it when:** you can hear your voice, not just hiss. If the file is quiet, that is normal for this microphone; the program will normalise loudness.

If the card does not appear, go to the troubleshooting table at the end.

### 3H. Check 16 kHz

The classifier uses 16,000 samples per second. Try recording directly at that rate:

```bash
arecord -D plughw:1 -c1 -r 16000 -f S16_LE -t wav -d 5 test16.wav
```

If this fails or sounds wrong, keep recording at 48 kHz and let the Python code shrink it to 16 kHz.

---

## Part 4. The little helper: ESP32 (optional)

The ESP32 is **not** needed for classification. It is a bonus helper that lights up when the Pi hears something.

**Simplest wiring: one USB cable.**

1. Plug the ESP32 into a Pi USB port with a data-capable USB cable.
2. The Pi finds it as `/dev/ttyUSB0` or `/dev/ttyACM0`. Check with:

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

No jumper wires are needed. The full guide is in `05_ESP32_AND_ALERTS.md`.

---

## Part 5. The juice box and babysitter, step by step (battery and solar)

**Big truth:** your solar parts are far too small to run the Pi. A Pi uses about 3 W; your panel gives 1.3 W at best. The solar setup in this project is a **small learning experiment** for charging a battery, perhaps to power the ESP32. Power the Pi from the wall adapter.

### 5A. Meet the solar panel

The panel's sticker says: 1.3 W, 7.5 V at best power, 180 × 90 mm.

- "7.5 V" is the voltage when the panel is working its hardest. With nothing connected, a panel makes a **higher** number (the open-circuit voltage). It can be roughly 9 V or more. You must measure it.

### 5B. Meet the battery

- A 2500 mAh Li-ion battery stores about 9 Wh (watt-hours).
- It has a + and a − end. The holder usually has a red wire (+) and a black wire (−).
- A bare 18650-style cell may have **no protection circuit**. Your solar module must provide protection, or the cell should be a protected one. See the missing-components file.

### 5C. Find out what your solar module is

Look for the **name printed on the board** (for example "DFR0559", "CN3791", "TP4056", "DFR0535"). Search the name online and find "input voltage" for the solar side.

**The panel and module must match.**

| If the module accepts about... | Your 7.5 V panel (open-circuit probably higher) |
|---|---|
| 4.4–6 V (for example DFRobot DFR0559). Its maker warns against going above about 6.5 V. [DFRobot](https://wiki.dfrobot.com/dfr0559/docs/21586) | **Do not connect. Too high. Risk of damage.** |
| 1–3 V (for example DFR0535) | **Do not connect.** |
| 7–30 V (for example DFRobot "9V/12V/18V" Solar Power Manager) [DFRobot](https://www.dfrobot.com/product-1714.html) | **Possible,** but 7.5 V is close to the bottom of its range, so it may stop charging when light is weak. |
| Unknown / no name | **Do not connect.** Ask the person who gave you the part, or replace it. |

### 5D. Measure the panel (needs a multimeter, ask an adult or lab staff)

1. Take the panel outside in bright sunlight, facing the sun.
2. Set the multimeter to **DC voltage** (a V with a straight line, 20 V range).
3. Touch the red probe to the panel's + wire and the black probe to −.
4. Write the number: ____ V. This is the open-circuit voltage.
5. Look at the module's maximum solar input. Is your number **safely below** it? If not, stop.

If the multimeter shows a negative number, you have the wires swapped. That is OK for measuring, but important to know before connecting.

### 5E. Plan the connection (only if the numbers match)

The module has labelled places, for example `SOLAR IN`, `BAT IN` and an output (USB or 5 V).

Typical order (always follow your module's manual first):
1. **Nothing is connected to the module.**
2. Connect the battery holder to `BAT IN`: red to +, black to −. Check twice.
3. Look at the module's lights (if any). Nothing should smell hot.
4. Connect the solar panel to `SOLAR IN`: panel + to +, panel − to −.
5. Place the panel in sunlight. A charging light should come on, depending on the module.
6. Plug a tiny load into the output, such as the ESP32 through a USB cable.

### 5F. Check that it charges

With the multimeter, measure the **battery voltage** (across the battery + and −) before and after an hour in the sun.
- A full Li-ion cell is about **4.2 V**.
- An empty one is about 3.0–3.3 V; do not run it flat.
- The number should slowly go up in sunshine.

**You did it when:** the battery voltage rises in sunlight, nothing gets hot, and the ESP32 stays powered from the module output.

### 5G. Things not to do
- Do not charge a battery from a module that is not designed for single-cell Li-ion.
- Do not run Pi power from the module's output. Its 5 V output is only about 1–1.5 A for the DFRobot 9/12/18 V module, and the Pi needs about 3 A peaks. [DFRobot](https://www.dfrobot.com/product-1714.html)
- Do not leave the battery charging unattended during first tests.
- Do not bend the panel.

---

## Part 6. Put it all together (the demo build)

### The demo setup (recommended)

```text
Wall socket -> 5.1 V 3 A USB-C adapter -> Raspberry Pi 4B
Microphone -> 6 wires -> Pi pins
(optional) ESP32 -> USB cable -> Pi
(optional, separate experiment) Solar panel -> module -> battery -> ESP32 or LED
```

Steps:
1. Put the Pi and microphone in a small box. Leave a hole for the microphone's little sound port (a tiny hole on the microphone board). Do not block it.
2. Keep wires short and tidy. Use tape or a zip tie so nothing wiggles loose.
3. Keep the microphone away from the Pi's fan or heatsink noise. A few centimetres helps.
4. Keep the ESP32 a little away from the microphone wires.
5. Keep the battery and solar module outside the Pi box and away from heat.

### Cooling
The Pi 4 gets warm when it runs a program all the time. A small heatsink or case with a fan helps. Check temperature:

```bash
vcgencmd measure_temp
```

---

## Part 7. Measure power (for your report)

If you have a USB power meter:
1. Plug it between the wall adapter and the Pi.
2. Record the watts when: idle, recording audio, running the classifier.

Do not guess battery life. Measure it.

---

## Troubleshooting

| What you see | What it probably means | What to try |
|---|---|---|
| `arecord -l` shows no new card | The I2S lines are not set up or wires are wrong | Check `config.txt` lines; check pins 12, 35, 38; reboot |
| Recording is silent | Mic not powered, L/R or SD wrong | Check VDD on pin 1 and GND on pin 6; check pin 38 |
| Recording is only crackle | Loose wires or very long wires | Shorten the wires; push connectors in firmly |
| Voice is very quiet | Normal for this microphone and 32-bit format | Speak close; the software boosts loudness |
| Pi restarts or shows a lightning bolt | Weak power supply | Use a 5.1 V 3 A adapter and a good cable |
| `vcgencmd get_throttled` is not `0x0` | The Pi has had power problems | Better adapter, shorter cable |
| ESP32 not found | Cable is charge-only or wrong port | Use a data USB cable; look at `ls /dev/tty*` |
| Solar module shows no charging | No sun, wrong polarity, panel too weak, voltage mismatch | Disconnect; measure panel; read module manual |
| Anything hot or smelly | Short circuit or wrong connection | **Unplug right away.** Do not touch the battery until cool. |

---

## Hardware completion checklist

- [ ] Pi boots and I can connect with SSH.
- [ ] Microphone is wired exactly as in Part 3C and double-checked.
- [ ] `arecord -l` shows the microphone and a 10 s recording sounds like speech.
- [ ] I tested recording at 16 kHz and 48 kHz and noted which worked.
- [ ] Pi runs from a 5.1 V 3 A adapter, not a weak charger.
- [ ] ESP32 is detected over USB (if used).
- [ ] Solar module model is identified, and its input range is written down.
- [ ] Panel open-circuit voltage is measured and is safely inside the module's range (or solar is skipped).
- [ ] Battery voltage was measured before and after charging.
- [ ] Wires are neat, tied down, and the microphone hole is clear.
