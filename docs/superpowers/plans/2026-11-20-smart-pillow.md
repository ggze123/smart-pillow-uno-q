# Silent Bruxism & Snore Interruptor (Smart Pillow v2) Implementation Plan

> **For agentic workers:** REQUIRED SUB-SKILL: Use superpowers:subagent-driven-development (recommended) or superpowers:executing-plans to implement this plan task-by-task. Steps use checkbox (`- [ ]`) syntax for tracking.

**Goal:** Build and verify a stage-ready **Arduino App Lab** Smart Pillow application for Arduino UNO Q 4GB (STM32U585 sketch + Python App Lab Bricks + Edge Impulse `.eim` + Stage Dashboard + Benchmark/Demo pack) in 6 days (Nov 20–25) for the Nov 26–27 Hackathon.

**Architecture:** Packaged directly as an Arduino App Lab project (`smart_pillow_app/` with `app.yaml`, `python/main.py`, `sketch/sketch.ino`, `sketch/sketch.yaml`, `assets/`). STM32U585 samples 14-bit ADC sensors (`A2` FSR presence, `A3` Piezo motion artifact), drives Soft-Start PWM coin vibration (`D5`), `MG996R` servo tilt (`D9`), and the `8×13` LED matrix while enforcing an autonomous 5 s safety watchdog and 35° hard angle clamp. The Qualcomm QRB2210 side uses official App Lab Bricks (`arduino:audio_classification` for offline Edge Impulse `.eim` inference, `arduino:web_ui` for live WebSocket dashboard, and `arduino.app_utils.Bridge` for STM32 RPC), wrapped around pure-Python core modules (`python/smart_pillow/`) so all logic is unit-tested on PC via `pytest` before running in App Lab.

**Tech Stack:** Arduino App Lab (`app.yaml`, `arduino:audio_classification`, `arduino:web_ui`, `arduino.app_utils.App`/`Bridge`), Python 3.11+ (`sqlite3`, `dataclasses`, `pytest`), Vanilla HTML/CSS/JS Canvas Dashboard in `assets/`, Arduino C++ (`Arduino_RouterBridge.h`, `Arduino_LED_Matrix.h`, `Servo.h`), `g++` for host safety unit tests, Edge Impulse Studio (`.eim`).

**Spec:** [2026-11-20-smart-pillow-design.md](../specs/2026-11-20-smart-pillow-design.md)

## Global Constraints

- Platform & Tooling: Arduino UNO Q 4GB, **developed and deployed via Arduino App Lab** (`smart_pillow_app/` layout with `app.yaml`, `python/main.py`, `sketch/sketch.ino`, `sketch/sketch.yaml`, `assets/`); 100% offline/local execution, no Cloud dependency.
- Electrical safety: UNO Q header logic is `3.3V`; `A0`, `A1`, and `D3` are non-5V-tolerant (max ~3.6V) and MUST NOT be used. Pinout: `A2` = FSR (3.3V divider, 10 kΩ pull-down), `A3` = Piezo (`1 MΩ` + `3.3V Zener`), `D5` = Coin Motor PWM (MOSFET/NPN + flyback diode), `D9` = Servo PWM (external `5V/3A` power bank + Common GND).
- Privacy by design: Runtime audio classification via `arduino:audio_classification` Brick runs offline in RAM only and NEVER writes `.wav` files to disk.
- Safety limits (STM32 firmware-enforced): Heartbeat watchdog timeout = `5000 ms`, max continuous haptic vibration = `3000 ms`, servo hard angle clamp = `[0, 35]` degrees.
- Sensor Fusion rules: `FSR` gates both `Bruxism` and `Snoring`; `Piezo` rejects `Bruxism` motion artifacts (`piezo_peak >= motion_spike_th` within last `1.5s`) and is NOT required to trigger `Snoring`; `Bruxism` requires 2 consecutive windows above `conf_th`; `Snoring` requires sliding-window ratio `>= 0.60` (`10s` Normal / `3s` Stage); `Blanking Window` = `3.0s`; `Cooldown` = `3` interventions per `600s` (Normal) / `5s` (Stage).
- Timeline & Attribution: 6-day build (Nov 20–25, competition Nov 26–27, Pitch 7 min + Q&A 3 min). Every third-party dataset, model, library, and hardware component must be attributed in docs.

## Review Focus

1. **Stale/Disconnected Bridge Telemetry:** If STM32 telemetry stops arriving for `> 1.5s` or `Bridge.call` raises an exception, the controller must NOT keep triggering actuators on cached `fsr_mean`, must emit a `BRIDGE_DISCONNECTED` fault to WebUI, and must keep the App Lab loop alive. *(Tested in Task 6)*
2. **Mid-Action Emergency Stop & Resume:** Pressing `Emergency Stop` while `INTERVENING` or `BLANKING` must immediately enter `ESTOP`, issue `stop_all`, suppress all triggers even if FSR/audio stay high, and require explicit `RESUME` back to `IDLE_NO_HEAD`. *(Tested in Task 4)*
3. **NaN / Out-of-Range Sensor or Audio Inputs:** Negative or `> 16383` ADC values, empty/NaN audio scores from the Audio Brick, or zero samples during calibration must be clamped/rejected cleanly without raising unhandled exceptions or corrupting thresholds. *(Tested in Task 3 & Task 4)*
4. **Snoring Sliding-Window Gaps & Mode Switching:** Silent 1–2 s exhale gaps must NOT reset snoring progress prematurely, and toggling `NORMAL ↔ STAGE` must evaluate strictly over the active mode's window without stale carryover. *(Tested in Task 4)*
5. **STM32 Watchdog Wrap-Around & Illegal Command Clamping:** Out-of-range commands (`angle=90`, `duration_ms=10000`, `intensity=150`) and 32-bit `millis()` wrap-around must remain clamped (`<=35°`, `<=3000 ms`) and trip `WATCHDOG_TRIPPED` cleanly at `5000 ms`. *(Tested in Task 8)*

