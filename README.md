# mfp-daemon-rs

A Rust implementation of Xiaomi's `mfp-daemon` fingerprint AIDL HAL. It exposes
the Android fingerprint interfaces, loads the legacy vendor fingerprint HAL,
implements the Xiaomi extension service, and reproduces the Xiaomi FOD engine
used for under-display fingerprint authentication.

This is an unofficial project intended for research, reverse engineering, and
development on devices you own and can recover.

## Target platform

The implementation targets the `peridot` platform (Xiaomi `24069RA21C`) with
Android 16 and an optical under-display sensor.

The FOD path is enabled when:

```text
ro.hardware.fp.fod.touch.ctl.version = 2.0
```

The code reads sensor geometry, brightness thresholds, and touch-controller
capabilities from the same vendor properties used by the stock daemon.

## Interfaces

| Interface | Instance | Notes |
| --- | --- | --- |
| `android.hardware.biometrics.fingerprint.IFingerprint` | `default` | Version 3 |
| `android.hardware.biometrics.fingerprint.ISession` | per session | Version 3 |
| `android.hardware.biometrics.fingerprint.ISessionCallback` | client-provided | Version 3 |
| `vendor.xiaomi.hardware.fingerprintextension.IXiaomiFingerprint` | `default` | Version 1 |

`IFingerprint/default` is registered as a lazy service and made persistent.
`IXiaomiFingerprint/default` is registered when a usable vendor HAL is loaded.

## Architecture

```text
┌──────────────────────────────────────────────────────────────────────┐
│ Android framework                                                    │
│  FingerprintService · SystemUI / FOD UI · Xiaomi framework           │
│    │ Binder: IFingerprint, ISession, ISessionCallback                │
│    │ Binder: IXiaomiFingerprint                                      │
└────┼─────────────────────────────────────────────────────────────────┘
     │
┌────▼─────────────────────────────────────────────────────────────────┐
│ /odm/bin/hw/mfp-daemon (this project)                                │
│                                                                      │
│  main.rs ───── loads HAL, starts Daemon, registers Binder services   │
│  service.rs ── AIDL state machine, worker queue, notifications       │
│  fod.rs ────── display/touch/evdev monitors, FOD state, HBM          │
│  touchfeature.rs ─ raw Binder client for TouchFeature                │
│  hal.rs ────── legacy vendor HAL loader and function-table wrappers  │
└───┬───────────────────────┬────────────────────────┬─────────────────┘
    │ dlopen / dlsym        │ Binder                 │ read/ioctl
┌───▼───────────────────┐ ┌─▼─────────────────────┐ ┌▼──────────────────┐
│ libhardware.so        │ │ TouchFeature service  │ │ disp_feature      │
│ fingerprint.<vendor>  │ │ setModeValue          │ │ touch sysfs       │
│ legacy HMI, ABI 0x201 │ │                       │ │ evdev touch input │
└───────────────────────┘ └───────────────────────┘ └───────────────────┘
```

Module dependency flow:

```text
main
├── hal          load libhardware, resolve vendor fingerprint device
└── service      Daemon, AIDL implementations, worker queue
    ├── fod      FOD engine and hardware monitors
    │   ├── hal
    │   └── touchfeature
    ├── hal
    ├── touchfeature
    └── mangled  generated AIDL types
```

The daemon is the only owner of the vendor fingerprint device. AIDL requests
are converted into `Job` values and serialized through the `hal-worker` thread,
so vendor calls and notifications never race. The FOD engine is independent of
the AIDL session lifetime and keeps the screen-off touch/HBM path active.

## Repository layout

Every tracked path is listed below. The local `.cargo/config.toml` is ignored
and is created from `.cargo/config.toml.example`. Vendor libraries, firmware,
decompiled output, device dumps, `target/`, `libs/`, `orig/`, `ref/`, and
`work/` are not part of the repository.

