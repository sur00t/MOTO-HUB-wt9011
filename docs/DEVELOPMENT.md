# Development guide

This document covers building, testing, packaging, extending and debugging the WT9011 module. It
describes **version 0.2.0, contract 43**. For the design, see [`ARCHITECTURE.md`](ARCHITECTURE.md);
for the wire format and calibration math, see [`PROTOCOL.md`](PROTOCOL.md).

## 1. Repository layout and build environment

This directory is a **module source tree**, not a standalone Gradle project. It has no `gradlew`
and no `settings.gradle.kts`. It is built from the MOTO-HUB SDK checkout, where it lives at
`modules/wt9011` and is included by the root build.

```text
wt9011-module/
├── build.gradle.kts                  module metadata and dependencies
├── LICENSE                           AGPL-3.0-only
├── README.md                         rider-facing install, calibration and usage
├── wt9011-0.2.0.mhm                  packaged, signed module artifact
├── docs/
│   ├── ARCHITECTURE.md
│   ├── DEVELOPMENT.md
│   └── PROTOCOL.md
├── backup-0.1.0/                     previous local tree, kept for reference
└── src/
    ├── main/
    │   ├── AndroidManifest.xml       intentionally empty
    │   └── java/io/github/sur00t/wt9011/
    │       ├── Wt9011Entry.kt
    │       ├── Wt9011Link.kt
    │       ├── Wt9011Packet.kt
    │       └── Packet.kt
    └── test/java/io/github/sur00t/wt9011/
        └── Wt9011PacketTest.kt
```

The README's build instructions assume:

- JDK 21 (the Kotlin/Java target is 17, but the SDK build uses a JDK 21 toolchain);
- an Android SDK with platform 36;
- the MOTO-HUB SDK checkout, optionally provisioned by `scratch/bootstrap-build-env.sh`.

```sh
export JAVA_HOME=.../scratch/jdk21/jdk-21.*
export ANDROID_HOME=.../scratch/android-sdk
export GRADLE_USER_HOME=.../scratch/gradle-home
```

If the SDK checkout was provisioned by the bootstrap script, those paths are under `scratch/`.

## 2. Build, test and package

Run these from the **SDK checkout root**, not from this directory.

| Task | Command | Purpose |
| --- | --- | --- |
| Unit tests | `./gradlew :modules:wt9011:testDebugUnitTest` | decoder, parser, zero, protocol and gyro tests on the JVM |
| Package | `./gradlew :modules:wt9011:modulePackage -Pmotohub.moduleKeyDir=.../scratch/motohub-keys` | writes `build/module/out/wt9011-0.2.0.mhm` |
| Linkage check | `./gradlew :modules:wt9011:checkModuleLinkage -PmotohubApk=/path/to/moto-hub.apk` | verifies the module against a real app build |

The packaged artifact is a zip:

```text
module.jar      classes.dex + META-INF/motohub-module.json
module.sig      signature
module.pub      public key
```

The 0.2.0 tree in this directory was verified with `:modules:wt9011:testDebugUnitTest` (27 tests) and
`:modules:wt9011:modulePackage`; the resulting `wt9011-0.2.0.mhm` is in the repository root.

Install it with **MOTO-HUB → Modules → Install from a file**, with developer mode on. The module
runs inside the host process and uses the host's Bluetooth permissions; the empty manifest is not a
mistake. It needs a MOTO-HUB that knows contract 43 (`ModuleMotionSource`).

## 3. Test suite

The tests are plain JUnit 4 on the JVM. They deliberately construct frames byte by byte, so a wrong
offset, missed sign extension, wrong scale or broken resync fails here rather than on the bike.

```sh
./gradlew :modules:wt9011:testDebugUnitTest
```

### 3.1 Inventory

The suite has **27 tests**.

