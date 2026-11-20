# WT9011DCL protocol reference

This document describes the subset of the WitMotion WT9011DCL protocol that the
`io.github.sur00t.wt9011` module actually uses. It mirrors the constants and decoding logic in
`src/main/java/io/github/sur00t/wt9011/Wt9011Packet.kt`; where the two disagree, the code is
authoritative.

The module treats the sensor as a BLE peripheral that streams as soon as notifications are enabled.
There is no request/response handshake for the stream, no checksum, no length field and no sequence
number. Register reads and their replies share the notification characteristic with the stream.

For the module-level architecture, see [`ARCHITECTURE.md`](ARCHITECTURE.md). For build and test
instructions, see [`DEVELOPMENT.md`](DEVELOPMENT.md).

## 1. GATT profile

### 1.1 UUIDs

The sensor advertises the WitMotion `ffe5`/`ffe4`/`ffe9` trio on a non-standard Bluetooth base UUID:

| Role | UUID | Properties used |
| --- | --- | --- |
| Service | `0000ffe5-0000-1000-8000-00805f9a34fb` | primary service |
| Data | `0000ffe4-0000-1000-8000-00805f9a34fb` | notifications |
| Command | `0000ffe9-0000-1000-8000-00805f9a34fb` | write with response |
| CCCD | `00002902-0000-1000-8000-00805f9b34fb` | enable notifications on the data characteristic |

Some units advertise the same 16-bit UUIDs on the standard base ending
`-00805f9b34fb` instead. The module therefore compares 16-bit UUIDs by
their short form (the first eight hex characters, for example `0000ffe5`), not by the full 128-bit
value. See `Wt9011Protocol.matchesShort`.

A second, unrelated WitMotion service is accepted as a fallback:

| Role | UUID |
| --- | --- |
| Service | `49535343-fe7d-4ae5-8fa9-9fafd205e455` |
| Data | `49535343-1e4d-4bd9-ba61-23c647249616` |
| Command | `49535343-8841-43f4-a8d4-ecbe34729bb3` |

These three UUIDs must be matched **exactly**. The data and command characteristics both begin with
`49535343`, so a short-form comparison would confuse them. `matchesShort` deliberately refuses to
shorten any UUID that does not start with `0000`.

### 1.2 Discovery rules

The module scans without a service filter. Before a sensor has been remembered, it accepts a device
whose advertised name starts with `WT` (`Wt9011Protocol.NAME_PREFIX`). After the first successful
connection it **pins the device address** and accepts only that address; the name is no longer
required, and another WT device is ignored.

After connecting it requires either the `ffe5` service or the fallback service; a device with
neither is rejected with `"That device has no WT sensor service - it is not the IMU."` and added to
the scan's rejected set. This two-stage check matters on a track with several WitMotion devices: a
name match alone is not enough, but filtering the scan by service UUID would hide a unit whose
advertisement is minimal.

When the documented service is present, both characteristics are looked up **inside that service**
by short UUID. When only the fallback service is present, both fallback characteristics are looked
up by exact UUID inside that service. The data and command characteristics are never mixed between
the two services.

## 2. Frame format

All payload values are signed 16-bit little-endian integers; the register base is an unsigned 16-bit
little-endian value. Both frame types are exactly 20 bytes long (`Wt9011Protocol.FRAME_SIZE`).

### 2.1 Data frame (`0x55 0x61`)

| Offset | Size | Field | Scale | Notes |
| --- | --- | --- | --- | --- |
| 0 | 1 | Header | - | always `0x55` |
| 1 | 1 | Flag | - | always `0x61` for a data frame |
| 2-3 | 2 | AX | raw × 16 / 32768 g | accelerometer X |
| 4-5 | 2 | AY | raw × 16 / 32768 g | accelerometer Y |
| 6-7 | 2 | AZ | raw × 16 / 32768 g | accelerometer Z |
| 8-9 | 2 | WX | raw × 2000 / 32768 °/s | gyro X |
| 10-11 | 2 | WY | raw × 2000 / 32768 °/s | gyro Y |
| 12-13 | 2 | WZ | raw × 2000 / 32768 °/s | gyro Z |
| 14-15 | 2 | Roll | raw × 180 / 32768 ° | vendor angle field |
| 16-17 | 2 | Pitch | raw × 180 / 32768 ° | vendor angle field |
| 18-19 | 2 | Yaw | raw × 180 / 32768 ° | vendor angle field |

