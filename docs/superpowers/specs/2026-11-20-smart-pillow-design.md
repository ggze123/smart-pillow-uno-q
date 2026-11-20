# Design Document — Silent Bruxism & Snore Interruptor (Smart Pillow v2: Stage-Ready Edition)

**วันที่:** 2026-11-20  
**กำหนดแข่งขัน:** 26–27 พฤศจิกายน 2026 (ระยะเวลาพัฒนาและทดสอบ 6 วัน)  
**แพลตฟอร์มเป้าหมาย:** Arduino UNO Q 4GB *(มีบอร์ดอยู่ในมือแล้ว — พัฒนาและ Deploy ผ่าน **Arduino App Lab**)*  
**เวทีเป้าหมาย:** Arduino UNO Q AIoT & Edge AI Hackathon — Health, Wellness and Smart Living (Arduino Education Day Thailand 2026)

---

## 1. Intent & The Why

เอกสารนี้ออกแบบการปรับปรุง **Smart Pillow** สำหรับหยุดอาการ **นอนกัดฟัน (Bruxism)** และ **นอนกรน (Snoring)** แบบไม่ปลุกให้ตื่น โดยยกระดับจากสเปกแนวคิดเดิมให้กลายเป็น **Working Prototype ที่สาธิตได้จริงและเสถียรบนเวที Hackathon ภายในเวลา 6 วัน** และสอดคล้องกับโครงสร้าง **Arduino App Lab** ของบอร์ด Arduino UNO Q อย่างสมบูรณ์

การตัดสินใจออกแบบทุกข้อผูกตรงกับโครงสร้างคะแนนและกติกาของรายการ:
* **Technical Integration (25 คะแนน):** ใช้สถาปัตยกรรม 2 สมองของ Arduino UNO Q และ **Arduino App Lab Bricks** อย่างเต็มประสิทธิภาพ (`arduino:audio_classification` + `arduino:web_ui` + `arduino.app_utils.Bridge` ฝั่ง Linux คุยกับ STM32U585 ที่อ่าน Analog 14-bit + ขับ PWM/Servo/LED Matrix + Watchdog)
* **Functional Prototype & Testing (20 คะแนน):** สาธิตสดได้จริง เสถียร ไม่ค้างกลางเวที พร้อมหลักฐานการทดลอง (Confusion Matrix, อัตราการตัด False Alarm ก่อน–หลังเปิด Sensor Fusion, และเรื่องเล่าการปรับแก้ทางวิศวกรรมจริง เช่น การเพิ่ม FSR แยกหน้าที่จาก Piezo)
* **Impact, Feasibility & Safety (10 คะแนน):** ป้องกันไฟกระชาก ป้องกันขา ADC 3.3V พัง มีกลไกตัดมอเตอร์อัตโนมัติ (Watchdog) และอธิบายข้อจำกัดของต้นแบบ vs เวอร์ชันใช้งานจริง (Air Bladder) ได้อย่างตรงไปตรงมา
* **Problem & User Need (15) + Innovation (15) + Presentation (10) + Documentation (5):** รองรับรูปแบบ Pitch **7 นาที + Q&A 3 นาที** (เวลาเดโมสดจริงประมาณ 2–3 นาที) ในห้องจัดงานที่มีเสียงรบกวน

---

## 2. Goals, Non-Goals & Anti-Goals

### Goals
1. **Triple-Signal Sensor Fusion ที่สอดคล้องกับฟิสิกส์จริง:**
   * **FSR (Force Sensitive Resistor)** ทำหน้าที่ **Head Presence Sensor** ยืนยันว่า *"มีศีรษะวางอยู่บนหมอนจริง"* แม้คนนอนนิ่ง ๆ (แก้ข้อจำกัดเดิมที่คิดจะใช้ Piezo วัดการมีคนนอน)
   * **Analog Piezo** ปรับหน้าที่เป็น **Motion Artifact Detector** จับแรงสั่นกระชากตอนพลิกตัว/ขยับแรง ๆ เพื่อปฏิเสธเสียงผ้าเสียดสีปลอกหมอนไม่ให้หลอก AI ว่าเป็นเสียงกัดฟัน (เหมาะกับธรรมชาติของ Piezo ที่จับแรงกระแทกชั่วขณะได้ดีโดยไม่ต้องใช้วงจร Preamp ซับซ้อน)
   * **USB Microphone + `arduino:audio_classification` Brick (Edge Impulse `.eim`)** แยกเสียง 4 คลาสแบบออฟไลน์ 100% โดยไม่บันทึกไฟล์เสียงลงดิสก์
