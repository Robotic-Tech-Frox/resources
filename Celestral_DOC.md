# Celeste Documentation

## Setup

```cpp
// everything including FTC SDK (for hardware access) and special features
#include "celeste.h"   
// Only FTC SDK functionality
#include "sdk.h"
```

Hardware, gamepads and telemetry only exist after the library has been loaded by `LibLoader`.
That means: construct devices inside the opmode function, never in a global.

Files:

| Path | Contents |
| --- | --- |
| `teamcode/header`, `teamcode/sources` | Team code and opmodes |
| `sdk/header`, `sdk/sources` | Shared SDK code |

Values use Units-master types:

```cpp
constexpr Power half    = 0.5_pow;    // -1..1
constexpr Second wait   = 250_ms;     // time
constexpr Degree sweep  = 90_deg;     // angle
constexpr Tick counts   = 8192_tick;  // encoder ticks
constexpr Millimetre travel = 4_mm;   // length
```

`.Value()` or `.V()` returns the raw number.

### settings.h

| Setting | What it does for you |
| --- | --- |
| `LUCARNE_ENABLED` | Starts the dashboard server when an opmode starts. When false, it starts only if an opmode calls a Lucarne function (`config::Live`, `field::draw_*`, `Send`, `On`). On also turns on the automatic gamepad and telemetry mirrors even if no browser is attached. |
| `TELEMETRY_MANAGER_ENABLED` | False makes `CreateChannel`, `AddLine`, `AddData`, `flush` and `Update` no-ops. Plain `sdk::telemetry` still works. |
| `LOG_TAG` | logcat tag used by `log_manager::LOG`; filter with `adb logcat -s "Celeste - Madlen"`. |
| `INCLUDE_SDK_TEST_OPMODES` | Read only by Astral tooling, not by native code. |

## OpModes

An opmode is a header with an annotation and a `.cpp` with one exported function.

`teamcode/header/my_teleop.h`:

```cpp
#pragma once
#include "celeste.h"

namespace my_teleop {
    /*
    * @TeleOp(name = "My Teleop", group = "Competition")
    */
    extern "C" void my_teleop();
}
```

`teamcode/sources/my_teleop.cpp`:

```cpp
#include "my_teleop.h"

using namespace sdk;
using namespace sdk::linear_op_mode;

namespace my_teleop {
    extern "C" void my_teleop() {
        WaitForStart();

        while (OpModeIsActive()) {
            // one control cycle
            Sleep(10);
        }
    }
}
```

The build task `updateJavaClasses` finds the annotation and the `extern "C"` declaration,
then writes `TeamCode/src/main/java/org/firstinspires/ftc/teamcode/opmodes/my_teleop.java`
with `@Photon`, the annotation, and `LibLoader.start(this)`.

Rules that matter:

* The generated Java file is overwritten every build; never edit it.
* When the function returns, threads are stopped and the libraries unloaded. Nothing survives
  to the next run except `cross_opmode_storage`.
* (Class name = function name. The declaration must have empty parentheses and appear after
    the annotation. `@Disabled` in that comment block generates a disabled opmode.)

## Hardware

`hardware_map::get<T>("name")` looks up the device by its robot-config name and returns a JNI
reference. Construct the wrapper once, inside the opmode function. The names in the table are
placeholders for your robot configuration.