## 6-Day Execution Roadmap

- **Day 1 (Nov 20):** Order/buy local BOM (`docs/hardware/bom-and-wiring.md`) + Tasks 1, 2, 3.
- **Day 2 (Nov 21):** Tasks 4, 5 + start Speaker-Matched Dataset recording (`tools/record_dataset.py`).
- **Day 3 (Nov 22):** Train Edge Impulse `.eim` model + deploy to App Lab + Tasks 6, 7.
- **Day 4 (Nov 23):** Assemble wiring/pillow rig + Task 8 (`sketch/sketch.ino` + `safety_logic.h` via App Lab).
- **Day 5 (Nov 24):** End-to-end UNO Q App Lab integration + Task 9 (20-trial A/B benchmark, Confusion Matrix, Watchdog fault test).
- **Day 6 (Nov 25):** Dry-run 7-min Pitch + 90s Stage Demo rehearsal + freeze App Lab build & print report.

## File Structure

- `docs/hardware/bom-and-wiring.md` — Day-1 shopping list, pinout table, 3.3V-safe schematics, power separation checklist.
- `pyproject.toml` — Root pytest configuration (`pythonpath = ["smart_pillow_app/python", "."]`) for PC unit testing.
- `smart_pillow_app/app.yaml` — Arduino App Lab manifest declaring `arduino:web_ui` and `arduino:audio_classification` (pointing to custom `.eim`).
- `smart_pillow_app/python/requirements.txt` — Minimal extra Python requirements (if needed inside App Lab).
- `smart_pillow_app/python/main.py` — Arduino App Lab entrypoint wiring `App`, `Bridge`, `AudioClassification`, `WebUI`, and `PillowController`.
- `smart_pillow_app/python/smart_pillow/models.py` — Typed enums and dataclasses shared across all modules.
- `smart_pillow_app/python/smart_pillow/config.py` — Threshold config, Normal/Stage presets, safety clamping.
- `smart_pillow_app/python/smart_pillow/storage.py` — SQLite event store, timeline query, summary stats, RFC 4180 CSV export.
- `smart_pillow_app/python/smart_pillow/calibration.py` — 5-second FSR/Piezo baseline & ambient confidence floor calibrator.
- `smart_pillow_app/python/smart_pillow/decision_engine.py` — Pure 5-State Machine + Triple-Signal Sensor Fusion logic (zero I/O).
- `smart_pillow_app/python/smart_pillow/bridge.py` — `HardwareBridge` protocol, `AppLabBridgeClient` (`arduino.app_utils.Bridge`), `MockBridgeClient`.
- `smart_pillow_app/python/smart_pillow/audio_adapter.py` — `AudioSource` protocol, `AppLabAudioSource` (`arduino.app_bricks.audio_classification.AudioClassification`), `MockAudioSource`.
- `smart_pillow_app/python/smart_pillow/controller.py` — Runtime loop orchestrator (10 Hz telemetry, 1 Hz heartbeat, audio polling, calibration, FSM → Bridge → SQLite → WebUI snapshot).
- `smart_pillow_app/python/smart_pillow/webui_adapter.py` — `WebUIChannel` protocol & message router mapping WebUI events (`set_mode`, `update_config`, `start_calibration`, `manual_test`, `estop`, `resume`, `request_export_csv`) to `PillowController`.
- `smart_pillow_app/assets/{index.html,app.js,styles.css}` — High-contrast Stage-Ready Web Dashboard served by `arduino:web_ui` Brick (with printable PDF report view).
- `smart_pillow_app/sketch/safety_logic.h` — Pure C++ hardware-independent safety watchdog, angle clamp (`<=35°`), 3 s haptic cap, and PWM ramp logic.
- `smart_pillow_app/sketch/sketch.ino` — STM32U585 sketch wiring `safety_logic.h` to ADC (`A2`, `A3`), PWM (`D5`), Servo (`D9`), `8×13` LED Matrix, and `Arduino_RouterBridge`.
- `smart_pillow_app/sketch/sketch.yaml` — Arduino App Lab sketch profile & library declarations.
- `firmware_tests/test_safety_logic.cpp` — Host-compiled C++ unit tests for `safety_logic.h`.
- `tools/record_dataset.py` — Offline PC/UNO Q tool to record labeled 1.5 s WAV clips via USB Mic for Edge Impulse upload (outside App Lab runtime).
- `tools/run_ab_benchmark.py` — Automated 20-trial A/B test runner (Audio-Only vs Sensor Fusion) producing CSV & Markdown summary table.
- `docs/dataset/edge-impulse-guide.md` — Dataset attribution, capture protocol, Edge Impulse → App Lab `.eim` deployment steps, Confusion Matrix log.
- `docs/demo/stage-demo-and-testing.md` — 90-second 4-beat Stage Demo script, verification log, and Q&A defense notes.
- `tests/test_*.py` — PC-runnable pytest suite covering every Python module and App Lab adapter via mocks.

---

### Task 1: App Lab Scaffold, Hardware BOM, Core Models & Config

**Files:**
- Create: `.gitignore`
- Create: `pyproject.toml`
- Create: `smart_pillow_app/app.yaml`
- Create: `smart_pillow_app/python/requirements.txt`
- Create: `smart_pillow_app/python/smart_pillow/__init__.py`
- Create: `smart_pillow_app/python/smart_pillow/models.py`
- Create: `smart_pillow_app/python/smart_pillow/config.py`
- Create: `docs/hardware/bom-and-wiring.md`
- Test: `tests/test_config_and_models.py`