2. **100% Arduino App Lab Compatible:** จัดโครงสร้างแอปตามมาตรฐาน App Lab (`app.yaml`, `python/main.py`, `sketch/sketch.ino`, `sketch/sketch.yaml`, `assets/`) พร้อมแยกแกนตรรกะ (`python/smart_pillow/`, `sketch/safety_logic.h`) เป็น Pure Python / Pure C++ เพื่อให้รัน Unit Test บนคอมฯ ได้ก่อนกด Run บน UNO Q
3. **Zero Self-Feedback Loop:** มี State Machine 5 สถานะ พร้อมช่วง `BLANKING` ตัดการตรวจจับระหว่างและหลังมอเตอร์/Servo ทำงาน ไม่ให้ระบบกระตุ้นตัวเองซ้ำ
4. **Hardware-Level Safety:** แยกแหล่งจ่ายไฟมอเตอร์ออกจากบอร์ด, ป้องกันแรงดันเกินบนขา Analog 3.3V, และมี Autonomous Watchdog บน STM32 ที่หยุดทุก Actuator เองเมื่อ Linux ค้างหรือ Bridge ขาดการติดต่อ
5. **Stage-Ready Live Demo & Evidence for Judges:** มี **Stage Demo Mode** และปุ่ม **1-Click Calibration** บน `WebUI` Brick ทำให้เดโมครบ 4 จังหวะจบได้ภายใน ~90 วินาที พร้อมบันทึก SQLite และ Export CSV / Printable Report สำหรับสไลด์ Pitch

### Non-Goals
* **ไม่ใช่ผลิตภัณฑ์ทางการแพทย์ (Non-Medical Device):** ไม่ได้วินิจฉัยภาวะหยุดหายใจขณะหลับ (OSA) และไม่ได้ออกแบบให้ผ่านมาตรฐานเครื่องมือแพทย์
* **ไม่ได้ออกแบบให้ทนการนอนจริง 8 ชั่วโมงต่อคืน หรือรับน้ำหนักศีรษะจริง 4–5 กก. ในเวอร์ชัน 6 วันนี้:** ต้นแบบออกแบบสำหรับ **Tabletop Demo (ใช้มือกดหรือตุ้มน้ำหนักจำลอง ~0.5–1.5 กก.)** เพื่อให้เห็นกลไกชัดเจน ปลอดภัย และประกอบทันภายใน 6 วัน
* **ไม่เชื่อมต่อ Cloud:** ระบบทั้งหมดทำงานแบบ Local / Offline บนบอร์ด UNO Q เพื่อรักษาความเป็นส่วนตัวและกันปัญหา Wi-Fi ในงานแข่งล่ม

### Anti-Goals (ถือว่าล้มเหลวแม้โค้ดจะรันผ่าน)
* **บอร์ดรีเซ็ตหรือ Brownout** ระหว่างเดโมตอน Servo เริ่มหมุน
* **มอเตอร์สั่นหรือ Servo กวนเซ็นเซอร์ตัวเอง** จนระบบสั่งงานวนลูปไม่หยุด
* **False Trigger จากเสียงผู้ชม/ลำโพงในฮอลล์** ตอนที่ไม่มีแรงกดบนหมอน (FSR = 0)
* **โมเดล AI เพี้ยนหน้างาน** เพราะเทรนเสียงคนละโดเมนกับลำโพงที่ใช้เดโม หรือไม่มีการปรับเทียบหน้างาน
* **เดโมรอนานเกินไป** (ต้องยืนรอกรน 10+ วินาทีแล้วยังไม่ติดเพราะจังหวะหายใจเว้นช่อง 1–2 วิ ทำให้ตัวนับรีเซ็ต)

---

## 3. Constraints

1. **แพลตฟอร์มและเครื่องมือบังคับ:** Arduino UNO Q 4GB (มีบอร์ดอยู่ในมือแล้ว) **ต้องเขียนและ Deploy ผ่าน Arduino App Lab เท่านั้น** และต้องสอดคล้องกับกรอบ `Sense → Connect → Analyze → Decide → Respond`
2. **ข้อจำกัดไฟฟ้าของ Arduino UNO Q (สำคัญมาก):**
   * ระดับลอจิกของ Header คือ **3.3V** (ไม่ใช่ 5V แบบ UNO R3)
   * ขา **A0, A1, และ D3 ไม่รองรับ 5V (Non-5V-Tolerant, max ~3.6V)** ส่วนขา Analog A0–A5 มีความละเอียด ADC 14-bit
3. **กรอบเวลาและสถานะของ:** เหลือเวลา **6 วัน (20–25 พ.ย.)** ก่อนวันแข่ง 26–27 พ.ย. — ยังไม่ได้จัดซื้ออุปกรณ์เสริมและยังไม่มี Audio Dataset ต้องเลือกของที่หาซื้อในไทยได้ทันทีและเก็บ Dataset ที่ตรงกับสภาพเดโมจริง
4. **เวลาการนำเสนอ:** Pitch 7 นาที + Q&A 3 นาที
5. **ความโปร่งใสและลิขสิทธิ์:** อุปกรณ์เสริม ซอฟต์แวร์ ไลบรารี ชุดข้อมูลเสียง (Dataset) และโมเดล AI ทั้งหมดต้องระบุแหล่งที่มาในเอกสาร Engineering Documentation

---

## 4. Approaches Considered