| Java | C++ |
| --- | --- |
| `hardwareMap.get(DcMotorEx.class, "motor_ex")` | `Dc_motor_ex motor_ex(hardware_map::get<Dc_motor_ex>("motor_ex"));` |
| `hardwareMap.get(DcMotor.class, "motor")` | `Dc_motor motor(hardware_map::get<Dc_motor>("motor"));` |
| `hardwareMap.get(DcMotorSimple.class, "motor_simple")` | `Dc_motor_simple motor_simple(hardware_map::get<Dc_motor_simple>("motor_simple"));` |
| `hardwareMap.get(Servo.class, "servo")` | `Servo servo(hardware_map::get<Servo>("servo"));` |
| `hardwareMap.get(CRServo.class, "cr_servo")` | `CR_servo cr_servo(hardware_map::get<CR_servo>("cr_servo"));` |
| `hardwareMap.get(LynxModule.class, "Control Hub")` | `Lynx_module hub(hardware_map::get<Lynx_module>("Control Hub"));` |
| `hardwareMap.get(BNO055IMU.class, "imu")` | `BNO055IMU imu(hardware_map::get<BNO055IMU>("imu"));` |
| `hardwareMap.get(TouchSensor.class, "touch")` | `Touch_sensor t(hardware_map::get<Touch_sensor>("touch"));` |
| `hardwareMap.get(WebcamName.class, "Webcam 1")` | `Webcam_name cam(hardware_map::get<Webcam_name>("Webcam 1"));` |
| `gamepad1` | `gamepads::gamepad1` or `gamepad_ex::gamepad1` |
| `telemetry.addData("x", v)` | `telemetry::AddData("x", v)` |
| `sleep(20)` | `linear_op_mode::Sleep(20)` |

### Motors and servos

`DcMotorEx` behaviour is unchanged; only the units are new.

```cpp
extern "C" void motors_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    Dc_motor_ex motor_ex(hardware_map::get<Dc_motor_ex>("motor_ex"));
    Servo servo(hardware_map::get<Servo>("servo"));

    motor_ex.SetMode(Dc_motor::Run_mode::RUN_USING_ENCODER);
    motor_ex.SetZeroPowerBehavior(Dc_motor::Zero_power_behavior::BRAKE);
    servo.SetDirection(Servo::Direction::REVERSE);
    servo.ScaleRange(0.2, 0.8);   // 0..1 now maps onto 0.2..0.8

    WaitForStart();

    while (OpModeIsActive()) {
        motor_ex.SetPower(0.75_pow);
        servo.SetPosition(0.5);

        telemetry::AddData("velocity (deg/s)", motor_ex.GetVelocity().Value());
        telemetry::AddData("current (A)", motor_ex.GetCurrent().Value());
        telemetry::Update();

        Sleep(20);
    }
}
```

### LynxModule and camera

```cpp
extern "C" void sensors_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    Lynx_module hub(hardware_map::get<Lynx_module>("Control Hub"));
    hub.SetBulkCachingMode(Lynx_module::Bulk_caching_mode::MANUAL);

    WaitForStart();

    while (OpModeIsActive()) {
        hub.GetBulkData();
        Sleep(10);
    }
}
```

* `hub.GetBulkData()` once per loop refreshes every motor/servo read on that hub. Without
  bulk caching, each `GetPower`/`GetCurrent` is a separate I2C read.
* `GetCachedVoltage()` reads the bus at most once per 500 ms. `DefaultControlHub()` returns
  the `"Control Hub"` instance, and is the default voltage source for `CMotorE`/`CMotorS`.