```text
.
├── .cargo
│   └── config.toml.example
├── aidl
│   └── vendor
│       └── xiaomi
│           └── hardware
│               └── fingerprintextension
│                   └── IXiaomiFingerprint.aidl
├── src
│   ├── bin
│   │   ├── ext_cmd.rs
│   │   └── test_client.rs
│   ├── gen
│   │   ├── android
│   │   │   └── hardware
│   │   │       ├── biometrics
│   │   │       │   ├── common
│   │   │       │   │   ├── AuthenticateReason.rs
│   │   │       │   │   ├── CommonProps.rs
│   │   │       │   │   ├── ComponentInfo.rs
│   │   │       │   │   ├── DisplayState.rs
│   │   │       │   │   ├── ICancellationSignal.rs
│   │   │       │   │   ├── OperationContext.rs
│   │   │       │   │   ├── OperationReason.rs
│   │   │       │   │   ├── SensorStrength.rs
│   │   │       │   │   └── WakeReason.rs
│   │   │       │   └── fingerprint
│   │   │       │       ├── AcquiredInfo.rs
│   │   │       │       ├── Error.rs
│   │   │       │       ├── FingerprintSensorType.rs
│   │   │       │       ├── IFingerprint.rs
│   │   │       │       ├── ISession.rs
│   │   │       │       ├── ISessionCallback.rs
│   │   │       │       ├── PointerContext.rs
│   │   │       │       ├── SensorLocation.rs
│   │   │       │       ├── SensorProps.rs
│   │   │       │       ├── SensorShape.rs
│   │   │       │       └── TouchDetectionParameters.rs
│   │   │       └── keymaster
│   │   │           ├── HardwareAuthToken.rs
│   │   │           ├── HardwareAuthenticatorType.rs
│   │   │           ├── SecurityLevel.rs
│   │   │           ├── Timestamp.rs
│   │   │           └── VerificationToken.rs
│   │   └── vendor
│   │       └── xiaomi
│   │           └── hardware
│   │               └── fingerprintextension
│   │                   └── IXiaomiFingerprint.rs
│   ├── fod.rs
│   ├── hal.rs
│   ├── lib.rs
│   ├── log.rs
│   ├── main.rs
│   ├── service.rs
│   └── touchfeature.rs
├── vendor
│   └── binder_ndk_sys
│       ├── src
│       │   ├── include_cpp
│       │   │   └── android
│       │   │       ├── binder_auto_utils.h
│       │   │       ├── binder_enums.h
│       │   │       ├── binder_interface_utils.h
│       │   │       ├── binder_internal_logging.h
│       │   │       ├── binder_parcel_utils.h
│       │   │       ├── binder_parcelable_utils.h
│       │   │       └── binder_to_string.h
│       │   ├── include_ndk
│       │   │   └── android
│       │   │       ├── binder_ibinder.h
│       │   │       ├── binder_ibinder_jni.h
│       │   │       ├── binder_parcel.h
│       │   │       ├── binder_parcel_jni.h
│       │   │       └── binder_status.h
│       │   ├── include_platform
│       │   │   └── android
│       │   │       ├── binder_ibinder_platform.h
│       │   │       ├── binder_libbinder.h
│       │   │       ├── binder_manager.h
│       │   │       ├── binder_parcel_platform.h
│       │   │       ├── binder_process.h
│       │   │       ├── binder_shell.h
│       │   │       └── binder_stability.h
│       │   ├── BinderBindings.hpp
│       │   ├── lib.rs
│       │   └── symbols.txt
│       ├── Cargo.toml
│       ├── Cargo.toml.orig
│       └── build.rs
├── .gitignore
├── Cargo.lock
├── Cargo.toml
└── README.md
```

Path responsibilities:

| Path | Purpose |
| --- | --- |
| `aidl/vendor/xiaomi/hardware/fingerprintextension/IXiaomiFingerprint.aidl` | Xiaomi fingerprint extension interface |
| `src/main.rs` | Process entry point, HAL load, service registration |
| `src/lib.rs` | Module graph and generated AIDL re-exports |
| `src/service.rs` | `IFingerprint` / `ISession` / `IXiaomiFingerprint`, worker queue, notifications, lockout |
| `src/hal.rs` | Legacy vendor HAL loader, ABI slots, property helpers |
| `src/fod.rs` | FOD state, display/brightness events, press pipeline, HBM |
| `src/touchfeature.rs` | Raw Binder client for the vendor TouchFeature service |
| `src/log.rs` | `logd!` / `logi!` / `logw!` / `loge!` macros |
| `src/bin/ext_cmd.rs` | Xiaomi `extCmd` diagnostic client |
| `src/bin/test_client.rs` | AIDL fingerprint session smoke client |
| `src/gen/android/` | Checked-in AIDL-generated Android biometric/keymaster types |
| `src/gen/vendor/xiaomi/hardware/fingerprintextension/` | Checked-in generated Xiaomi extension type |
| `vendor/binder_ndk_sys/` | Patched Binder NDK bindings and target stub build |
| `.cargo/config.toml.example` | Template for the machine-specific NDK linker path |
| `Cargo.toml` | Crate manifest and local `binder_ndk_sys` patch |
| `Cargo.lock` | Reproducible dependency versions |

## AIDL binding

The custom interface source is kept in
`aidl/vendor/xiaomi/hardware/fingerprintextension/`. The Rust bindings in
`src/gen/android` and `src/gen/vendor` are checked in because the repository
does not run an AIDL generator at build time. `src/lib.rs` mounts each file with
`#[path = ...]` and re-exports the mangled names through `pub mod mangled`.

| Interface | Version | Hash |
| --- | ---: | --- |
| `android.hardware.biometrics.fingerprint.IFingerprint` | 3 | `637371b53fb7faf9bd43aa51b72c23852d6e6d96` |
| `android.hardware.biometrics.fingerprint.ISession` | 3 | `637371b53fb7faf9bd43aa51b72c23852d6e6d96` |
| `android.hardware.biometrics.fingerprint.ISessionCallback` | 3 | `637371b53fb7faf9bd43aa51b72c23852d6e6d96` |
| `vendor.xiaomi.hardware.fingerprintextension.IXiaomiFingerprint` | 1 | `1ed45089cc89154986c15fc1c591d72e9ac7ae0d` |

## Process architecture

`main.rs` performs the following sequence:

1. Sets the binder thread-pool limit to 0.
2. Creates the HAL notification channel.
3. Loads the vendor fingerprint HAL.
4. Starts the daemon worker.
5. Registers `IFingerprint/default` as a lazy service.
6. Enables persistent lazy services.
7. Registers `IXiaomiFingerprint/default` when a HAL is available.
8. Joins the binder thread pool.

Runtime threads:

| Thread | Responsibility |
| --- | --- |
| binder pool | AIDL entry points, Xiaomi `extCmd` |
| `hal-worker` | Serializes HAL operations and notification dispatch |
| `hal-notify` | Moves HAL callbacks from the vendor thread into the worker |
| `fod-display` | Polls `/dev/mi_display/disp_feature` |
| `fod-touch` | Polls `/sys/class/touch/touch_dev/fod_press_status` |
| `fod-evdev` | Reads touch coordinates from the vendor touch input device |
| `lockout-timer` | Expires timed lockout after 30 seconds |

The daemon keeps one global client callback and one active session identifier,
matching the single-client model used by the vendor daemon.

## Control flow

1. `main` loads the vendor HAL, installs the legacy notification callback,
   starts `Daemon`, and registers the AIDL services.
2. `createSession` stores the client callback and session id, links a death
   recipient, and calls the vendor `set_active_group` with
   `/data/vendor_de/<userId>/fpdata`.
3. Every session operation marks the session status and enqueues a `Job`.
   `hal-worker` drains the queue and invokes one vendor function at a time.
4. The vendor HAL reports results through `fingerprint_msg_t`. The callback
   copies the message into an `mpsc` channel, `hal-notify` forwards it as
   `Job::Notify`, and the worker translates it into an AIDL callback.
5. Notification handling also updates global status, enrollment counts,
   lockout state, and FOD/TouchFeature state.
6. Xiaomi `extCmd` values `1..8` update `FodEngine` conditions. The touch
   monitor gates FOD presses; a valid down selects HBM and drives the vendor
   `extCmd` sequence, while an up or reset clears HBM and touch state.