| Test | What it guarantees |
| --- | --- |
| `decodesOneFrameWithTheDocumentedOffsets` | field offsets, scales, seq and arrival timestamp |
| `appliesTheAxisCandidateWithoutLosingTheOtherField` | lean and nose come from different fields, whichever `LEAN_IS_PITCH` selects |
| `mapsForwardAndRightGravityPerAxisCandidate` | forward/right g mapping for both candidates |
| `readsNegativeWordsAsSignedValues` | signed little-endian conversion, including -1 g and -180° |
| `rejectsFramesThatAreNotDataFrames` | wrong length, header or flag is rejected |
| `parserCountsSequencesAcrossChunks` | seq and per-chunk timestamps |
| `parserReassemblesAFrameSplitAcrossNotifications` | a frame split across two notifications |
| `parserDeliversTwoFramesFromOneNotification` | two frames in one notification |
| `parserResynchronisesAfterGarbage` | garbage prefix is discarded |
| `parserDropsOnlyOneByteForADoubleHeader` | `55 55 61 ...` keeps the real header |
| `parserHandsRegisterRepliesOutAsTheyArrive` | mixed sample/register/sample ordering |
| `readsEightRegistersFromTheEchoedBase` | eight-register window and out-of-range nulls |
| `readsBatteryFromItsOwnReplyRange` | battery reply based at `0x64` |
| `batteryPercentFollowsTheVoltageBucket` | clamp, interpolation and truncation |
| `zeroAtRestKeepsAnUprightSampleUntouched` | an upright zero is the identity for angles and g |
| `zeroSubtractsTheBracketTiltAsAPlainOffset` | angles are offsets, not rotations |
| `zeroRefusesSamplesThatCannotBeZeroes` | total-g and upright thresholds, refusal messages |
| `mountingSwitchesFlipTheirOwnPairOnly` | switch signs; yaw, vertical g and yaw rate untouched |
| `wrapsAnglesIntoTheSignedRange` | `wrap180` edge cases, including NaN and 180/360 |
| `bearingIsTheCompassReadingOfTheYaw` | 0…360 bearing conversion |
| `setupBlockUnlocksConfiguresAndSavesInThatOrder` | setup block order and command size |
| `commandsAreTheDocumentedBytes` | exact command bytes, including register low/high order |
| `acceptsBothWitMotionBases` | `…9a34fb` and `…9b34fb` short matching |
| `keepsTheFallbackServiceExact` | fallback UUIDs are not short-matched |
| `decodesTheGyroIntoRadiansAboutForwardLeftAndUp` | WX/WY/WZ scale, axis mapping and °/s → rad/s |
| `theSwitchesFlipTheRatesWithTheirAnglesAndLeaveTheYawRate` | mounting signs for roll/pitch rate, yaw rate untouched |
| `theZeroLevelsTheRatesWithTheSameRotationAsTheGForces` | a turn about tilted up becomes a pure yaw rate |

The three angular-rate tests were added in 0.2.0; 0.1.0 had 24 tests and did not decode the gyro.

### 3.2 What is not tested

- `Wt9011Link` and the GATT callback paths (no Android instrumentation tests).
- Sensor pinning, rejection of non-IMU WT devices, and `forgetSensor()`.
- Motion-listener registration, delivery and exception isolation.
- The Compose diagnostics screen and the Ride pill.
- Properties persistence and corruption recovery.
- The setup sequencing timers and register retry timing.

If you change `Wt9011Link`, test on a device. The link is intentionally thin: most logic is pushed
into the pure layer precisely so the JVM tests can cover it.

## 4. Working on the decoder

### 4.1 Adding a decoded field

1. Add the field to `Wt9011Raw` (raw sensor channel) and/or `Packet` (ride model).
2. Decode it in `decodeSample` with the right offset and scale from
   [`PROTOCOL.md`](PROTOCOL.md).
3. If it is directional, decide whether `Packet.oriented` should flip it. Do not flip yaw, vertical
   g or yaw rate; they describe the world.
4. If it belongs in the ride packet, update the diagnostics Ride section and the pill if relevant.
5. If it must reach MOTO-HUB, add it to `ModuleMotionSample` in `publish()` and confirm the host
   contract supports it. Contract 43 carries lean, pitch, three g axes, three rates and heading;
   a new field needs a newer contract.
6. Add a test that uses a non-trivial value and at least one negative value.
7. Update [`PROTOCOL.md`](PROTOCOL.md) and the README table if the rider can see it.

There is no recorder header to update any more. Do not add module-side trip files: contract 43 is
the recording path.