**Interfaces:**
- Consumes: None
- Produces:
  - `models.py`: `SystemState` (`IDLE_NO_HEAD`, `MONITORING`, `INTERVENING`, `BLANKING`, `COOLDOWN`, `ESTOP`), `OperatingMode` (`NORMAL`, `STAGE`), `ActuatorState` (`IDLE`, `HAPTIC_ACTIVE`, `SERVO_TILTED`, `WATCHDOG_TRIPPED`), `EventOutcome` (`TRIGGERED_BRUXISM`, `TRIGGERED_SNORING`, `IGNORED_NO_HEAD`, `IGNORED_MOTION_ARTIFACT`, `SUPPRESSED_COOLDOWN`, `ESTOP_ACTIVATED`, `BRIDGE_DISCONNECTED`), `LedIcon` (`IDLE`, `MONITOR`, `BRUXISM`, `SNORE`, `FALSE_ALARM`, `ESTOP`), `TelemetryFrame(timestamp_s: float, fsr_mean: int, piezo_peak: int, actuator_state: ActuatorState, servo_angle: int)`, `AudioPrediction(timestamp_s: float, scores: dict[str, float])`, `BridgeCommand(kind: str, params: dict[str, int | str])`, `DecisionEvent(timestamp_s: float, state: SystemState, outcome: EventOutcome | None, audio_label: str, confidence: float, fsr_mean: int, piezo_peak: int, mode: OperatingMode)`.
  - `config.py`: `PillowConfig` dataclass + `PillowConfig.for_mode(mode: OperatingMode) -> PillowConfig` + `PillowConfig.with_updates(**kwargs) -> PillowConfig` (clamps `servo_angle_deg` to `[5, 35]`, `haptic_intensity_pct` to `[20, 100]`, `haptic_duration_ms` to `[500, 3000]`, `ai_conf_th` to `[0.3, 0.99]`).

- [ ] **Step 1: Create App Lab manifest, `.gitignore`, and root `pyproject.toml`**
  - Create `.gitignore` excluding `.venv/`, `__pycache__/`, `*.db`, `*.eim`, `dataset_raw/`, `*.exe`.
  - Create `pyproject.toml` with `[tool.pytest.ini_options] pythonpath = ["smart_pillow_app/python", "."]`.
  - Create `smart_pillow_app/app.yaml` declaring `name: "Silent Bruxism & Snore Interruptor"`, `ports: [5001]`, and `bricks:` for `arduino:web_ui: {}` and `arduino:audio_classification:` with `EI_AUDIO_CLASSIFICATION_MODEL: /home/arduino/.arduino-bricks/ei-models/smart_pillow.eim` (placeholder path updated on Day 3).
  - Create `smart_pillow_app/python/requirements.txt` and `smart_pillow_app/python/smart_pillow/__init__.py`.

- [ ] **Step 2: Write the failing test in `tests/test_config_and_models.py`**

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
  - Run: `pytest tests/test_config_and_models.py -v`
  - Expected: FAIL (`ModuleNotFoundError`).

- [ ] **Step 4: Implement `models.py`, `config.py`, and `docs/hardware/bom-and-wiring.md`**
  - Implement all enums and dataclasses listed in `Produces` (`fsr_mean` / `piezo_peak` use 14-bit ADC range `0..16383`; defaults: `head_present_th=1200`, `motion_spike_th=4500`, `ai_conf_th=0.70`, `servo_angle_deg=20`, `haptic_intensity_pct=65`, `haptic_duration_ms=1500`).
  - Write `docs/hardware/bom-and-wiring.md` with the Day-1 local shopping list, exact pinout (`A2`, `A3`, `D5`, `D9`, `8×13 LED Matrix`, avoiding `A0`, `A1`, `D3`), ASCII 3.3V-safe schematics (FSR 3.3V divider, Piezo `1 MΩ + 3.3V Zener`, MOSFET/NPN + flyback coin motor, external `5V/3A` Servo rail + Common GND), and pre-power-on multimeter checklist.

- [ ] **Step 5: Run test to verify it passes**
  - Run: `pytest tests/test_config_and_models.py -v`
  - Expected: PASS (2 passed).

- [ ] **Step 6: Commit**
  - `git add .gitignore pyproject.toml smart_pillow_app/app.yaml smart_pillow_app/python/requirements.txt smart_pillow_app/python/smart_pillow/__init__.py smart_pillow_app/python/smart_pillow/models.py smart_pillow_app/python/smart_pillow/config.py tests/test_config_and_models.py docs/hardware/bom-and-wiring.md`
  - `git commit -m "feat: scaffold App Lab project, core models, config presets, and hardware BOM"`

---

### Task 2: SQLite Event Store & CSV/Summary Export

**Files:**
- Create: `smart_pillow_app/python/smart_pillow/storage.py`
- Test: `tests/test_storage.py`

**Interfaces:**
- Consumes: `DecisionEvent`, `OperatingMode`, `SystemState`, `EventOutcome` from `smart_pillow.models`.
- Produces:
  - `EventStore(db_path: str | Path)`
  - `EventStore.log_event(event: DecisionEvent) -> int`
  - `EventStore.list_events(limit: int = 100) -> list[dict]` (newest first)
  - `EventStore.get_summary() -> dict[str, int]` (counts per `EventOutcome` plus `total`)
  - `EventStore.export_csv() -> str` (RFC 4180 CSV with header `id,timestamp_s,mode,state,outcome,audio_label,confidence,fsr_mean,piezo_peak`)
  - `EventStore.clear() -> None`