7. `close`, cancellation, client death, or a HAL error stops the active path
   and resets the FOD engine.

## HAL integration

`hal.rs` loads `libhardware.so` and resolves `hw_get_module_by_class`.

Loading order:

1. Read `persist.vendor.sys.fp.vendor`.
2. Try the configured vendor.
3. Try the stock vendor list:
   `goodix_us`, `qcom_us`, `goodix_fod`, `fpc_fod`, `goodix_fod6`,
   `goodix_fod7s`, `goodix`, `fpc`, `jiiov`.
4. Fall back to direct `dlopen` of
   `/odm/lib64/hw/fingerprint.<vendor>.so` or
   `/vendor/lib64/hw/fingerprint.<vendor>.so`.
5. Resolve the legacy `HMI` symbol from the directly loaded library.

Each candidate must expose legacy device version `0x201` and a `set_notify`
slot. Unknown or incompatible candidates are rejected.

Legacy function table offsets:

```text
set_notify                   0x80
generate_challenge           0x88
revoke_challenge             0x90
enroll                       0x98
get_authenticator_id         0xa0
invalidate_authenticator_id  0xa8
cancel                       0xb0
enumerate                    0xb8
remove                       0xc0
set_active_group             0xc8
authenticate                 0xd0
reset_lockout                0xd8
onPointerDown                0xe0
onPointerUp                  0xe8
extCmd                       0xf0
detect_interaction           0x110
```

The vendor notification callback copies `fingerprint_msg_t` into a channel. The
worker consumes notifications in order with other daemon work.

## Authentication

### Sensor properties

`getSensorProps` returns a single optical under-display sensor:

| Property | Value |
| --- | --- |
| Sensor ID | `5` |
| Strength | `STRONG` |
| Maximum enrollments | `5` |
| Type | `UNDER_DISPLAY_OPTICAL` |
| Location | derived from vendor properties |
| Shape | circle |
| Radius | half of the configured width |
| `halHandlesDisplayTouches` | `true` |
| `halControlsIllumination` | `false` |
| `supportsDetectInteraction` | `false` |

Geometry is read from:

```text
persist.vendor.sys.fp.fod.location.X_Y
persist.vendor.sys.fp.fod.size.width_height
```

The reported center is:

```text
x = location.x + width / 2
y = location.y + height / 2
```

The component list and strings match the vendor daemon's AIDL response.

### Session lifecycle

`createSession(sensorId, userId, callback)`:

1. Allocates a session id.
2. Stores the callback and session id in the global state.
3. Registers a binder death recipient for the callback.
4. Calls the vendor `set_active_group(userId, "/data/vendor_de/<userId>/fpdata")`.
5. Returns an `ISession`.

Session operations are placed on the worker queue, so vendor HAL calls never
overlap. A request is ignored after the session has been closed.

### Session operations

| Operation | Behavior |
| --- | --- |
| `generateChallenge` | Calls the vendor challenge slot; the reply arrives as a notification |
| `revokeChallenge` | Calls the vendor revoke slot |
| `enroll` | Converts the HAT and calls the vendor enroll slot |
| `authenticate` | Calls the vendor authenticate slot with the operation id |
| `detectInteraction` | Completes immediately; the sensor does not support it |
| `enumerateEnrollments` | Calls the vendor enumerate slot |
| `removeEnrollments` | Limits the list to five ids and calls the vendor remove slot |
| `getAuthenticatorId` | Calls the vendor getter; the reply is a notification |
| `invalidateAuthenticatorId` | Calls the vendor invalidator |
| `resetLockout` | Converts the HAT and calls the vendor reset slot |
| `close` | Marks the session closed, resets FOD state, and sends `onSessionClosed` |
| `cancel` | Stops the active HAL operation and resets FOD state |

Pointer and UI-ready calls (`onPointerDown`, `onPointerUp`, `onUiReady`, and the
context variants) reset the session status. Pointer events are not forwarded to
the vendor HAL in this implementation.

### Legacy authentication token

The vendor HAL consumes a packed 69-byte `hw_auth_token_t`:

| Field | Offset | Encoding |
| --- | ---: | --- |
| `version` | 0 | `u8` |
| `challenge` | 1 | `u64` little-endian |
| `user_id` | 9 | `u64` little-endian |
| `authenticator_id` | 17 | `u64` little-endian |
| `authenticator_type` | 25 | `u32` big-endian |
| `timestamp` | 29 | `u64` big-endian |
| `mac` | 37 | 32 bytes |

The mixed byte order is required by the vendor interface.

### Notification mapping

The daemon translates legacy notifications into AIDL callbacks:

| Legacy notification | AIDL callback |
| --- | --- |
| `typ = 1` | `onAcquired` |
| `typ = 3` | `onEnrollmentProgress` |
| `typ = 4` | `onEnrollmentsRemoved` |
| `typ = 5` | `onAuthenticationSucceeded` or `onAuthenticationFailed` |
| `typ = 6` | `onEnrollmentsEnumerated` |
| `typ = 7` | `onChallengeGenerated` |
| `typ = 8` | `onChallengeRevoked` |
| `typ = 9` | `onAuthenticatorIdRetrieved` |
| `typ = 10` | `onAuthenticatorIdInvalidated` |
| `typ = 11` | `onLockoutCleared` |
| `typ = -1` | `onError` |

Acquired values:

| Raw value | Reported |
| --- | --- |
| `0` | `GOOD` |
| `1` | `PARTIAL` |
| `2` | `INSUFFICIENT` |
| `3` | `SENSOR_DIRTY` |
| `4` | `TOO_SLOW` |
| `5` | `TOO_FAST` |
| `>= 1000` | `VENDOR` with `raw - 1000` |
| other | `INSUFFICIENT` |

Enrollment progress, authentication success, and authentication failure are
suppressed while the global state is idle. Enrollment and removal notification
lists ignore zero ids.

Errors are mapped to `HW_UNAVAILABLE`, `UNABLE_TO_PROCESS`, `TIMEOUT`,
`NO_SPACE`, `CANCELED`, `UNABLE_TO_REMOVE`, or `VENDOR`.

### Lockout

Failed authentication attempts are counted for under-display sensor types.
Five failures enter a timed lockout for 30 seconds. Twenty failures enter a
permanent lockout. A successful authentication or an explicit lockout reset
clears the counter. `onLockoutTimed`, `onLockoutPermanent`, and
`onLockoutCleared` are delivered to the current callback.

## Screen-off FOD engine

`fod.rs` owns the screen-off fingerprint pipeline. It is created before the
AIDL services are registered and runs for the lifetime of the process.

### Activation

When `ro.hardware.fp.fod.touch.ctl.version` is `2.0`:

1. Open `/dev/mi_display/disp_feature`.
2. Set local HBM to off.
3. Start the display, touch, and evdev monitors.

When the property is absent or different, the FOD engine stays disabled and
only the standard fingerprint HAL path is active.

### Condition state

The framework drives the FOD engine through Xiaomi `extCmd` values `1..8`:

| Command | Meaning | Effect |
| --- | --- | --- |
| `1` | gxzw animation | Updates animation state and re-evaluates touch mode |
| `2` | color fade | Updates color-fade state and re-evaluates touch mode |
| `3` | power mode | Runs the pre-power check, stores the mode, re-evaluates, and forwards HAL `extCmd 14` |
| `4` | fingerprint/HAL mode | Adjusts HBM and stores the current mode |
| `5` | lockout mode | Stores lockout state and re-evaluates touch mode |
| `6..8` | auxiliary modes | Logged |

For command `4`:

| Mode | HBM action |
| --- | --- |
| `1`, `3` | HBM off if currently on |
| `0`, `2`, `4` | `HBM_OFF_AUTH_STOP` |
| other | unchanged |

The current mode is also used as the fingerprint mode for HBM selection and
touch eligibility.

### Touch mode

Touch mode is recomputed from the FOD state:

```text
if !gxzwAnim:
    touchEnable = 0
else if powerMode != 2:
    touchEnable = 1
else if colorfade:
    touchEnable = cached value
else if lockoutMode != 0 or fpMode in {1, 3, 5}:
    touchEnable = 1
else:
    touchEnable = 0
```

