# Agent Handoff: Reactive Laser Target (ESP32 + PlatformIO)

**Owner:** Tom Matthews (tmatthews@testo.de)
**Date of handoff:** 2026-10-22
**Your role:** Take over this project as an expert embedded systems engineer. Build, verify, flash and refine the firmware, and guide the physical build and tuning.

---

## 1. Mission

Build a standalone **reactive laser target** for dry-fire training. A 10x10 cm target face detects a brief (10-50 ms) low-power laser training pulse. On a hit it gives instant feedback: a buzzer chirp and an LED flash.

### Background
- The owner built a proof of concept with an Arduino and a standard LDR (5 mm sensing area).
- The sensing area was too small to hit reliably.
- A light-diffuser box with semi-translucent plastic was tried to widen the area. A standard LDR was too slow (30 ms+) and too insensitive for the faint scattered light.

### Chosen architecture
1. **High-speed sensing:** a 4-pin photodiode module (VCC, GND, DO, AO) with an LM393 comparator and an onboard threshold potentiometer. DO is used for interrupt-driven edge detection. AO is used for background ambient calibration.
2. **Optical front face:** 10x10 cm prismatic acrylic (fluorescent-light diffuser panel), or clear acrylic with frosted privacy film. Target transmission is about 85-90 %, with the beam scattered into the cavity.
3. **Reflective cavity:** the box interior is lined with aluminium foil or mirror tape (integrating-sphere effect), channelling scattered light to the sensor.

### Hard constraints
- MCU: **ESP32-WROOM-32D** dev board (30 or 38 pin).
- Toolchain: **PlatformIO** (VS Code), `esp32dev` environment, **Arduino framework**.
- Outputs: 1x active piezo buzzer, 1x high-brightness indicator LED with a 220-330 ohm series resistor.
- No blocking `delay()` in the monitoring path (it is allowed in `setup()` only).
- Software cooldown of about 500 ms after a hit.
- Ambient-light handling so room-lighting changes do not cause false triggers.

---

## 2. Current status

| Item | State |
|---|---|
| Wiring design and GPIO selection | Done (section 3) |
| `platformio.ini` | Done (section 4), **not yet built** |
| Firmware (`src/main.cpp`) | Written (section 5). **Never compiled and never run on hardware.** It was reviewed by reading only, because the authoring environment had no C++ toolchain. |
| Physical box | Not built or not verified by the agent. See the build and tuning guide in section 7. |

Treat the firmware as a strong first draft. Your first job is to prove that it compiles and works.

---

## 3. Hardware and wiring (source of truth)

### Bill of materials
| Qty | Part | Notes |
|---|---|---|
| 1 | ESP32 DevKit, ESP32-WROOM-32D (30/38 pin) | `esp32dev`-compatible |
| 1 | 4-pin photodiode module (LM393, VCC/GND/DO/AO) | Prefer a **clear-lens**, visible-light-sensitive photodiode. Black-epoxy types are IR-only and respond poorly to a red laser. |
| 1 | Active piezo buzzer, 3.3-5 V | Or a passive piezo for a true 1500 Hz tone |
| 1 | NPN transistor (2N2222 / PN2222 / BC337) | Buzzer switch. Check the C/B/E pinout in the datasheet. |
| 1 | 1 kohm resistor | Transistor base |
| 1 | 10 kohm resistor (optional) | Base pull-down, prevents a boot chirp |
| 1 | High-brightness LED (red/green/amber, 5 or 10 mm) | White and blue need a transistor driver (see below) |
| 1 | 220 ohm resistor | LED current limit (330 ohm is also fine) |
| 1 | 100 nF ceramic capacitor | Across module VCC-GND |
| 1 | Prismatic acrylic panel 100x100 mm, or clear acrylic plus frosted film | Front face |
| - | Mirror tape or foil, black tape or paint, perfboard, wire, USB cable | |