| แนวทาง | ภาพรวม | จุดเด่น | ข้อจำกัด / Trade-off |
| :--- | :--- | :--- | :--- |
| **A: Lean Demo-Proof** | คงเซ็นเซอร์เดิม (`Mic + Piezo`) เพิ่มแค่แยกไฟ Servo + Watchdog + State Machine/Demo Mode | งบต่ำสุด ทำเร็วสุด ตรงเอกสารเดิมเป๊ะ | Piezo วัดการมีหัวนอนนิ่ง ๆ ไม่ได้จริง เสี่ยงพังหน้างานและโดนจี้ตอน Q&A |
| **B: True Sensor Fusion & Stage-Ready *(ที่เลือก)*** | เพิ่ม **FSR** ทำหน้าที่วัดการมีคนนอน (Head Presence) แทน Piezo + ปรับ **Piezo** ไปจับ Motion Artifact ตอนพลิกตัว + **MG996R Metal-Gear Servo** แยกไฟ 5V/3A + วงจรกันไฟเกินขา ADC + State Machine 5 สถานะ + App Lab Bricks (`audio_classification` + `web_ui`) + Stage Demo Mode & Benchmark Tools | ปิดทุกจุดอ่อนทางฟิสิกส์และความปลอดภัย ตอบคำถาม Q&A ได้ขาด เดโมเห็นภาพชัดและไวใน 90 วิ ประกอบทันใน 6 วัน กวาดคะแนน 55 แต้มหลักได้คุ้มสุด | ซื้ออุปกรณ์เพิ่ม 2–3 ชิ้น (FSR, Servo เฟืองเหล็ก, MOSFET, Power Bank 5V/3A) และต้องจูน Threshold เพิ่ม 1 ตัว |
| **C: Silent Air-Bladder Pillow** | เหมือนแนวทาง B แต่เปลี่ยน Servo เป็นถุงลม + Micro Air Pump 5V + Solenoid Valve | เหมือนสินค้าจริงมาก นุ่ม รับน้ำหนักหัวจริงได้สบาย คะแนน Innovation สูง | เสี่ยงลมรั่ว ปั๊มพองช้าเกินไปสำหรับ Pitch 7 นาที และเสี่ยงทำไม่ทันใน 6 วัน |

**เหตุผลที่เลือกแนวทาง B (การตัดสินใจของผู้ใช้):** บาลานซ์ที่สุดระหว่างความสมจริงทางวิศวกรรม ความเสถียรบนเวทีเดโม 7 นาที และความเร็วในการสร้างภายใน 6 วัน ส่วน **แนวทาง C (Air Bladder)** จะถูกวางเป็น **Future Work / Product Roadmap** ในสไลด์ Pitch เพื่อเสริมคะแนน Feasibility & Impact

---

## 5. System Design (Approach B — Arduino App Lab Edition)

### 5.1 ภาพรวมสถาปัตยกรรม (End-to-End Architecture)

```mermaid
flowchart TD
    subgraph SENSE["1. SENSE"]
        MIC["USB Microphone<br/>(RAM only via Audio Brick)"]
        FSR["FSR Pressure Pad x1-2 (3.3V Divider)<br/>ยืนยันมีคนนอนอยู่บนหมอน (Presence)"]
        PIEZO["Analog Piezo + 3.3V Clamp<br/>ตรวจจับ Motion Artifact (พลิกตัวแรง)"]
    end

    subgraph MCU["STM32U585 — sketch/sketch.ino"]
        ADC["ADC Sampling & Filtering<br/>(50 Hz Window Mean / Peak)"]
        WD["Autonomous Safety Watchdog<br/>(safety_logic.h: 5s Timeout + <=35 deg)"]
        DRV_V["MOSFET/NPN + Flyback<br/>Coin Vibration Motor (Soft-Start PWM)"]
        DRV_S["External 5V/3A Power Bank<br/>MG996R Metal-Gear Servo"]
        LED["8x13 Blue LED Matrix<br/>Status Icons"]
    end

    subgraph APPLAB["Qualcomm QRB2210 — Arduino App Lab (app.yaml)"]
        AI["Brick: arduino:audio_classification<br/>Edge Impulse .eim (4 Classes)"]
        CAL["5s Baseline & Confidence Calibration"]
        FSM["Sensor Fusion & 5-State Engine<br/>(python/smart_pillow/)"]
        DB["SQLite Event & Benchmark Log"]
        WEB["Brick: arduino:web_ui (assets/)<br/>WebSocket send_message / on_message<br/>Normal / Stage Mode + E-Stop"]
    end

    FSR --> ADC
    PIEZO --> ADC
    ADC <-->|"2. CONNECT<br/>Arduino_RouterBridge ↔ app_utils.Bridge"| FSM
    MIC -->|"3. ANALYZE"| AI
    AI --> FSM
    CAL --> FSM
    FSM -->|"4. DECIDE → 5. RESPOND"| WD
    WD --> DRV_V
    WD --> DRV_S
    WD --> LED
    FSM <--> WEB
    FSM --> DB
    DB --> WEB
```

---

### 5.2 Hardware BOM, Pinout & Electrical Safety