* Odometry is Pedro's job, not the opmode's. Put the pod in `drive_config.json`, then read the
  pose with `pedro::getPose()`; see [Pedro](#pedro).

For vision, subclass the processor and hand it to a portal. The portal owns the camera and the
processor calls happen on the vision thread.

```cpp
class Example_processor : public sdk::Vision_processor {
public:
    void init(const int &width, const int &height) const override {}
    void ProcessFrame(const cv::Mat &input, const long &capture_time_nanos) const override {
        // detect something
    }
};

extern "C" void vision_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    Example_processor processor;
    Webcam_name webcam(hardware_map::get<Webcam_name>("Webcam 1"));
    Vision_portal portal = Vision_portal::EasyCreateWithDefaults(webcam, {&processor});

    WaitForStart();
    while (OpModeIsActive()) {
        Sleep(10);
    }
}
```

Wrappers that exist: `Analog_input`, `BNO055IMU`, `Color_range_sensor`, `Color_sensor`,
`CR_servo`, `Dc_motor`, `Dc_motor_ex`, `Dc_motor_simple`, `Digital_channel`,
`Distance_sensor`, `Gamepad`, `Gyro_sensor`, `Light_sensor`,
`Lynx_module`, `Modern_robotics_color_beacon`, `PIDF_coefficients`, `Potentiometer`,
`Pwm_control`, `Servo`, `Servo_controller`, `Servo_impl_ex`, `Touch_sensor`, `Camera_name`,
`Webcam_name`, and the navigation types.

## Cmotors

Use `CMotorE`/`CMotorS` when a mechanism should hold a torque instead of a percentage of
power, they also add some other high level functionality.

```cpp
extern "C" void cmotor_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    DefaultMotorSpec() = Motor_spec{
        .torque_constant      = Divide<Torque, Ampere>::Type(0.0184),  // Kt, N*m/A
        .resistance           = Ohm(0.13),                             // R
        .back_emf_constant    = Divide<Volt, Divide<Radian, Second>::Type>::Type(0.0184),
        .ticks_per_revolution = Divide<Tick, Revolution>::Type(537.7),
    };

    CMotorE motor_e("motor_e", 1.0);   // name, output-per-motor gear ratio
    CMotorS motor_s("motor_s", 1.0);

    WaitForStart();

    while (OpModeIsActive()) {
        motor_e.SetTorque(0.5_Nm);   // re-evaluated against the current bus voltage
        motor_s.SetTorque(0.2_Nm);

        telemetry::AddData("torque (N*m)", motor_e.GetCurrentTorque().Value());
        telemetry::AddData("stall %", motor_e.GetStallPercentage());
        telemetry::Update();

        Sleep(20);
    }
}
```

The values in the example are placeholders; use your motor's datasheet. `DefaultMotorSpec()`
starts at zero and nothing in this repo sets it. A zero `Kt` divides by zero in the conversion,
so assign it before constructing a default-configured motor.

Reads:

```cpp
motor_e.GetMaxTorque();       // stall torque at the current bus voltage
motor_e.GetCurrentTorque();   // Kt * gear ratio * measured current
motor_e.GetCurrentSpeed();    // output-shaft speed, deg/s
motor_e.GetCurrentAngle();
motor_e.SetStallThreshold(0.7);
motor_e.IsStalling();
```

`CMotorE` needs a `DcMotorEx` (encoder and current). `CMotorS` works on a `DcMotorSimple`
(no encoder, no current), so it can only do resistive feedforward; it overdrives once the
motor spins.

What `SetTorque` does: `p = (tau * R) / (gear_ratio * Kt * V)` plus, on `CMotorE`, a
back-EMF term. `V` is the cached bus voltage. `SetTorque(torque)` uses the measured output
speed; `SetTorque(torque, speed)` uses the speed you pass.

**Known issue:** `GetStallPercentage()` compares the speed in deg/s (`AngularSpeed`) against
a no-load speed in rad/s, so it reads about 57x too high (`IsStalling()` uses it). Don't trust
either for real stall detection yet.

## Gamepad

```cpp
extern "C" void controls_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    bool toggle = false;
    float trigger = 0.0f;

    gamepad_ex::init();   // binds the pads and clears their listeners

    gamepad_ex::gamepad1.a.AddPressListener([&] { toggle = !toggle; });
    gamepad_ex::gamepad1.right_trigger.AddUpdateListener(
        [&](float v) { trigger = v; });

    WaitForStart();

    while (OpModeIsActive()) {
        gamepad_ex::Update();   // the one JNI poll per loop

        telemetry::AddData("toggle", toggle);
        telemetry::AddData("trigger", trigger);
        telemetry::Update();

        if (gamepads::gamepad1.b) {   // raw level, not an edge
            // ...
        }

        Sleep(10);
    }
}
```

| Object | Listener |
| --- | --- |
| `Button` | `AddPressListener`, `AddReleaseListener` |
| `Trigger` | press/release at 0.5, plus `AddUpdateListener(float)` |
| `Sticks` | one listener `(lx, ly, rx, ry)` with per-listener monitor flags |

Details:

* `GamepadEx` button members (`a`, `b`, `trigger`) shadow the inherited `bool`/`float` fields.
  Read `gamepads::gamepad1` for levels, `gamepad_ex::gamepad1` for edges.
* `Sticks` passes Y negated, so `ly > 0` means up. Raw `left_stick_y` is negative when up.
* `Remove*Listener` compares addresses and never matches. Use `Clear*Listeners`.
* Captures: a reference to an object that lives for the whole opmode is safe; don't reference
  something that goes out of scope while the callback can still fire.

## Telemetry Manager

Channels keep subsystems from overwriting each other's lines.

```cpp
extern "C" void telemetry_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    telemetry_manager::channel a = telemetry_manager::CreateChannel("A");
    telemetry_manager::channel b = telemetry_manager::CreateChannel();

    WaitForStart();

    while (OpModeIsActive()) {
        telemetry_manager::AddData(a, "value1", 0.4);
        telemetry_manager::AddData(a, "value2", 0.4);
        telemetry_manager::AddLine(b, "ready");

        telemetry_manager::flush(a);
        telemetry_manager::flush(b);
        telemetry_manager::Update();

        Sleep(10);
    }
}
```

`AddLine`/`AddData` append to pending; `flush` moves pending to current; `Update` prints every
non-empty current buffer and calls `telemetry::Update()` once. Current is not cleared, so a
channel keeps its last frame until the next flush.

`FlushMode`: `DONT` (default) appends only, `FLUSH` appends and flushes that channel, `UPDATE`
also renders. Don't use `FLUSH` on several lines in a row — each one replaces the previous
frame. Add all lines, flush once, then `Update()`.

`CreateChannels({&a, &b})` creates several unnamed channels. `init()` clears the channel
list, so don't call it after creating handles. Everything is a no-op when
`TELEMETRY_MANAGER_ENABLED` is false.

## Lucarne

Lucarne is a WebSocket server on port **1708**. Connect the dashboard to the robot's address.
One client at a time, no auth, no TLS; it is tuning infrastructure.

```cpp
namespace {
    double gain = 0.0;
    double value = 0.0;
}

extern "C" void tune_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    lucarne::config::Live("gain", gain);
    lucarne::graph::Live("value", value);

    lucarne::recorder::start();

    WaitForStart();

    while (OpModeIsActive()) {
        value += 0.01;

        lucarne::field::DrawRobot(0.0, 0.0, 0.0);

        Sleep(10);
    }

    lucarne::recorder::stop();
}
```

* `config::Live` supports `double`, `bool`, `std::string`; it publishes every 200 ms. The
  variable must outlive the registration. The registry is locked; the variable is not, so
  don't mutate a registered `std::string` from another thread.
* `graph::Live` is `double` only, sampled about 20 times a second. The dashboard derives
  integral/derivative from the `(t, value)` stream.
* `field` calls pass coordinates through; the dashboard field is inches with a corner origin,
  matching Pedro. `DrawHistoryAndTarget(..., {}, {})` omits the target path.
* `recorder` writes every outgoing message to disk and sends the file as one base64 message
  when `stop()` runs. `stop()` uses the socket, so keep the dashboard connected until it
  returns.

The gamepad and telemetry mirrors are automatic: `gamepads::Update()` sends both pads, and
`telemetry::Update()` sends the lines. The dashboard can also drive a virtual gamepad; input
is merged on top of the physical pad and dropped after 300 ms of silence (not legal match
control).

Messages are `{"type", "data"}`:

| Type | Direction | Data |
| --- | --- | --- |
| `config` / `config_set` | both | values, or `{name, value}` |
| `graph` | robot → dashboard | `{t, values}` |
| `field` | robot → dashboard | `{robot, history, target}` |
| `gamepad` / `gamepad_input` | both | pad fields |
| `telemetry` | robot → dashboard | `{lines}` |
| `recording` / `recording_status` | robot → dashboard | blob, or `{recording}` |

For a custom panel: `lucarne::Send("my_panel", {{"value", 42}})` and
`lucarne::On("my_panel_set", handler)`. The handler runs on the socket thread; don't touch
motors there.

## Task Manager

```cpp
#include <atomic>

namespace {
    std::atomic<int> ticks{0};
}

extern "C" void task_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;
    using namespace std::chrono_literals;

    task_scheduler::RegisterTask([] { ticks++; return false; }, 20ms, "example", false, Core::C2);

    WaitForStart();

    while (OpModeIsActive()) {
        task_scheduler::Execute();   // pumps C1 only

        telemetry::AddData("ticks", ticks.load());
        telemetry::Update();

        Sleep(10);
    }

    task_scheduler::ClearTasks(Core::C2);
}
```

* `Core::C1` is the opmode thread; call `Execute()` in the loop. Each call runs at most one C1
  task, so a 10 ms loop limits C1 to one task per 10 ms.
* `Core::C2`/`C3` get their own pinned thread on first registration and tick themselves every
  500 µs. Nothing to pump.
* `RegisterTask(callable, interval, name, run_direct, core, end_function)`. Return `true` to
  finish and run `end_function`; return `false` (or `void`) to repeat.
* `run_direct = true` runs once immediately before queuing.
* An interval `<= 0` is rejected with a warning.
* Keep hardware and telemetry on C1. C2/C3 are plain C++ threads; pass data back with
  `std::atomic`.
* `RunAfter(fn, 0.75_s)` schedules a one-shot on a core, and `RegisterFunctionSleepChain`
  runs steps in order with a delay before each.

`IsRealtime(Core::C2)` tells you whether the `SCHED_FIFO` request succeeded;
`rt::Probe().sched_fifo` tells you whether the process can grant it at all.

## Sequencer

```cpp
extern "C" void sequence_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    pedro::init(pedro::p(0, 0, 0), Alliance::RED);
    pedro::Path path = pedro::lp(pedro::p(0, 24), pedro::p(24, 24, 90));

    SequencerEx seq;
    seq.f(path);              // start the path and wait for it
    seq.d([] { /* action */ });
    seq.w(0.4_s);

    WaitForStart();

    while (OpModeIsActive()) {
        seq.Update();
        pedro::update();
        Sleep(10);
    }
}
```

| Call | Meaning |
| --- | --- |
| `u(condition, on_start, timeout)` | Wait until true; advance after the timeout anyway. |
| `w(duration, on_start, on_end)` | Wait a fixed time. |
| `d(callback, timeout)` | Run once and advance. |
| `f(path)` | `followPath(path)` and wait until Pedro is free. |
| `f(on_start)` | Wait until Pedro is free, running `on_start` first. |

`Update()` must be called every loop; callbacks run on the opmode thread, so keep them short.
A finished sequencer wakes up when you append another step. `reset()` rewinds, `clear()`
deletes, `IsDone()` reports the end.

## Pedro

Pedro 3 runs on the Java side; `sdk::pedro` is the C++ face of it. There is no `Follower` object in
your opmode and no `Constants.java` to edit — one JSON file describes the robot, and `pedro::init()`
builds the follower from it.

### drive_config.json

`TeamCode/drive_config.json` is the only Pedro file you edit. The build validates it, exposes typed
C++ values as `sdk::drive_config`, and embeds the text in `libsdk.so`; `pedro::init()` passes that
JSON to the Java follower, so a typo fails the build at the key name and a config edit needs only
the native rebuild.

| Block | What it sets |
| --- | --- |
| `teleOp` | how `pedro::teleOpUpdate()` drives: `driveType`, `robotCentric`, `turnMultiplier`, `cancelAutoDriveButton`, Sparky anti-brake nudge |
| `localizer` | Pedro's localizer: hardware `name`, `podType`, pod directions, `xPodOffset`/`yPodOffset` with `offsetUnits`, `ticksPerUnit` (`null` uses the pod's built-in resolution), `resetMode` |
| `drivetrain` | mecanum motor names and directions, `manualBrakeMode`, `powerThreshold` |
| `foresight` | every Pedro 3 follower setting: heading and translational feedback, brake and coast controllers, tolerances, limits, achievable velocities and decelerations |