The acceleration range is ±16 g, the gyro range is ±2000 °/s, and the angle range is ±180° over
32768 counts, so the scale factors are `16 / 32768`, `2000 / 32768` and `180 / 32768`
(`ACCEL_SCALE`, `GYRO_SCALE`, `ANGLE_SCALE`).

`Wt9011Raw` keeps all nine decoded fields (`ax`, `ay`, `az`, `wx`, `wy`, `wz`, `rollDeg`,
`pitchDeg`, `yawDeg`). The rate fields stay in degrees per second; `decodeSample` converts them to
radians per second for `Packet`. `Wt9011Sample` pairs the raw view with the ride-model `Packet`.

### 2.2 Register reply (`0x55 0x71`)

| Offset | Size | Field | Notes |
| --- | --- | --- | --- |
| 0 | 1 | Header | always `0x55` |
| 1 | 1 | Flag | always `0x71` |
| 2-3 | 2 | Base register | little-endian; the register named in the read request |
| 4-19 | 16 | Payload | eight signed 16-bit little-endian registers, `base` through `base + 7` |

`RegisterReply` exposes the echoed base, the number of registers (`payload.size / 2`, always 8 for a
well-formed frame), and `short(register)`, which returns `null` when the requested register is
outside the reply's window. The reply to a read of `0x64` therefore carries `0x64` through `0x6B`;
the reply to a read of `0x40` carries `0x40` through `0x47` and cannot answer for the battery.

### 2.3 No integrity checking

There is no checksum, no length byte and no sequence number. A corrupted frame cannot be detected as
such. The parser can only:

- resynchronise on the `0x55` header,
- check the flag after the header,
- drop exactly one byte when a header does not lead to a known frame,
- stamp each completed frame with the phone's arrival time.

The module's `seq` counter exists only on the phone side. It increments for decoded data frames and
wraps at `0x7FFFFFFF`. Register replies do not consume a sequence number. `seq` continues across
reconnects; it is a session counter, not a per-link one.

## 3. Register map

| Register | Name | Used by the module | Meaning |
| --- | --- | --- | --- |
| `0x2E` | `REG_VERSION1` | no | firmware version (defined for reference) |
| `0x2F` | `REG_VERSION2` | no | firmware version (defined for reference) |
| `0x3A` | `REG_HX` | no | magnetic field X (defined for reference) |
| `0x3D` | `REG_ROLL` | no | roll angle (the stream carries it) |
| `0x3E` | `REG_PITCH` | no | pitch angle (the stream carries it) |
| `0x3F` | `REG_YAW` | no | yaw angle (the stream carries it) |
| `0x40` | `REG_TEMP` | yes | chip temperature, raw / 100 °C |
| `0x64` | `REG_BATTERY` | yes | supply voltage, raw / 100 V |
| `0x96` | `REG_AGPVSEL` | yes | output-content selection |

A register read is the five-byte command `FF AA 27 <low> <high>`, for example
`FF AA 27 64 00` for the battery register. The device answers with a `0x71` frame.

Reads are retried up to three times, 200 ms apart (`REGISTER_TRIES`, `REGISTER_TIMEOUT_MS`). If no
reply covers the register within that window, the link logs
`"no reply for register 0xNN"` and gives up. Replies are matched by the echoed register range, not
by arrival order.

### 3.1 Temperature

`temperatureC = raw / 100f`, in degrees Celsius. This is the sensor's chip temperature, not air
temperature.

### 3.2 Battery

The battery register carries the sensor's supply voltage in hundredths of a volt. The vendor
documents buckets (above 413 → 100 %, 393–397 → 80 %, below 342 → 0 %) while the vendor's Android
app interpolates between 3.40 V and 3.99 V. This module uses the linear interpolation:

```
volts   = raw / 100
percent = (volts - 3.40) / (3.99 - 3.40) * 100
percent = percent.toInt().coerceIn(0, 100)
```

The result is truncated toward zero, not rounded, so 3.69 V reads 49 %. This is the sensor's own
supply, not the motorcycle battery.

## 4. Command set

Commands are always five bytes beginning `FF AA`. They are written to the command characteristic
with **write with response** (`WRITE_TYPE_DEFAULT`). The device needs a moment between settings
writes and only accepts settings for ten seconds after the unlock command.

| Purpose | Bytes | Notes |
| --- | --- | --- |
| Unlock settings | `FF AA 69 88 B5` | mandatory before any settings write; valid for 10 s |
| Output content | `FF AA 96 00 00` | acceleration + angular rate + angle |
| Output rate 20 Hz | `FF AA 03 07 00` | used by the setup block |
| Output rate 10 Hz | `FF AA 03 06 00` | device default; defined for reference/tests |
| Bandwidth 42 Hz | `FF AA 1F 03 00` | at least twice the 20 Hz output rate |
| 9-axis algorithm | `FF AA 24 00 00` | the only mode with an absolute heading |
| Horizontal install | `FF AA 23 00 00` | Z-up; the vertical install mode is never used |
| Save to flash | `FF AA 00 00 00` | persists the settings |
| Read register | `FF AA 27 <reg low> <reg high>` | answered by a `0x71` frame |

### 4.1 The setup block

`Wt9011Protocol.setup()` returns the block in this exact order:

```
FF AA 69 88 B5   unlock
FF AA 96 00 00   output = acceleration + angular rate + angle
FF AA 03 07 00   20 Hz
FF AA 1F 03 00   42 Hz bandwidth
FF AA 24 00 00   9-axis algorithm
FF AA 23 00 00   horizontal install
FF AA 00 00 00   save
```

Writes are sent one at a time with a 150 ms gap (`WRITE_GAP_MS`). The whole block fits inside the
sensor's ten-second configuration window after the unlock.

The settings are flash-backed on the device, so the block is a **one-time installation step**, not a
per-connection step. The link stores a `configured` flag in `setup.properties`; on later connections
it only reads the output-content register (`0x96`) and repairs that register if necessary. The flag
is set only when every write in the block succeeded; an interrupted block runs again on the next
connection.

After the block the module reads `0x96` to verify the output content. If the register still reports
something other than acceleration + gyro + angle, the output-content fact shows the value and
displacement mode raises the screen warning. The module never assumes the configuration took effect.

## 5. Parser and resynchronisation

`Wt9011FrameParser` is the only stateful part of the wire layer. It is fed one notification's bytes
at a time and returns whole frames. It is **not thread safe**; the link calls it only from the GATT
callback thread. `reset()` clears the buffer and is called from `onSubscribed()`, when notifications
have just been enabled and the next byte cannot be part of a frame from before the subscription.

Algorithm:

```text
append chunk to buffer
loop forever:
    drop leading bytes until buffer[0] == 0x55 (or the buffer is empty)
    if buffer has fewer than 2 bytes: stop and wait for more
    flag = buffer[1]
    length = 20 if flag is 0x61 or 0x71, otherwise 0
    if length == 0:
        drop one byte (the 0x55) and continue
    if buffer has fewer than 20 bytes: stop and wait for more
    take exactly 20 bytes out of the buffer
    if flag == 0x61: decode a data sample
    if flag == 0x71: decode a register reply
```

Two details are deliberate and are called out in the code as places where the vendor's own Android
SDK goes wrong:

1. **Drop exactly one byte when resynchronising.** A truncated frame followed by a whole one can
   produce `55 55 61 ...`. Dropping two bytes would swallow the real header. The vendor resolver
   tests two conditions with `&&` where it needs `||` and can leave the buffer permanently out of
   sync.
2. **Dispatch on the flag after the header, not on the header alone.** Register replies travel on
   the same characteristic as the stream. Treating every `0x55` as a data frame would drop `0x71`
   replies as noise.

Unknown flags are treated as noise: drop the `0x55` and resync. This also handles the `55 5F`
11-byte frames of another protocol generation without guessing their length.

