# ESP32 Alert Beacon (optional)

## 1. What the ESP32 does here

The Pi does all the listening and thinking. The ESP32 is a small second device that receives a short message from the Pi and shows an alert: it blinks an LED, beeps a buzzer, or sends the alert onward over Wi-Fi.

| Role | Status |
|---|---|
| Run the sound classifier | **No.** The Pi does this. |
| Show a local alert (LED / buzzer) | Yes, easiest. |
| Send alerts over Wi-Fi or ESP-NOW | Possible with a second ESP32. |
| Long-range LoRa | Not possible with the parts you have. You would need a LoRa radio module (see `06_MISSING_COMPONENTS.md`). |

## 2. Why not put the model on the ESP32?

The ESP32 (without S3 vector hardware) is much weaker than a Pi 4B. A tiny model could be made to fit, but it needs different model sizing and a different audio pipeline. Keeping the classifier on the Pi is simpler and more reliable for your first working version.

## 3. Connection

One USB data cable from the ESP32 to a Pi USB port. The cable carries power and serial text.

```text
Pi USB port ---- USB data cable ---- ESP32
```

Find the port:

```bash
ls /dev/ttyUSB* /dev/ttyACM*
```

## 4. ESP32 sketch (Arduino IDE)

Install Arduino IDE on the laptop, add the ESP32 board package, choose your board (for example "ESP32 Dev Module"), and upload:

```cpp
// alert_beacon.ino
const int LED_PIN = 2;   // many ESP32 dev boards use GPIO2 for the on-board LED; check yours

void blinkAlert() {
  for (int i = 0; i < 6; i++) {
    digitalWrite(LED_PIN, !digitalRead(LED_PIN));
    delay(150);
  }
  digitalWrite(LED_PIN, LOW);
}

void setup() {
  Serial.begin(115200);
  pinMode(LED_PIN, OUTPUT);
  Serial.println("BEACON_READY");
}

void loop() {
  if (Serial.available()) {
    String line = Serial.readStringUntil('\n');
    line.trim();
    if (line.startsWith("ALERT")) {
      blinkAlert();
      Serial.println("ACK " + line);
    }
  }
}
```

If you hold the BOOT button while uploading is required on your board, follow its documentation.

## 5. Pi side: send an alert

```python
# send_alert.py
import serial, time
ser = serial.Serial("/dev/ttyUSB0", 115200, timeout=1)   # may be /dev/ttyACM0
time.sleep(2)                                              # let the ESP32 start
def send(cls, conf):
    ser.write(f"ALERT,{cls},{conf:.2f}\n".encode())
    print(ser.readline().decode().strip())
send("chainsaw", 0.93)
```

Call `send(...)` from `run_sg.py` when an event is accepted.

Serial permission: add your user to the `dialout` group.

```bash
sudo usermod -a -G dialout $USER && sudo reboot
```

## 6. Optional: a second ESP32 receiver

If you buy a second ESP32, one can send **ESP-NOW** messages (a Wi-Fi-based direct link) to the other. That gives a wireless alert across a room or building, but it is short-range and not LoRa. Document the range you measure; do not promise forest range.

## 7. Optional: ESP32 on the solar battery

The ESP32 can be powered from the solar module's output, as a learning experiment.

- A Wi-Fi ESP32 can briefly draw a few hundred milliamps when transmitting. Use the multimeter or a USB meter to measure; do not assume.
- If the ESP32 runs all day on Wi-Fi it can use more energy than a 1.3 W panel can reliably supply. Measure over time.

## 8. Tests

| Test | Pass |
|---|---|
| Send `ALERT,chainsaw,0.93` from the Pi | LED blinks and the Pi prints `ACK` |
| Disconnect and reconnect the ESP32 | The Pi script reconnects or reports an error |
| 30 alerts in a row | No missed acknowledgements |

## 9. Limitations

- USB serial is local: the ESP32 is attached to the Pi.
- There is no encryption or authentication.
- This is an alert demonstration, not a safety system.
