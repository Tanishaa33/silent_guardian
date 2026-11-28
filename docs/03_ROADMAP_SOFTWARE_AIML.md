# Software and AI/ML Roadmap

Step-by-step, from empty folder to live classification on the Raspberry Pi 4B.

Scripts here are reference code. They have not been run against your real audio. Test each on a small sample first, and check array shapes.

---

## Phase 0: Laptop setup (half a day)

**Goal:** A clean Python environment that can train and convert models.

```bash
python -m venv .venv
# Windows: .venv\Scripts\activate     Linux/Mac: source .venv/bin/activate
pip install numpy pandas scipy soundfile librosa scikit-learn matplotlib tqdm requests
pip install tensorflow           # for training + TFLite conversion
pip freeze > requirements.lock.txt
```

Create the folders from the README layout. Put `combined_shuffled-1.csv` in `data/`.

**Done when:** `python -c "import tensorflow, librosa; print('ok')"` prints `ok`.

---

## Phase 1: Build the manifest (1 day)

**Goal:** One clean table that says which audio file is which class, and which group it belongs to.

Run `ml/build_manifest.py`:

```python
import pandas as pd, re
d = pd.read_csv("data/combined_shuffled-1.csv")
d["xc"] = d.url.str.extract(r"xeno-canto\.org/(\d+)")[0]

bad = d.dropna(subset=["xc"]).groupby("xc").primary_label.nunique()
d = d[~d.xc.isin(bad[bad > 1].index)]                 # 38 conflicting rows

def label4(r):
    p = r.primary_label
    return "chainsaw" if p == "chainsaw" else "gunshot" if p.startswith("gunshot") else "bird"
d["label4"] = d.apply(label4, axis=1)

syn = (d.label4 == "gunshot") & d.url.str.contains("Codec-SUPERB")
d = d[~syn | d.filename.str.startswith("original_")]   # keep originals only

def group(r):
    if r.label4 == "bird": return f"xc_{r.xc}"
    if r.label4 == "chainsaw": return re.sub(r"_\d+-\d+\.wav$", "", r.filename)
    if "Codec-SUPERB" in r.url: return "syn_" + re.search(r"_(\d+)\.wav", r.filename).group(1)
    return "kaggle_" + r.filename
d["group"] = d.apply(group, axis=1)

bird = d[d.label4 == "bird"].groupby("primary_label").head(60)   # cap per species
d = pd.concat([bird, d[d.label4 != "bird"]]).reset_index(drop=True)
d.to_csv("data/manifest_raw.csv", index=False)
print(d.label4.value_counts())
```

**Done when:** the printed counts look sensible (about 8,049 bird, 3,500 chainsaw, some gunshot rows) and no `group` is empty.

Check the synthetic gunshot filename pattern in the CSV before running; the `original_` prefix and the number pattern are taken from the CSV filenames.

---

## Phase 2: Download the pilot data (1–2 days)

**Goal:** Audio files on disk for every row in `manifest_raw.csv`.

| Source | How |
|---|---|
| Xeno-canto birds | Download from the page URLs in the CSV. Check Xeno-canto's current terms and use their official API if bulk access is blocked. Rate-limit your requests (for example, one per second). |
| Chainsaw (`rfcx/frugalai`) | Gated dataset. Create a Hugging Face account, accept the conditions, `huggingface-cli login`, then load it with the `datasets` library and save only the filenames in your manifest. |
| Synthetic gunshots | Download the 88 `original_*.wav` files using the URLs in the CSV. |
| Kaggle gunshots | `pip install kaggle`, add your API token, then `kaggle datasets download emrahaydemr/gunshot-audio-dataset`. |
| Background | ESC-50 (rain, wind, insects, engine and similar) plus your own recordings (Phase 4). |

Put files under `data/raw/<class>/` and record the final local path in a new `path` column.

**Done when:** every manifest row has a file that exists and opens. Write a script that lists missing files.

---

## Phase 3: Clean the audio (1–2 days)

**Goal:** Remove unusable and duplicate files.