- [ ] **Step 1: Write the failing test in `tests/test_storage.py`**

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
  - Run: `pytest tests/test_storage.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `EventStore` in `smart_pillow_app/python/smart_pillow/storage.py`**
  - Use stdlib `sqlite3` with WAL mode enabled (zero extra pip packages needed inside App Lab), creating table `events` on `__init__` if absent, and stdlib `csv` + `io.StringIO` for `export_csv()`.

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest tests/test_storage.py -v`
  - Expected: PASS.

- [ ] **Step 5: Commit**
  - `git add smart_pillow_app/python/smart_pillow/storage.py tests/test_storage.py`
  - `git commit -m "feat: add SQLite EventStore with summary and CSV export"`

---

### Task 3: 5-Second Sensor Baseline & Ambient Confidence Calibrator

**Files:**
- Create: `smart_pillow_app/python/smart_pillow/calibration.py`
- Test: `tests/test_calibration.py`

**Interfaces:**
- Consumes: `AudioPrediction`, `TelemetryFrame` from `smart_pillow.models`.
- Produces:
  - `CalibrationResult(fsr_baseline: int, piezo_baseline: int, ambient_max_trigger_conf: float, recommended_head_present_th: int, recommended_motion_spike_th: int, recommended_ai_conf_th: float)`
  - `Calibrator(duration_s: float = 5.0)`
  - `Calibrator.start(now_s: float) -> None`
  - `Calibrator.is_active(now_s: float) -> bool`
  - `Calibrator.add_sample(audio: AudioPrediction | None = None, telemetry: TelemetryFrame | None = None) -> None`
  - `Calibrator.finish() -> CalibrationResult`

