# Distributed Face-Locked Servo System

Ever wanted your robot to actually look at you? This project does exactly that. It's a fully distributed system that tracks your face in real-time and uses a servo motor to follow you around. Think of it as a tiny robotic head that's always watching.

## 🎯 What's This About?

The system has four main pieces working together:

1. **Vision Node (PC)** - Sits on your computer with a camera, spots faces using OpenCV
2. **Edge Controller (ESP8266)** - A tiny microcontroller that physically moves the servo
3. **Backend Bridge** - Connects everything through MQTT and serves data to the web
4. **Web Dashboard** - Your control center in the browser showing everything in real-time

## 🏗️ How It All Works

The system uses MQTT as the nervous system - everything talks through a central broker. Your PC detects faces and sends commands to the ESP8266, which moves the servo. The backend relays all this info to your browser dashboard in real-time.

```
┌─────────────┐         ┌──────────────┐         ┌──────────────┐
│   PC with   │         │   VPS with   │         │   Browser    │
│   Camera    │ ──MQTT──│ MQTT Broker  │──WSS────│  Dashboard   │
│             │         │  + Backend   │         │              │
└─────────────┘         └──────────────┘         └──────────────┘
       │                        │
       │                        │
       │                        │ MQTT
       │                        │
       │                  ┌─────▼────┐
       └──────MQTT────────│ ESP8266  │
                          │  Servo   │
                          └──────────┘
```

## ✅ What You'll Need

### For Your PC (Vision Node)

- Python 3.8 or newer
- A working webcam
- OpenCV (for face detection)
- MQTT client library

### For the ESP8266 Board

- ESP8266 board (NodeMCU works great)
- MicroPython installed
- A servo motor (SG90 is perfect)
- Power supply for the servo (5V)

### For the Server/VPS

- Node.js 16 or later
- MQTT broker running (Mosquitto is solid)
- A public IP or domain name

### For the Dashboard

- Any modern browser (Chrome, Firefox, Edge)
- Internet connection

## 🚀 Getting Started

### Step 1: Set Up the Vision Node (Your PC)

Start here - this is what sees your face:

```bash
cd manage_vision

# Create a virtual environment (good practice)
python3 -m venv venv
source venv/bin/activate  # Windows: venv\Scripts\activate

# Install what you need
pip install -r requirements.txt

# Open vision_node.py and update these:
# - TEAM_ID: Give your team a unique name
# - MQTT_BROKER: Put your VPS IP here
```

**Fire it up:**

```bash
python vision_node.py
```

### Step 2: Flash the ESP8266

This tiny board will control your servo:

**Hardware setup:**

```
ESP8266 (NodeMCU)     Servo Motor
─────────────────     ───────────
5V              ──► VCC (Red)
GND               ──► GND (Brown/Black)
D2 (GPIO4)        ──► Signal (Orange/Yellow)
```

**Upload the code:**

```bash
# First, install ampy
pip install adafruit-ampy

# Edit main.py with your WiFi details and MQTT settings
# Then upload it
ampy --port /dev/ttyUSB0 put main.py

# Reboot and check the serial output
screen /dev/ttyUSB0 115200
```

### Step 3: Get the Backend Running

This bridges everything together:

**First, set up MQTT (on your VPS):**

```bash
sudo apt update
sudo apt install mosquitto mosquitto-clients
sudo systemctl start mosquitto
sudo systemctl enable mosquitto
```

**Then start your backend:**

```bash
cd back-end

npm install

# Update TEAM_ID in server.js to match your other components

npm start
# Or use: npm run dev (for development with auto-restart)
```

### Step 4: Get Your Dashboard Online

View everything from your browser:

**Edit dashboard.html:**

```javascript
const TEAM_ID = "your-team-name"; // Match your team ID
```

**Choose your deployment:**

_Option 1: Local Server_

```bash
cd web-dashboard
python3 -m http.server 8000
# Open: http://localhost:8000/dashboard.html
```

_Option 2: On Your VPS_

```bash
sudo apt install nginx
sudo cp dashboard.html /var/www/html/
# Access: http://YOUR_VPS_IP/dashboard.html
```

_Option 3: GitHub Pages_
Just push dashboard.html to GitHub and enable Pages - done!

## � How to Configure Everything

### Topics You Need to Know About

Since everyone uses the same broker, we isolate teams with unique topic names. This is **super important**:

```
vision/<TEAM_ID>/movement     → Face position commands
vision/<TEAM_ID>/heartbeat    → System health checks
iot/status/<CLIENT_ID>        → Online/offline status
```

**Real example for team01:**

```
vision/team01/movement
vision/team01/heartbeat
iot/status/esp8266_a1b2c3d4
```

### The Messages Your System Sends