`ml/clean_audio.py` should, for every file:
1. Decode with `soundfile` (for `.ogg` and `.wav`).
2. Record: duration, original sample rate, channels, RMS, peak, fraction of samples near full scale.
3. Compute a SHA-256 hash of the file bytes.
4. Mark as bad: decode error, duration under 0.5 s, RMS near zero, extremely clipped.
5. Mark duplicates by hash. Keep one copy.
6. Save `manifest_clean.csv` plus a `cleaning_report.csv` (what was removed and why).

Then:
- Listen to 10 random files per class.
- Plot 5 spectrograms per class.
- Look for near-duplicates among Kaggle gunshots (similar spectrogram embeddings or identical waveforms after trimming).

**Done when:** you can explain every removed file, and the class counts are still reasonable.

---

## Phase 4: Add background data and your own microphone (2–3 days)

**Goal:** Handle "none of the above" and test the real microphone.

1. Add ESC-50 non-target classes to the `background` class. Keep them as separate source groups.
2. After the hardware is wired (hardware roadmap Phase 3), record on the Pi:
   - 20–30 minutes of quiet room and outdoor background.
   - Speech, fan, door slam, clap, keyboard (hard negatives).
   - Playback of bird, chainsaw and gunshot-like clips from a phone at several volumes and distances.
3. Save these in `data/own_mic/` with a session ID in the filename.
4. Split sessions: some for training, some **only for the final own-microphone test**.

**Done when:** you have an own-microphone test set (at least 100 windows per class) that training never sees.

Ask people before recording voices, and delete any private speech you do not need.

---

## Phase 5: Split and extract features (1 day)

**Goal:** `features.npz` with features, labels and split.

1. Assign splits by group (70/15/15, fixed seed) and write them into the manifest.
2. Use the shared `sg_features.py`:

```python
import numpy as np, librosa
SR = 16000; N = 16000
def logmel(x):
    x = x.astype(np.float32) - np.mean(x)
    rms = np.sqrt(np.mean(x * x)) + 1e-8
    x = np.clip(x * min(10.0, 0.1 / rms), -1, 1)
    m = librosa.feature.melspectrogram(
        y=x, sr=SR, n_fft=1024, hop_length=256, n_mels=64,
        fmin=50, fmax=8000, power=2.0)
    return librosa.power_to_db(m, ref=1.0, top_db=80).astype(np.float32)   # (64, 63)
```

3. Run `ml/preprocess.py` (windowing rules in the AI/ML workflow). Resample with `librosa.resample` when the file is not 16 kHz.

**Checks before moving on:**
- `X.shape[1:] == (64, 63)`.
- Counts per class and split are printed and balanced enough.
- No group appears in two splits. Write an assert for this.

---

## Phase 6: Train the baseline and CNN (2–3 days)

**Goal:** A trained model and an honest evaluation.

1. Baseline: 13 MFCC means and standard deviations → logistic regression. Record macro-F1.
2. CNN: use the small Keras model in `02_AIML_WORKFLOW.md`.

```python
import numpy as np, tensorflow as tf
from tensorflow.keras import layers as L
d = np.load("data/features.npz"); X, y, s = d["X"][..., None], d["y"], d["split"]
tr, va, te = [s == k for k in ("train", "val", "test")]
mu, sd = X[tr].mean(), X[tr].std(); X = (X - mu) / sd
m = tf.keras.Sequential([
  L.Input((64, 63, 1)),
  L.Conv2D(16, 3, padding="same", activation="relu"), L.BatchNormalization(), L.MaxPool2D(2),
  L.Conv2D(32, 3, padding="same", activation="relu"), L.BatchNormalization(), L.MaxPool2D(2),
  L.Conv2D(64, 3, padding="same", activation="relu"), L.BatchNormalization(), L.MaxPool2D(2),
  L.GlobalAveragePooling2D(), L.Dropout(0.3), L.Dense(4)])
m.compile("adam", tf.keras.losses.SparseCategoricalCrossentropy(from_logits=True), ["accuracy"])
cw = {i: len(y[tr]) / (4 * max(1, (y[tr] == i).sum())) for i in range(4)}
m.fit(X[tr], y[tr], validation_data=(X[va], y[va]), epochs=60, batch_size=64,
      class_weight=cw,
      callbacks=[tf.keras.callbacks.EarlyStopping(patience=8, restore_best_weights=True)])
np.save("artifacts/norm.npy", np.array([mu, sd]))
```