### Wiring
```
                ESP32 DevKit
            +--------------------+
 PHOTODIODE |                    |
 MODULE     |                    |
  VCC <-----|-- 3V3              |      (3.3 V ONLY, never 5 V)
  GND <-----|-- GND              |
  DO  ----->|-- GPIO27           |      interrupt input
  AO  ----->|-- GPIO34           |      ADC1_CH6, input-only
            |                    |
 LED        |                    |
  anode <---|-- GPIO26 --[220R]--|-->  LED anode
  cathode ---------------------------> GND
            |                    |
 BUZZER     |                    |
            |   VIN (5 V USB) ---|---> Buzzer (+)
            |                    |     Buzzer (-) ---> NPN Collector
            |   GPIO25 --[1k]----|---> NPN Base ---[10k]--- GND (optional)
            |   GND -------------|---> NPN Emitter
            +--------------------+

 100 nF across module VCC-GND, close to the module.
 All grounds (ESP32, module, LED cathode, transistor emitter) are common.
```
Direct drive option: a 3.3 V buzzer that draws less than 10 mA can go straight from GPIO25 to the buzzer + pin, with the buzzer - pin to GND.

### Pin rationale
| Function | GPIO | Reason |
|---|---|---|
| Sensor DO | 27 | No boot role and not a strapping pin. It supports interrupts. |
| Sensor AO | 34 | Input-only, on ADC1 (channel 6), so it is unaffected by ADC2/Wi-Fi conflicts |
| Buzzer | 25 | Normal output with no boot role |
| LED | 26 | Normal output with no boot role |

Avoided pins:
- **GPIO 0, 2, 5, 12, 15:** strapping pins.
- **GPIO 6-11:** internal flash.
- **GPIO 1, 3:** USB serial.
- **GPIO 34-39 as outputs:** input-only.

### Critical electrical notes
1. **Power the sensor module from 3V3.** At 5 V, DO swings to 5 V and AO can exceed 3.3 V. ESP32 GPIOs are not 5 V tolerant.
2. **An active buzzer has a fixed pitch.** It cannot play 1500 Hz. The firmware does a 150 ms on/off chirp. For a true 1500 Hz tone, use a passive piezo and set `BUZZER_ACTIVE 0`.
3. **LED colour:** red, green and amber (about 2 V forward voltage) draw about 6 mA from 3.3 V through 220 ohm. White and blue LEDs (about 3.0-3.2 V) would get only about 1 mA, so drive them from 5 V through a second transistor and a roughly 150 ohm resistor.
4. **Keep the LED and the module's own LEDs out of the optical cavity.** Otherwise they re-trigger the sensor.

---

## 4. `platformio.ini`

```ini
[env:esp32dev]
platform      = espressif32@^6.9.0   ; Arduino-ESP32 core 2.x, so the LEDC calls are stable
board         = esp32dev
framework     = arduino

monitor_speed = 115200
upload_speed  = 921600
monitor_filters = default

build_type    = release
build_flags   =
    -O2
    -DCORE_DEBUG_LEVEL=0
```
If the upload hangs on "Connecting...", hold the board's BOOT button until the upload starts.

---

## 5. Firmware: `src/main.cpp`