`feed(chunk, now)` stamps every sample completed from that chunk with the same `now`. A chunk can
contain two frames, and the phone has no better timestamp for either one than the moment the
notification arrived. The link converts the notification's `elapsedRealtimeNanos()` to milliseconds
for the internal sample timestamp and passes the original nanoseconds to the motion listener.

## 6. Axis and sign model

The datasheet's axis diagram is left-handed, and the vendor's own apps disagree about which way is
forward. The module therefore treats the axis mapping as **empirical** and settles it with six bench
tests. All of the mapping is in `Wt9011Axes` in `Wt9011Packet.kt`.

### 6.1 The two candidate conventions

One boolean, `LEAN_IS_PITCH`, selects the whole triad, including the gyro axes:

| | Candidate B (`LEAN_IS_PITCH = false`, current default) | Candidate A (`LEAN_IS_PITCH = true`) |
| --- | --- | --- |
| Positive X | forward | right |
| Positive Y | left | forward |
| Lean field | Roll (bytes 14-15) | Pitch (bytes 16-17) |
| Nose field | Pitch (bytes 16-17) | Roll (bytes 14-15) |
| Forward g | AX | AY |
| Rightward g | -AY | AX |
| Forward rate (roll) | WX | WY |
| Left rate (pitch) | WY | -WX |
| Up rate (yaw) | WZ | WZ |

The parser applies this mapping in `decodeSample`:

- `leanField` and `noseField` are chosen from Roll/Pitch.
- `forwardAxis` and `rightAxis` are chosen from AX/AY.
- `forwardRate` and `leftRate` are chosen from WX/WY.
- `rightAxis` is negated for candidate B because the packet's lateral g is right-positive while the
  sensor's +Y is left.

The rate vector is in the motorcycle frame: x forward, y left, z up. Its components are radians per
second:

```
rollRate  = forwardRate * GYRO_SIGN * (π / 180)
pitchRate = leftRate    * GYRO_SIGN * (π / 180)
yawRate   = raw.wz      * GYRO_SIGN * (π / 180)
```

### 6.2 Sign constants

| Constant | Default | Effect | Calibration test |
| --- | --- | --- | --- |
| `LEAN_IS_PITCH` | `false` | selects the candidate triad above | T2/T3 |
| `LEAN_SIGN` | `1f` | flips lean left/right | T2 |
| `PITCH_SIGN` | `1f` | flips nose up/down | T3 |
| `YAW_SIGN` | `1f` | flips heading sense | T4 |
| `YAW_OFFSET_DEG` | `0f` | rotates the heading to the bike's nose | T4 |
| `ACC_SIGN` | `1f` | flips all three g channels together | T1/T5 |
| `GYRO_SIGN` | `1f` | flips all three angular rates together | T6 |

`YAW_SIGN` and `YAW_OFFSET_DEG` are applied before `wrap180`:

```
yawDeg = wrap180(raw.yawDeg * YAW_SIGN + YAW_OFFSET_DEG)
```

`YAW_OFFSET_DEG` is a **compile-time constant**. Unlike the zero and the mounting switches, it is
not persisted or editable at runtime; changing it requires rebuilding/reflashing the module.

`GYRO_SIGN` is one switch for all three rates. If T6 shows a left turn reading a negative yaw rate,
flip it; the roll and pitch rates follow the same sensor handedness. Do not flip individual rate
signs to compensate for a swapped axis; change `LEAN_IS_PITCH` instead.

### 6.3 Bench tests

| Test | Procedure | Expected |
| --- | --- | --- |
| T1 | bike upright, stationary | vertical g ≈ +1.000 g |
| T2 | lean the bike right by 15-20° | lean positive, pitch barely moves |
| T3 | lift the front wheel | pitch positive, lean barely moves |
| T4 | point the sensor's forward axis at magnetic north, then 90° right | heading ≈ 0°, then ≈ 90° |
| T5 | upright at rest, then push the bike forward | vertical ≈ +1.0 g; forward push gives positive longitudinal g |
| T6 | on the bench, turn the sensor left by hand (or ride one left turn) | yaw rate positive |