3. Evaluate on the test split only once the model is chosen. Save: classification report, confusion matrix image, training curves.
4. Add augmentation only if needed: random gain, background mixing at 5–20 dB SNR, small time shift. Apply it to training data only.

**Done when:** you have `results/test_report.txt` and `results/confusion_matrix.png`, and you can explain the biggest confusion.

---

## Phase 7: Evaluate on your own microphone (1 day)

**Goal:** Learn whether the model works outside the dataset.

1. Extract features from `data/own_mic/test/` using the same normalisation.
2. Report the same metrics as Phase 6, in a separate table.
3. Compare the two. A large gap means domain mismatch.
4. Fix the biggest problems by adding **training** sessions of your own recordings, not by touching the test sessions.

**Done when:** you have both tables and a short written explanation of the gap.

---

## Phase 8: Convert to TFLite (half a day)

```python
c = tf.lite.TFLiteConverter.from_keras_model(m)
open("artifacts/sg.tflite", "wb").write(c.convert())
```

Check the converted model on the laptop: load it with `tf.lite.Interpreter`, run the test set, and compare predictions with the Keras model. They should match almost exactly.

Optionally try dynamic-range quantisation (`c.optimizations = [tf.lite.Optimize.DEFAULT]`) and compare accuracy and speed.

---

## Phase 9: Pi software setup (half a day)

Complete hardware Phase 3 first so the microphone works.

```bash
sudo apt update && sudo apt full-upgrade -y
sudo apt install -y python3-venv python3-pip libsndfile1 libportaudio2 libatlas-base-dev
python3 -m venv ~/sg --system-site-packages && source ~/sg/bin/activate
pip install numpy librosa soundfile sounddevice pyserial
pip install ai-edge-litert        # if this fails on your Python version: pip install tflite-runtime
```

Copy `sg.tflite`, `norm.npy` and `sg_features.py` from the laptop (`scp` or a USB drive).

**Done when:** `python -c "from sg_features import logmel; import numpy as np; print(logmel(np.random.randn(16000)).shape)"` prints `(64, 63)`.

---

## Phase 10: Live inference on the Pi (2 days)

**Goal:** Real-time classification from the microphone.

Reference loop (adjust the audio device and rate to what Phase 3 of the hardware roadmap found works):

```python
import numpy as np, sounddevice as sd, time, csv
from collections import deque
try:    from ai_edge_litert.interpreter import Interpreter
except: from tflite_runtime.interpreter import Interpreter
from sg_features import logmel, SR, N

LABELS = ["background", "bird", "chainsaw", "gunshot"]
THRESH = {"bird": 0.75, "chainsaw": 0.85, "gunshot": 0.90}
mu, sd_ = np.load("norm.npy")
it = Interpreter("sg.tflite"); it.allocate_tensors()
inp, out = it.get_input_details()[0], it.get_output_details()[0]

buf = deque(maxlen=N)
def cb(indata, frames, t, status):
    buf.extend(indata[:, 0])

log = open("events.csv", "a", newline=""); w = csv.writer(log)
recent = deque(maxlen=2)
with sd.InputStream(samplerate=SR, channels=1, blocksize=4000, callback=cb):
    while True:
        time.sleep(0.5)
        if len(buf) < N: continue
        t0 = time.time()
        x = np.array(buf, dtype=np.float32)
        f = ((logmel(x) - mu) / sd_)[None, :, :, None].astype(np.float32)
        it.set_tensor(inp["index"], f); it.invoke()
        z = it.get_tensor(out["index"])[0]
        p = np.exp(z - z.max()); p /= p.sum()
        k = int(p.argmax()); name = LABELS[k]
        ok = name != "background" and p[k] >= THRESH[name]
        recent.append(name if ok else "none")
        ms = (time.time() - t0) * 1000
        if len(recent) == 2 and recent[0] == recent[1] != "none":
            w.writerow([time.strftime("%Y-%m-%d %H:%M:%S"), name, f"{p[k]:.3f}", f"{ms:.0f}"]); log.flush()
            print(time.strftime("%H:%M:%S"), name.upper(), f"{p[k]:.2f}", f"{ms:.0f} ms")
```