#### รายการอุปกรณ์หลัก (BOM — หาซื้อในไทยได้ทันทีสำหรับกรอบ 6 วัน)
* **Main Board:** Arduino UNO Q 4GB *(มีอยู่แล้ว)* + สาย/อะแดปเตอร์ USB-C 5V/3A
* **Sense (Audio):** USB Condenser/Omnidirectional Microphone แบบตั้งใกล้ขอบหมอน (ต่อเข้าฝั่ง Linux ผ่าน USB-C Hub)
* **Sense (Head Presence):** **FSR402** (หรือ FSR แถบยาว) 1–2 แผ่น วางใต้ปลอกหมอนบริเวณกลางหมอน (ต่อขนานเข้าตัวต้านทาน Pull-down `10 kΩ` เพื่อขยายพื้นที่รับแรงกด)
* **Sense (Motion Artifact):** Analog Piezo Disc (27–35 mm) 1 แผ่น วางใต้ปลอกหมอน + วงจรป้องกัน (`1 MΩ` bleed resistor + `3.3V Zener diode` ขนานขา Piezo) — ใช้จับแรงกระแทก/พลิกตัวแรง ๆ ซึ่งให้สัญญาณแรงพอโดยไม่ต้องเพิ่มวงจร Preamp
* **Respond (Bruxism):** Coin Vibration Motor (3V–5V) 1–2 ตัว ขับผ่าน **N-Channel MOSFET (IRLZ44N / 2N7000) หรือ NPN Transistor (S8050 / 2N2222)** + ไดโอดกันไฟย้อน (**1N4148 / 1N5819 Flyback Diode**) + ตัวต้านทาน Gate/Base (`1 kΩ`)
* **Respond (Snoring):** **MG996R Metal-Gear Servo** (~10 kg·cm ที่ 5–6V) ยึดกับฐานแผ่นแข็งบานพับใต้หมอนด้านข้างเพื่อยกมุมหมอนเอียงขึ้น 10°–25° สำหรับ Tabletop Demo *(หากต้องการรับน้ำหนักศีรษะคนจริง 4–5 กก. ให้เปลี่ยนเป็น `DS3218 20kg` + แหล่งจ่ายไฟ `6.8V–7.4V 3A`)*
* **Power Separation (บังคับเพื่อความปลอดภัย):** แหล่งจ่ายไฟแยก **5V ≥ 3A** (เช่น Power Bank 5V/3A หรือ Adapter 5V/3A + คาปาซิเตอร์กรองไฟ `470µF–1000µF` คร่อมรางไฟ Servo) ต่อ **GND ร่วม (Common Ground)** กับ Arduino UNO Q

#### การจัดขา (Pin Mapping — เคารพข้อจำกัด 3.3V ของ UNO Q) *(My Call: หลีกเลี่ยงขา A0, A1, D3 ที่ไม่รองรับ 5V)*
| สัญญาณ | ขา STM32 | หมายเหตุทางไฟฟ้า |
| :--- | :--- | :--- |
| FSR Analog Input | `A2` | ต่อวงจรแบ่งแรงดันจากขา **3.3V** ของบอร์ด (ห้ามใช้ 5V) เพื่อให้แรงดันเข้าไม่เกิน 3.3V |
| Piezo Analog Input | `A3` | ขนาน `1 MΩ` + `3.3V Zener` ป้องกัน Spike แรงดันสูงตอนกด/ตบหมอนแรง ๆ |
| Coin Motor PWM | `D5` (PWM) | ขับผ่าน MOSFET/Transistor + Flyback Diode เท่านั้น ห้ามต่อมอเตอร์ตรงเข้าขา MCU |
| Servo PWM Signal | `D9` (PWM) | ใช้ไฟเลี้ยง Servo จากราง 5V/3A แยก (ต่อเฉพาะสาย Signal + Common GND เข้าบอร์ด) |
| Status Display | Onboard `8×13 LED Matrix` | ควบคุมตรงจาก STM32 |

---

### 5.3 Arduino App Lab Packaging & Bridge Protocol (Connect)

* **โครงสร้างโปรเจกต์ Arduino App Lab (บังคับตามเครื่องมือ UNO Q):**
  ```text
  smart_pillow_app/
  ├── app.yaml                  # ประกาศ Bricks: arduino:web_ui, arduino:audio_classification
  ├── python/
  │   ├── main.py               # Entry point ของ App Lab (App.run(), WebUI, AudioClassification, Bridge)
  │   ├── requirements.txt      # ไลบรารีเสริม (ถ้าจำเป็น)
  │   └── smart_pillow/         # Pure Python modules (เทสต์ pytest บน PC ได้โดยไม่ต้องมีบอร์ด)
  ├── sketch/
  │   ├── sketch.ino            # เฟิร์มแวร์ STM32U585 (Arduino_RouterBridge + ADC + PWM + Servo + LED)
  │   ├── safety_logic.h        # Pure C++ watchdog & hard clamps (เทสต์บน PC ได้)
  │   └── sketch.yaml           # Manifest ระบุบอร์ด UNO Q และไลบรารีของ Sketch
  └── assets/                   # หน้าเว็บ Dashboard (index.html, app.js, styles.css) สำหรับ WebUI Brick
  ```