- [ ] **Step 1: Write failing tests in `tests/test_calibration.py` (pins Review Focus #3 NaN/empty-buffer resilience)**

```python
import math
from smart_pillow.calibration import Calibrator
from smart_pillow.models import ActuatorState, AudioPrediction, TelemetryFrame

def test_calibrator_computes_safe_thresholds():
    cal = Calibrator(duration_s=5.0)
    cal.start(0.0)
    assert cal.is_active(2.5) is True
    assert cal.is_active(5.1) is False
    cal.add_sample(audio=AudioPrediction(1.0, {"Snoring": 0.40, "Bruxism": 0.35, "Quiet_Noise": 0.25}))
    cal.add_sample(audio=AudioPrediction(2.0, {"Snoring": 0.55, "Bruxism": 0.20, "Quiet_Noise": 0.25}))
    for fsr, pz in [(200, 100), (250, 150)]:
        cal.add_sample(telemetry=TelemetryFrame(1.0, fsr, pz, ActuatorState.IDLE, 0))
    res = cal.finish()
    assert res.fsr_baseline == 225
    assert res.recommended_head_present_th > res.fsr_baseline
    assert res.recommended_motion_spike_th > res.piezo_baseline
    assert res.recommended_ai_conf_th >= 0.70

def test_calibrator_ignores_nan_and_handles_empty_buffers():
    cal = Calibrator(duration_s=5.0)
    cal.start(0.0)
    cal.add_sample(audio=AudioPrediction(1.0, {"Snoring": float("nan"), "Bruxism": -1.0}))
    cal.add_sample(telemetry=TelemetryFrame(1.0, -50, 99999, ActuatorState.IDLE, 0))
    res = cal.finish()
    assert math.isfinite(res.recommended_ai_conf_th) and 0.65 <= res.recommended_ai_conf_th <= 0.95
    assert 0 <= res.fsr_baseline <= 16383
    assert 0 <= res.recommended_head_present_th <= 16383
    assert 0 <= res.recommended_motion_spike_th <= 16383
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest tests/test_calibration.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `Calibrator` and `CalibrationResult` in `smart_pillow_app/python/smart_pillow/calibration.py`**
  - Filter out non-finite/out-of-range confidence values; clamp ADC readings to `[0, 16383]`; set `recommended_ai_conf_th = min(0.95, max(0.70, ambient_max_trigger_conf + 0.15))`, `recommended_head_present_th = min(15000, fsr_baseline + 800)`, `recommended_motion_spike_th = min(16000, max(piezo_baseline * 3, piezo_baseline + 2500))`.

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest tests/test_calibration.py -v`
  - Expected: PASS (2 passed).

- [ ] **Step 5: Commit**
  - `git add smart_pillow_app/python/smart_pillow/calibration.py tests/test_calibration.py`
  - `git commit -m "feat: add 5s sensor baseline and ambient confidence calibrator"`

---

### Task 4: 5-State Machine & Triple-Signal Sensor Fusion Decision Engine

**Files:**
- Create: `smart_pillow_app/python/smart_pillow/decision_engine.py`
- Test: `tests/test_decision_engine.py`

**Interfaces:**
- Consumes: `PillowConfig` (`config.py`), `SystemState`, `OperatingMode`, `EventOutcome`, `LedIcon`, `TelemetryFrame`, `AudioPrediction`, `BridgeCommand`, `DecisionEvent` (`models.py`).
- Produces:
  - `StepResult(state: SystemState, commands: list[BridgeCommand], event: DecisionEvent | None)`
  - `DecisionEngine(config: PillowConfig)`
  - `DecisionEngine.update_config(config: PillowConfig) -> None` (clears snoring sliding-window history when `mode` changes)
  - `DecisionEngine.snore_window_count` (read-only `int` property returning current sliding-window sample count)
  - `DecisionEngine.step(now_s: float, telemetry: TelemetryFrame, audio: AudioPrediction | None) -> StepResult`
  - `DecisionEngine.trigger_estop(now_s: float) -> StepResult`
  - `DecisionEngine.resume_from_estop(now_s: float) -> StepResult`

- [ ] **Step 1: Write failing tests in `tests/test_decision_engine.py` covering all spec rules + Review Focus #2, #3, #4**

```python
from smart_pillow.config import PillowConfig
from smart_pillow.decision_engine import DecisionEngine
from smart_pillow.models import ActuatorState, AudioPrediction, EventOutcome, OperatingMode, SystemState, TelemetryFrame

def _tel(t, fsr, piezo=100, act=ActuatorState.IDLE):
    return TelemetryFrame(t, fsr, piezo, act, 0)

def _aud(t, label, conf=0.90):
    return AudioPrediction(t, {label: conf})

def test_no_head_rejects_snoring_and_bruxism():
    eng = DecisionEngine(PillowConfig.for_mode(OperatingMode.STAGE))
    res = eng.step(1.0, _tel(1.0, fsr=100), _aud(1.0, "Snoring", 0.95))
    assert res.state == SystemState.IDLE_NO_HEAD
    assert res.event.outcome == EventOutcome.IGNORED_NO_HEAD
    assert not any(c.kind in ("haptic_pulse", "servo_tilt") for c in res.commands)

def test_bruxism_triggers_after_two_windows_and_enters_blanking():
    eng = DecisionEngine(PillowConfig.for_mode(OperatingMode.STAGE))
    eng.step(0.0, _tel(0.0, fsr=3000), None)
    eng.step(1.1, _tel(1.1, fsr=3000), None)
    r1 = eng.step(1.5, _tel(1.5, fsr=3000), _aud(1.5, "Bruxism", 0.9))
    assert r1.state == SystemState.MONITORING
    r2 = eng.step(2.0, _tel(2.0, fsr=3000), _aud(2.0, "Bruxism", 0.9))
    assert r2.state == SystemState.INTERVENING
    assert any(c.kind == "haptic_pulse" for c in r2.commands)
    assert r2.event.outcome == EventOutcome.TRIGGERED_BRUXISM
    r3 = eng.step(3.6, _tel(3.6, fsr=3000), _aud(3.6, "Bruxism", 0.99))
    assert r3.state == SystemState.BLANKING
    assert not any(c.kind == "haptic_pulse" for c in r3.commands)

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
    eng.step(2.0, _tel(2.0, fsr=3000), _aud(2.0, "Normal_Breathing", 0.8))
    eng.step(2.5, _tel(2.5, fsr=3000), _aud(2.5, "Snoring", 0.9))
    r = eng.step(3.5, _tel(3.5, fsr=3000), _aud(3.5, "Snoring", 0.9))
    assert r.state == SystemState.INTERVENING
    assert any(c.kind == "servo_tilt" for c in r.commands)
    assert r.event.outcome == EventOutcome.TRIGGERED_SNORING
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
    assert any(c.kind == "stop_all" for c in estop.commands)
    still_estop = eng.step(2.5, _tel(2.5, fsr=5000), _aud(2.5, "Bruxism", float("nan")))
    assert still_estop.state == SystemState.ESTOP
    resumed = eng.resume_from_estop(3.0)
    assert resumed.state == SystemState.IDLE_NO_HEAD
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest tests/test_decision_engine.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `DecisionEngine` in `smart_pillow_app/python/smart_pillow/decision_engine.py`**
  - Implement all transitions per Section 5.5 of the spec using exact Bridge command names (`haptic_pulse`, `servo_tilt`, `servo_home`, `stop_all`, `led_icon`):
    - `IDLE_NO_HEAD → MONITORING` after `FSR >= head_present_th` continuously for `1.0s`; `MONITORING → IDLE_NO_HEAD` after `FSR < head_present_th` continuously for `3.0s`.
    - Clamp incoming ADC values to `[0, 16383]`; ignore audio predictions with empty or non-finite scores.
    - Track `last_motion_spike_s` whenever `piezo_peak >= motion_spike_th`; reject `Bruxism` if `(now_s - last_motion_spike_s) <= 1.5`.
    - Prune `_snore_history` to `config.snore_window_s`; trigger `Snoring` once coverage `>= 0.75 * config.snore_window_s` (or `>= 3` samples) and positive ratio `>= config.snore_ratio_th`.
    - During `INTERVENING`: finish haptic after `haptic_duration_ms / 1000.0`; finish servo tilt (`servo_home`) after `Snoring` is absent for `3.0s` or max hold `15.0s`; transition to `BLANKING` (`3.0s`) then `COOLDOWN` / `MONITORING`.
    - Enforce cooldown quota (`cooldown_max_count` within `cooldown_window_s`) logging `SUPPRESSED_COOLDOWN`.

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest tests/test_decision_engine.py -v`
  - Expected: PASS (5 passed).

- [ ] **Step 5: Commit**
  - `git add smart_pillow_app/python/smart_pillow/decision_engine.py tests/test_decision_engine.py`
  - `git commit -m "feat: implement 5-state sensor fusion decision engine"`

---

### Task 5: App Lab Bridge & AudioClassification Brick Adapters

**Files:**
- Create: `smart_pillow_app/python/smart_pillow/bridge.py`
- Create: `smart_pillow_app/python/smart_pillow/audio_adapter.py`
- Test: `tests/test_adapters.py`

**Interfaces:**
- Consumes: `ActuatorState`, `AudioPrediction`, `BridgeCommand`, `TelemetryFrame` (`models.py`).
- Produces:
  - `bridge.py`: `HardwareBridge` (Protocol with `get_telemetry() -> TelemetryFrame`, `send_heartbeat() -> None`, `send_command(cmd: BridgeCommand) -> None`), `AppLabBridgeClient` (lazily imports `arduino.app_utils.Bridge`; calls `Bridge.call("get_telemetry", timeout=0.5)` and `Bridge.notify(cmd.kind, ...)`), `MockBridgeClient` (in-memory simulator with `inject_telemetry(...)`, `sent_commands: list[BridgeCommand]`, `heartbeat_count: int`, `simulate_error: Exception | None = None`).
  - `audio_adapter.py`: `AudioSource` (Protocol with `poll(now_s: float) -> AudioPrediction | None`), `AppLabAudioSource` (lazily wraps `arduino.app_bricks.audio_classification.AudioClassification` and normalizes Brick prediction list `[{"label": ..., "value": ...}]` into `AudioPrediction(now_s, scores)`), `MockAudioSource(scores: dict[str, float])`.

- [ ] **Step 1: Write failing tests in `tests/test_adapters.py`**

```python
from smart_pillow.audio_adapter import MockAudioSource, normalize_brick_predictions
from smart_pillow.bridge import MockBridgeClient
from smart_pillow.models import ActuatorState, BridgeCommand

def test_normalize_brick_predictions_converts_app_lab_format():
    pred = normalize_brick_predictions(
        10.0,
        [{"label": "Snoring", "value": 0.87}, {"label": "Quiet_Noise", "value": 0.13}],
    )
    assert pred is not None
    assert pred.scores["Snoring"] == 0.87
    assert normalize_brick_predictions(10.0, []) is None

def test_mock_bridge_and_audio_source_contract():
    bridge = MockBridgeClient()
    bridge.inject_telemetry(fsr_mean=2500, piezo_peak=300)
    tel = bridge.get_telemetry()
    assert tel.fsr_mean == 2500 and tel.actuator_state == ActuatorState.IDLE
    bridge.send_command(BridgeCommand("haptic_pulse", {"intensity_pct": 65, "duration_ms": 1500}))
    assert bridge.sent_commands[-1].kind == "haptic_pulse"
    src = MockAudioSource({"Snoring": 0.92})
    assert src.poll(1.0).scores["Snoring"] == 0.92
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest tests/test_adapters.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `bridge.py` and `audio_adapter.py`**
  - Keep imports of `arduino.app_utils` and `arduino.app_bricks.audio_classification` lazy inside `AppLabBridgeClient` / `AppLabAudioSource` constructors so tests run cleanly on any PC without App Lab installed.
  - Handle STM32 CSV/list telemetry return format (`"fsr_mean,piezo_peak,actuator_state,servo_angle"`) safely inside `AppLabBridgeClient.get_telemetry()`.

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest tests/test_adapters.py -v`
  - Expected: PASS (2 passed).

- [ ] **Step 5: Commit**
  - `git add smart_pillow_app/python/smart_pillow/bridge.py smart_pillow_app/python/smart_pillow/audio_adapter.py tests/test_adapters.py`
  - `git commit -m "feat: add Arduino App Lab Bridge and AudioClassification adapters"`

---

### Task 6: Runtime Controller (10 Hz Telemetry, 1 Hz Heartbeat, Fault Isolation)

**Files:**
- Create: `smart_pillow_app/python/smart_pillow/controller.py`
- Test: `tests/test_controller.py`

**Interfaces:**
- Consumes: `PillowConfig`, `EventStore`, `Calibrator`, `DecisionEngine`, `HardwareBridge`, `AudioSource`.
- Produces:
  - `PillowController(config: PillowConfig, bridge: HardwareBridge, audio_source: AudioSource, store: EventStore, now_fn: Callable[[], float] = time.monotonic)`
  - `PillowController.tick_once() -> dict` (polls telemetry at 10 Hz, sends 1 Hz heartbeat, polls audio source at 2 Hz, feeds calibration if active, steps FSM, dispatches commands, logs events to SQLite, returns JSON-serializable telemetry snapshot dict)
  - `PillowController.set_mode(mode: OperatingMode) -> dict`
  - `PillowController.update_thresholds(**kwargs) -> dict`
  - `PillowController.start_calibration() -> dict`
  - `PillowController.manual_test(action: str) -> dict` (`"haptic"` | `"servo_tilt"` | `"servo_home"`)
  - `PillowController.emergency_stop() -> dict` / `resume() -> dict`
  - `PillowController.get_export_csv() -> str` / `get_report_data() -> dict`

- [ ] **Step 1: Write failing tests in `tests/test_controller.py` (pins Review Focus #1 Stale/Disconnected Bridge)**

```python
from smart_pillow.audio_adapter import MockAudioSource
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
        MockAudioSource({"Bruxism": 0.95}),
        EventStore(tmp_path / "c.db"),
        now_fn=lambda: now[0],
    )
    ctrl.tick_once()
    now[0] = 2.0
    bridge.simulate_error = RuntimeError("bridge timeout")
    snap = ctrl.tick_once()
    assert snap["bridge_connected"] is False
    assert snap["last_outcome"] == EventOutcome.BRIDGE_DISCONNECTED.value
    assert not any(c.kind == "haptic_pulse" for c in bridge.sent_commands)

def test_controller_sends_heartbeat_every_1s_and_supports_estop(tmp_path):
    now = [0.0]
    bridge = MockBridgeClient()
    ctrl = PillowController(
        PillowConfig.for_mode(OperatingMode.STAGE),
        bridge,
        MockAudioSource({"Quiet_Noise": 0.95}),
        EventStore(tmp_path / "c2.db"),
        now_fn=lambda: now[0],
    )
    ctrl.tick_once()
    now[0] = 1.1
    ctrl.tick_once()
    assert bridge.heartbeat_count >= 2
    ctrl.emergency_stop()
    assert bridge.sent_commands[-1].kind == "stop_all"
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest tests/test_controller.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `PillowController` in `smart_pillow_app/python/smart_pillow/controller.py`**
  - On `bridge.get_telemetry()` error or telemetry age `> 1.5s`, mark `bridge_connected=False`, log `BRIDGE_DISCONNECTED` once per outage, skip FSM actuation triggers, and keep returning snapshots safely.
  - Auto-apply `Calibrator.finish()` thresholds into `PillowConfig` once the 5 s calibration finishes.

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest tests/test_controller.py -v`
  - Expected: PASS (2 passed).

- [ ] **Step 5: Commit**
  - `git add smart_pillow_app/python/smart_pillow/controller.py tests/test_controller.py`
  - `git commit -m "feat: add PillowController with 1Hz heartbeat and bridge disconnect protection"`

---

### Task 7: Arduino App Lab Entrypoint, WebUI Brick Adapter & Stage Dashboard

**Files:**
- Create: `smart_pillow_app/python/smart_pillow/webui_adapter.py`
- Create: `smart_pillow_app/python/main.py`
- Create: `smart_pillow_app/assets/index.html`
- Create: `smart_pillow_app/assets/app.js`
- Create: `smart_pillow_app/assets/styles.css`
- Test: `tests/test_webui_adapter.py`

**Interfaces:**
- Consumes: `PillowController` (`controller.py`), `OperatingMode` (`models.py`), `arduino.app_utils.App`, `arduino.app_bricks.web_ui.WebUI` (in `main.py`).
- Produces:
  - `webui_adapter.py`: `WebUIChannel` (Protocol with `on_message(event: str, handler)` and `send_message(event: str, payload: dict)`), `register_webui_handlers(ui: WebUIChannel, controller: PillowController) -> None` wiring client messages `set_mode`, `update_config`, `start_calibration`, `manual_test`, `estop`, `resume`, `request_export_csv`, `request_report`, and emitting `telemetry`, `config_updated`, `csv_export`, `report_data`.
  - `main.py`: App Lab entrypoint instantiating `WebUI()`, `AppLabAudioSource()`, `AppLabBridgeClient()`, `EventStore()`, `PillowController()`, registering WebUI handlers, spawning the 10 Hz background ticker thread pushing `ui.send_message("telemetry", snap)`, and calling `App.run()`.
  - `assets/{index.html,app.js,styles.css}`: Stage-Ready Web Dashboard using App Lab WebUI client WebSocket messages.

- [ ] **Step 1: Write failing tests in `tests/test_webui_adapter.py`**

```python
from smart_pillow.audio_adapter import MockAudioSource
from smart_pillow.bridge import MockBridgeClient
from smart_pillow.config import PillowConfig
from smart_pillow.controller import PillowController
from smart_pillow.models import OperatingMode
from smart_pillow.storage import EventStore
from smart_pillow.webui_adapter import register_webui_handlers

class FakeWebUI:
    def __init__(self):
        self.handlers = {}
        self.sent = []
    def on_message(self, event, fn=None):
        def _reg(func):
            self.handlers[event] = func
            return func
        return _reg(fn) if fn else _reg
    def send_message(self, event, payload):
        self.sent.append((event, payload))

def test_webui_handlers_route_mode_estop_and_csv_export(tmp_path):
    ctrl = PillowController(
        PillowConfig.for_mode(OperatingMode.NORMAL),
        MockBridgeClient(),
        MockAudioSource({"Quiet_Noise": 0.9}),
        EventStore(tmp_path / "ui.db"),
    )
    ui = FakeWebUI()
    register_webui_handlers(ui, ctrl)
    ui.handlers["set_mode"]("sid1", {"mode": "STAGE"})
    assert ui.sent[-1][0] == "config_updated"
    assert ui.sent[-1][1]["snore_window_s"] == 3.0
    ui.handlers["estop"]("sid1", {})
    assert ui.sent[-1][1]["state"] == "ESTOP"
    ui.handlers["request_export_csv"]("sid1", {})
    assert ui.sent[-1][0] == "csv_export"
    assert "timestamp_s,mode,state,outcome" in ui.sent[-1][1]["csv"]
```

- [ ] **Step 2: Run test to verify it fails**
  - Run: `pytest tests/test_webui_adapter.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `webui_adapter.py`, `main.py`, and Stage Dashboard (`assets/index.html`, `app.js`, `styles.css`)**
  - Dashboard MUST display: large color-coded State Banner, prominent `NORMAL / STAGE DEMO MODE` badge + toggle, live FSR gauge + threshold line, Piezo motion-spike meter, 4-class AI confidence bars, 1-Click `Calibrate (5s)` button, `Test Haptic` / `Test Servo` buttons, large red `EMERGENCY STOP` / `RESUME` button, real-time threshold sliders, live Event Timeline, `Export CSV` button (Blob download), and `Print Summary Report` (`@media print` view).

- [ ] **Step 4: Run test to verify it passes**
  - Run: `pytest tests/test_webui_adapter.py -v`
  - Expected: PASS.

- [ ] **Step 5: Commit**
  - `git add smart_pillow_app/python/smart_pillow/webui_adapter.py smart_pillow_app/python/main.py smart_pillow_app/assets tests/test_webui_adapter.py`
  - `git commit -m "feat: add App Lab entrypoint, WebUI Brick adapter, and Stage Dashboard"`

---

### Task 8: STM32U585 App Lab Sketch (`sketch/`) & Host C++ Safety Tests

**Files:**
- Create: `smart_pillow_app/sketch/safety_logic.h`
- Create: `smart_pillow_app/sketch/sketch.ino`
- Create: `smart_pillow_app/sketch/sketch.yaml`
- Test: `firmware_tests/test_safety_logic.cpp`

**Interfaces:**
- Consumes: `Arduino_RouterBridge.h`, `Arduino_LED_Matrix.h`, `Servo.h` (on STM32 target inside App Lab).
- Produces:
  - `safety_logic.h`: pure C++ `SafetyController` struct with `clamp_angle(int deg) -> int` (`[0, 35]`), `clamp_haptic_ms(uint32_t ms) -> uint32_t` (`[0, 3000]`), `clamp_intensity(int pct) -> int` (`[0, 100]`), `on_heartbeat(uint32_t now_ms)`, `start_haptic(int pct, uint32_t dur_ms, uint32_t now_ms)`, `start_servo(int target_deg)`, `stop_all()`, `tick(uint32_t now_ms)` (enforces 5000 ms watchdog with unsigned subtraction wrap-around safety, 3000 ms continuous vibration cap, and 500 ms Soft-Start PWM ramp from 20% to target intensity).
  - `sketch.ino`: registers `Bridge.provide` handlers (`get_telemetry`, `heartbeat`, `haptic_pulse`, `servo_tilt`, `servo_home`, `stop_all`, `led_icon`), samples `A2` (FSR) and `A3` (Piezo) at 50 Hz with `analogReadResolution(14)`, drives `D5` PWM and `D9` Servo, and updates 6 custom 8×13 LED Matrix icons.
  - `sketch.yaml`: App Lab sketch manifest referencing UNO Q STM32 board & `Servo` / `Arduino_RouterBridge` / `Arduino_LED_Matrix`.

- [ ] **Step 1: Write failing host C++ safety tests in `firmware_tests/test_safety_logic.cpp` (pins Review Focus #5)**

```cpp
#include <cassert>
#include <cstdint>
#include "../smart_pillow_app/sketch/safety_logic.h"

int main() {
    SafetyController sc;
    sc.on_heartbeat(0);
    assert(sc.clamp_angle(90) == 35);
    assert(sc.clamp_angle(-10) == 0);
    assert(sc.clamp_haptic_ms(10000) == 3000);
    sc.start_haptic(150, 10000, 100);
    sc.tick(200);
    assert(sc.current_pwm_pct() >= 20 && sc.current_pwm_pct() < 100);
    sc.tick(3200);
    assert(sc.current_pwm_pct() == 0);
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
  - Expected: FAIL (`safety_logic.h: No such file or directory`).

- [ ] **Step 3: Implement `safety_logic.h`, `sketch.ino`, and `sketch.yaml`**
  - Keep `safety_logic.h` header-only with zero Arduino dependencies so it compiles identically on host `g++` and STM32 Zephyr/Arduino Core.
  - Never touch `A0`, `A1`, or `D3`; step Servo smoothly (`1°–2°` per `step_delay_ms`) toward `sc.target_servo_deg()` and snap to `0°` on `stop_all` or watchdog trip.

- [ ] **Step 4: Compile & run host C++ test to verify it passes**
  - Run: `g++ -std=c++17 firmware_tests/test_safety_logic.cpp -o firmware_tests/test_safety_logic.exe && firmware_tests/test_safety_logic.exe`
  - Expected: Exit code `0` (all assertions pass).

- [ ] **Step 5: Commit**
  - `git add smart_pillow_app/sketch firmware_tests/test_safety_logic.cpp`
  - `git commit -m "feat: add App Lab STM32U585 sketch and verified safety watchdog logic"`

---

### Task 9: Speaker-Matched Dataset Tool, 20-Trial A/B Benchmark & Stage Demo Pack

**Files:**
- Create: `tools/__init__.py`
- Create: `tools/record_dataset.py`
- Create: `tools/run_ab_benchmark.py`
- Create: `docs/dataset/edge-impulse-guide.md`
- Create: `docs/demo/stage-demo-and-testing.md`
- Test: `tests/test_benchmark_script.py`

**Interfaces:**
- Consumes: `DecisionEngine`, `PillowConfig`, `TelemetryFrame`, `AudioPrediction`.
- Produces:
  - `tools/run_ab_benchmark.py`: `evaluate_trials(trials: list[dict], config: PillowConfig) -> dict` returning `audio_only_fp_rate`, `fusion_fp_rate`, `audio_only_tp_rate`, `fusion_tp_rate`, plus CLI writing `benchmark_results.csv` and Markdown table for Pitch slides.
  - `tools/record_dataset.py`: offline PC CLI (`--label {Snoring,Bruxism,Normal_Breathing,Quiet_Noise} --count N --duration 1.5 --out dataset_raw`) recording 16 kHz mono WAVs via USB Mic for Edge Impulse Studio upload (never shipped inside `smart_pillow_app/`).
  - `docs/dataset/edge-impulse-guide.md` & `docs/demo/stage-demo-and-testing.md`: dataset attribution, Edge Impulse → App Lab `.eim` deployment steps, 90-second 4-beat Stage Demo cue sheet, and Q&A defense answers.

- [ ] **Step 1: Write failing test in `tests/test_benchmark_script.py`**

```python
from tools.run_ab_benchmark import evaluate_trials
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
  - Run: `pytest tests/test_benchmark_script.py -v`
  - Expected: FAIL (`ImportError`).

- [ ] **Step 3: Implement `record_dataset.py`, `run_ab_benchmark.py`, and both documentation guides**
  - Ensure `docs/dataset/edge-impulse-guide.md` documents exact Edge Impulse → Arduino App Lab model upload/path mapping (`app.yaml`) and full source attribution (Kaggle Snoring Dataset, Custom Team Bruxism/Noise recordings, Edge Impulse `.eim`, Arduino App Lab Bricks) for the 5-pt Engineering Documentation rubric.

- [ ] **Step 4: Run full test suite to verify everything passes**
  - Run: `pytest tests -v`
  - Expected: ALL tests PASS.

- [ ] **Step 5: Commit**
  - `git add tools tests/test_benchmark_script.py docs/dataset/edge-impulse-guide.md docs/demo/stage-demo-and-testing.md`
  - `git commit -m "feat: add dataset capture tool, A/B benchmark runner, and stage demo docs"`