The README's "Calibration checks" section maps each failure to the constant to change. The
important rule is: if T2 and T3 are swapped, change `LEAN_IS_PITCH`, not `LEAN_SIGN`; the two
signs belong to different fields. If T6 is inverted, change `GYRO_SIGN`.

### 6.4 Mounting switches

`Mounting` is the rider's two-switch correction, persisted in `mounting.properties`:

| Switch | Flips | Does not touch |
| --- | --- | --- |
| `flipNose` | pitch, longitudinal g, pitch rate | lean, lateral g, roll rate, yaw, vertical g, yaw rate |
| `flipSide` | lean, lateral g, roll rate | pitch, longitudinal g, pitch rate, yaw, vertical g, yaw rate |

`Packet.oriented` applies the signs after the zero:

```
leanDeg   = leanDeg   * (flipSide ? -1 : +1)
pitchDeg  = pitchDeg  * (flipNose ? -1 : +1)
longG     = longG     * (flipNose ? -1 : +1)
latG      = latG      * (flipSide ? -1 : +1)
rollRate  = rollRate  * (flipSide ? -1 : +1)
pitchRate = pitchRate * (flipNose ? -1 : +1)
```

The heading, vertical g and yaw rate are deliberately left alone: they describe the world, not the
bracket. The yaw rate is about up, which neither switch changes.

## 7. Processing math

### 7.1 Heading

`Packet.yawDeg` keeps the signed range -180…180. `bearingDeg` converts it to the 0…360 reading a
rider expects:

```
wrapped = wrap180(yawDeg)
bearing = if (wrapped < 0) wrapped + 360 else wrapped
```

`wrap180` maps into [-180, 180), returns 0 for non-finite input, maps 180 to -180 and 360 to 0.
This is a **magnetic** bearing; the module never subtracts declination.

### 7.2 Zero capture

`Zero.of(packet, takenAt)` refuses a sample unless:

- total gravity `sqrt(longG² + latG² + vertG²)` is between 0.7 g and 1.3 g, and
- the vertical axis reads at least 0.5 g in magnitude (`abs(vertG) >= 0.5`).

`zeroRefusal` returns a rider-readable reason for either refusal. The current magnitude check also
accepts an upside-down sensor (vertical ≈ -1 g); see the review notes.

The captured zero stores the lean and pitch readings and the three g components, all in the raw
decoded packet's coordinate frame (before the mounting switches).

### 7.3 Applying the zero

The angles are a **plain offset**, not a rotation:

```
leanDeg  = packet.leanDeg  - zero.leanDeg
pitchDeg = packet.pitchDeg - zero.pitchDeg
```

A rotation of the angle pair would cancel the real lean along with the mounting error, because it
would rotate the sideways g towards vertical by the whole measured angle. The rider expectation is
"zero it now" → the screen reads zero now.

The g vector, however, gets a genuine minimum-angle rotation. Let

```
u = (forwardG, -latG, upG) / |(forwardG, -latG, upG)|
```

be the zeroed "up" direction in a right-handed frame (x forward, y left, z up). The module rotates
the current g vector by the rotation that takes `u` to `+Z`, about the axis

```
k = u × z = (uy, -ux, 0)
sin = |k|
cos = uz
```

and applies Rodrigues' formula:

```
v' = v cos + (k̂ × v) sin + k̂ (k̂ · v) (1 - cos)
```

This removes both a roll error and a small base-level error without changing gravity's magnitude. If
the zero vector is already aligned (`sin < 1e-6`) or degenerate (`norm < 1e-3`), only the angle
offsets are applied.

### 7.4 Levelling the angular rate

The gyro sits in the same tilted bracket frame as the accelerometer, so the same rotation that levels
gravity must level the rate vector. The rate is already in the motorcycle frame's component order
(`rollRate` about forward, `pitchRate` about left, `yawRate` about up), so it goes through the same
Rodrigues rotation as the g vector:

```
w  = (rollRate, pitchRate, yawRate)
p  = k̂ = (uy, -ux, 0) / sin
w' = w cos + (p × w) sin + p (p · w) (1 - cos)
```