```cpp
/**
 * ============================================================================
 *  Reactive Laser Target  -  ESP32-WROOM-32D  |  PlatformIO  |  Arduino core
 * ============================================================================
 *  Sensor : 4-pin photodiode module (LM393 comparator)  VCC/GND/DO/AO
 *  Output : active (or passive) piezo buzzer + indicator LED
 *
 *  Architecture
 *  ------------
 *   DO pin  --(hardware interrupt, active edge only)--> ISR stores a timestamp
 *   loop()  --(non-blocking state machine)------------> validate -> hit -> cooldown
 *   AO pin  --(slow ADC sampling, background)---------> ambient baseline / drift
 *
 *  State machine:
 *   CALIBRATING -> ARMED -> VALIDATING -> COOLDOWN -> WAIT_CLEAR -> ARMED ...
 *
 *  No delay() is used after setup(); all timing uses millis()/micros().
 * ============================================================================
 */
#include <Arduino.h>
#include <math.h>

// ============================== USER CONFIG =================================
// ---- Pin assignment (see wiring guide for why these are safe) --------------
#define PIN_SENSOR_DO   27   // Digital out of photodiode module -> interrupt
#define PIN_SENSOR_AO   34   // Analog out (ADC1_CH6, input-only pin)
#define PIN_BUZZER      25   // Buzzer (via NPN transistor, or direct if <10 mA)
#define PIN_LED         26   // Indicator LED (through 220 ohm resistor)

// ---- Behaviour switches ----------------------------------------------------
#define SENSOR_ACTIVE_LOW  1  // 1: DO goes LOW on light (most LM393 modules)
                              // 0: DO goes HIGH on light (check your module!)
#define BUZZER_ACTIVE      1  // 1: active buzzer (fixed pitch, simple on/off)
                              // 0: passive buzzer (true 1500 Hz tone via LEDC)
#define USE_AO_CONFIRM     0  // 1: require AO to deviate from ambient baseline
                              //    before accepting a hit (enable after testing)

// ---- Timing ----------------------------------------------------------------
static constexpr uint32_t COOLDOWN_MS        = 500;   // lock-out after a hit
static constexpr uint32_t GLITCH_FILTER_US   = 1000;  // DO must stay active this long
static constexpr uint32_t REARM_QUIET_MS     = 50;    // DO must be inactive this long
static constexpr uint32_t STUCK_WARN_MS      = 2000;  // "sensor saturated" warning

// ---- Feedback --------------------------------------------------------------
static constexpr uint32_t BUZZER_MS          = 150;
static constexpr uint16_t BUZZER_HZ          = 1500;  // only used if BUZZER_ACTIVE 0
static constexpr uint32_t LED_FLASH_MS       = 250;

// ---- Ambient calibration / tracking ---------------------------------------
static constexpr uint16_t CAL_SAMPLES        = 200;   // 200 x 5 ms = 1 s
static constexpr uint32_t CAL_INTERVAL_MS    = 5;
static constexpr uint32_t BOOT_CLEAR_TIMEOUT = 60000; // max wait for DO to go idle
static constexpr uint32_t TRACK_INTERVAL_MS  = 10;    // background AO sampling
static constexpr float    TRACK_ALPHA        = 0.01f; // EMA, ~1 s time constant
static constexpr float    DRIFT_WARN_COUNTS  = 150.0f;// ADC counts (~120 mV)
static constexpr uint32_t DRIFT_REPEAT_MS    = 10000;
static constexpr float    AO_MIN_MARGIN      = 20.0f; // min deviation for confirm
static constexpr float    AO_NOISE_K         = 4.0f;  // margin = K x measured noise
// ============================================================================

#if defined(ESP_ARDUINO_VERSION_MAJOR) && (ESP_ARDUINO_VERSION_MAJOR >= 3)
  #define ARDUINO_CORE_V3 1
#else
  #define ARDUINO_CORE_V3 0
#endif

#if !BUZZER_ACTIVE && !ARDUINO_CORE_V3
  static constexpr uint8_t BUZZER_LEDC_CH = 0;
#endif

enum class State : uint8_t { CALIBRATING, ARMED, VALIDATING, COOLDOWN, WAIT_CLEAR };

// ------------------------------ Shared with ISR -----------------------------
static volatile bool     g_armed       = false;  // ISR only records when true
static volatile bool     g_edgePending = false;
static volatile uint32_t g_edgeUs      = 0;

// ------------------------------ Application state ---------------------------
static State    g_state       = State::CALIBRATING;
static uint32_t g_hitCount    = 0;
static uint32_t g_glitchCount = 0;
static uint32_t g_rejectCount = 0;
static uint32_t g_hitMs       = 0;

struct Ambient {
  float    baseline    = 0;   // live (slowly tracking) ambient AO level
  float    calBaseline = 0;   // level measured at boot
  float    noise       = 0;   // half peak-to-peak noise at boot
  uint32_t lastDriftMs = 0;
} g_amb;

struct Feedback {
  bool     buzzerOn  = false;
  bool     ledOn     = false;
  uint32_t buzzerEnd = 0;
  uint32_t ledEnd    = 0;
} g_fb;

static uint32_t g_waitStartMs  = 0;
static uint32_t g_clearSinceMs = 0;
static bool     g_clearing     = false;
static bool     g_stuckWarned  = false;
static uint32_t g_lastTrackMs  = 0;

// =============================== Helpers ====================================
static inline bool sensorActive() {
  return digitalRead(PIN_SENSOR_DO) == (SENSOR_ACTIVE_LOW ? LOW : HIGH);
}

static uint16_t readAo(uint8_t n = 4) {
  uint32_t sum = 0;
  for (uint8_t i = 0; i < n; i++) sum += analogRead(PIN_SENSOR_AO);
  return (uint16_t)(sum / n);
}

static inline bool timeReached(uint32_t now, uint32_t target) {
  return (int32_t)(now - target) >= 0;   // rollover-safe
}

// ================================ ISR =======================================
// Keep it tiny: store the timestamp of the FIRST active edge, nothing else.
void IRAM_ATTR onSensorEdge() {
  if (g_armed && !g_edgePending) {
    g_edgeUs      = micros();
    g_edgePending = true;
  }
}

static void armSensor() {
  g_edgePending = false;
  g_armed       = true;
}

// ============================== Buzzer / LED ================================
static void buzzerWrite(bool on) {
#if BUZZER_ACTIVE
  digitalWrite(PIN_BUZZER, on ? HIGH : LOW);
#elif ARDUINO_CORE_V3
  ledcWriteTone(PIN_BUZZER, on ? BUZZER_HZ : 0);
#else
  ledcWriteTone(BUZZER_LEDC_CH, on ? BUZZER_HZ : 0);
#endif
}

static void buzzerInit() {
#if BUZZER_ACTIVE
  pinMode(PIN_BUZZER, OUTPUT);
#elif ARDUINO_CORE_V3
  ledcAttach(PIN_BUZZER, BUZZER_HZ, 10);
#else
  ledcSetup(BUZZER_LEDC_CH, BUZZER_HZ, 10);
  ledcAttachPin(PIN_BUZZER, BUZZER_LEDC_CH);
#endif
  buzzerWrite(false);
}

// Start chirp + LED flash at the same instant; returns immediately.
static void startFeedback() {
  const uint32_t now = millis();
  digitalWrite(PIN_LED, HIGH);
  buzzerWrite(true);
  g_fb.ledOn     = true;  g_fb.ledEnd    = now + LED_FLASH_MS;
  g_fb.buzzerOn  = true;  g_fb.buzzerEnd = now + BUZZER_MS;
}

// Called every loop(): switches outputs off when their time is up.
static void serviceFeedback() {
  const uint32_t now = millis();
  if (g_fb.buzzerOn && timeReached(now, g_fb.buzzerEnd)) {
    buzzerWrite(false);
    g_fb.buzzerOn = false;
  }
  if (g_fb.ledOn && timeReached(now, g_fb.ledEnd)) {
    digitalWrite(PIN_LED, LOW);
    g_fb.ledOn = false;
  }
}

// ========================= Ambient calibration ==============================
// Measures the AO baseline and noise for ~1 s. Called once in setup().
// NOTE: The LM393 threshold itself is a hardware potentiometer; software
// cannot move it. Software instead (a) verifies the sensor is idle at boot,
// (b) tracks ambient drift, (c) optionally gates hits against the baseline.
static void calibrateAmbient() {
  // 1) The comparator must be idle. If DO is already active, the pot is set
  //    too sensitive for the current light (or SENSOR_ACTIVE_LOW is wrong).
  const uint32_t t0 = millis();
  bool warned = false;
  while (sensorActive() && (millis() - t0) < BOOT_CLEAR_TIMEOUT) {
    if (!warned) {
      Serial.println(F("[CAL] WARNING: DO is ACTIVE at boot. Lower sensitivity (pot) "
                       "or check SENSOR_ACTIVE_LOW."));
      warned = true;
    }
    digitalWrite(PIN_LED, (millis() / 100) & 1);   // fast blink = fault
    delay(10);
  }
  digitalWrite(PIN_LED, LOW);

  // 2) Sample the analog output.
  float    sum = 0;
  uint16_t mn = 4095, mx = 0;
  for (uint16_t i = 0; i < CAL_SAMPLES; i++) {
    const uint16_t v = readAo(4);
    sum += v;
    if (v < mn) mn = v;
    if (v > mx) mx = v;
    delay(CAL_INTERVAL_MS);
  }
  g_amb.baseline    = sum / CAL_SAMPLES;
  g_amb.calBaseline = g_amb.baseline;
  g_amb.noise       = (mx - mn) / 2.0f;

  Serial.printf("[CAL] AO baseline=%.0f  noise=+/-%.0f counts  (min=%u max=%u)\n",
                g_amb.baseline, g_amb.noise, mn, mx);
  if (g_amb.baseline > 4000.0f || g_amb.baseline < 30.0f) {
    Serial.println(F("[CAL] NOTE: AO is near a rail - ambient may be too bright/dark "
                     "for a useful AO reading."));
  }
}

// Background tracking: runs only while armed and idle, so a laser pulse can
// never pollute the baseline. Slow EMA follows daylight changes.
static void trackAmbient() {
  const uint32_t now = millis();
  if ((now - g_lastTrackMs) < TRACK_INTERVAL_MS) return;
  g_lastTrackMs = now;
  if (sensorActive()) return;

  const float v = (float)readAo(2);
  g_amb.baseline += TRACK_ALPHA * (v - g_amb.baseline);

  if (fabsf(g_amb.baseline - g_amb.calBaseline) > DRIFT_WARN_COUNTS &&
      (now - g_amb.lastDriftMs) > DRIFT_REPEAT_MS) {
    g_amb.lastDriftMs = now;
    Serial.printf("[AMB] Ambient drifted: now %.0f vs %.0f at boot. "
                  "Re-tune the pot if false/missed hits occur.\n",
                  g_amb.baseline, g_amb.calBaseline);
  }
}

[[maybe_unused]] static float aoMargin() {
  const float m = AO_NOISE_K * g_amb.noise;
  return (m > AO_MIN_MARGIN) ? m : AO_MIN_MARGIN;
}

// ============================== Hit handling ================================
static void registerHit(uint32_t edgeUs, int aoDelta) {
  startFeedback();                              // feedback FIRST, logging after
  const uint32_t latencyUs = micros() - edgeUs;
  g_hitCount++;
  g_hitMs = millis();
  Serial.printf("[HIT #%lu] latency=%lu us  AO delta=%d  baseline=%.0f  "
                "(glitches=%lu rejected=%lu)\n",
                (unsigned long)g_hitCount, (unsigned long)latencyUs, aoDelta,
                g_amb.baseline, (unsigned long)g_glitchCount,
                (unsigned long)g_rejectCount);
}

static void enterWaitClear() {
  g_state        = State::WAIT_CLEAR;
  g_clearing     = false;
  g_stuckWarned  = false;
  g_waitStartMs  = millis();
}

// ================================ setup() ===================================
void setup() {
  Serial.begin(115200);
  delay(200);
  Serial.println(F("\n=== Reactive Laser Target ==="));

  pinMode(PIN_LED, OUTPUT);
  digitalWrite(PIN_LED, LOW);
  buzzerInit();
  pinMode(PIN_SENSOR_DO, INPUT);        // module has its own pull-up

  analogReadResolution(12);                         // 0..4095
  analogSetPinAttenuation(PIN_SENSOR_AO, ADC_11db); // ~0..3.1 V range

  // Power-on self-test: short blip so you know LED + buzzer work.
  digitalWrite(PIN_LED, HIGH); buzzerWrite(true);
  delay(60);
  digitalWrite(PIN_LED, LOW);  buzzerWrite(false);

  calibrateAmbient();

  attachInterrupt(digitalPinToInterrupt(PIN_SENSOR_DO), onSensorEdge,
                  SENSOR_ACTIVE_LOW ? FALLING : RISING);

  armSensor();
  g_state = State::ARMED;
  Serial.println(F("[RUN] Armed. Waiting for hits..."));
}

// ================================ loop() ====================================
void loop() {
  serviceFeedback();

  switch (g_state) {

    case State::ARMED:
      trackAmbient();
      if (g_edgePending) g_state = State::VALIDATING;
      break;

    case State::VALIDATING:
      // Wait (non-blocking) until the glitch-filter window has elapsed.
      if ((uint32_t)(micros() - g_edgeUs) >= GLITCH_FILTER_US) {
        g_armed = false;                        // stop ISR from recording more

        if (!sensorActive()) {                  // pulse too short: noise/flicker
          g_glitchCount++;
          armSensor();
          g_state = State::ARMED;
          break;
        }

        const int aoDelta = (int)readAo(4) - (int)g_amb.baseline;
#if USE_AO_CONFIRM
        if (fabsf((float)aoDelta) < aoMargin()) {   // DO fired but AO disagrees
          g_rejectCount++;
          enterWaitClear();
          break;
        }
#endif
        registerHit(g_edgeUs, aoDelta);
        g_state = State::COOLDOWN;
      }
      break;

    case State::COOLDOWN:
      if ((millis() - g_hitMs) >= COOLDOWN_MS) enterWaitClear();
      break;

    case State::WAIT_CLEAR: {
      const uint32_t now = millis();
      if (sensorActive()) {
        g_clearing = false;                     // beam / light still present
        if (!g_stuckWarned && (now - g_waitStartMs) > STUCK_WARN_MS) {
          g_stuckWarned = true;
          Serial.println(F("[WARN] DO stuck active - ambient too bright or pot "
                           "too sensitive. Target disarmed until it clears."));
        }
      } else if (!g_clearing) {
        g_clearing = true;
        g_clearSinceMs = now;
      } else if ((now - g_clearSinceMs) >= REARM_QUIET_MS) {
        armSensor();
        g_state = State::ARMED;
      }
      break;
    }

    default:
      g_state = State::ARMED;
      break;
  }

  yield();
}
```