Only `PINPOINT` and `MECANUM` are implemented by the adapter; anything else is rejected at build
time. The fields you match to your robot are in `drivetrain`:

```json
"drivetrain": {
  "type": "MECANUM",
  "mecanum": {
    "frontLeftName": "leftFront",
    "backLeftName": "leftRear",
    "frontRightName": "rightFront",
    "backRightName": "rightRear",
    "frontLeftDirection": "REVERSE",
    "backLeftDirection": "REVERSE",
    "frontRightDirection": "FORWARD",
    "backRightDirection": "FORWARD",
    "manualBrakeMode": true,
    "powerThreshold": 0.01
  }
}
```

All four blocks must be present, and `foresight` must contain exactly the Pedro 3 fields — the
generator rejects anything missing or extra, so a Pedro upgrade means updating that block.

Offsets, directions and the follower numbers come from Pedro's AutoTune: open
`http://<robot-ip>:10158`, run the procedure from `Tuning.java`, and paste the result into the
matching JSON block. AutoTune does not write `drive_config.json`; rerun `download.sh` after pasting.
AutoTune itself runs on the Java side, so a normal APK build is needed before AutoTune sees values
added to the JSON; the opmodes use them as soon as `download.sh` finishes.

### Build the config

1. Edit `TeamCode/drive_config.json`.
2. `bash TeamCode/src/main/cpp/download.sh` (or `--compile-only` when no robot is connected).
   It runs `:TeamCode:generateDriveConfigBridge` first; that validates the file and writes
   `TeamCode/build/generated/native/DriveConfig.generated.cpp` and `.hpp`, then rebuilds and pushes
   the native libraries.