Expanded in the code:

```
wDot      = px·wx + py·wy
rollRate' = wx·cos + py·wz·sin + px·wDot·(1 - cos)
pitchRate'= wy·cos - px·wz·sin + py·wDot·(1 - cos)
yawRate'  = wz·cos + (px·wy - py·wx)·sin
```

For a pure turn about the tilted up axis, this comes out as a pure turn about true up: the roll and
pitch rates vanish and the yaw rate survives. `theZeroLevelsTheRatesWithTheSameRotationAsTheGForces`
asserts exactly that.

### 7.5 Worked example

A parked bike reads lean 4°, pitch -1°, vertical 0.99 g. Capturing that zero stores
`leanDeg = 4`, `pitchDeg = -1`. A later riding sample reads lean 19°, pitch 2°; applying the zero
gives lean 15°, pitch 3°. This is exactly the behaviour asserted by
`zeroSubtractsTheBracketTiltAsAPlainOffset`.

A sample with raw candidate-B channels `ax = 0.10 g`, `ay = -0.20 g`, `az = 0.97 g` maps to
`longG = +0.10 g`, `latG = +0.20 g`, `vertG = +0.97 g` before any zero or mounting correction.

A raw gyro reading `wx = 10 °/s`, `wy = -20 °/s`, `wz = 30 °/s` maps, with candidate B and
`GYRO_SIGN = 1`, to `rollRate = 10 °/s`, `pitchRate = -20 °/s`, `yawRate = 30 °/s`, all converted
to radians per second. The sign convention follows the right-hand rule about forward, left and up:
positive roll leans right, positive pitch sends the nose down, positive yaw turns left.

## 8. Failure modes and limits

| Condition | Effect | Module response |
| --- | --- | --- |
| No frames for 2 s | link is dead even if GATT still says connected | watchdog tears it down and rescans |
| All three g axes below 0.5 g in magnitude for 2 s | sensor is probably in displacement mode | `displacementSuspected` flag and a screen warning |
| Register read unanswered for 3 × 200 ms | value unavailable | log line, field stays at its last value |
| Output content register != 0 | stream is not acceleration + gyro + angle | one automatic unlock/output/save fix per connection |
| Garbage bytes before a header | parser must resync | drop bytes until `0x55`, then drop one byte at a time |
| Corrupted frame with a valid header/flag | cannot be detected | it is decoded as if valid |
| BLE write larger than 20 bytes | stack rejects/truncates | module only writes 5-byte commands |
| Slow phone / no flow control | frames are dropped by the phone | sensor keeps streaming at its configured rate |
| Slow host motion listener | GATT callback is delayed | frames may be dropped; the module has no queue |
| Magnetic disturbance near the bracket | heading wanders | no software fix; mounting location is the fix |

## 9. Interoperability notes

- **Two base UUIDs.** WitMotion's SDK uses `…00805f9a34fb`, some units use the standard
  `…00805f9b34fb`. The module accepts both by comparing the 16-bit part.
- **Fallback UUIDs are exact.** The `49535343-…` data and command characteristics share their first
  four bytes; short matching would mix them up.
- **Register replies share the stream.** A parser that dispatches on `0x55` alone will treat `0x71`
  replies as noise. This module dispatches on the flag.
- **The vendor's resolver has a resync bug.** Its loop combines two conditions with `&&` where it
  needs `||`; the module uses the Python SDK's accumulate/test/drop-one model instead.
- **No ACK for the stream.** A live sample is proven by a frame arriving, not by an acknowledgement.
  The watchdog is the only liveness signal.
- **The heading is magnetic.** The sensor's 9-axis fusion produces it; the module never derives a
  heading from raw `Hx`/`Hy`, and never applies declination.
- **Angular rates are the sensor's own.** The module does not integrate them; it only converts
  °/s to rad/s, applies `GYRO_SIGN`, levels them with the zero rotation and applies the mounting
  switches. MOTO-HUB consumes them as `rollRateRadiansPerSecond`, `pitchRateRadiansPerSecond` and
  `yawRateRadiansPerSecond`.