A change is sent to TouchFeature:

```text
setModeValue(0, 10, value)   for values other than 5 and 6
setModeValue(0, 35, value)   for values 5 and 6
```

Before the power mode is stored, command `3` performs the pre-power transition.
If the new mode is `0`, the fingerprint mode is `3` or gxzw animation is on, and
the cached touch mode is `0`, the daemon sends touch mode `3` immediately.

### Display events and HBM

The display monitor registers for display and brightness events on
`/dev/mi_display/disp_feature`:

```c
ioctl(fd, 0x400c4407, {u64 cookie; u32 type})
```

with cookies `0` and `1 << 32`, and types `2` (display) and `5` (brightness).

Event records are 16 bytes:

| Offset | Field |
| ---: | --- |
| 0 | reserved |
| 4 | type |
| 8 | size (`16`) |
| 12 | value |

Display event value bits:

| Bit | Meaning |
| --- | --- |
| `value & 4` | low brightness |
| `(value & 3) == 3` or `(value & 8) != 0` | capture start |

HBM is controlled through:

```c
ioctl(fd, 0x400c440e, {u64 1; u32 value})
```

| Value | State |
| ---: | --- |
| `0` | off |
| `1` | off for authentication stop |
| `2` | 1000 nit class / normal |
| `3` | low-light level |
| `4` | mode-5 level |

On capture start, the daemon sends the current FOD coordinates and HAL
`extCmd 10` with `6` for low brightness or `1` otherwise. On capture stop it
sends HAL `extCmd 10` with `0`.

### Press pipeline

`fod_press_status` is the authoritative press source. The touch monitor reads
the node every 20 ms and treats any value other than `'0'` as pressed.

On a down transition:

1. Require HAL mode `1`, `3`, or `5`.
2. Mark the touch as down.
3. Send HAL `extCmd 2 = 0` and `extCmd 3 = 0`.
4. Select and enable local HBM.
5. Send HAL `extCmd 1 = 1`.
6. Send the stored coordinates through `extCmd 2` and `extCmd 3`.
7. Notify TouchFeature of authentication state `0`.

On an up transition:

1. Send the last coordinates through `extCmd 2` and `extCmd 3`.
2. Disable local HBM if it was enabled.
3. Send HAL `extCmd 1 = 0`.

The thread never calls the down/up path from the input device.

### Coordinates and status-bar guard

The evdev monitor reads the vendor touch device found in
`/proc/bus/input/devices`, falling back to `/dev/input/event7`. It reads
`ABS_MT_POSITION_X` (`0x35`) and `ABS_MT_POSITION_Y` (`0x36`) and divides each
value by the touch-controller resolution factor from
`/sys/class/touch/touch_dev/resolution_factor`.

The input device is used only for coordinates. `BTN_TOUCH` events update an
internal debug flag and never start or stop a fingerprint operation.

This separation is the status-bar guard:

- The system touch mode is `0` whenever the framework is not requesting FOD
  authentication, including while the screen is on and the notification shade
  or Control Center is open.
- A touch in the FOD area while the screen is on does not emit a FOD press
  status transition.
- Even if the input device reports `BTN_TOUCH`, the daemon does not call the
  authentication pipeline.
- The `fod_press_status` gate, together with the HAL mode check, ensures that
  authentication only starts for a real FOD press in a mode that permits it.

Coordinates are logged only when they are inside the configured FOD rectangle
plus a 120 px margin and the engine is ready or already handling a touch. This
keeps coordinate tracking local and avoids emitting FOD coordinates for
unrelated UI touches.

### HBM selection

On finger down, HBM is enabled only when the fingerprint mode is eligible:

```text
fpMode < 6 and (1 << fpMode & 0x2a) != 0
```

Eligible modes are `1`, `3`, and `5`.

The HBM level is selected as:

| Condition | Level |
| --- | --- |
| mode `3` and display brightness `<= 411` | `3` (low light) |
| mode `5` | `4` |
| other eligible modes | `2` |

HBM is not re-sent if it is already on. On finger up, HBM is turned off only if
this daemon enabled it.

