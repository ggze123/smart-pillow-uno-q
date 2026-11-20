# Silent Bruxism & Snore Interruptor (Smart Pillow v2) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and verify a stage-ready Arduino UNO Q 4GB Smart Pillow prototype (STM32U585 firmware + Native Debian Linux Python Edge AI/Sensor Fusion engine + Local Web Dashboard + Benchmark/Demo tooling) in 6 days (Nov 20–25) for the Nov 26–27 Hackathon.

**Architecture:** STM32U585 samples 14-bit ADC sensors (`A2` FSR head presence, `A3` Piezo motion artifact), drives Soft-Start PWM coin vibration (`D5`), `MG996R` servo tilt (`D9`), and the `8×13` LED matrix, while enforcing an autonomous 5 s safety watchdog and 35° angle clamp. Native Debian Linux on the Qualcomm QRB2210 runs a RAM-only Edge Impulse `.eim` 4-class audio classifier, a 5-state Sensor Fusion engine, SQLite logging, and a FastAPI + WebSocket Stage Dashboard, communicating with STM32 via `Arduino_RouterBridge` / `arduino-router`.

**Tech Stack:** Python 3.11+ (`fastapi`, `uvicorn`, `websockets`, `numpy`, `sounddevice`, `edge_impulse_linux`, `arduino.app_utils.Bridge`, `sqlite3`, `pytest`, `httpx`), Vanilla HTML/CSS/JS Canvas Dashboard, Arduino C++ (STM32U585 Zephyr/Arduino Core, `Arduino_RouterBridge.h`, `Arduino_LED_Matrix.h`, `Servo.h`), g++ for host safety unit tests.

**Spec:** [2026-11-20-smart-pillow-design.md](../specs/2026-11-20-smart-pillow-design.md)

## Global Constraints

- Platform: Arduino UNO Q 4GB (Qualcomm QRB2210 Native Debian Linux + STM32U585 Zephyr/Arduino Core); no Cloud dependency, 100% offline/local execution.
- Electrical safety: UNO Q header logic is `3.3V`; `A0`, `A1`, and `D3` are non-5V-tolerant (max ~3.6V) and MUST NOT be used for sensor/actuator connections. Pinout: `A2` = FSR (3.3V divider, 10 kΩ pull-down), `A3` = Piezo (`1 MΩ` + `3.3V Zener`), `D5` = Coin Motor PWM (via MOSFET/NPN + flyback diode), `D9` = Servo PWM (external `5V/3A` power bank + Common GND).
- Privacy by design: Runtime audio inference MUST process `1.0–1.5s` windows (0.5s overlap) in RAM only and NEVER persist audio `.wav` files to disk.
- Safety limits (STM32 firmware-enforced): Heartbeat watchdog timeout = `5000 ms`, max continuous haptic vibration = `3000 ms`, servo hard angle clamp = `[0, 35]` degrees.
- Sensor Fusion rules: `FSR` gates both `Bruxism` and `Snoring`; `Piezo` rejects `Bruxism` motion artifacts (`piezo_peak >= motion_spike_th` within last `1.5s`) and is NOT required to trigger `Snoring`; `Bruxism` requires 2 consecutive windows above `conf_th`; `Snoring` requires sliding-window ratio `>= 0.60` (`10s` Normal / `3s` Stage); `Blanking Window` = `3.0s`; `Cooldown` = `3` interventions per `600s` (Normal) / `5s` (Stage).
- Timeline & Attribution: 6-day build (Nov 20–25, competition Nov 26–27, Pitch 7 min + Q&A 3 min). Every third-party dataset, model, library, and hardware component must be attributed in docs.

## Review Focus

1. **Stale/Disconnected Bridge Telemetry:** If STM32 telemetry stops arriving for `> 1.5s` or `Bridge` raises an error, the controller must NOT keep triggering actuators on cached `fsr_mean`, must emit a `BRIDGE_DISCONNECTED` fault on WebSocket, and must keep the FastAPI loop alive. *(Tested in Task 6)*
2. **Mid-Action Emergency Stop & Resume:** Pressing `Emergency Stop` while `INTERVENING` or `BLANKING` must immediately enter `ESTOP`, issue `CMD_STOP_ALL`, suppress all triggers even if FSR/audio stay high, and require explicit `RESUME` back to `IDLE_NO_HEAD`. *(Tested in Task 4)*
3. **NaN / Out-of-Range Sensor or Audio Inputs:** Negative or `> 16383` ADC values, empty/NaN audio scores, or empty audio buffers during calibration must be safely clamped/rejected without raising exceptions or corrupting thresholds. *(Tested in Task 3 & Task 4)*
4. **Snoring Sliding-Window Gaps & Mode Switching:** Silent 1–2 s exhale gaps must NOT reset snoring progress prematurely, and toggling `NORMAL ↔ STAGE` must evaluate strictly over the active mode's window without stale carryover. *(Tested in Task 4)*
5. **STM32 Watchdog Wrap-Around & Illegal Command Clamping:** Out-of-range commands (`angle=90`, `duration_ms=10000`, `intensity=150`) and 32-bit `millis()` wrap-around must remain clamped (`<=35°`, `<=3000 ms`) and trip `WATCHDOG_TRIPPED` cleanly at `5000 ms`. *(Tested in Task 8)*

## 6-Day Execution Roadmap

- **Day 1 (Nov 20):** Order/buy local BOM (`docs/hardware/bom-and-wiring.md`) + Tasks 1, 2, 3.
- **Day 2 (Nov 21):** Tasks 4, 5 + start Speaker-Matched Dataset recording (`record_dataset.py`).
- **Day 3 (Nov 22):** Train Edge Impulse `.eim` model + Tasks 6, 7.
- **Day 4 (Nov 23):** Assemble wiring/pillow rig + Task 8 (STM32 firmware + hardware test).
- **Day 5 (Nov 24):** End-to-end UNO Q integration + Task 9 (20-trial A/B benchmark, Confusion Matrix, Watchdog fault test).
- **Day 6 (Nov 25):** Dry-run 7-min Pitch + 90s Stage Demo rehearsal + freeze build & print report.

## File Structure