* **สัญญาณ STM32 → Linux (RPC `Bridge.call("get_telemetry")` ทุก 100 ms / 10 Hz):**
  * `fsr_mean` (ค่าเฉลี่ยแรงกด 0–16383)
  * `piezo_peak` (ค่าจุดสูงสุดในหน้าต่าง 100 ms ใช้จับ Motion Artifact ตอนพลิกตัว/กระแทกหมอน)
  * `actuator_state` (`IDLE`, `HAPTIC_ACTIVE`, `SERVO_TILTED`, `WATCHDOG_TRIPPED`)
  * `servo_angle` (องศาปัจจุบัน)
* **สัญญาณ Linux → STM32 (RPC `Bridge.notify` / `Bridge.call` + Heartbeat ทุก 1 วินาที):**
  * `heartbeat()` — ยืนยันว่าฝั่ง Linux ยังทำงานปกติ
  * `haptic_pulse(intensity_pct, duration_ms)` — สั่งสั่นแบบ Soft-Start PWM (ค่อย ๆ ไต่จาก 20% → `intensity_pct` ใน 500 ms ค้างไว้แล้วลดลง)
  * `servo_tilt(target_angle, step_delay_ms)` — ค่อย ๆ ขยับ Servo จาก `0°` → `target_angle` ทีละ 1–2° เพื่อลดเสียงเฟืองและแรงกระชาก (Hard Limit ในเฟิร์มแวร์ไม่เกิน `35°`)
  * `servo_home(step_delay_ms)` — ค่อย ๆ คืน Servo กลับ `0°`
  * `stop_all()` — ตัดมอเตอร์สั่นทันที + คืน Servo กลับ `0°` (Emergency Stop)
  * `led_icon(icon_id)` — เปลี่ยนไอคอนบน 8×13 LED Matrix (`IDLE`, `MONITOR`, `BRUXISM`, `SNORE`, `FALSE_ALARM`, `ESTOP`)
* **Autonomous Safety Watchdog บน STM32:**
  * หากไม่ได้รับ `heartbeat` จาก Linux เกิน **5 วินาที** หรือมอเตอร์สั่นทำงานเกิน **3 วินาทีต่อเนื่อง** → STM32 ตัด PWM มอเตอร์เป็น 0 ทันที คืน Servo กลับ `0°` และขึ้นไอคอนเตือนบน LED Matrix โดยไม่ต้องรอคำสั่งจาก Linux

---

### 5.4 Edge AI Audio Pipeline & Dataset Strategy (Analyze)

* **การรันบน App Lab:** ใช้ **Audio Classification Brick (`arduino:audio_classification`)** ที่ผูกกับโมเดล **Edge Impulse `.eim`** ของทีมใน `app.yaml` — App Lab จัดการเชื่อมต่อ USB Microphone และรัน Inference แบบออฟไลน์ 100% บน Qualcomm QRB2210 โดยไม่บันทึกไฟล์เสียงลงดิสก์ (Privacy by Design)
* **4 คลาสเสียง & แผนสร้าง Dataset ภายใน Day 1–2 *(My Call: Domain-Matched กับลำโพงเดโม)*:**
  * **รูปแบบเดโมหลัก:** เปิดคลิปเสียงผ่าน **ลำโพงขนาดเล็กวางข้างหมอน** (ควบคุมจังหวะซ้ำได้ 100% ในฮอลล์แข่งที่เสียงดัง และไม่พลาดหน้ากรรมการ)
  * **วิธีเก็บ Dataset ให้ไร้ Domain Shift:** ใช้สคริปต์ช่วยอัดบน PC/UNO Q ผ่าน **USB Mic ตัวจริงที่ใช้แข่ง** โดยเปิดคลิปเสียงต้นฉบับผ่านลำโพงเดโมตัวจริงที่ระยะห่างเท่าวันแข่ง + ผสมเสียงพื้นหลังห้องแข่ง (crowd/hall ambient noise) แล้วอัปโหลดขึ้น Edge Impulse Studio เพื่อเทรนและส่ง `.eim` เข้า App Lab
    1. `Snoring`: ใช้ Public Snoring Dataset (เช่น Kaggle Snoring Dataset — ระบุแหล่งที่มาในเอกสาร) เปิดผ่านลำโพงเข้า USB Mic + คลิปกรนหลายโทน
    2. `Bruxism`: อัดชุดเสียงกัดฟัน/ขบกรามจำลอง (เสียงฟันเสียดสีความถี่สูงสั้น ๆ) ทีมอัดเอง + เปิดผ่านลำโพงเข้า USB Mic (ระบุเป็น Custom Team Dataset ในเอกสาร)
    3. `Normal_Breathing`: เสียงหายใจปกติอัดผ่าน USB Mic + ลำโพง
    4. `Quiet_Noise`: เสียงห้องเงียบ, เสียงพูดคุยรอบข้าง, เสียงแอร์/ฮอลล์งานแฟร์, และเสียงผ้าเสียดสีปลอกหมอน
  * **Target ขั้นต่ำ:** คลาสละ ~3–5 นาที (แบ่งเป็นคลิป 1–1.5 วิ ได้ ~150–250 samples/คลาส) เพียงพอสำหรับ Edge Impulse MFE + 1D-CNN ให้ได้ Accuracy > 85–90% ภายใน 1 วัน