### Reset

The FOD engine is reset on cancellation, client death, and session close:

- Pending capture sends HAL `extCmd 10 = 0`.
- A pending touch sends HAL `extCmd 1 = 0`.
- HBM is turned off if it was on.
- Touch state, coordinates, and HBM target are cleared.

## Fingerprint animation

The daemon does not draw the fingerprint animation. The animation is rendered
by the framework and SystemUI. The daemon supplies the events and hardware state
that drive it:

1. The framework creates an authentication session and starts the FOD UI.
2. Xiaomi `extCmd` updates set the animation, color-fade, power, fingerprint,
   and lockout conditions.
3. The FOD engine enables the touch controller and local HBM when a valid press
   occurs.
4. The vendor HAL reports acquired values. Values `22` and `23` are forwarded as
   `VENDOR` acquired events and are used by the UI for touch feedback.
5. `onUiReady` resets the session status so the UI can accept the result.
6. Authentication success or failure is delivered through the callback, and
   TouchFeature authentication state is set to `0` or `1`.
7. On finger up, HBM and the FOD touch state are cleared.

The display event monitor keeps HBM synchronized with the panel state, including
the low-brightness path used during screen-off authentication.

## Xiaomi extension

`extCmd(cmd, param)` handles:

| Command | Behavior |
| ---: | --- |
| `1..8` | FOD condition update |
| `10` | Ignored while the FOD engine is enabled; otherwise forwarded to the vendor HAL |
| `11` | Updates `persist.vendor.sys.fp.fp_ocp_control` and forwards HAL `extCmd 11` |
| `12` | Forwarded to the vendor HAL |
| `13` | Clears failed authentication attempts |
| `14` | Updates anti-mistouch configuration and forwards HAL `extCmd 14` |
| `15` | Low-brightness authentication cloud update; log-only, matching the vendor handler |
| `17` | Adds a failed authentication attempt |
| other | Logged and ignored |

## Anti-mistouch

Anti-mistouch configuration is loaded from:

```text
persist.vendor.sys.fp.fp_anti_mistouch
persist.vendor.sys.fp.valid_times_per_screenlock
persist.vendor.sys.fp.side_unlock_type
ro.hardware.fp.fod
ro.hardware.fp.fod.c
```

The default threshold is `3`.

Cloud command `14`:

- Param `0` or `16` sets the property to `false` or `true`.
- If FOD is present, enabling the property enables anti-mistouch.
- Without FOD, it is enabled only when side-unlock type is zero.
- The result is stored and forwarded to the vendor HAL as `extCmd 14`.
- If bit 0 is set, bits 4..7 become the new threshold and
  `persist.vendor.sys.fp.valid_times_per_screenlock` is updated.

When the vendor reports raw acquired code `1028` (`VENDOR 28`) while
anti-mistouch is enabled and the count is below the threshold:

1. Local HBM is enabled.
2. `onAcquired(VENDOR, 28)` is sent.
3. The mistouch counter is incremented.
4. A mistouch flag suppresses the matching authentication failure callback.

A successful authentication resets the counter. Vendor raw code `1040`
(`VENDOR 40`) disables HBM and is not forwarded, matching the UI-ready timeout
path.

## Building

Requirements:

- Rust with Edition 2024 support (Rust 1.85 or newer).
- Android NDK r28 or a compatible NDK with API 34 clang wrappers.
- `libclang` for the vendored `bindgen` build.
- `aarch64-linux-android` Rust target.
- No vendor library is required at build time.

The generated AIDL bindings are committed under `src/gen`.

```bash
rustup target add aarch64-linux-android

export ANDROID_NDK_ROOT="$HOME/android-ndk-r28"
export LIBCLANG_PATH="$ANDROID_NDK_ROOT/toolchains/llvm/prebuilt/linux-x86_64/lib"
export BINDGEN_EXTRA_CLANG_ARGS="--target=aarch64-linux-android34 --sysroot=$ANDROID_NDK_ROOT/toolchains/llvm/prebuilt/linux-x86_64/sysroot"
```

