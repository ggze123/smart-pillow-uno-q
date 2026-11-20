# Silent Bruxism & Snore Interruptor (Smart Pillow v2)

โปรเจกต์หมอนอัจฉริยะหยุดอาการนอนกัดฟัน (Bruxism) และนอนกรน (Snoring) บนแพลตฟอร์ม **Arduino UNO Q 4GB** (Qualcomm Dragonwing QRB2210 Debian Linux + STM32U585) สำหรับการแข่งขัน **Arduino UNO Q AIoT & Edge AI Hackathon — Health, Wellness and Smart Living (Arduino Education Day Thailand 2026)**

## เอกสารออกแบบและแผนการพัฒนา

* **Design Document:** [docs/superpowers/specs/2026-11-20-smart-pillow-design.md](docs/superpowers/specs/2026-11-20-smart-pillow-design.md)
* **6-Day Implementation Plan:** [docs/superpowers/plans/2026-11-20-smart-pillow.md](docs/superpowers/plans/2026-11-20-smart-pillow.md)

## ภาพรวมสถาปัตยกรรม (`Sense → Connect → Analyze → Decide → Respond`)

* **Sense:** USB Microphone (RAM-only), FSR Pressure Pad (`A2`, 3.3V divider) สำหรับยืนยันการมีศีรษะวางอยู่บนหมอน, และ Analog Piezo (`A3`, `1 MΩ` + `3.3V Zener`) สำหรับตรวจจับ Motion Artifact ตอนพลิกตัว
* **Connect:** `Arduino_RouterBridge` (STM32) ↔ `arduino-router` (Native Debian Linux Python)
* **Analyze:** Edge Impulse `.eim` 4-Class Audio Classification (`Quiet_Noise`, `Normal_Breathing`, `Snoring`, `Bruxism`) แบบออฟไลน์ 100%
* **Decide:** 5-State Machine + Triple-Signal Sensor Fusion (`Blanking Window 3s`, `Cooldown Quota`, `Emergency Stop`)
* **Respond:** Soft-Start PWM Coin Vibration Motor (`D5`), MG996R Servo Tilt แยกไฟเลี้ยง 5V/3A (`D9`), `8×13` LED Matrix, STM32 Autonomous Watchdog (`5s`), และ Local Web Dashboard (`FastAPI + WebSocket + SQLite`)
