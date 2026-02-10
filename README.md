# TalkyRock: High-Fidelity TTS for Roborock S5

This project documents how to transform a rooted **Roborock S5 (running Valetudo)** into a high-quality AI-powered smart speaker. By using Squeezelite, Logitech Media Server (LMS), and ElevenLabs, your vacuum can speak with natural, high-fidelity voices.

**Status:** Verified and tested on Roborock S5.

---

## 🏗️ Architecture

* **Device:** Roborock S5 (Rooted, Valetudo).
* **Audio Client:** `Squeezelite-armv7` running as a background service.
* **Audio Server:** Logitech Media Server (LMS / Lyrion) via Home Assistant.
* **TTS Engine:** ElevenLabs (via Home Assistant Integration).
* **Networking:** IOT VLAN to Management LAN communication.

---

## 📦 Required Files

To ensure long-term availability, it is recommended to keep these files in your own repository:

1.  `squeezelite-armv7`: The binary for the vacuum.
2.  `S99squeezelite`: The startup and watchdog script.

---

## 1. Vacuum Setup (SSH)

### Installation
Move the Squeezelite binary to the vacuum's persistent data partition and make it executable:

    cd /mnt/data
    # Upload or download the binary here
    chmod +x squeezelite-armv7

### Service & Watchdog Script
Create the file `/etc/init/S99squeezelite`. This script acts as an "Immortal Watchdog," checking every 15 seconds if the process is running and restarting it if necessary.

    #!/bin/sh

    LMS_IP="10.10.10.101" # CHANGE THIS to your HA/LMS IP
    PLAYER_NAME="Roborock"

    run_watchdog() {
        # Wait for network to stabilize after boot
        sleep 15
        while true; do
            if ! ps | grep -v grep | grep -q "squeezelite-armv7"; then
                /mnt/data/squeezelite-armv7 -n "$PLAYER_NAME" -s $LMS_IP -o hw:0,0 -u M -a 40:4:: > /dev/null 2>&1
            fi
            sleep 15
        done
    }

    case "$1" in
        start)
            run_watchdog &
            ;;
        stop)
            killall squeezelite-armv7
            pkill -f S99squeezelite
            ;;
        restart)
            $0 stop
            $0 start
            ;;
    esac

**Apply permissions and start:**
    
    chmod +x /etc/init/S99squeezelite
    /etc/init/S99squeezelite start

---

## 2. Firewall Configuration (Crucial)

If the vacuum is on an IOT VLAN, you **must** allow traffic to the Home Assistant IP on these ports:

| Port | Protocol | Purpose |
| :--- | :--- | :--- |
| **8123** | **TCP** | **Internal TTS Proxy (Vacuum downloads the audio file here)** |
| 3483 | TCP/UDP | SlimProto Control & Discovery |
| 9000 | TCP | LMS Web Interface & Streaming |

*Note: Without port 8123, the vacuum will connect to LMS but will be unable to download the TTS audio, leading to 504 Gateway Timeouts.*

---

## 3. Home Assistant Implementation

### ElevenLabs Action (YAML)
To get the best results on an S5, use leading periods (padding) to wake up the speaker hardware and specify the multilingual model to avoid language detection issues.

    action: tts.speak
    target:
      entity_id: tts.elevenlabs
    data:
      media_player_entity_id: media_player.roborock
      message: ".......... Hello, I am ready to clean your floors."
      cache: true
      options:
        model_id: "eleven_multilingual_v2"
        voice: "Rachel"

---

## 💡 Lessons Learned & Tips

* **Mono Mixing:** The Roborock S5 has a single speaker. The `-u M` flag in Squeezelite is mandatory to properly downmix stereo content.
* **The "Danish" Trap:** If your Norwegian or Swedish text sounds Danish, ensure `model_id: "eleven_multilingual_v2"` is explicitly defined in your call.
* **Cloudflare/DNS:** If using a reverse proxy (like Cloudflare), ensure Home Assistant's **Internal URL** is set to the local IP. The vacuum cannot download TTS files through Cloudflare's 504 timeout filters.
* **Audio Padding:** The S5 audio gate takes ~1-2 seconds to open. Always start your TTS messages with `..........` (periods).