---

## 6. Firmware design: what it does and why

| Requirement | Implementation |
|---|---|
| Non-blocking, instant detection | The ISR on DO's active edge stores only a microsecond timestamp. `loop()` is a state machine, and `delay()` appears only in `setup()`. |
| Noise and flicker rejection | A hit is valid only if DO is still active `GLITCH_FILTER_US` (1 ms) after the edge. |
| Debounce and cooldown | 500 ms cooldown. After it, the target re-arms only once DO has been idle for 50 ms (`REARM_QUIET_MS`). |
| Ambient calibration | At boot: refuse to arm while DO is active (fast LED blink plus a serial message), then measure the AO baseline and noise over about 1 s. In the background: a slow moving average tracks drift, sampling only while idle, and warns on large drift. Optional AO confirmation gate (`USE_AO_CONFIRM`). |
| Feedback | The buzzer is on for 150 ms and the LED for 250 ms. Both are started in the same call, and hit latency is logged. |

**Known limitation:** the LM393 threshold is a physical potentiometer, so software cannot retune it. The firmware can only detect and report drift, and optionally gate on AO. A sudden light step (lamp switched on, torch) can cause one false hit, and the stuck-DO logic then prevents repeats.

---

## 7. Physical build and tuning guide

### Mounting the module in the 10x10 cm box
Use a cavity depth of 5-8 cm.