- `docs/hardware/bom-and-wiring.md` — Day-1 shopping list, pinout table, 3.3V-safe schematics, power separation checklist.
- `linux/pyproject.toml` — Python package metadata and dependencies.
- `linux/smart_pillow/models.py` — Typed enums and dataclasses shared across all Linux modules.
- `linux/smart_pillow/config.py` — Threshold config, Normal/Stage presets, validation/clamping.
- `linux/smart_pillow/storage.py` — SQLite event & calibration store, timeline query, A/B stats, CSV export.
- `linux/smart_pillow/calibration.py` — 5-second ambient RMS/dBFS and FSR/Piezo baseline calculator.
- `linux/smart_pillow/decision_engine.py` — Pure 5-State Machine + Triple-Signal Sensor Fusion logic (zero I/O).
- `linux/smart_pillow/bridge.py` — `HardwareBridge` protocol, `RouterBridgeClient` (`arduino.app_utils.Bridge`), `MockBridgeClient`.
- `linux/smart_pillow/audio_pipeline.py` — RAM-only sliding-window audio ring buffer + `.eim` classifier adapter + `MockAudioClassifier`.
- `linux/smart_pillow/controller.py` — Async runtime orchestrator wiring Bridge, Audio, Calibrator, FSM, SQLite, and WebSocket subscribers.
- `linux/smart_pillow/server.py` — FastAPI REST + WebSocket API and printable HTML summary report.
- `linux/smart_pillow/static/{index.html,app.js,styles.css}` — High-contrast Stage-Ready Web Dashboard.
- `firmware/smart_pillow_mcu/safety_logic.h` — Pure C++ hardware-independent safety state, watchdog, angle clamp, and PWM ramp logic.
- `firmware/smart_pillow_mcu/smart_pillow_mcu.ino` — STM32U585 sketch wiring `safety_logic.h` to ADC, PWM, Servo, 8×13 LED Matrix, and `Arduino_RouterBridge`.
- `firmware_tests/test_safety_logic.cpp` — Host-compiled C++ unit tests for `safety_logic.h`.
- `linux/scripts/record_dataset.py` — Offline tool to record labeled 1.5 s clips via USB Mic for Edge Impulse upload.
- `linux/scripts/run_ab_benchmark.py` — 20-trial A/B test runner (Audio-Only vs Sensor Fusion) producing CSV & summary metrics.
- `docs/dataset/edge-impulse-guide.md` — Dataset attribution, capture protocol, DSP/model settings, Confusion Matrix template.
- `docs/demo/stage-demo-and-testing.md` — 90-second 4-beat Stage Demo script, verification log, and Q&A defense notes.

---

### Task 1: Project Scaffolding, BOM Spec, Core Models & Config

**Files:**
- Create: `linux/pyproject.toml`
- Create: `linux/smart_pillow/__init__.py`
- Create: `linux/smart_pillow/models.py`
- Create: `linux/smart_pillow/config.py`
- Create: `docs/hardware/bom-and-wiring.md`
- Test: `linux/tests/test_config_and_models.py`

**Interfaces:**
- Consumes: None
- Produces:
  - `models.py`: `SystemState` (`IDLE_NO_HEAD`, `MONITORING`, `INTERVENING`, `BLANKING`, `COOLDOWN`, `ESTOP`), `OperatingMode` (`NORMAL`, `STAGE`), `ActuatorState` (`IDLE`, `HAPTIC_ACTIVE`, `SERVO_TILTED`, `WATCHDOG_TRIPPED`), `EventOutcome` (`TRIGGERED_BRUXISM`, `TRIGGERED_SNORING`, `IGNORED_NO_HEAD`, `IGNORED_MOTION_ARTIFACT`, `SUPPRESSED_COOLDOWN`, `ESTOP_ACTIVATED`, `BRIDGE_DISCONNECTED`), `LedIcon` (`IDLE`, `MONITOR`, `BRUXISM`, `SNORE`, `FALSE_ALARM`, `ESTOP`), `TelemetryFrame(timestamp_s: float, fsr_mean: int, piezo_peak: int, actuator_state: ActuatorState, servo_angle: int)`, `AudioPrediction(timestamp_s: float, scores: dict[str, float], rms_energy: float)`, `BridgeCommand(kind: str, params: dict[str, int | str])`, `DecisionEvent(timestamp_s: float, state: SystemState, outcome: EventOutcome | None, audio_label: str, confidence: float, fsr_mean: int, piezo_peak: int, mode: OperatingMode)`
  - `config.py`: `PillowConfig` dataclass + `PillowConfig.for_mode(mode: OperatingMode) -> PillowConfig` + `PillowConfig.with_updates(**kwargs) -> PillowConfig` (clamps `servo_angle_deg` to `[5, 35]`, `haptic_intensity_pct` to `[20, 100]`, `haptic_duration_ms` to `[500, 3000]`, `ai_conf_th` to `[0.3, 0.99]`).

- [ ] **Step 1: Initialize repo & Python package scaffold**
  - Run `git init` in `d:\hackathon ahahh` (create `.gitignore` excluding `.venv/`, `__pycache__/`, `*.db`, `*.eim`, `dataset_raw/`).
  - Create `linux/pyproject.toml` with dependencies: `fastapi`, `uvicorn[standard]`, `numpy`, `sounddevice`, `pydantic`, dev deps `pytest`, `pytest-asyncio`, `httpx`, and `[tool.pytest.ini_options] pythonpath = ["."]`.
  - Create `linux/smart_pillow/__init__.py`.

- [ ] **Step 2: Write the failing test in `linux/tests/test_config_and_models.py`**

```python
from smart_pillow.config import PillowConfig
from smart_pillow.models import OperatingMode, SystemState

def test_mode_presets_match_spec():
    normal = PillowConfig.for_mode(OperatingMode.NORMAL)
    stage = PillowConfig.for_mode(OperatingMode.STAGE)
    assert normal.snore_window_s == 10.0
    assert normal.snore_ratio_th == 0.60
    assert normal.cooldown_max_count == 3
    assert normal.cooldown_window_s == 600.0
    assert normal.blanking_window_s == 3.0
    assert stage.snore_window_s == 3.0
    assert stage.cooldown_window_s == 5.0

def test_config_updates_clamp_safety_bounds():
    cfg = PillowConfig.for_mode(OperatingMode.STAGE).with_updates(
        servo_angle_deg=90, haptic_duration_ms=9999, haptic_intensity_pct=150
    )
    assert cfg.servo_angle_deg == 35
    assert cfg.haptic_duration_ms == 3000
    assert cfg.haptic_intensity_pct == 100
    assert SystemState.IDLE_NO_HEAD.value == "IDLE_NO_HEAD"
```

