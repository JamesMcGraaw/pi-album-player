NFC Spotify Album Player — Final Build Documentation
Hardware: Raspberry Pi 5 + Waveshare PN532 NFC HAT + MOSWAG USB Audio Adapter  
Built by: jamesmcgraaw | Hostname: albumplayer
---
Hardware
Raspberry Pi 5
Waveshare PN532 NFC HAT
MOSWAG USB to 3.5mm audio adapter (card 0 on this build)
NTAG NFC sticker tags
Speakers/headphones via 3.5mm jack
---
Part 1 — Raspberry Pi OS Setup
Flash the SD Card
Open Raspberry Pi Imager
Choose Device → Raspberry Pi 5
Choose OS → Raspberry Pi OS (other) → Raspberry Pi OS Lite (64-bit)
Click Edit Settings before writing:
Hostname: `albumplayer`
Enable SSH → password authentication
Username: your own
Password: avoid special characters
WiFi: enter SSID and password, country `GB`
Timezone: `Europe/London`
Write to SD card
First Boot & Update
```bash
ssh yourusername@albumplayer.local
sudo apt update && sudo apt full-upgrade -y
sudo reboot
```
> If you get a "WARNING: REMOTE HOST IDENTIFICATION HAS CHANGED" error after re-flashing, run this on your computer first:
> ```bash
> ssh-keygen -R albumplayer.local
> ```
---
Part 2 — Raspotify (Spotify Connect)
Install
```bash
sudo apt-get -y install curl
curl -sL https://dtcooper.github.io/raspotify/install.sh | sh
```
Configure `/etc/raspotify/conf`
The working configuration (non-commented lines only):
```
LIBRESPOT_BACKEND="alsa"
LIBRESPOT_BITRATE=320
LIBRESPOT_DEVICE="plughw:0,0"
LIBRESPOT_DISABLE_AUDIO_CACHE=
LIBRESPOT_DISABLE_CREDENTIAL_CACHE=
LIBRESPOT_ENABLE_VOLUME_NORMALISATION=
LIBRESPOT_NAME="Album Player"
LIBRESPOT_QUIET=
TMPDIR=/tmp
```
> **Important:** Use `plughw:0,0` not `hw:0,0`. The MOSWAG adapter doesn't natively support 44100Hz — `plughw` tells ALSA to handle the format conversion automatically. Using `hw:0,0` directly causes an "Unsupported Sample Rate" error and raspotify crashes.
> **Finding your audio device:** Run `aplay -l` to list playback devices. The MOSWAG USB adapter appeared as `card 0` on this build. If yours is different, update `plughw:X,0` accordingly.
```bash
sudo systemctl restart raspotify
sudo systemctl status raspotify   # Should show active (running)
```
---
Part 3 — Waveshare PN532 NFC HAT
Key Pi 5 Notes
The Pi 5's SPI GPIO header bus appears as `/dev/spidev10.0` (not `spidev0.0` as on older Pi models)
Use Waveshare's own Python library — third-party libraries (e.g. Adafruit CircuitPython PN532) have GPIO compatibility issues on Pi 5
The chip select pin is BCM GPIO 4 (not CE0/GPIO8 as you might expect)
The reset pin is BCM GPIO 20
HAT Physical Configuration
Jumpers (I0 and I1):
```
       I1  I0
UART:  L   L
I2C:   L   H
SPI:   H   L   ← this one
```
Left jumper (I1) = H
Right jumper (I0) = L
The two jumpers must be in different positions
DIP Switches:
Switch	Label	Position
1	SCK	ON
2	MISO	ON
3	MOSI	ON
4	NSS	ON
5	SCL	OFF
6	SDA	OFF
7	RX	OFF
8	TX	OFF
RSTPDN jumper: Connect the RSTPDN pin to D20 using the jumper cap on the HAT.
Seating the HAT: Press down firmly and evenly on all four corners. The Pi 5 has slightly shorter GPIO pins than older models — the HAT requires more force than expected to fully seat.
`/boot/firmware/config.txt`
The relevant lines that must be present:
```
dtparam=i2c_arm=on
dtparam=spi=on
```
The `[all]` section at the bottom is empty — no additional overlays are needed when using the Waveshare library with cs=4.
Download Waveshare Library
```bash
cd ~
sudo apt-get install -y p7zip-full python3-lgpio
wget https://files.waveshare.com/upload/6/67/pn532-nfc-hat-code.7z
7z x pn532-nfc-hat-code.7z -r -o./pn532-nfc-hat-code
sudo chmod 777 -R pn532-nfc-hat-code/
```
Test NFC Reading
```bash
cd ~/pn532-nfc-hat-code/Pn532-nfc-hat-code/raspberrypi/python/
nano example_get_uid.py
```
Make sure this line is active (others commented out):
```python
pn532 = PN532_SPI(debug=False, reset=20, cs=4)
```
Run:
```bash
sudo python3 example_get_uid.py
```
Tap a tag. Output looks like:
```
Found PN532 with firmware version: 1.6
Waiting for RFID/NFC card...
Found card with UID: ['0xb7', '0xa8', '0x87', '0x3']
```
Converting the UID for use in config: Remove `0x` prefixes, brackets and quotes, join together, pad single digits with a leading zero. So `['0xb7', '0xa8', '0x87', '0x3']` becomes `b7a88703`.
Press `Ctrl+C` to stop.
---
Part 4 — Spotify API Setup
Create Developer App
Go to developer.spotify.com/dashboard
Create app with:
Redirect URI: `http://127.0.0.1:8888/callback`
Web API checkbox ticked
Copy Client ID and Client Secret from Settings
Get Refresh Token
> **Pi 5 SSH port forwarding required:** Because the Pi is headless, the Spotify OAuth redirect needs to come back to the Pi. SSH in using port forwarding so the redirect hits the Pi's local server:
> ```bash
> ssh -L 8888:localhost:8888 jamesmcgraaw@192.168.50.16
> ```
> Then open the auth URL in your browser on your computer — the redirect to `127.0.0.1:8888` will be tunnelled through SSH to the Pi.
Install requests:
```bash
pip3 install requests --break-system-packages
```
Create `~/spotify_auth.py`:
```python
import requests
import base64
import urllib.parse
import http.server
import threading

CLIENT_ID     = "YOUR_CLIENT_ID_HERE"
CLIENT_SECRET = "YOUR_CLIENT_SECRET_HERE"
REDIRECT_URI  = "http://127.0.0.1:8888/callback"
SCOPE         = "user-modify-playback-state user-read-playback-state"

auth_code = None

class Handler(http.server.BaseHTTPRequestHandler):
    def do_GET(self):
        global auth_code
        params = dict(urllib.parse.parse_qsl(urllib.parse.urlparse(self.path).query))
        auth_code = params.get("code")
        self.send_response(200)
        self.end_headers()
        self.wfile.write(b"Got it! You can close this tab.")

    def log_message(self, *args):
        pass

server = http.server.HTTPServer(("", 8888), Handler)
thread = threading.Thread(target=server.handle_request)
thread.start()

params = urllib.parse.urlencode({
    "client_id": CLIENT_ID,
    "response_type": "code",
    "redirect_uri": REDIRECT_URI,
    "scope": SCOPE,
})
url = f"https://accounts.spotify.com/authorize?{params}"
print(f"\nOpen this URL in your browser:\n\n{url}\n")
thread.join()

creds = base64.b64encode(f"{CLIENT_ID}:{CLIENT_SECRET}".encode()).decode()
resp = requests.post("https://accounts.spotify.com/api/token", data={
    "grant_type": "authorization_code",
    "code": auth_code,
    "redirect_uri": REDIRECT_URI,
}, headers={"Authorization": f"Basic {creds}"})

tokens = resp.json()
print(f"\n✅ Refresh token: {tokens.get('refresh_token', 'ERROR')}")
```
Run it (via the port-forwarded SSH session):
```bash
python3 ~/spotify_auth.py
```
---
Part 5 — Player Scripts
`~/nfc_player_config.py`
```python
# ── Spotify credentials ──────────────────────────────────────────────────
CLIENT_ID     = "YOUR_CLIENT_ID_HERE"
CLIENT_SECRET = "YOUR_CLIENT_SECRET_HERE"
REFRESH_TOKEN = "YOUR_REFRESH_TOKEN_HERE"

# ── Must match LIBRESPOT_NAME in /etc/raspotify/conf ─────────────────────
SPOTIFY_DEVICE_NAME = "Album Player"
PAUSE_TAG = "your_pause_tag_uid_here"

# ── Tag UID → Spotify URI mapping ────────────────────────────────────────
# Get UIDs by running example_get_uid.py and tapping each tag
# Get Spotify URIs by right-clicking an album in the desktop app
# → Share → hold Alt → Copy Spotify URI
TAG_MAP = {
    "your_tag_uid": "spotify:album:xxxxxx",  # Album - Artist
}
```
`~/nfc_player.py`
```python
#!/usr/bin/env python3
"""NFC Spotify Album Player — Pi 5 + Waveshare PN532 HAT"""

import sys
import time
import requests
import base64

sys.path.insert(0, '/home/jamesmcgraaw/pn532-nfc-hat-code/Pn532-nfc-hat-code/raspberrypi/python')
from pn532 import *

import nfc_player_config as cfg

# ── Spotify token management ──────────────────────────────────────────────
_access_token = None
_token_expiry  = 0

def get_access_token():
    global _access_token, _token_expiry
    if _access_token and time.time() < _token_expiry - 60:
        return _access_token
    creds = base64.b64encode(f"{cfg.CLIENT_ID}:{cfg.CLIENT_SECRET}".encode()).decode()
    resp = requests.post("https://accounts.spotify.com/api/token", data={
        "grant_type": "refresh_token",
        "refresh_token": cfg.REFRESH_TOKEN,
    }, headers={"Authorization": f"Basic {creds}"})
    data = resp.json()
    _access_token = data["access_token"]
    _token_expiry  = time.time() + data["expires_in"]
    return _access_token

def spotify_headers():
    return {"Authorization": f"Bearer {get_access_token()}",
            "Content-Type": "application/json"}

def get_device_id():
    resp = requests.get("https://api.spotify.com/v1/me/player/devices",
                        headers=spotify_headers())
    devices = resp.json().get("devices", [])
    for d in devices:
        if d["name"] == cfg.SPOTIFY_DEVICE_NAME:
            return d["id"]
    return None

def play_uri(spotify_uri):
    device_id = get_device_id()
    if not device_id:
        print(f"⚠️  Device '{cfg.SPOTIFY_DEVICE_NAME}' not found. Is raspotify running?")
        return
    context_type = spotify_uri.split(":")[1]
    resp = requests.put(
        f"https://api.spotify.com/v1/me/player/play?device_id={device_id}",
        headers=spotify_headers(),
        json={"context_uri": spotify_uri, "offset": {"position": 0}, "position_ms": 0}
    )
    if resp.status_code in (200, 204):
        print(f"▶  Playing {context_type}: {spotify_uri}")
    else:
        print(f"❌ Playback error {resp.status_code}: {resp.text}")

def get_playback_state():
    resp = requests.get("https://api.spotify.com/v1/me/player",
                        headers=spotify_headers())
    if resp.status_code == 204 or not resp.content:
        return None
    return resp.json()

def toggle_pause():
    state = get_playback_state()
    if state is None:
        print("⚠️  Nothing currently playing")
        return
    device_id = get_device_id()
    if not device_id:
        print(f"⚠️  Device '{cfg.SPOTIFY_DEVICE_NAME}' not found")
        return
    if state.get("is_playing"):
        requests.put("https://api.spotify.com/v1/me/player/pause",
                     headers=spotify_headers())
        print("⏸  Paused")
    else:
        requests.put("https://api.spotify.com/v1/me/player/play",
                     headers=spotify_headers())
        print("▶  Resumed")

# ── NFC setup ─────────────────────────────────────────────────────────────
pn532 = PN532_SPI(debug=False, reset=20, cs=4)
ic, ver, rev, support = pn532.get_firmware_version()
print(f"PN532 firmware v{ver}.{rev} — ready")
pn532.SAM_configuration()

# ── Main loop ─────────────────────────────────────────────────────────────
print("🎵 NFC Player running — tap a tag to play music!\n")

last_uid  = None
last_time = 0
DEBOUNCE_SECONDS = 5

while True:
    try:
        uid = pn532.read_passive_target(timeout=0.5)

        if uid is None:
            last_uid = None
            continue

        uid_str = ''.join([hex(i)[2:].zfill(2) for i in uid])
        now = time.time()

        if uid_str == last_uid and (now - last_time) < DEBOUNCE_SECONDS:
            continue

        last_uid  = uid_str
        last_time = now

       print(f"Tag scanned: {uid_str}")
       
       if uid_str == cfg.PAUSE_TAG:
           toggle_pause()
       elif uid_str in cfg.TAG_MAP:
           play_uri(cfg.TAG_MAP[uid_str])
       else:
           print(f"   (No album mapped to this tag — add it to nfc_player_config.py)")

    except KeyboardInterrupt:
        print("\nStopping.")
        break
    except Exception as e:
        print(f"Error: {e}")
        time.sleep(1)
```
---
Part 6 — Systemd Service
`/etc/systemd/system/nfc-player.service`
```ini
[Unit]
Description=NFC Spotify Player
After=network-online.target raspotify.service
Wants=network-online.target

[Service]
Type=simple
User=jamesmcgraaw
WorkingDirectory=/home/jamesmcgraaw
Environment=PYTHONUNBUFFERED=1
ExecStart=/usr/bin/python3 -u /home/jamesmcgraaw/nfc_player.py
Restart=always
RestartSec=5

[Install]
WantedBy=multi-user.target
```
> **Note:** `PYTHONUNBUFFERED=1` and the `-u` flag on python3 are both required to see log output in journalctl. Without them the service runs silently.
Enable and start:
```bash
sudo systemctl daemon-reload
sudo systemctl enable nfc-player
sudo systemctl start nfc-player
```
Check it's running:
```bash
sudo systemctl status nfc-player
```
View live logs:
```bash
sudo journalctl -u nfc-player -f
```
---
Adding New Tags
Run the test script and tap the tag:
```bash
   sudo python3 ~/pn532-nfc-hat-code/Pn532-nfc-hat-code/raspberrypi/python/example_get_uid.py
   ```
