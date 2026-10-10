# Spotify Now Playing Display (Arduino Uno + 1602A LCD)

Show what you're listening to on a 16x2 LCD, with a progress bar, and control Spotify by waving your hand over an ultrasonic sensor.

The Arduino Uno has no WiFi, so a small Python script on your computer talks to Spotify and sends the data to the Uno over the USB cable.

```
Spotify  ->  Python script (PC)  ->  USB serial  ->  Arduino Uno  ->  LCD
                    ^                                      |
                    +------ "NEAR" / "NEXT" messages ------+  (ultrasonic sensor)
```

## Features

- Scrolling `Title - Artist` on the top row
- Elapsed time and a 10-block progress bar on the bottom row (`01:35 ####------`)
- Shows `Paused` and keeps the last song visible when music stops
- Quick hand tap over the sensor: **play / pause**
- Hold your hand over the sensor for 1 second: **next song**
- Endless-loop title scrolling, short titles stay still

## Requirements

**Hardware**
- Arduino Uno R3
- 1602A LCD (16-pin, no I2C backpack)
- 10k potentiometer (LCD contrast)
- 220 ohm resistor (backlight)
- HC-SR04 ultrasonic sensor (optional, for play/pause/skip)
- Breadboard and jumper wires (about 12 for the LCD alone, a jumper pack costs a few euros)
- USB cable