* **5-Second Ambient & Sensor Calibration:** ปุ่ม 1-Click บน Dashboard เก็บค่า Baseline ของ FSR, Piezo และค่าความมั่นใจเสียงรบกวน (`Quiet_Noise` / false confidence ceiling) ของห้องแข่ง 5 วินาทีก่อนเริ่มเดโม เพื่อตั้งค่า Threshold อัตโนมัติ (กันเสียงในฮอลล์ทริกเกอร์ผิดพลาด)
* **หลักฐานการทดสอบ:** เก็บ **Confusion Matrix และ F1-Score** จาก Edge Impulse Test Set ไว้ใส่สไลด์และเอกสาร Engineering Documentation

---

### 5.5 Sensor Fusion & 5-State Decision Engine (Decide)

เพื่อแก้ปัญหา Self-Feedback, การพลิกตัวหลอกเป็นเสียงกัดฟัน และจังหวะเว้นหายใจตอนกรน ระบบตัดสินใจใน `python/smart_pillow/decision_engine.py` ใช้ **State Machine 5 สถานะ**:

```mermaid
stateDiagram-v2
    [*] --> IDLE_NO_HEAD
    IDLE_NO_HEAD --> MONITORING : FSR >= head_present_th (ค้าง 1s)
    MONITORING --> IDLE_NO_HEAD : FSR < head_present_th (ค้าง 3s)
    MONITORING --> INTERVENING : Bruxism OR Snore Trigger (ผ่าน Fusion)
    INTERVENING --> BLANKING : ครบระยะเวลาสั่น/เอียง หรือ เสียงเงียบ
    BLANKING --> COOLDOWN : ผ่าน Blanking Window (3s)
    COOLDOWN --> MONITORING : ครบ Cooldown หรือยังต่ำกว่าโควตา
    IDLE_NO_HEAD --> ESTOP : กด Emergency Stop
    MONITORING --> ESTOP : กด Emergency Stop
    INTERVENING --> ESTOP : กด Emergency Stop
    BLANKING --> ESTOP : กด Emergency Stop
    COOLDOWN --> ESTOP : กด Emergency Stop
    ESTOP --> IDLE_NO_HEAD : กด Reset / Resume
```

#### กฎการตัดสินใจ (Sensor Fusion Rules)
| เงื่อนไข | ตรรกะ Sensor Fusion | การตอบสนอง |
| :--- | :--- | :--- |
| **No Head (False Alarm จากคนข้าง ๆ / เสียงในฮอลล์)** | AI พบ `Snoring` หรือ `Bruxism` แต่ **`FSR < head_present_th`** | บันทึกเป็น `IGNORED_NO_HEAD` (แสดงไฟ/ป้าย False Alarm) ไม่สั่งงานฮาร์ดแวร์ |
| **Motion Artifact Rejection (พลิกตัว/ขยับหมอนแรง)** | **`piezo_peak >= motion_spike_th`** ในช่วง 1.5 วิล่าสุด (มีการพลิกตัว/กระแทกหมอนแรง) | ระงับการ Trigger `Bruxism` ชั่วคราว บันทึกเป็น `IGNORED_MOTION_ARTIFACT` ไม่สั่งงานฮาร์ดแวร์ |
| **Bruxism Trigger (นอนกัดฟันจริง)** | `FSR >= head_present_th` **AND** `piezo_peak < motion_spike_th` **AND** AI พบ `Bruxism >= conf_th` (ยืนยัน 2 หน้าต่างติดกันใน ~1.5–2 วิ) | สั่ง `haptic_pulse` (Soft-Start PWM 1.5–2 วิ) → เข้า `BLANKING` (3 วิ) |
| **Snoring Trigger (นอนกรนต่อเนื่อง)** | `FSR >= head_present_th` **AND** **Snore Sliding-Window Ratio ≥ 60%** (Normal Mode: กรนอย่างน้อย 6 จาก 10 วิล่าสุด / **Stage Mode**: กรนอย่างน้อย 2 จาก 3 วิล่าสุด) *(ไม่บังคับ Piezo เพื่อให้ทำงานสม่ำเสมอทั้งตอนนอนจริงและตอนเปิดเสียงเดโม)* | สั่ง `servo_tilt` (เอียงช้า ๆ ตามองศาที่ตั้ง) ค้างไว้จนเสียงกรนหายต่อเนื่อง 3–5 วิ (หรือครบ Max Hold Time 15 วิ) แล้วค่อย `servo_home` → เข้า `BLANKING` (3 วิ) |
| **Cooldown Quota** | สั่งงานครบโควตาในรอบเวลา (Normal: สูงสุด 3 ครั้ง / 10 นาที; Stage Mode: ปรับลดเป็น 5 วิได้) | ระงับการสั่งงานชั่วคราว บันทึก `SUPPRESSED_COOLDOWN` |
| **Blanking Window (3 วินาที)** | ระหว่างที่มอเตอร์สั่น/Servo กำลังหมุน และต่อเนื่องอีก 3 วินาทีหลังหยุด | **ระงับการนำผล AI และ Piezo ไป Trigger** เพื่อป้องกัน Self-Feedback Loop 100% |