### 4.2 Adding a register

1. Add a `REG_*` constant to `Wt9011Protocol`.
2. If it is not already polled, add a read in `Wt9011Link` (or in the diagnostics poll for a
   low-rate value).
3. Handle the value in `onRegisterReply`; remember that a reply covers eight registers starting at
   its echoed base, so `reply.short(reg)` may legitimately return null.
4. Add a test that builds a `RegisterReply` with that base and checks the value and an
   out-of-range null.

### 4.3 Changing the axis mapping

Do not change `LEAN_SIGN` to fix a swapped lean/nose field. The mapping is one boolean:

- If T2 and T3 are swapped, set `LEAN_IS_PITCH` to the other candidate. The gyro axes move with it.
- If lean is the right field but the sign is wrong, change `LEAN_SIGN`.
- If nose is the right field but the sign is wrong, change `PITCH_SIGN`.
- If heading runs backwards, change `YAW_SIGN`; if it is consistently 180° out, change
  `YAW_OFFSET_DEG`.
- If vertical g is inverted, change `ACC_SIGN`; this also flips forward/lateral g, so re-check T5.
- If a left turn reads a negative yaw rate, change `GYRO_SIGN`; this flips all three rates, so
  re-check roll and pitch rates with T6 and a bench turn.

All of these are compile-time constants, so a single rebuild and one pass of T1-T6 verifies the
choice. The rider-facing corrections (zero and mounting) stay runtime settings.

### 4.4 Parser changes

The parser is the most safety-critical pure code because the protocol has no checksum, length or
sequence number. Preserve these invariants:

- Drop **one** byte when a candidate header does not lead to a known frame, then retry.
- Dispatch on the flag after the header, not on the header alone.
- Wait for 20 bytes before consuming a frame; a notification may contain half a frame or two frames.
- Treat unknown flags as noise; do not guess a length.
- Keep `seq` parser-side and increment it only for decoded samples.
- Stamp every frame completed from a chunk with the same `now`.
- Call `reset()` when notifications are first enabled; `onSubscribed()` already does.

## 5. Working on the link

### 5.1 Threading rules

- Everything public on `Wt9011Link` is expected to be called on the main looper, except
  `addMotionListener`/`removeMotionListener` (any thread) and `release` (module lifecycle).
- The parser runs on the GATT callback thread. Do not feed it from anywhere else.
- `publish()` also runs on the GATT callback thread. A listener must return quickly; a slow one
  delays BLE callbacks and can drop frames.
- UI-visible state is `@Volatile`. If you add a new field read by `snapshot()` or the screen, mark
  it `@Volatile` too, or confine it to one thread.
- Guard `recent` with `synchronized(recent)`.
- Motion listeners are a `CopyOnWriteArrayList`; keep the per-listener `try/catch` so a host bug
  cannot kill the Bluetooth callback.
- Do not call BLE operations from the GATT callback thread. Post to `main` first.

### 5.2 Timers

All timers are main-looper `postDelayed` runnables. If you add one, make sure `shutDown()` removes
it; otherwise a stale timeout can fire after a reconnect. The current teardown removes the watchdog,
diagnostics poll, connect timeout and scan timeout. The parser is reset on subscription, not in
teardown, because the next bytes cannot arrive before notifications are enabled.

The scan timeout retries forever while `autoConnect` is true: every 2 s until three attempts have
been made, then every 30 s (`IDLE_RETRY_MS`). `retryLater()` still has a 2 s rate limit; see the
review notes.

### 5.3 Setup sequencing

`writeSequence` sends one command every 150 ms and stops early if the link goes down or the stack
refuses a write. It reports a `complete` boolean to its callback. `runSetupBlock` sets
`configured = true` **only** when the whole block went out; an incomplete block is logged as such
and runs again on the next connection. This is the 0.2.0 fix for the 0.1.0 behaviour.

The device's configuration window is ten seconds after the unlock, so keep the whole block under
that budget. If you add a command, account for the extra 150 ms.

`configureSensor()` sends the same block by hand and re-reads the output content. It does not clear
`configured` when it fails while already true: the previous flash configuration is still there.

### 5.4 Motion publication