3. Run the opmode. `pedro::init()` hands the new JSON to Java, so no APK rebuild is needed.

To regenerate only, run `./gradlew :TeamCode:generateDriveConfigBridge`. The generated native files
are build outputs; never edit them.

### Init

`pedro::init(pose, alliance)` builds the follower from `drive_config.json`, sets the start pose and
ticks once. Call it during init, before `WaitForStart()`: `lp`/`tp` chain from the pose you pass.

### Autonomous

Poses are inches and radians; `p(x, y, deg)` converts the angle.

```cpp
extern "C" void auto_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    pedro::init(pedro::p(0, 0, 0), Alliance::RED);
    pedro::Path path = pedro::lp(pedro::p(0, 24), pedro::p(24, 24, 90));

    WaitForStart();
    pedro::followPath(path);

    while (OpModeIsActive()) {
        pedro::update();
        if (!pedro::isBusy()) break;

        const pedro::Pose pose = pedro::getPose();
        const pedro::PoseHistory trail = pedro::getPoseHistory();
        const pedro::PoseHistory target = pedro::getPathDrawingPoints(path);

        lucarne::field::DrawHistoryAndTarget(pose.x, pose.y, pose.heading,
                                             trail.x, trail.y, target.x, target.y);

        Sleep(10);
    }
}
```