**Software**
- [Arduino IDE](https://www.arduino.cc/en/software)
- [Python 3](https://www.python.org/downloads/) (on Windows, tick "Add Python to PATH")
- Python libraries: `spotipy`, `pyserial`
- A Spotify account. **Spotify Premium is required** for play/pause and skip. Showing the current song works on free accounts.

## 1. Wiring

### Power rails
| From | To |
|---|---|
| Uno 5V | `+` rail |
| Uno GND | `-` rail |

### LCD (1602A)
| LCD pin | Name | Connect to |
|---|---|---|
| 1 | VSS | GND |
| 2 | VDD | 5V |
| 3 | V0 | Potentiometer middle pin |
| 4 | RS | Uno pin 12 |
| 5 | RW | GND |
| 6 | E | Uno pin 11 |
| 7-10 | D0-D3 | not connected |
| 11 | D4 | Uno pin 5 |
| 12 | D5 | Uno pin 4 |
| 13 | D6 | Uno pin 3 |
| 14 | D7 | Uno pin 2 |
| 15 | A (backlight +) | 5V through a 220 ohm resistor |
| 16 | K (backlight -) | GND |

The potentiometer's two outer pins go to 5V and GND. Turn it until the text is clear.

The LCD pins D4-D7 are crossed over on purpose. The sketch line `LiquidCrystal lcd(12, 11, 5, 4, 3, 2);` lists the Uno pins in the order RS, E, D4, D5, D6, D7, so wiring and code must agree.

**Brighter backlight:** use a 100 ohm resistor instead of 220 ohm on pin 15. Do not go below about 47 ohm unless your module has a resistor on the back.

### Ultrasonic sensor (optional, no extra wires)

The HC-SR04 plugs straight into the Uno's analog header, and the code uses two of those pins as 5V and GND:

| Uno pin | Sensor pin | Job |
|---|---|---|
| A0 | VCC | set HIGH in code (5V) |
| A1 | Trig | output |
| A2 | Echo | input |
| A3 | GND | set LOW in code (GND) |

Check that the printed labels on your sensor match this order before plugging it in. The sensor draws about 15 mA, which a Uno pin can supply. This is a handy shortcut, but not ideal for permanent builds. If you have jumper wires, you can wire VCC and GND to the 5V and GND rails instead.

## 2. Install the libraries

Open **Command Prompt** (not the Python shell, the prompt should NOT show `>>>`) and run:

```
pip install spotipy pyserial
```

## 3. Create a Spotify app

1. Go to the [Spotify Developer Dashboard](https://developer.spotify.com/dashboard) and log in.
2. Click **Create app**. Name and description can be anything, leave Website empty.
3. Add this **Redirect URI** and click Add: `http://127.0.0.1:8888/callback`
4. Tick **Web API**, accept the terms, and save.
5. Open **Settings** and copy your **Client ID** and **Client secret**.

**Keep your Client secret private.** Never post it in a public repository, issue, or chat. If it leaks, reset it in the dashboard.

## 4. Upload the Arduino sketch

1. Plug in the Uno.
2. In the Arduino IDE, select *Tools -> Board -> Arduino Uno* and pick your port under *Tools -> Port* (for example `COM3`). Remember the port name.
3. Paste the sketch below and click **Upload**.
4. The screen should show `Waiting...` or `00:00 ----------`. If you see only blocks or nothing, turn the potentiometer.

Use the copy button on the code box and make sure the **last line is a single `}`**. A cut-off paste causes errors like `expected '}' at end of input`.

```cpp
#include <LiquidCrystal.h>

LiquidCrystal lcd(12, 11, 5, 4, 3, 2);   // RS, E, D4, D5, D6, D7
const int TRIG = A1, ECHO = A2;

String text = "Waiting...";
int posT = 0;
unsigned long lastScroll = 0, lastClock = 0, lastPing = 0;
long baseSec = 0, durSec = 0;
unsigned long baseMs = 0;
bool playing = false;

bool nearNow = false, fired = false;
unsigned long nearStart = 0;
int misses = 0;

void drawTop() {
  lcd.setCursor(0, 0);
  int len = text.length();
  if (len <= 16) {
    lcd.print(text);
    for (int i = len; i < 16; i++) lcd.print(' ');
    return;
  }
  String loopText = text + "   ";          // gap before the title repeats
  int n = loopText.length();
  for (int i = 0; i < 16; i++) lcd.print(loopText[(posT + i) % n]);
}

void updateScroll() {
  if (millis() - lastScroll < 400) return;   // title speed: ms per step
  lastScroll = millis();
  if (text.length() > 16) {
    posT = (posT + 1) % (text.length() + 3);
  }
  drawTop();
}

void showBar(long cur) {
  if (cur > durSec) cur = durSec;
  char buf[7];
  sprintf(buf, "%02d:%02d ", (int)((cur / 60) % 100), (int)(cur % 60));
  lcd.setCursor(0, 1);
  lcd.print(buf);
  if (!playing && durSec > 0) {
    lcd.print("Paused    ");
    return;
  }
  int filled = durSec > 0 ? (cur * 10) / durSec : 0;
  for (int i = 0; i < 10; i++) {
    if (i < filled) lcd.write((uint8_t)255);   // solid block
    else lcd.print('-');
  }
}

long readCm() {
  digitalWrite(TRIG, LOW);
  delayMicroseconds(2);
  digitalWrite(TRIG, HIGH);
  delayMicroseconds(10);
  digitalWrite(TRIG, LOW);
  long us = pulseIn(ECHO, HIGH, 25000);
  if (us == 0) return 0;
  return us / 58;
}

void setup() {
  Serial.begin(9600);
  lcd.begin(16, 2);
  pinMode(A0, OUTPUT); digitalWrite(A0, HIGH);   // sensor VCC
  pinMode(A3, OUTPUT); digitalWrite(A3, LOW);    // sensor GND
  pinMode(TRIG, OUTPUT);
  pinMode(ECHO, INPUT);
  drawTop();
  showBar(0);
}

void loop() {
  // Receive: "title - artist|seconds|duration|playing"
  if (Serial.available()) {
    String line = Serial.readStringUntil('\n');
    int p3 = line.lastIndexOf('|');
    int p2 = line.lastIndexOf('|', p3 - 1);
    int p1 = line.lastIndexOf('|', p2 - 1);
    if (p1 > 0 && p2 > p1 && p3 > p2) {
      String newText = line.substring(0, p1);
      baseSec = line.substring(p1 + 1, p2).toInt();
      durSec = line.substring(p2 + 1, p3).toInt();
      playing = line.substring(p3 + 1).toInt() == 1;
      baseMs = millis();
      if (newText != text) {
        text = newText;
        posT = 0;
        drawTop();
      }
    }
  }

  // Ultrasonic gestures
  if (millis() - lastPing > 100) {
    lastPing = millis();
    long d = readCm();
    bool isNear = (d > 0 && d < 15);          // hand closer than 15 cm
    if (isNear) {
      misses = 0;
      if (!nearNow) {
        nearNow = true;
        fired = false;
        nearStart = millis();
      } else if (!fired && millis() - nearStart > 1000) {
        fired = true;
        Serial.println("NEXT");               // held 1 second: skip
      }
    } else if (nearNow && ++misses >= 3) {
      if (!fired) Serial.println("NEAR");     // quick tap: play/pause
      nearNow = false;
    }
  }

  updateScroll();

  if (millis() - lastClock > 200) {            // time and bar redraw rate
    lastClock = millis();
    showBar(playing ? baseSec + (millis() - baseMs) / 1000 : baseSec);
  }
}
```

## 5. The Python script

Save this as `nowplaying.py`. Fill in your Client ID, Client secret, and COM port.

If you use the Arduino IDE to edit it by accident, it may create a folder with that name. Use Notepad (*Save as type: All files*) or any code editor instead.

```python
import time, serial, spotipy
from spotipy.oauth2 import SpotifyOAuth

sp = spotipy.Spotify(auth_manager=SpotifyOAuth(
    client_id="YOUR_CLIENT_ID",
    client_secret="YOUR_CLIENT_SECRET",
    redirect_uri="http://127.0.0.1:8888/callback",
    scope="user-read-currently-playing user-read-playback-state user-modify-playback-state",
    cache_path="spotify_token.cache"))   # use a full path if you get a cache error

ser = serial.Serial("COM3", 9600, timeout=0.1)   # Mac/Linux: "/dev/ttyACM0" or similar
time.sleep(2)   # the Uno resets when the port opens

is_playing = False
last_poll = 0
last_info = None   # (text, seconds, duration) of the last song seen

while True:
    # 1) Messages from the ultrasonic sensor
    if ser.in_waiting:
        msg = ser.readline().decode(errors="ignore").strip()
        try:
            if msg == "NEAR":
                if is_playing:
                    sp.pause_playback()
                else:
                    sp.start_playback()
                last_poll = 0
                time.sleep(0.5)
            elif msg == "NEXT":
                sp.next_track()
                last_poll = 0
                time.sleep(0.5)
        except Exception as e:
            print(e)

    # 2) Update the LCD every 2 seconds
    if time.time() - last_poll > 2:
        last_poll = time.time()
        try:
            t = sp.current_user_playing_track()
            if t and t.get("item"):
                item = t["item"]
                is_playing = bool(t.get("is_playing"))
                last_info = (
                    f'{item["name"]} - {item["artists"][0]["name"]}',
                    t["progress_ms"] // 1000,
                    item["duration_ms"] // 1000,
                )
            else:
                is_playing = False   # keep showing the last song

            if last_info:
                text, secs, dur = last_info
                line = f'{text}|{secs}|{dur}|{1 if is_playing else 0}'
            else:
                line = "Nothing playing|0|0|0"
            ser.write((line + "\n").encode("ascii", "replace"))
        except Exception as e:
            print(e)

    time.sleep(0.05)
```

## 6. Run it

1. Plug in the Uno and **close the Arduino IDE Serial Monitor** (only one program can use the port).
2. Open **Command Prompt** and run:
   ```
   python path\to\nowplaying.py
   ```
3. The first time, a browser opens. Click **Agree**. If the page afterwards cannot be reached, copy the full URL from the address bar and paste it into the Command Prompt window when asked.
4. Play a song on Spotify (desktop app, phone, or web player all work, as long as it is the same account).
5. The song appears on the LCD within a couple of seconds.

Keep the Command Prompt window open. Closing it stops the updates.

## Gestures

| Gesture | Action |
|---|---|
| Quick tap, hand within 15 cm, then away | Play / pause |
| Hold hand within 15 cm for 1 second | Next song |

## Settings you can tweak

All in the Arduino sketch:

| Setting | Where | Effect |
|---|---|---|
| `lastScroll < 400` | `updateScroll()` | Title scroll speed. Smaller is faster. |
| `lastClock > 200` | `loop()` | How often time and bar redraw. Only affects the bottom row. |
| `d < 15` | `loop()` | How close your hand must be (cm). |
| `> 1000` | `loop()` | How long to hold for "next song" (ms). |
| `"   "` and `+ 3` | `drawTop()` / `updateScroll()` | Gap between title repeats. Keep both in sync. |
| `(cur * 10) / durSec` | `showBar()` | Bar length. Change `10` and the loop to resize it. |

## Troubleshooting

| Problem | Fix |
|---|---|
| `>>>` prompt and `IndentationError` or `NameError` | You typed a command inside Python. Type `exit()`, then run it in Command Prompt. |
| `can't open file ...` | Wrong folder. Use the full path to the script. |
| `can't find '__main__' module` | The "file" is actually a folder. Re-save the script under a new name. |
| `could not open port` | Close the Serial Monitor and any other running copy of the script. Check the COM port. |
| `invalid_client` | Wrong Client ID or secret. Copy both again from the same app, with no spaces. |
| `Couldn't write token to cache` | Set a full, writable `cache_path`, or run from a folder you own. |
| Screen only shows `Waiting...` | Sketch and script versions do not match, or the script is not running. |
| `expected '}' at end of input` | The sketch was cut off when pasting. Clear the editor and paste again. |
| Blocks or blank screen | Turn the contrast potentiometer. Recheck pins 1, 2, 5, 15, 16. |
| Garbled text | Data pins 11-14 are wired in the wrong order. |
| `Nothing playing` while music plays | Same account? Private Session off? Music actually playing? |
| 403 / "Premium required" | Play, pause, and skip need Spotify Premium. |
| `NO_ACTIVE_DEVICE` | Open Spotify on any device and play something once. |
| Gestures do nothing | Check sensor orientation (A0 = VCC, A3 = GND) and the `d < 15` distance. |
| Need new permissions | Delete `spotify_token.cache` and log in again. |

## Limitations

- The computer must be on and the script running.
- The LCD cannot show accented or non-Latin characters, they appear as `?`.
- Play, pause, and skip need Spotify Premium.
- The display updates about every 2 seconds, and the timer is estimated between updates.

## Going further

- Use an ESP32 (it has WiFi) to run it without a computer.
- Add a bigger display, such as a TM1637 clock module or a 5641AS 4-digit 7-segment display.
- Add buttons for previous track or volume.

## Credits

Built with [Spotipy](https://spotipy.readthedocs.io/), [pySerial](https://pyserial.readthedocs.io/), and the Arduino `LiquidCrystal` library.