`onSample()` calls `publish(rideOf(packet), nowNanos)` only when at least one listener is
registered. `rideOf` applies the zero and then the mounting switches; `publish` converts the result
to `ModuleMotionSample`:

```kotlin
ModuleMotionSample(
    elapsedRealtimeNanos = nowNanos,
    leanDegrees = ride.leanDeg,
    pitchDegrees = ride.pitchDeg,
    forwardG = ride.longG,
    leftG = -ride.latG,          // packet is right-positive
    upG = ride.vertG,
    rollRateRadiansPerSecond = ride.rollRate,
    pitchRateRadiansPerSecond = ride.pitchRate,
    yawRateRadiansPerSecond = ride.yawRate,
    magneticHeadingDegrees = bearingDeg(ride.yawDeg),
    zeroed = zero != null
)
```

Keep the contract axes right-handed (x forward, y left, z up) and keep the pitch-rate sign
convention (+ = nose down). The host does the physics; the module only reports what the bike
measures. Samples are delivered at 20 Hz on the GATT callback thread; do not add a queue without
considering the timestamp and back-pressure semantics.

## 6. Working on the UI

- The screen reads `link.snapshot()` every 100 ms; it does not collect a flow. If you add a field,
  add it to `Wt9011Snapshot`, fill it in both `snapshot()` and `emptySnapshot()` if relevant, and
  draw it.
- The pill is updated by a 500 ms ticker in the module, not by the screen, so it keeps working when
  the screen is closed. It hides after 1 s without frames; the link watchdog tears down at 2 s.
- **Forget this sensor** is shown only when `pinnedAddress` is non-null. Keep it that way: before the
  first connection there is nothing to forget.
- **Reconnect** calls `link.reconnect()` when connected and `link.connect()` otherwise. Do not call
  `connect()` while connected; it is deliberately a no-op.
- Keep rider-facing values non-technical (`Fact.technical = false`) and raw/counter values technical.
- Use `Locale.ROOT` for every `String.format`.
- Do not block the main thread on BLE or file I/O in a composable.

## 7. Debugging checklist

| Symptom | First check | Likely cause |
| --- | --- | --- |
| Nothing happens on a fresh install | Connect button | no sensor pinned yet; auto-connect waits for the first manual Connect |
| "This phone reports no Bluetooth adapter." | Link status | emulator/no BLE hardware |
| "Bluetooth permission is not granted to MOTO-HUB." | App permissions | `BLUETOOTH_SCAN`/`BLUETOOTH_CONNECT` missing |
| "Bluetooth is off." | system | adapter disabled |
| "No WT sensor answered." | sensor power, name | sensor off, out of range, name not starting `WT` |
| "Sensor not in range. Looking again every 30 s." | after 3 scans | the module keeps retrying; bring the pinned sensor back or tap Forget |
| Connects to the wrong WT device | pinned address / recent log | pin was taken in a crowd; tap Forget this sensor and reconnect |
| "That device has no WT sensor service" | sensor model | another WitMotion device, or missing `ffe5`/fallback |
| "The WT service is missing its data or command characteristic." | firmware/unit | unexpected service layout |
| "Could not subscribe to the data characteristic." | GATT/CCCD | descriptor missing or stack refusal |
| Connected but "No data for N s." | frames/s, last frame | sensor not streaming, output content wrong, watchdog teardown |
| Readings frozen but link says connected | last-frame age | watchdog has not fired yet; wait 2 s or check GATT |
| Lean/nose swapped | raw Roll/Pitch | change `LEAN_IS_PITCH`, not `LEAN_SIGN` |
| Lean sign wrong | T2 | `LEAN_SIGN` |
| Nose sign wrong | T3 | `PITCH_SIGN` |
| Heading backwards | T4 | `YAW_SIGN` |
| Heading 180° out | bracket direction | `YAW_OFFSET_DEG = 180f`, or remount |
| Vertical g ≈ -1 | T1 | `ACC_SIGN = -1f` |
| Left turn reads negative yaw rate | T6 | `GYRO_SIGN = -1f` |
| Zero refused: "gravity reads ..." | total g | bike moving or sensor not at rest |
| Zero refused: "Sensor is not upright" | vertical g | bracket not Z-up |
| Warning: no axis reads gravity | output content | sensor in displacement mode; Send set-up |
| Battery/temperature missing | recent log | register read timed out; wait for the 30 s poll or reconnect |
| Trip records nothing from the bike | host capability / listener | contract below 43, or the host never registered a listener |
| Ride Dashboard still shows phone sensors | motion listener | module connected but not streaming; check frames/s |
| Replaced sensor inherits old zero | zero state | known finding: `forgetSensor()` does not clear the zero; tap Clear zero and re-zero |