The linker is selected through `.cargo/config.toml`. Create it locally; the
repository ignores the machine-specific file and ships
`.cargo/config.toml.example`:

```toml
[target.aarch64-linux-android]
linker = "/path/to/android-ndk-r28/toolchains/llvm/prebuilt/linux-x86_64/bin/aarch64-linux-android34-clang"
```

Compile:

```bash
cargo build --release --target aarch64-linux-android
```

Artifacts:

```text
target/aarch64-linux-android/release/mfp-daemon
target/aarch64-linux-android/release/ext_cmd
target/aarch64-linux-android/release/test_client
```

## Deployment

### Bind-mount over the stock binary

This keeps the init service definition and SELinux domain intact while replacing
the executable:

```bash
adb push target/aarch64-linux-android/release/mfp-daemon /data/local/tmp/mfp-daemon.rust
adb shell 'su -c "stop mfp-daemon"'
adb shell 'su -c "cp /data/local/tmp/mfp-daemon.rust /data/local/tmp/mfp-daemon.ctx"'
adb shell 'su -c "chmod 755 /data/local/tmp/mfp-daemon.ctx"'
adb shell 'su -c "chcon u:object_r:vendor_mfp_daemon_exec:s0 /data/local/tmp/mfp-daemon.ctx"'
adb shell 'su -c "nsenter -t 1 -m -- mount --bind /data/local/tmp/mfp-daemon.ctx /odm/bin/hw/mfp-daemon"'
adb shell 'su -c "start mfp-daemon"'
```

Verify registration and logs:

```bash
adb shell 'service list | grep -E "fingerprint.IFingerprint|IXiaomiFingerprint"'
adb logcat -s mfp-daemon
```

Restore the stock daemon:

```bash
adb shell 'su -c "stop mfp-daemon"'
adb shell 'su -c "nsenter -t 1 -m -- umount /odm/bin/hw/mfp-daemon"'
adb shell 'su -c "rm -f /data/local/tmp/mfp-daemon.ctx"'
adb shell 'su -c "start mfp-daemon"'
```

### Direct root execution

For development without replacing the stock service:

```bash
adb push target/aarch64-linux-android/release/mfp-daemon /data/local/tmp/mfp-daemon
adb shell 'su -c "stop mfp-daemon"'
adb shell 'su -c "nohup /data/local/tmp/mfp-daemon >/data/local/tmp/mfp.log 2>&1 < /dev/null &"'
```

Stop the test daemon and restart the stock service afterwards:

```bash
adb shell 'su -c "pkill -f /data/local/tmp/mfp-daemon || true"'
adb shell 'su -c "start mfp-daemon"'
```

### Test clients

`test_client` connects to the registered fingerprint service and exercises the
session API. `ext_cmd` sends Xiaomi extension commands:

```bash
adb push target/aarch64-linux-android/release/test_client /data/local/tmp/
adb push target/aarch64-linux-android/release/ext_cmd /data/local/tmp/
adb shell 'su -c "chmod 755 /data/local/tmp/test_client /data/local/tmp/ext_cmd"'
adb shell 'su -c /data/local/tmp/test_client'
adb shell 'su -c "/data/local/tmp/ext_cmd 4 1"'
```

`test_client` replaces the daemon's global session and callback. Do not run it
while the framework is actively using biometrics.

## Runtime scope

- The daemon registers the fingerprint service and the Xiaomi extension service.
  It consumes the vendor TouchFeature service but does not provide it.
- The FOD engine implements the optical path used by `peridot`. TDDI-specific
  condition handling, side-cap sensor monitoring, and film-mode forwarding are
  not part of this path.
- The daemon assumes a single active client callback, matching the vendor
  service model.
- Deployment uses the existing init service and file label. SELinux enforcing
  mode requires a policy that permits the replacement binary.
- Permanent installation is not provided. The stock binary is on a read-only
  `/odm` partition.

## Safety

This software talks directly to a vendor fingerprint HAL and runs as a
privileged Android service. Use it only on devices you own and can recover.
Incorrect installation can break fingerprint functionality or prevent the device
from booting normally.

## License

Apache License 2.0. See [LICENSE](LICENSE).