* `lp`/`tp` chain from the last built pose and mirror for `Alliance::BLUE` using a 144 in
  field. `tp(true, ...)` reverses direction.
* Two poses per segment make a line, three or more make a Bezier. Paths are cached by content
  for the run.
* For per-segment headings use `PathBuilder`; it is never mirrored:

```cpp
pedro::Path path = pedro::PathBuilder()
    .segment({pedro::p(0, 0), pedro::p(24, 24)}, pedro::Heading::tangent())
    .segment({pedro::p(24, 24), pedro::p(48, 24)}, pedro::Heading::facingPoint(48, 48))
    .build();
```

* `update()` must run every loop in auto, or the pose goes stale. `breakFollowing()` stops,
  `atParametricEnd()` reports the last path's end, `getTotalHeading()` gives unwrapped radians.

### Teleop

```cpp
extern "C" void teleop_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    if (pedro::hasStoredPoseAndAlliance(120.0)) {
        pedro::InitWithSaved(120.0);   // also calls init()
    } else {
        pedro::init(pedro::p(0, 0, 0), Alliance::RED);
    }

    pedro::startTeleOpDrive(true);     // true = brake mode

    WaitForStart();

    while (OpModeIsActive()) {
        pedro::teleOpUpdate();         // reads gamepads, ticks the pose
        Sleep(10);
    }
}
```

`teleOpUpdate()` polls the pads and runs Pedro's teleop drive. What it does comes from `teleOp`
in `drive_config.json` (`robotCentric`, `turnMultiplier`, `cancelAutoDriveButton`), so changing
how teleop feels is a JSON edit and a rebuild, not a code edit.

At the end of auto, call `pedro::storeCurrentPoseAndAlliance()` if teleop should continue from
where auto stopped; `pedro::InitWithSaved(...)` restores it while the stored pose is still valid.