The host log receives every `note()` as `wt9011: ...`, so a full diagnostics report can be one file:
the screen's "Recent" list plus the host log.

## 8. Review findings and suggested fixes

The full table with severities and effects is in
[`ARCHITECTURE.md` §15](ARCHITECTURE.md#15-review-findings). The remaining items, in priority order:

1. Make the zero's upright check require a positive vertical g, not `abs()`.
2. Require a fresh, connected sample before capturing a zero.
3. Decide whether `forgetSensor()` should clear the zero and the "zeroed" flag for a replacement
   unit.
4. Clear `displacementSuspected` when gravity returns or after a verified repair.
5. Make the log clock thread-safe.
6. Serialise or key the pending register reads.
7. Fix the `retryLater()` rate limit so a fast establish/drop cannot leave auto-connect idle.

Resolved in 0.2.0: the recorder header/newline and pre-correction recording findings are gone with
the recorder itself; the parser is reset on subscription; `configured` is only set for a complete
setup block; Reconnect now tears the link down; recording I/O is no longer on the frame path.

None of the remaining items changes the wire protocol. Item 1 is a one-line comparison; item 2 needs
a freshness check; item 3 is a product decision; items 5-7 are small concurrency/timer fixes.

## 9. Release checklist

- [ ] Bump `version` in `build.gradle.kts` **and** `Wt9011Module.manifest`; they must match.
- [ ] Bump `contractVersion` only if the module starts using a newer host API.
- [ ] Run `testDebugUnitTest` (27 tests).
- [ ] Run `checkModuleLinkage` against the target app build.
- [ ] Package with the module key directory; the artifact is `wt9011-0.2.0.mhm`.
- [ ] Install the `.mhm` on a device with developer mode and a contract-43 host.
- [ ] Smoke test: first Connect pins the sensor, 20 Hz frames, setup runs once, Forget/Connect
      selects a new unit, zero, both mounting switches, heading, battery/temperature, Disconnect,
      Reconnect, watchdog teardown, out-of-range 30 s retry.
- [ ] Confirm the host records a trip from the module and the Ride Dashboard shows its lean.
- [ ] Run the six bench tests T1-T6 if any axis or gyro constant changed.
- [ ] Update `README.md`, [`ARCHITECTURE.md`](ARCHITECTURE.md) and [`PROTOCOL.md`](PROTOCOL.md) for
      any behavioural or protocol change.

## 10. Conventions

- **License headers.** Every Kotlin file starts with the AGPL-3.0-only SPDX line and the MOTO-HUB
  and sur00t copyright lines. Keep them.
- **Pure logic stays Android-free.** `Wt9011Packet.kt` and `Packet.kt` must remain importable on a
  plain JVM. This is what makes the decoder and correction math testable.
- **The link owns BLE; the module owns UI.** Do not put Compose or host UI calls in the link, and do
  not put GATT calls in the composable.
- **The host owns recording.** Publish through `ModuleMotionSource`; do not add a module-side trip
  file or CSV recorder.
- **Constants over literals.** Timings and limits live in the companion objects; protocol values
  live in `Wt9011Protocol`; empirical signs live in `Wt9011Axes`.
- **Explain the "why".** The existing comments document vendor quirks, empirical choices and
  failure modes. Preserve that style; a comment that only restates the code is noise.
- **Use `Locale.ROOT`** for every formatted number or date that could be read as a diagnostic.
- **Do not assume the sensor accepted a write.** Read back and report; the UI must not claim a
  configuration took effect when it did not.
- **No silent failure.** Permission, adapter, service, subscription, register, listener and storage
  failures all become a status line, a log event, or both.