1. **Line the cavity.** Cover the walls and back with mirror tape or foil, shiny side in. Smooth out wrinkles.
2. **Mount the photodiode on the back wall, facing the front panel,** at the centre or in a corner.
3. **Keep the module PCB outside the cavity.** Drill a hole just big enough for the photodiode lens. The module's power LED and DO LED would otherwise light the cavity.
4. **If the PCB must be inside,** cover both onboard LEDs with black tape or desolder them.
5. **Keep the potentiometer reachable** through a small access hole in the back or side wall. Plug the hole with tape after tuning.
6. **Keep the indicator LED and buzzer outside the cavity** (or optically shielded) so feedback never feeds back into the sensor.
7. **Seal the front panel edge** with black foam or tape. Paint or tape the outer edges black.
8. **Add a short black hood,** 2-3 cm deep, around the front face against stray light.
9. **Optional:** put a red gel filter behind the front panel for a red laser (about 650 nm), or an IR-pass filter for an IR laser.
10. **Fit the 100 nF capacitor** across the module's VCC-GND and keep the sensor wires short.

### Tuning the LM393 potentiometer
The direction that raises sensitivity varies by module. Use the module's DO LED as the guide and do not assume a direction.

1. Close the box in its final form: panel on, hood fitted, in its real location. Power it from USB and open the serial monitor.
2. Turn the pot to its **least sensitive** extreme. The DO LED should be off.
3. Set the worst-case ambient light: all room lights on, plus bright daylight if the target sits near a window.
4. In small steps, turn towards **more sensitive**. Stop where the DO LED first lights or flickers. This is the ambient trip point.
5. Back off until the DO LED is off and stable, then back off a further 5-10 degrees of rotation. This is the safety margin.
6. Reboot the ESP32. Boot calibration must finish without the fast-blink fault.
7. Fire the laser at the centre, edges and corners from the real training distance. Use a 3x3 grid and at least 5 pulses per zone. Every pulse should give a DO LED flash, a buzzer chirp and a `[HIT #...]` log line.
8. If edges or corners miss, increase sensitivity by one tiny step and repeat steps 3 and 7. If ambient then causes false hits, fix the optics (hood, sealing, colour filter) rather than raising sensitivity further.
9. Stress-test for about 10 minutes. Wave a hand over the face, toggle the room lights, and shine a phone torch from the side. You should see no `[HIT]` lines. A few filtered `glitches` are fine.
10. Lock the pot with nail polish or hot glue, then retest.