**What movement looks like (JSON):**

```json
{
  "status": "MOVE_LEFT",
  "confidence": 0.87,
  "timestamp": 1730000000,
  "face_position": {
    "x": 320,
    "y": 240
  }
}
```

**Status codes your servo understands:**

- `MOVE_LEFT` → Face moved to the right, servo turns left to track
- `MOVE_RIGHT` → Face moved to the left, servo turns right to track
- `CENTERED` → Got you in the middle, hold position
- `NO_FACE` → Can't see any face right now

## 🧪 Testing Your Setup

### Test One Component at a Time

**Check your vision node:**

```bash
cd manage_vision
python vision_node.py
```

You should see: `✓ Connected to MQTT broker`
Move your face around and watch the output change.

**Check the ESP8266:**

```bash
screen /dev/ttyUSB0 115200
```

Look for messages like:

- `✓ Wi-Fi connected`
- `✓ MQTT connected`
- `← MOVE_LEFT (conf: 0.87)`

**Check your backend:**

```bash
cd back-end
npm start
```

Should show:

- `✓ Connected to MQTT broker`
- `✓ WebSocket server listening on port 9002`

**Check the dashboard:**
Open it in your browser. You should see:

- Status says "Connected"
- Real-time movement indicators
- Message log showing incoming data

### Full System Test

1. Start MQTT on your VPS
2. Start the backend
3. Power on the ESP8266 (servo should point to 90°)
4. Run the vision node
5. Open the dashboard
6. Move your face - everything should respond

### Troubleshooting Quick Reference

| Problem                       | Fix                                                  |
| ----------------------------- | ---------------------------------------------------- |
| PC won't connect to MQTT      | Check firewall, verify broker IP is correct          |
| ESP8266 not connecting        | Check WiFi password, look at serial output           |
| Dashboard says "Disconnected" | Verify port 9002 is open, check WebSocket URL        |
| Servo won't budge             | Check servo wiring, verify MQTT messages are flowing |
| Wrong team's data showing up  | Make sure TEAM_ID is the same everywhere             |

## ⚡ Making It Snappier

### Reduce Lag

- Publish every 100ms max (tweak in vision_node.py)
- Servo smoothing is on by default (less jittery movement)
- MQTT QoS set to 1 (guaranteed delivery but still fast)

### Stop the Jitter

In ESP8266 main.py:

```python
SMOOTHING_ENABLED = True  # Gradual movement instead of jerky
MOVE_DELAY_MS = 50       # Small delay between servo steps
```

### Network Tweaks

- Use retained messages for state tracking
- Don't spam the same message repeatedly
- Enable WebSocket compression if your network supports it

## �️ Where This is Headed

### Right Now (Phase 1) ✅

- Camera stays fixed on your PC
- Servo moves to follow your face
- Camera doesn't move, only the servo rotates
- This proves the whole concept works

### Next Level (Phase 2) 🔜

- Mount the camera directly on the servo
- Camera rotates WITH the servo
- Creates a true feedback loop
- Add fancy PID control for super smooth tracking

## 💡 What You'll Learn

This isn't just a project - it's a masterclass in modern development:

1. **Distributed Systems** - How independent pieces talk without direct connections
2. **Message-Driven Architecture** - MQTT as the central nervous system
3. **Real-Time Control** - Getting commands to hardware with minimal delay
4. **Edge Computing** - Splitting work between PC (thinking) and ESP8266 (doing)
5. **Web Tech** - WebSockets bringing live data to browsers
6. **Embedded Programming** - Making resource-limited devices do cool stuff

## 📚 Resources & Attribution

- Gabriel Baziramwabo (2025). "ESP8266-Based Environmental Monitoring & Actuator Control Using MQTT"
- Gabriel Baziramwabo (2025). "Mastering ESP8266 with MicroPython" DOI:10.13140/RG.2.2.35581.88800
- MicroPython Docs: https://docs.micropython.org
- MQTT Protocol: https://mqtt.org/
- OpenCV: https://docs.opencv.org/

## 👥 Built By

- **TUYISHIME Christian** - Vision system, ESP8266 hardware
- **NTARE KAYITARE Prince** - Backend & server setup
- **BAGABO Bonny** - Web dashboard & docs

## 📄 License

MIT - Use it, learn from it, build on it.

## 🙏 Thanks to

- Gabriel Baziramwabo for ESP8266 guidance
- Rwanda Coding Academy for the framework
- OpenCV community for vision tools

---

## 📞 Need Help?

1. See the troubleshooting section above
2. Check out Gabriel's ResearchGate materials
3. Watch BenaxMedia YouTube tutorials
4. Reach out to Gabriel Baziramwabo

---

**Last Updated:** February 2026  
**Version:** 1.0.0