- [ ] **Step 3: Run test to verify it fails**
  - Run: `pytest linux/tests/test_config_and_models.py -v`
  - Expected: FAIL (`ModuleNotFoundError` / `ImportError`).

- [ ] **Step 4: Implement `linux/smart_pillow/models.py`, `linux/smart_pillow/config.py`, and `docs/hardware/bom-and-wiring.md`**
  - Implement all enums and dataclasses listed in `Produces` above (`fsr_mean` / `piezo_peak` use 14-bit ADC range `0..16383`, default `head_present_th=1200`, `motion_spike_th=4500`, `ai_conf_th=0.70`, `audio_energy_min=0.01`, `servo_angle_deg=20`, `haptic_intensity_pct=65`, `haptic_duration_ms=1500`).
  - Write `docs/hardware/bom-and-wiring.md` with the exact BOM, pinout (`A2`, `A3`, `D5`, `D9`, `8×13 LED Matrix`, avoiding `A0`, `A1`, `D3`), ASCII wiring diagrams (3.3V FSR divider, `1 MΩ + 3.3V Zener` Piezo clamp, MOSFET + flyback coin motor, external `5V/3A` Servo rail + Common GND), and pre-power-on multimeter checklist.

- [ ] **Step 5: Run test to verify it passes**
  - Run: `pytest linux/tests/test_config_and_models.py -v`
  - Expected: PASS (2 passed).

- [ ] **Step 6: Commit**
  - `git add .gitignore linux/pyproject.toml linux/smart_pillow/__init__.py linux/smart_pillow/models.py linux/smart_pillow/config.py linux/tests/test_config_and_models.py docs/hardware/bom-and-wiring.md`
  - `git commit -m "feat: scaffold smart pillow models, config presets, and hardware BOM"`

---

### Task 2: SQLite Event Store & CSV/Summary Export

**Files:**
- Create: `linux/smart_pillow/storage.py`
- Test: `linux/tests/test_storage.py`

**Interfaces:**
- Consumes: `DecisionEvent`, `OperatingMode`, `SystemState`, `EventOutcome` from `smart_pillow.models`.
- Produces:
  - `EventStore(db_path: str | Path)`
  - `EventStore.log_event(event: DecisionEvent) -> int`
  - `EventStore.list_events(limit: int = 100) -> list[dict]` (newest first)
  - `EventStore.get_summary() -> dict[str, int]` (counts per `EventOutcome` plus `total`)
  - `EventStore.export_csv() -> str` (RFC 4180 CSV with header `id,timestamp_s,mode,state,outcome,audio_label,confidence,fsr_mean,piezo_peak`)
  - `EventStore.clear() -> None`

- [ ] **Step 1: Write the failing test in `linux/tests/test_storage.py`**

```python
from smart_pillow.models import DecisionEvent, EventOutcome, OperatingMode, SystemState
from smart_pillow.storage import EventStore

def test_log_query_summary_and_csv(tmp_path):
    store = EventStore(tmp_path / "events.db")
    store.log_event(DecisionEvent(100.0, SystemState.MONITORING, EventOutcome.IGNORED_NO_HEAD, "Snoring", 0.91, 100, 50, OperatingMode.STAGE))
    store.log_event(DecisionEvent(105.0, SystemState.INTERVENING, EventOutcome.TRIGGERED_BRUXISM, "Bruxism", 0.88, 3200, 400, OperatingMode.STAGE))
    rows = store.list_events(limit=10)
    assert len(rows) == 2
    assert rows[0]["outcome"] == "TRIGGERED_BRUXISM"
    summary = store.get_summary()
    assert summary["IGNORED_NO_HEAD"] == 1
    assert summary["TRIGGERED_BRUXISM"] == 1
    assert summary["total"] == 2
    csv_text = store.export_csv()
    assert "timestamp_s,mode,state,outcome,audio_label,confidence,fsr_mean,piezo_peak" in csv_text
    assert "TRIGGERED_BRUXISM" in csv_text
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest linux/tests/test_storage.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `EventStore` in `linux/smart_pillow/storage.py`**
  - Use stdlib `sqlite3` with WAL mode enabled, creating table `events` on `__init__` if absent, and stdlib `csv.StringIO` for `export_csv()`.

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest linux/tests/test_storage.py -v`
  - Expected: PASS.

- [ ] **Step 5: Commit**
  - `git add linux/smart_pillow/storage.py linux/tests/test_storage.py`
  - `git commit -m "feat: add SQLite EventStore with summary and CSV export"`

---

### Task 3: 5-Second Ambient Noise & Sensor Baseline Calibrator

**Files:**
- Create: `linux/smart_pillow/calibration.py`
- Test: `linux/tests/test_calibration.py`

**Interfaces:**
- Consumes: `TelemetryFrame` from `smart_pillow.models`.
- Produces:
  - `CalibrationResult(audio_energy_min: float, fsr_baseline: int, piezo_baseline: int, recommended_head_present_th: int, recommended_motion_spike_th: int)`
  - `Calibrator(duration_s: float = 5.0)`
  - `Calibrator.start(now_s: float) -> None`
  - `Calibrator.is_active(now_s: float) -> bool`
  - `Calibrator.add_sample(rms_energy: float | None = None, telemetry: TelemetryFrame | None = None) -> None`
  - `Calibrator.finish() -> CalibrationResult`

