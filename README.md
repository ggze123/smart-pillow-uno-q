# Silent Bruxism & Snore Interruptor (Smart Pillow v2 — Arduino App Lab Edition)

โปรเจกต์หมอนอัจฉริยะหยุดอาการนอนกัดฟัน (Bruxism) และนอนกรน (Snoring) บนแพลตฟอร์ม **Arduino UNO Q 4GB** พัฒนาและ Deploy ผ่าน **Arduino App Lab** สำหรับการแข่งขัน **Arduino UNO Q AIoT & Edge AI Hackathon — Health, Wellness and Smart Living (Arduino Education Day Thailand 2026)**

## เอกสารออกแบบและแผนการพัฒนา

* **Design Document (Arduino App Lab Edition):** [docs/superpowers/specs/2026-11-20-smart-pillow-design.md](docs/superpowers/specs/2026-11-20-smart-pillow-design.md)
* **6-Day Implementation Plan:** [docs/superpowers/plans/2026-11-20-smart-pillow.md](docs/superpowers/plans/2026-11-20-smart-pillow.md)

## ภาพรวมสถาปัตยกรรม (`Sense → Connect → Analyze → Decide → Respond`)

* **Sense:** USB Microphone (RAM-only via `arduino:audio_classification` Brick), FSR Pressure Pad (`A2`, 3.3V divider) สำหรับยืนยันการมีศีรษะวางอยู่บนหมอน, และ Analog Piezo (`A3`, `1 MΩ` + `3.3V Zener`) สำหรับตรวจจับ Motion Artifact ตอนพลิกตัว
* **Connect:** `Arduino_RouterBridge` (`sketch/sketch.ino`) ↔ `arduino.app_utils.Bridge` (`python/main.py`)
* **Analyze:** Edge Impulse `.eim` 4-Class Audio Classification (`Quiet_Noise`, `Normal_Breathing`, `Snoring`, `Bruxism`) รันออฟไลน์ 100% ผ่าน `arduino:audio_classification` Brick
* **Decide:** 5-State Machine + Triple-Signal Sensor Fusion (`Blanking Window 3s`, `Cooldown Quota`, `Emergency Stop`)
* **Respond:** Soft-Start PWM Coin Vibration Motor (`D5`), MG996R Servo Tilt แยกไฟเลี้ยง 5V/3A (`D9`), `8×13` LED Matrix, STM32 Autonomous Watchdog (`5s` + Hard Clamp `≤ 35°`), และ Stage-Ready Web Dashboard ผ่าน `arduino:web_ui` Brick (`assets/`) + `SQLite`