### Pitfalls
- **Mains flicker:** Germany's 50 Hz supply makes fluorescent and LED lamps flicker at 100 Hz, and a fast photodiode sees it. If DO chatters near the trip point, move the threshold further from ambient. You can also raise `GLITCH_FILTER_US` to 2000-3000.
- **Laser wavelength:** many cheap photodiode modules use IR-optimised parts that are about half as sensitive to red light. Use a clear-lens visible-light photodiode for a red laser.
- **Wrong DO polarity:** if the module's DO LED blinks on a shot but no hit is logged, set `SENSOR_ACTIVE_LOW 0`.

---

## 8. Your task list (in order)

### Phase A: Project setup and compile
1. Create the PlatformIO project: `platformio.ini` (section 4) and `src/main.cpp` (section 5).
2. Run `pio run`. Fix any compile or link errors.
3. Compile every configuration combination. Do this by temporarily toggling the macros or by adding build-flag variants:
   - `BUZZER_ACTIVE` 1 and 0
   - `USE_AO_CONFIRM` 1 and 0
   - `SENSOR_ACTIVE_LOW` 1 and 0
4. Confirm there are no warnings in the default build.
5. Confirm the core version in use (Arduino-ESP32 2.x with the pinned platform). The `ARDUINO_CORE_V3` branch (`ledcAttach`, `ledcWriteTone(pin, ...)`) is also untested. Test it with a Core 3 platform if you can, or note it as unverified.