- [ ] **Step 1: Write the failing test in `linux/tests/test_calibration.py` (including Review Focus #3 NaN/empty-buffer resilience)**

```python
import math
from smart_pillow.calibration import Calibrator
from smart_pillow.models import ActuatorState, TelemetryFrame

def test_calibrator_computes_safe_thresholds():
    cal = Calibrator(duration_s=5.0)
    cal.start(0.0)
    assert cal.is_active(2.5) is True
    assert cal.is_active(5.1) is False
    for rms in [0.02, 0.03, 0.025]:
        cal.add_sample(rms_energy=rms)
    for fsr, pz in [(200, 100), (250, 150)]:
        cal.add_sample(telemetry=TelemetryFrame(1.0, fsr, pz, ActuatorState.IDLE, 0))
    res = cal.finish()
    assert res.audio_energy_min > 0.03
    assert res.fsr_baseline == 225
    assert res.recommended_head_present_th > res.fsr_baseline
    assert res.recommended_motion_spike_th > res.piezo_baseline

def test_calibrator_ignores_nan_and_handles_empty_buffers():
    cal = Calibrator(duration_s=5.0)
    cal.start(0.0)
    cal.add_sample(rms_energy=float("nan"))
    cal.add_sample(rms_energy=-5.0)
    cal.add_sample(telemetry=TelemetryFrame(1.0, -50, 99999, ActuatorState.IDLE, 0))
    res = cal.finish()
    assert math.isfinite(res.audio_energy_min) and res.audio_energy_min >= 0.005
    assert 0 <= res.fsr_baseline <= 16383
    assert 0 <= res.recommended_head_present_th <= 16383
    assert 0 <= res.recommended_motion_spike_th <= 16383
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest linux/tests/test_calibration.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `Calibrator` and `CalibrationResult` in `linux/smart_pillow/calibration.py`**
  - Filter out non-finite/negative `rms_energy` values; clamp ADC readings to `[0, 16383]`; set `audio_energy_min = max(0.005, mean_rms * 1.5 + 0.005)` (fallback `0.01` when empty), `recommended_head_present_th = min(15000, fsr_baseline + 800)`, `recommended_motion_spike_th = min(16000, max(piezo_baseline * 3, piezo_baseline + 2500))`.

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest linux/tests/test_calibration.py -v`
  - Expected: PASS (2 passed).

- [ ] **Step 5: Commit**
  - `git add linux/smart_pillow/calibration.py linux/tests/test_calibration.py`
  - `git commit -m "feat: add 5s ambient noise and sensor baseline calibrator"`

---

### Task 4: 5-State Machine & Triple-Signal Sensor Fusion Decision Engine

**Files:**
- Create: `linux/smart_pillow/decision_engine.py`
- Test: `linux/tests/test_decision_engine.py`

**Interfaces:**
- Consumes: `PillowConfig` (`config.py`), `SystemState`, `OperatingMode`, `EventOutcome`, `LedIcon`, `TelemetryFrame`, `AudioPrediction`, `BridgeCommand`, `DecisionEvent` (`models.py`).
- Produces:
  - `StepResult(state: SystemState, commands: list[BridgeCommand], event: DecisionEvent | None)`
  - `DecisionEngine(config: PillowConfig)`
  - `DecisionEngine.update_config(config: PillowConfig) -> None` (clears snoring window history when `mode` changes)
  - `DecisionEngine.snore_window_count` (read-only `int` property returning current sliding-window sample count)
  - `DecisionEngine.step(now_s: float, telemetry: TelemetryFrame, audio: AudioPrediction | None) -> StepResult`
  - `DecisionEngine.trigger_estop(now_s: float) -> StepResult`
  - `DecisionEngine.resume_from_estop(now_s: float) -> StepResult`

- [ ] **Step 1: Write failing tests in `linux/tests/test_decision_engine.py` covering all spec rules + Review Focus #2, #3, #4**

```python
from smart_pillow.config import PillowConfig
from smart_pillow.decision_engine import DecisionEngine
from smart_pillow.models import ActuatorState, AudioPrediction, EventOutcome, OperatingMode, SystemState, TelemetryFrame

def _tel(t, fsr, piezo=100, act=ActuatorState.IDLE):
    return TelemetryFrame(t, fsr, piezo, act, 0)

def _aud(t, label, conf=0.90, rms=0.10):
    return AudioPrediction(t, {label: conf}, rms)

def test_no_head_rejects_snoring_and_bruxism():
    eng = DecisionEngine(PillowConfig.for_mode(OperatingMode.STAGE))
    res = eng.step(1.0, _tel(1.0, fsr=100), _aud(1.0, "Snoring", 0.95))
    assert res.state == SystemState.IDLE_NO_HEAD
    assert res.event.outcome == EventOutcome.IGNORED_NO_HEAD
    assert not any(c.kind in ("CMD_HAPTIC_PULSE", "CMD_SERVO_TILT") for c in res.commands)

def test_bruxism_triggers_after_two_windows_and_enters_blanking():
    eng = DecisionEngine(PillowConfig.for_mode(OperatingMode.STAGE))
    eng.step(0.0, _tel(0.0, fsr=3000), None)
    eng.step(1.1, _tel(1.1, fsr=3000), None)  # enters MONITORING after 1s FSR hold
    r1 = eng.step(1.5, _tel(1.5, fsr=3000), _aud(1.5, "Bruxism", 0.9))
    assert r1.state == SystemState.MONITORING
    r2 = eng.step(2.0, _tel(2.0, fsr=3000), _aud(2.0, "Bruxism", 0.9))
    assert r2.state == SystemState.INTERVENING
    assert any(c.kind == "CMD_HAPTIC_PULSE" for c in r2.commands)
    assert r2.event.outcome == EventOutcome.TRIGGERED_BRUXISM
    # Finish 1.5s haptic -> BLANKING for 3s -> ignores self-feedback audio
    r3 = eng.step(3.6, _tel(3.6, fsr=3000), _aud(3.6, "Bruxism", 0.99))
    assert r3.state == SystemState.BLANKING
    assert not any(c.kind == "CMD_HAPTIC_PULSE" for c in r3.commands)

def test_piezo_motion_spike_rejects_bruxism_within_1_5s():
    eng = DecisionEngine(PillowConfig.for_mode(OperatingMode.STAGE))
    eng.step(0.0, _tel(0.0, fsr=3000), None)
    eng.step(1.1, _tel(1.1, fsr=3000, piezo=8000), None)
    r = eng.step(2.0, _tel(2.0, fsr=3000, piezo=100), _aud(2.0, "Bruxism", 0.95))
    assert r.event.outcome == EventOutcome.IGNORED_MOTION_ARTIFACT
    assert r.state == SystemState.MONITORING

def test_snoring_sliding_window_survives_exhale_gap_and_resets_on_mode_switch():
    eng = DecisionEngine(PillowConfig.for_mode(OperatingMode.STAGE))
    eng.step(0.0, _tel(0.0, fsr=3000), None)
    eng.step(1.1, _tel(1.1, fsr=3000), None)
    eng.step(1.5, _tel(1.5, fsr=3000), _aud(1.5, "Snoring", 0.9))
    eng.step(2.0, _tel(2.0, fsr=3000), _aud(2.0, "Normal_Breathing", 0.8))  # silent gap
    eng.step(2.5, _tel(2.5, fsr=3000), _aud(2.5, "Snoring", 0.9))
    r = eng.step(3.5, _tel(3.5, fsr=3000), _aud(3.5, "Snoring", 0.9))  # >=60% over 3s window
    assert r.state == SystemState.INTERVENING
    assert any(c.kind == "CMD_SERVO_TILT" for c in r.commands)
    assert r.event.outcome == EventOutcome.TRIGGERED_SNORING
    # Mode switch resets sliding window history cleanly
    eng.update_config(PillowConfig.for_mode(OperatingMode.NORMAL))
    assert eng.snore_window_count == 0

def test_estop_mid_intervention_requires_explicit_resume_and_handles_nan():
    eng = DecisionEngine(PillowConfig.for_mode(OperatingMode.STAGE))
    eng.step(0.0, _tel(0.0, fsr=3000), None)
    eng.step(1.1, _tel(1.1, fsr=3000), None)
    eng.step(1.5, _tel(1.5, fsr=3000), _aud(1.5, "Bruxism", 0.9))
    eng.step(2.0, _tel(2.0, fsr=3000), _aud(2.0, "Bruxism", 0.9))
    estop = eng.trigger_estop(2.2)
    assert estop.state == SystemState.ESTOP
    assert any(c.kind == "CMD_STOP_ALL" for c in estop.commands)
    still_estop = eng.step(2.5, _tel(2.5, fsr=5000), _aud(2.5, "Bruxism", float("nan")))
    assert still_estop.state == SystemState.ESTOP
    resumed = eng.resume_from_estop(3.0)
    assert resumed.state == SystemState.IDLE_NO_HEAD
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest linux/tests/test_decision_engine.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `DecisionEngine` in `linux/smart_pillow/decision_engine.py`**
  - Implement all transitions per Section 5.5 of the spec:
    - `IDLE_NO_HEAD → MONITORING` after `FSR >= head_present_th` continuously for `1.0s`; `MONITORING → IDLE_NO_HEAD` after `FSR < head_present_th` continuously for `3.0s`.
    - Clamp incoming ADC values to `[0, 16383]`; ignore audio predictions whose `rms_energy < audio_energy_min` or contain non-finite scores.
    - Track `last_motion_spike_s` whenever `piezo_peak >= motion_spike_th`; reject `Bruxism` if `(now_s - last_motion_spike_s) <= 1.5`.
    - Maintain `_snore_history: list[tuple[float, bool]]` pruned to `config.snore_window_s`; trigger `Snoring` once window coverage `>= 0.75 * config.snore_window_s` (or `>= 3` samples) and positive ratio `>= config.snore_ratio_th`.
    - During `INTERVENING`: for haptic, finish after `haptic_duration_ms / 1000.0`; for servo tilt, issue `CMD_SERVO_HOME` once `Snoring` is absent continuously for `3.0s` or max hold `15.0s` elapses; transition to `BLANKING` (`3.0s`) then `COOLDOWN` / `MONITORING`.
    - Enforce cooldown quota (`cooldown_max_count` within `cooldown_window_s`), logging `SUPPRESSED_COOLDOWN` when exceeded.
    - Emit `CMD_LED_ICON` whenever the logical LED icon changes.

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest linux/tests/test_decision_engine.py -v`
  - Expected: PASS (5 passed).

- [ ] **Step 5: Commit**
  - `git add linux/smart_pillow/decision_engine.py linux/tests/test_decision_engine.py`
  - `git commit -m "feat: implement 5-state sensor fusion decision engine"`

---

### Task 5: Hardware Bridge Abstraction & RAM-Only Edge Impulse Audio Pipeline

**Files:**
- Create: `linux/smart_pillow/bridge.py`
- Create: `linux/smart_pillow/audio_pipeline.py`
- Test: `linux/tests/test_bridge_and_audio.py`

**Interfaces:**
- Consumes: `ActuatorState`, `AudioPrediction`, `BridgeCommand`, `TelemetryFrame` (`models.py`).
- Produces:
  - `bridge.py`: `HardwareBridge` (Protocol with `get_telemetry() -> TelemetryFrame`, `send_heartbeat() -> None`, `send_command(cmd: BridgeCommand) -> None`), `RouterBridgeClient` (wraps `arduino.app_utils.Bridge.call` / `Bridge.notify` to STM32 RPC endpoints `get_telemetry`, `heartbeat`, `haptic_pulse`, `servo_tilt`, `servo_home`, `stop_all`, `led_icon`), `MockBridgeClient` (in-memory simulator with `inject_telemetry(...)`, `sent_commands: list[BridgeCommand]`, `heartbeat_count: int`, and `simulate_error: Exception | None = None`).
  - `audio_pipeline.py`: `AudioClassifier` (Protocol with `classify(samples: np.ndarray, sample_rate: int) -> dict[str, float]`), `EdgeImpulseClassifier(model_path: str)` (wraps `edge_impulse_linux.audio.AudioImpulseRunner` / `ImpulseRunner` in RAM only), `MockAudioClassifier`, `SlidingAudioBuffer(sample_rate: int = 16000, window_s: float = 1.5, hop_s: float = 0.5)` with `push(chunk: np.ndarray) -> np.ndarray | None` and `compute_rms(samples: np.ndarray) -> float`.

- [ ] **Step 1: Write failing tests in `linux/tests/test_bridge_and_audio.py`**

```python
import numpy as np
from smart_pillow.audio_pipeline import MockAudioClassifier, SlidingAudioBuffer
from smart_pillow.bridge import MockBridgeClient
from smart_pillow.models import ActuatorState, BridgeCommand

def test_sliding_audio_buffer_emits_window_on_hop_without_disk_io():
    buf = SlidingAudioBuffer(sample_rate=16000, window_s=1.0, hop_s=0.5)
    assert buf.push(np.ones(8000, dtype=np.float32) * 0.5) is None
    win1 = buf.push(np.ones(8000, dtype=np.float32) * 0.5)
    assert win1 is not None and len(win1) == 16000
    assert abs(buf.compute_rms(win1) - 0.5) < 1e-4
    win2 = buf.push(np.zeros(8000, dtype=np.float32))
    assert win2 is not None and len(win2) == 16000

def test_mock_bridge_and_classifier_contract():
    bridge = MockBridgeClient()
    bridge.inject_telemetry(fsr_mean=2500, piezo_peak=300)
    tel = bridge.get_telemetry()
    assert tel.fsr_mean == 2500 and tel.actuator_state == ActuatorState.IDLE
    bridge.send_command(BridgeCommand("CMD_HAPTIC_PULSE", {"intensity_pct": 65, "duration_ms": 1500}))
    assert bridge.sent_commands[-1].kind == "CMD_HAPTIC_PULSE"
    clf = MockAudioClassifier({"Snoring": 0.92})
    assert clf.classify(np.zeros(16000, dtype=np.float32), 16000)["Snoring"] == 0.92
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest linux/tests/test_bridge_and_audio.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `linux/smart_pillow/bridge.py` and `linux/smart_pillow/audio_pipeline.py`**
  - In `RouterBridgeClient`, lazily import `arduino.app_utils.Bridge` so unit tests run cleanly on non-UNO-Q machines; map each `BridgeCommand.kind` to exact STM32 RPC method names (`haptic_pulse`, `servo_tilt`, `servo_home`, `stop_all`, `led_icon`).
  - In `EdgeImpulseClassifier`, lazily import `edge_impulse_linux.runner.ImpulseRunner` and run inference strictly on in-memory float/int16 PCM arrays (never touch filesystem).

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest linux/tests/test_bridge_and_audio.py -v`
  - Expected: PASS (2 passed).

- [ ] **Step 5: Commit**
  - `git add linux/smart_pillow/bridge.py linux/smart_pillow/audio_pipeline.py linux/tests/test_bridge_and_audio.py`
  - `git commit -m "feat: add Arduino RouterBridge client and RAM-only audio pipeline"`

---

### Task 6: Async Runtime Controller (10 Hz Telemetry, 1 Hz Heartbeat, Fault Isolation)

**Files:**
- Create: `linux/smart_pillow/controller.py`
- Test: `linux/tests/test_controller.py`

**Interfaces:**
- Consumes: `PillowConfig`, `EventStore`, `Calibrator`, `DecisionEngine`, `HardwareBridge`, `AudioClassifier`, `SlidingAudioBuffer`.
- Produces:
  - `PillowController(config: PillowConfig, bridge: HardwareBridge, classifier: AudioClassifier, store: EventStore, now_fn: Callable[[], float] = time.monotonic)`
  - `PillowController.tick_once(audio_chunk: np.ndarray | None = None) -> dict` (executes one telemetry poll, optional heartbeat, optional audio classification, calibration sampling, FSM step, command dispatch, SQLite logging, and returns JSON-serializable snapshot dict)
  - `PillowController.set_mode(mode: OperatingMode) -> PillowConfig`
  - `PillowController.update_thresholds(**kwargs) -> PillowConfig`
  - `PillowController.start_calibration() -> None`
  - `PillowController.manual_test(action: str) -> None` (`"haptic"` | `"servo_tilt"` | `"servo_home"`)
  - `PillowController.emergency_stop() -> None` / `resume() -> None`
  - `PillowController.start_background() -> None` / `stop_background() -> None` (10 Hz async loop + WebSocket subscriber fan-out via `subscribe() -> asyncio.Queue`).

- [ ] **Step 1: Write failing tests in `linux/tests/test_controller.py` (including Review Focus #1 Stale/Disconnected Bridge)**

```python
import numpy as np
from smart_pillow.audio_pipeline import MockAudioClassifier
from smart_pillow.bridge import MockBridgeClient
from smart_pillow.config import PillowConfig
from smart_pillow.controller import PillowController
from smart_pillow.models import EventOutcome, OperatingMode
from smart_pillow.storage import EventStore

def test_controller_handles_bridge_disconnect_without_false_trigger(tmp_path):
    now = [0.0]
    bridge = MockBridgeClient()
    bridge.inject_telemetry(fsr_mean=4000, piezo_peak=100)
    ctrl = PillowController(
        PillowConfig.for_mode(OperatingMode.STAGE),
        bridge,
        MockAudioClassifier({"Bruxism": 0.95}),
        EventStore(tmp_path / "c.db"),
        now_fn=lambda: now[0],
    )
    ctrl.tick_once()
    now[0] = 2.0
    bridge.simulate_error = RuntimeError("router disconnected")
    snap = ctrl.tick_once(audio_chunk=np.ones(24000, dtype=np.float32) * 0.2)
    assert snap["bridge_connected"] is False
    assert snap["last_outcome"] == EventOutcome.BRIDGE_DISCONNECTED.value
    assert not any(c.kind == "CMD_HAPTIC_PULSE" for c in bridge.sent_commands)

def test_controller_sends_heartbeat_every_1s_and_supports_manual_estop(tmp_path):
    now = [0.0]
    bridge = MockBridgeClient()
    ctrl = PillowController(
        PillowConfig.for_mode(OperatingMode.STAGE),
        bridge,
        MockAudioClassifier({"Quiet_Noise": 0.95}),
        EventStore(tmp_path / "c2.db"),
        now_fn=lambda: now[0],
    )
    ctrl.tick_once()
    now[0] = 1.1
    ctrl.tick_once()
    assert bridge.heartbeat_count >= 2
    ctrl.emergency_stop()
    assert bridge.sent_commands[-1].kind == "CMD_STOP_ALL"
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest linux/tests/test_controller.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `PillowController` in `linux/smart_pillow/controller.py`**
  - On `bridge.get_telemetry()` failure or telemetry age `> 1.5s`, mark `bridge_connected=False`, emit `BRIDGE_DISCONNECTED` once per outage, skip FSM actuation triggers, and keep returning snapshots safely.
  - Auto-apply `Calibrator.finish()` results into `PillowConfig` once 5 s elapse.

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest linux/tests/test_controller.py -v`
  - Expected: PASS (2 passed).

- [ ] **Step 5: Commit**
  - `git add linux/smart_pillow/controller.py linux/tests/test_controller.py`
  - `git commit -m "feat: add async PillowController with heartbeat and disconnect protection"`

---

### Task 7: FastAPI Server & Stage-Ready Local Web Dashboard

**Files:**
- Create: `linux/smart_pillow/server.py`
- Create: `linux/smart_pillow/static/index.html`
- Create: `linux/smart_pillow/static/app.js`
- Create: `linux/smart_pillow/static/styles.css`
- Test: `linux/tests/test_server.py`

**Interfaces:**
- Consumes: `PillowController` (`controller.py`), `EventStore` (`storage.py`), `OperatingMode` (`models.py`).
- Produces:
  - `create_app(controller: PillowController) -> FastAPI`
  - REST endpoints:
    - `GET /` → serves `static/index.html`
    - `GET /api/status` → current snapshot + config + event summary
    - `POST /api/mode` (`{"mode": "NORMAL" | "STAGE"}`)
    - `POST /api/config` (threshold/actuator sliders)
    - `POST /api/calibrate` (triggers 5 s ambient/baseline calibration)
    - `POST /api/manual/{action}` (`haptic`, `servo_tilt`, `servo_home`)
    - `POST /api/estop` & `POST /api/resume`
    - `GET /api/events` & `GET /api/export/csv` & `GET /report` (printable PDF-ready summary page)
  - WebSocket: `WS /ws/telemetry` streaming live 10 Hz snapshots.

- [ ] **Step 1: Write failing API & WebSocket tests in `linux/tests/test_server.py`**

```python
from fastapi.testclient import TestClient
from smart_pillow.audio_pipeline import MockAudioClassifier
from smart_pillow.bridge import MockBridgeClient
from smart_pillow.config import PillowConfig
from smart_pillow.controller import PillowController
from smart_pillow.models import OperatingMode
from smart_pillow.server import create_app
from smart_pillow.storage import EventStore

def test_rest_controls_export_and_report(tmp_path):
    ctrl = PillowController(
        PillowConfig.for_mode(OperatingMode.NORMAL),
        MockBridgeClient(),
        MockAudioClassifier({"Quiet_Noise": 0.9}),
        EventStore(tmp_path / "api.db"),
    )
    client = TestClient(create_app(ctrl))
    assert client.get("/").status_code == 200
    r = client.post("/api/mode", json={"mode": "STAGE"})
    assert r.status_code == 200 and r.json()["snore_window_s"] == 3.0
    assert client.post("/api/estop").status_code == 200
    assert client.get("/api/status").json()["state"] == "ESTOP"
    assert client.post("/api/resume").status_code == 200
    assert client.get("/api/export/csv").headers["content-type"].startswith("text/csv")
    assert client.get("/report").status_code == 200
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest linux/tests/test_server.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `server.py` and the Stage-Ready Dashboard (`index.html`, `app.js`, `styles.css`)**
  - Dashboard MUST display: huge color-coded State Banner (`IDLE_NO_HEAD`, `MONITORING`, `INTERVENING`, `BLANKING`, `COOLDOWN`, `ESTOP`), prominent `NORMAL / STAGE DEMO MODE` badge + toggle, live FSR gauge + threshold marker, Piezo motion-spike meter, 4-class AI confidence bars, 1-Click `Calibrate (5s)` button, `Test Haptic` / `Test Servo` buttons, large red `EMERGENCY STOP` / `RESUME` button, real-time threshold sliders, live Event Timeline, and `Export CSV` / `Print Report` links.

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest linux/tests/test_server.py -v`
  - Expected: PASS.

- [ ] **Step 5: Commit**
  - `git add linux/smart_pillow/server.py linux/smart_pillow/static linux/tests/test_server.py`
  - `git commit -m "feat: add FastAPI WebSocket server and Stage-Ready dashboard"`

---

### Task 8: STM32U585 Firmware & Host C++ Safety Unit Tests

**Files:**
- Create: `firmware/smart_pillow_mcu/safety_logic.h`
- Create: `firmware/smart_pillow_mcu/smart_pillow_mcu.ino`
- Test: `firmware_tests/test_safety_logic.cpp`

**Interfaces:**
- Consumes: `Arduino_RouterBridge.h`, `Arduino_LED_Matrix.h`, `Servo.h` (on STM32 target).
- Produces:
  - `safety_logic.h`: pure C++ `SafetyController` struct with `clamp_angle(int deg) -> int` (`[0, 35]`), `clamp_haptic_ms(uint32_t ms) -> uint32_t` (`[0, 3000]`), `clamp_intensity(int pct) -> int` (`[0, 100]`), `on_heartbeat(uint32_t now_ms)`, `start_haptic(int pct, uint32_t dur_ms, uint32_t now_ms)`, `start_servo(int target_deg)`, `stop_all()`, `tick(uint32_t now_ms)` (enforces 5000 ms watchdog with unsigned subtraction wrap-around safety, 3000 ms continuous vibration cap, and 500 ms Soft-Start PWM ramp from 20% to target intensity).
  - `smart_pillow_mcu.ino`: registers `Bridge.provide` handlers (`get_telemetry`, `heartbeat`, `haptic_pulse`, `servo_tilt`, `servo_home`, `stop_all`, `led_icon`), samples `A2` (FSR) and `A3` (Piezo) at 50 Hz with 14-bit resolution (`analogReadResolution(14)`), drives `D5` PWM and `D9` Servo, and updates 6 custom 8×13 LED Matrix icons.

- [ ] **Step 1: Write failing host C++ safety tests in `firmware_tests/test_safety_logic.cpp` (pins Review Focus #5)**

```cpp
#include <cassert>
#include <cstdint>
#include "../firmware/smart_pillow_mcu/safety_logic.h"

int main() {
    SafetyController sc;
    sc.on_heartbeat(0);
    assert(sc.clamp_angle(90) == 35);
    assert(sc.clamp_angle(-10) == 0);
    assert(sc.clamp_haptic_ms(10000) == 3000);
    sc.start_haptic(150, 10000, 100);
    sc.tick(200);  // mid Soft-Start ramp
    assert(sc.current_pwm_pct() >= 20 && sc.current_pwm_pct() < 100);
    sc.tick(3200); // > 3000ms cap -> motor MUST auto-cut
    assert(sc.current_pwm_pct() == 0);
    // Test 32-bit millis() wrap-around watchdog at 5000ms
    uint32_t near_wrap = 0xFFFFF800u;
    sc.on_heartbeat(near_wrap);
    sc.start_servo(30);
    sc.tick(near_wrap + 4999u);
    assert(!sc.watchdog_tripped());
    sc.tick(near_wrap + 5001u);
    assert(sc.watchdog_tripped());
    assert(sc.target_servo_deg() == 0 && sc.current_pwm_pct() == 0);
    return 0;
}
```

- [ ] **Step 2: Compile & run test to verify it fails**
  - Run: `g++ -std=c++17 firmware_tests/test_safety_logic.cpp -o firmware_tests/test_safety_logic.exe`
  - Expected: FAIL (`safety_logic.h: No such file or directory`). *(If `g++` is not installed locally on Windows, verify syntax/run via WSL or UNO Q Debian `g++`.)*

- [ ] **Step 3: Implement `firmware/smart_pillow_mcu/safety_logic.h` and `firmware/smart_pillow_mcu/smart_pillow_mcu.ino`**
  - Keep `safety_logic.h` header-only with zero Arduino dependencies so it compiles identically on host `g++` and STM32 Zephyr/Arduino Core.
  - In `smart_pillow_mcu.ino`, never touch `A0`, `A1`, or `D3`; step Servo smoothly (`1°–2°` per `step_delay_ms`) toward `sc.target_servo_deg()` and snap to `0°` on `stop_all` or watchdog trip.

- [ ] **Step 4: Compile & run host C++ test to verify it passes**
  - Run: `g++ -std=c++17 firmware_tests/test_safety_logic.cpp -o firmware_tests/test_safety_logic.exe && firmware_tests/test_safety_logic.exe`
  - Expected: Exit code `0` (all assertions pass).

- [ ] **Step 5: Commit**
  - `git add firmware/smart_pillow_mcu/safety_logic.h firmware/smart_pillow_mcu/smart_pillow_mcu.ino firmware_tests/test_safety_logic.cpp`
  - `git commit -m "feat: add STM32U585 firmware and verified safety watchdog logic"`

---

### Task 9: Speaker-Matched Dataset Tool, 20-Trial A/B Benchmark & Stage Demo Pack

**Files:**
- Create: `linux/scripts/__init__.py`
- Create: `linux/scripts/record_dataset.py`
- Create: `linux/scripts/run_ab_benchmark.py`
- Create: `docs/dataset/edge-impulse-guide.md`
- Create: `docs/demo/stage-demo-and-testing.md`
- Test: `linux/tests/test_benchmark_script.py`

**Interfaces:**
- Consumes: `DecisionEngine`, `PillowConfig`, `EventStore`, `TelemetryFrame`, `AudioPrediction`.
- Produces:
  - `run_ab_benchmark.py`: `evaluate_trials(trials: list[dict], config: PillowConfig) -> dict` returning `audio_only_fp_rate`, `fusion_fp_rate`, `audio_only_tp_rate`, `fusion_tp_rate`, and CLI writing `benchmark_results.csv` + Markdown table for Pitch slides.
  - `record_dataset.py`: CLI (`--label {Snoring,Bruxism,Normal_Breathing,Quiet_Noise} --count N --duration 1.5 --out dataset_raw`) recording 16 kHz mono WAVs via USB Mic for Edge Impulse upload (separate offline tool; never imported by runtime).
  - `docs/dataset/edge-impulse-guide.md` & `docs/demo/stage-demo-and-testing.md`: step-by-step dataset attribution, Edge Impulse MFE/1D-CNN export to `.eim`, 90-second 4-beat live demo cue sheet, and Q&A defense answers.

- [ ] **Step 1: Write failing test in `linux/tests/test_benchmark_script.py`**

```python
from scripts.run_ab_benchmark import evaluate_trials
from smart_pillow.config import PillowConfig
from smart_pillow.models import OperatingMode

def test_ab_benchmark_shows_fusion_eliminates_no_head_false_positives():
    trials = (
        [{"head_present": False, "label": "Snoring", "conf": 0.92, "motion_spike": False}] * 10
        + [{"head_present": True, "label": "Snoring", "conf": 0.92, "motion_spike": False}] * 10
    )
    metrics = evaluate_trials(trials, PillowConfig.for_mode(OperatingMode.STAGE))
    assert metrics["audio_only_fp_rate"] == 1.0
    assert metrics["fusion_fp_rate"] == 0.0
    assert metrics["fusion_tp_rate"] == 1.0
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest linux/tests/test_benchmark_script.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `record_dataset.py`, `run_ab_benchmark.py`, and both documentation guides**
  - Ensure `docs/dataset/edge-impulse-guide.md` explicitly records attribution sections for Kaggle Snoring Dataset, Custom Team Bruxism/Noise recordings, Edge Impulse `.eim`, Arduino UNO Q libraries, and FastAPI/SQLite to satisfy the 5-pt Engineering Documentation rubric.

- [ ] **Step 4: Run full test suite to verify everything passes**
  - Run: `pytest linux/tests -v`
  - Expected: ALL tests PASS.

- [ ] **Step 5: Commit**
  - `git add linux/scripts linux/tests/test_benchmark_script.py docs/dataset/edge-impulse-guide.md docs/demo/stage-demo-and-testing.md`
  - `git commit -m "feat: add dataset capture tool, A/B benchmark runner, and stage demo docs"`