If the microphone only works at 48 kHz, record at 48 kHz and resample each window to 16 kHz with `librosa.resample` before `logmel`.

Tests:
1. Silence: nothing alarming should appear.
2. Speech, clapping, a fan: note false alarms.
3. Phone playback of bird, chainsaw, gunshot-like clips at 3 volumes and 3 distances.
4. A 30-minute run. Record dropped windows and the highest per-window time.

Check CPU, RAM and temperature with `htop` and `vcgencmd measure_temp`. Check power problems with `vcgencmd get_throttled` (it should print `0x0`).

---

## Phase 11: Dashboard and logging (1–2 days)

Simple, reliable option: let `run_sg.py` write `events.csv`, and run a small Streamlit page on the Pi that reads it.

```python
# dashboard.py
import pandas as pd, streamlit as st
st.title("Silent Guardian")
@st.fragment(run_every="2s")
def view():
    try: df = pd.read_csv("events.csv", names=["time", "class", "confidence", "ms"])
    except Exception: st.info("No events yet"); return
    last = df.iloc[-1]
    c1, c2, c3 = st.columns(3)
    c1.metric("Latest", last["class"]); c2.metric("Confidence", f"{last.confidence:.2f}")
    c3.metric("Events logged", len(df))
    st.bar_chart(df["class"].value_counts()); st.dataframe(df.tail(30))
view()
```

```bash
pip install streamlit
streamlit run dashboard.py --server.address 0.0.0.0
```

Open `http://<pi-ip>:8501` from the laptop on the same network. Add a "last heartbeat" time later so a crashed program is visible.

---

## Phase 12: Run automatically on boot (half a day)

Create `/etc/systemd/system/silentguardian.service`:

```ini
[Unit]
Description=Silent Guardian classifier
After=sound.target network.target

[Service]
User=pi
WorkingDirectory=/home/pi/silent-guardian/pi
ExecStart=/home/pi/sg/bin/python run_sg.py
Restart=always

[Install]
WantedBy=multi-user.target
```

```bash
sudo systemctl daemon-reload && sudo systemctl enable --now silentguardian
journalctl -u silentguardian -f
```

Replace `pi` with your actual username if different.

---

## Phase 13: Final measurements and report (3–4 days)

Fill in the PRD acceptance table, plus:

| Measurement | Value |
|---|---|
| Held-out dataset macro-F1 | ____ |
| Own-microphone macro-F1 | ____ |
| False alerts per hour (background) | ____ |
| Median and worst per-window time | ____ ms |
| CPU, RAM, temperature during run | ____ |
| Pi average power (USB meter) | ____ W |
| 30-minute run: crashes, dropped windows | ____ |

Include limitations and a short list of the next improvements.

---

## Common problems

| Problem | Likely cause | Fix |
|---|---|---|
| Very high test score, poor live results | Dataset style shortcut, leakage, no background data | Own-microphone data, group splits, background class |
| Everything predicted `chainsaw` | Loud noise looks similar, or normalisation mismatch | Check `norm.npy` is the training one; add hard negatives |
| Different features on laptop and Pi | Different `sg_features.py` or sample rate | Make copies identical; compare feature arrays |
| Gunshots never detected | Only 1 window rule, synthetic data mismatch | Test the single-window rule; add real recordings |
| Pi program slows down | Heavy `librosa` import work per window or SD writes | Time each stage; process every 0.5 s only |
| Random reboots or freezes | Weak power supply | Use the 3 A supply; check throttling |