### How this differs from the official Pedro docs

The [official pages](https://pedropathing.com/docs/pathing) still describe the path API, units
and tuning process correctly. What changes here is where the configuration lives and who owns
the follower:

| Official docs | This repo |
| --- | --- |
| Install by copying the `pedro` package, then write `Constants.java` | The package is already vendored; robot values live in `TeamCode/drive_config.json` |
| `Constants.create(hardwareMap)` | `pedro::init(startPose, alliance)` builds the follower from JSON |
| Create a `Follower` field, call `follower.update()` every loop | No follower object: call `pedro::update()` every loop |
| Build paths in the visualizer, save a `.pp`, load it in the opmode | Build paths in C++ with `lp`/`tp`/`PathBuilder`; nothing loads `.pp` files |
| `.with()` modifiers and end constraints per path | Not exposed; use more segments or `PathBuilder` headings for the same motion |
| Ivy / `SequentialCommandGroup` | `SequencerEx` (`f`, `d`, `w`, `u`) |
| AutoTune writes values you paste into `Constants.java` | Paste AutoTune output into `drive_config.json`, then rebuild |
| Foresight settings are Java builder calls | `foresight` must match the Pedro 3 field names exactly; the generator rejects missing or unknown keys |

If a Pedro upgrade adds a follower setting, add it to `foresight` (or the block it belongs to)
before building — otherwise `generateDriveConfigBridge` fails and names the offending key.

## Cross-opmode storage

```cpp
extern "C" void storage_demo() {
    using namespace sdk;
    using namespace sdk::linear_op_mode;

    WaitForStart();

    cross_opmode_storage::store("example", {
        {"value", 1.0}
    });

    if (auto saved = cross_opmode_storage::load("example", 120.0)) {
        telemetry::AddData("saved", (*saved)["value"]);
        telemetry::Update();
    }
}
```

One file per key, JSON contents, wall-clock age limit (60 s default). `IsValid(key, age)`
checks without parsing; `Invalidate(key)` deletes. Pedro uses the same mechanism internally.

## Math, constants, logging

`sdk::math` is constexpr:

```cpp
if (sdk::math::WillOvershoot(AngularSpeed(2.0), AngularAcceleration(4.0), Degree(90))) {
    // ...
}

const Newton pull = sdk::math::TorqueToForce(Torque(1.2), Metre(0.05));
const bool close = sdk::math::InProportionalRange(0.95, 1.0, 0.1);
const int dir = sdk::math::Signum(-1.0);
```

`sdk::hardware_constants` has servo PWM ranges and ranges of motion, REV Through Bore ticks
per revolution, camera focal length, and `GRAVITY`.

```cpp
sdk::log_manager::LOG("autonomous started");
sdk::log_manager::LOG("pull", pull.Value());
```

It writes synchronously at `ANDROID_LOG_FATAL` under `settings::LOG_TAG`; don't call it per
loop.

## Build and deploy
For C++ and driver config changes use the C++ Only launch config with:
* No flags for normal launch
* `-f` for force compiling and pushing everything again
* `-w` for pushing over cable instead of Wi-Fi
* `-c`for only compiling no pushing
If you added a new opmode, deleted one or changed a java file, use the FULL launch config with:
* `-l` to launch it to your robot
* `-b` to only build it

## How it works, briefly

An opmode is one exported C++ function. The generated Java class loads the libraries, calls
`init_sdk()` to resolve the Java classes, then calls that function; when it returns, threads
are stopped and the libraries are unloaded. That is why everything is constructed per run.

`hardware_map::get` returns a JNI reference owned by the wrapper, so a device is valid for the
opmode's lifetime. Lucarne is a small WebSocket server; config, graph, field and recorder are
publishers on top of it, while the gamepad and telemetry mirrors hook the existing update
functions. C1 is pumped by `Execute()`, C2/C3 run their own pinned threads. `CMotorE`/`CMotorS`
convert torque to power with the motor constants and cached bus voltage. The Pedro bridge hands the
validated JSON to Java once at init, keeps the pose buffer and path cache natively, and only crosses
JNI on build/update.
