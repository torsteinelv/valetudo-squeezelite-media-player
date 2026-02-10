# TalkyRock: High-Fidelity Audio for Roborock S5

This project documents how to transform a rooted **Roborock S5 (running Valetudo)** into a high-quality smart speaker. By using Squeezelite and Logitech Media Server (LMS), your vacuum can play music, radio, and Text-to-Speech (TTS) directly from Home Assistant.

**Status:** Verified and tested on Roborock S5. Highly responsive audio gate (no padding required).

---

## Architecture

* **Device:** Roborock S5 (Rooted, Valetudo).
* **Audio Client:** `Squeezelite-armv7` running as a background service.
* **Audio Server:** Logitech Media Server (LMS / Lyrion) via Home Assistant.
* **TTS Engine:** Any Home Assistant compatible TTS service.

---

## Required Files

To ensure long-term availability, it is recommended to keep these files in your own repository:

1.  `squeezelite-armv7`: The binary for the vacuum.
2.  `S99squeezelite`: The startup and watchdog script.

---

## 1. Vacuum Setup (SSH)

### Installation
Move the Squeezelite binary to the vacuum's persistent data partition and make it executable:

    cd /mnt/data
    wget https://github.com/torsteinelv/valetudo-squeezelite-media-player/raw/refs/heads/main/squeezelite-armv7
    chmod +x squeezelite-armv7

### Service & Watchdog Script
Create the file `/etc/init/S99squeezelite`. This script acts as an "Immortal Watchdog," checking every 15 seconds if the process is running and restarting it if necessary.

    #!/bin/sh

    LMS_IP="10.10.10.101" # CHANGE THIS to your HA/LMS IP
    PLAYER_NAME="Roborock"

    run_watchdog() {
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

---

## 3. Home Assistant Implementation

### Generic TTS Action (YAML)

    action: tts.speak
    target:
      entity_id: tts.google_en_com  # Or your preferred TTS provider
    data:
      media_player_entity_id: media_player.roborock
      message: "The vacuuming is complete. Returning to the dock."
      cache: true

---

## 💡 Lessons Learned & Tips

* **Mono Mixing:** The Roborock S5 has a single speaker. The `-u M` flag in Squeezelite is mandatory to properly downmix stereo content.
* **Firewall blockage:** If the vacuum connects to LMS but remains silent during TTS, check if port 8123 is blocked. The vacuum needs to reach the HA web server to fetch the generated audio file.
* **Internal URL:** Ensure Home Assistant's **Internal URL** is correctly configured to the local IP. If HA tries to serve the file via an external proxy (like Cloudflare), the vacuum might fail to download the audio.