### Phase B: Refactor (only after A passes)
- Move build-time switches into `build_flags` (`-DBUZZER_ACTIVE=0` etc.) and create named environments, for example `esp32dev_active` and `esp32dev_passive`, while keeping `esp32dev` working.
- Optionally split the code into `config.h`, `sensor`, `feedback` and `calibration` modules, with no behaviour change.
- Add a native (host) unit test for the pure logic if practical. Candidates are `timeReached()` rollover safety, drift detection, and the cooldown and re-arm state transitions. Abstract time and pin access to make this possible.

### Phase C: Hardware verification (needs the physical rig)
1. Flash the board and open the serial monitor at 115200 baud. Expect the `=== Reactive Laser Target ===` banner, a `[CAL] AO baseline=...` line and `[RUN] Armed`.
2. Verify the self-test blip (LED and buzzer) at power-up.
3. Trigger the sensor with a phone torch or the laser. Check that each shot gives one `[HIT]` line, a chirp and an LED flash. Check that `latency` is about 1.0-1.1 ms.
4. Confirm the cooldown: hold the light on and verify there are no repeat hits. Check for the `[WARN] DO stuck active` message after about 2 s. Then remove the light and confirm the target re-arms.
5. Check that the DO polarity setting is correct (see the pitfalls above).
6. Log several hits and read the `AO delta` values against the boot `noise`. Decide with the owner whether `USE_AO_CONFIRM` should be enabled.
7. Test with a real 10-50 ms laser pulse source if available.

### Phase D: Report back to the owner
Provide:
- the final `platformio.ini` and `src/main.cpp`
- the build output summary (flash and RAM use)
- the results of each hardware test
- any changes you made and why
- remaining risks

---

## 9. Acceptance criteria

- [ ] Clean build with no warnings on the `esp32dev` environment
- [ ] Boot banner, calibration line and armed message appear on serial
- [ ] A 10 ms light pulse produces exactly one hit, with chirp and LED flash
- [ ] Time from edge to feedback is under 2 ms (logged latency)
- [ ] No repeat hits within 500 ms of a hit, and no repeats from a held light
- [ ] The target re-arms automatically after the light is removed
- [ ] No false hits during a 10-minute ambient stress test (lights toggled, hand waved)
- [ ] No `delay()` calls outside `setup()`
- [ ] The GPIO assignments match section 3, with no strapping, flash or UART pins used

---

## 10. Open questions for the owner (ask only if blocked)

1. Is the laser **visible red (about 650 nm)** or **IR**? This affects the photodiode choice and optional filters.
2. Is the buzzer **active or passive**? This determines `BUZZER_ACTIVE`.
3. What is the indicator LED's colour? White or blue needs the 5 V transistor driver.
4. What is the typical firing distance and lighting (windows, LED panels, fluorescent)?
5. Should hit counts or a score be shown or sent anywhere later, such as Wi-Fi or a display? If so, keep the ADC1-only choice for AO.