---

### 5.6 Local Web Dashboard via `arduino:web_ui` Brick (Respond)

เสิร์ฟหน้าเว็บจากโฟลเดอร์ `assets/` ผ่าน **Arduino App Lab `WebUI` Brick** สื่อสารสองทางแบบเรียลไทม์ผ่าน `ui.send_message(...)` และ `@ui.on_message(...)` (เข้าถึงได้ทันทีจากเบราว์เซอร์บน Local Hotspot หรือ USB/LAN โดยไม่ง้ออินเทอร์เน็ต):

1. **Live Telemetry View (`send_message("telemetry", ...)` ทุก 100 ms):**
   * แถบสถานะ State Machine ปัจจุบัน (ตัวอักษรใหญ่ สีชัดเจน อ่านง่ายจากระยะไกลบนจอโปรเจกเตอร์)
   * กราฟ/เกจเรียลไทม์ของ `FSR (Head Presence + เส้น Threshold)`, `Piezo (Motion Spike + เส้น Reject Threshold)`, และ `Audio AI 4-Class Confidence Bars`
   * แถบแสดง Event ล่าสุด (`TRIGGERED`, `IGNORED_NO_HEAD`, `IGNORED_MOTION_ARTIFACT`, `SUPPRESSED_COOLDOWN`, `BRIDGE_DISCONNECTED`)
2. **Mode Switch & Stage Controls (`@ui.on_message(...)`):**
   * `set_mode`: สลับ **Normal Sleep Mode** (Snore window 10 วิ, Cooldown 3 ครั้ง/10 นาที) ↔ **Stage Demo Mode** (Snore window 3 วิ, Cooldown 5 วิ — พร้อมป้ายบอกชัดเจนบนหน้าจอเพื่อความโปร่งใสต่อกรรมการ)
   * `start_calibration`: ปุ่ม **1-Click Calibrate (5s)** วัด Baseline ของ FSR, Piezo และเสียงห้องแข่งหน้างาน
   * `manual_test`: ปุ่มทดสอบฮาร์ดแวร์ (`haptic`, `servo_tilt`, `servo_home`)
   * `estop` / `resume`: ปุ่มแดงขนาดใหญ่ **EMERGENCY STOP / RESUME**
   * `update_config`: Sliders ปรับ Thresholds (`FSR`, `Piezo Motion Spike`, `AI Confidence`, `PWM Intensity`, `Servo Angle`) แบบ Real-time
3. **Sleep Timeline & Benchmark Export (`SQLite` + `request_export_csv`):**
   * บันทึกทุก Event ลง **SQLite** ในโฟลเดอร์ข้อมูลของแอป
   * ปุ่ม **Export CSV** (ดาวน์โหลดไฟล์ CSV ตรงจากเบราว์เซอร์) และปุ่ม **Print Summary Report** (`@media print` layout พร้อมพิมพ์เป็น PDF แนบเอกสารประกวด)

---

### 5.7 Stage Demo Flow (90 วินาทีภายใน Pitch 7 นาที) & แผนการทดสอบ

#### บทเดโมสด 4 จังหวะ (~90 วินาที)
1. **ฉากที่ 1 — False Alarm Rejection (20 วิ):** เปิดเสียงกรนจากลำโพงข้างหมอน *โดยไม่วางมือกดหมอน* → หน้าเว็บโชว์ AI จับ `Snoring` สูง แต่ `FSR = 0` → สถานะขึ้น `IGNORED_NO_HEAD` (LED Matrix โชว์ไอคอน Ignore) มอเตอร์ไม่ทำงาน
2. **ฉากที่ 2 — Bruxism Haptic Interrupt + Motion Artifact (25 วิ):** วางมือกดหมอน (FSR ติด) + เปิดเสียงกัดฟัน → สถานะเข้า `INTERVENING (haptic_pulse)` มอเตอร์สั่นนุ่ม ๆ แบบ Soft-Start 1.5 วิ → เข้า `BLANKING` 3 วิ (โชว์ว่าไม่วนลูป) *(เสริมได้: ตบ/ขยับหมอนแรง ๆ โชว์ว่า Piezo ตัดเป็น `IGNORED_MOTION_ARTIFACT`)*
3. **ฉากที่ 3 — Snoring Positional Tilt (25 วิ):** วางมือกดหมอน + เปิดเสียงกรน 3 วิ (ใน Stage Mode) → Servo ค่อย ๆ ยกมุมหมอนเอียงขึ้นอย่างนุ่มนวล
4. **ฉากที่ 4 — Safety / Emergency Stop (20 วิ):** กดปุ่มแดง **Emergency Stop** บนหน้าเว็บ โชว์ว่าทุกอย่างหยุดและ Servo คืนองศา `0°` ทันที พร้อมอธิบาย Autonomous Watchdog 5 วิฝั่ง STM32