Note the UID — e.g. `['0xb7', '0xa8', '0x87', '0x3']` → `b7a88703`
Get the Spotify URI — right-click album in Spotify desktop app → Share → hold Alt → Copy Spotify URI
Add to `TAG_MAP` in `~/nfc_player_config.py`:
```python
   "b7a88703": "spotify:album:XXXXXX",  # Album - Artist
   ```
Restart the service:
```bash
   sudo systemctl restart nfc-player
   ```
---
Troubleshooting
Problem	Fix
SSH host identification error	Run `ssh-keygen -R albumplayer.local` on your computer
Wrong password on SSH	Re-flash SD card; avoid special characters in password
NFC not detected	Check jumpers (I1=H, I0=L), DIP switches (1-4 ON, 5-8 OFF), HAT fully seated
`Failed to detect PN532`	Confirm `dtparam=spi=on` is in `/boot/firmware/config.txt`
Raspotify "Unsupported Sample Rate"	Use `plughw:0,0` not `hw:0,0` in raspotify conf
Raspotify "Device Invalid/Busy"	Wrong card number — run `aplay -l` to find correct one
"Album Player" not in Spotify devices	Play something through it manually from the Spotify app first to wake it up
Tag scans but nothing plays	Check `get_device_id()` has a `return` statement; check device name matches exactly
No logs from service	Add `PYTHONUNBUFFERED=1` to service and `-u` flag to python3 in ExecStart
Audio drops / buffering	Network congestion (e.g. large downloads on same WiFi); reduce bitrate to 160 temporarily
Spotify OAuth redirect fails	SSH in with port forwarding: `ssh -L 8888:localhost:8888 user@pi_ip_address`
---
How It All Fits Together
```
NFC tag tapped on HAT
    └─► Waveshare PN532 HAT reads tag UID (SPI, cs=BCM4, reset=BCM20)
        └─► nfc_player.py looks up UID in TAG_MAP
            └─► Calls Spotify Web API /me/player/play
                └─► Raspotify (librespot) receives playback command
                    └─► Streams audio from Spotify
                        └─► MOSWAG USB adapter (plughw:0,0) → speakers
```