#### ชุดการทดสอบที่ต้องทำจริงก่อนวันแข่ง (เก็บคะแนน Functional Prototype & Testing 20 แต้ม)
1. **Model Evaluation:** เก็บ Accuracy & Confusion Matrix ของ `.eim` บนชุดทดสอบ 4 คลาส
2. **Sensor Fusion A/B Test:** ทดสอบเปิดเสียงกรน/กัดฟัน 20 ครั้ง (10 ครั้งไม่มีแรงกดบนหมอน + 10 ครั้งมีแรงกดบนหมอน) เทียบ False Positive Rate ระหว่าง *Audio-Only* vs *Audio + FSR + Piezo*
3. **Latency & Stability Test:** วัดเวลาตั้งแต่เริ่มเสียงจน Actuator ตอบสนอง, ทดสอบ Self-Feedback 10 รอบติดกัน, และทดสอบกดหยุดแอปใน App Lab เพื่อยืนยันว่า STM32 Watchdog ตัดมอเตอร์ภายใน 5 วิจริง

---

## 6. Decisions Made vs. Engineering Calls vs. Left to the Builder

### สิ่งที่ผู้ใช้ตัดสินใจแล้ว
* เป้าหมายคือ **Working Demo สำหรับ Arduino UNO Q AIoT & Edge AI Hackathon** (แข่งวันที่ 26–27 พ.ย. 2026, Pitch 7 นาที + Q&A 3 นาที)
* มีบอร์ด **Arduino UNO Q 4GB อยู่ในมือแล้ว** และ **ต้องเขียนและรันผ่าน Arduino App Lab เท่านั้น**
* เลือก **แนวทาง B (True Sensor Fusion & Stage-Ready Upgrade)** และวาง Air Bladder ไว้เป็น Future Work
* เจตนาเดิมของเซ็นเซอร์สัมผัสคือ **วัดว่ามีคนนอนอยู่บนหมอนจริงหรือไม่** (จึงอัปเกรดเป็น FSR เป็นตัวหลัก และคง Piezo ไว้จับ Motion Artifact)

### การตัดสินใจทางวิศวกรรมที่เคาะให้ (My Calls)
1. **App Lab Bricks + Testable Core Pattern:** ใช้ Official Bricks (`arduino:audio_classification`, `arduino:web_ui`, `arduino.app_utils.Bridge`) ที่ชั้น Entrypoint (`python/main.py`) และแยกแกนหลักทั้งหมดไว้ใน `python/smart_pillow/` กับ `sketch/safety_logic.h` ให้รัน Unit Test (`pytest` / `g++`) บนคอมฯ ได้ 100% ก่อนนำเข้า App Lab
2. **Tabletop Demo Rig (`MG996R` + Power Bank 5V/3A):** ออกแบบสำหรับวางโต๊ะเดโมใช้มือกด/น้ำหนักจำลอง (~0.5–1.5 กก.) เพื่อประกอบทันและปลอดภัยใน 6 วัน
3. **แยกหน้าที่เซ็นเซอร์ไม่ให้ขัดกับเดโม:** `Snoring` ใช้ `FSR + Audio AI` (ไม่บังคับ Piezo), `Bruxism` ใช้ `FSR + Audio AI` พร้อมให้ **Piezo** ทำหน้าที่ **Motion Artifact Rejection**
4. **Speaker-Matched Dataset Strategy:** เก็บและเทรน Dataset เสียง 4 คลาสผ่านลำโพงเดโมเข้า USB Mic ตัวจริง เพื่อตัดปัญหา Domain Shift ตอนแข่งในฮอลล์เสียงดัง
5. **Electrical & Safety Defaults:** หลีกเลี่ยงขา `A0, A1, D3`, บังคับ 3.3V divider บน FSR, `1 MΩ + 3.3V Zener` บน Piezo, MOSFET/NPN + Flyback Diode บน Coin Motor, STM32 Watchdog Timeout `5s`, Blanking Window `3s`, และ Snoring Sliding-Window Ratio `≥ 60%`

### สิ่งที่ปล่อยให้ผู้ลงมือสร้าง (Builder) ตัดสินใจตอนวางแผน/ประกอบจริง
* โครงสร้างยึดจับทางกล (Mechanical Mount) ระหว่าง Servo กับแผ่นดันมุมหมอน ตามวัสดุที่หาได้ไวที่สุด
* รายละเอียด Hyperparameters ของ Edge Impulse DSP (เช่น MFE vs MFCC) ระหว่างจูนให้ได้ F1-Score สูงสุด
* ธีมสี เลย์เอาต์หน้า Web Dashboard ใน `assets/` และลายพิกเซล 8×13 บน LED Matrix
* วิธีเชื่อมต่อเครื่องพรีเซนต์เข้า WebUI ของ App Lab หน้างาน (เช่น มือถือปล่อย Hotspot หรือสาย USB/LAN)
