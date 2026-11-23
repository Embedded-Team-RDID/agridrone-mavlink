[![Build Status](https://github.com/mavlink/mavlink/workflows/Test%20and%20deploy/badge.svg)](https://github.com/mavlink/mavlink/actions?query=branch%3Amaster)

# MAVLink

MAVLink -- Micro Air Vehicle Message Marshalling Library.

MAVLink is a very lightweight, header-only message library for communication between drones and/or ground control stations. It consists primarily of message-set specifications for different systems ("dialects") defined in XML files, and Python tools that convert these into appropriate source code for [supported languages](https://mavlink.io/en/#supported_languages). There are additional Python scripts providing examples and utilities for working with MAVLink data.

> **Tip** MAVLink is very well suited for applications with very limited communication bandwidth. Its reference implementation in C is highly optimized for resource-constrained systems with limited RAM and flash memory. It is field-proven and deployed in many products where it serves as interoperability interface between components of different manufacturers.


## Quick start

### Generate C headers

To install the minimal MAVLink environment on Ubuntu LTS 20.04, 22.04, or 24.04, enter the following on a terminal:

```bash
# Dependencies
sudo apt install python3-pip

# Clone mavlink into the directory of your choice
git clone https://github.com/mavlink/mavlink.git --recursive
cd mavlink

python3 -m pip install -r pymavlink/requirements.txt
```

You can then build the MAVLink2 C-library for `message_definitions/v1.0/common.xml` from the `/mavlink` directory as shown:

```bash
python3 -m pymavlink.tools.mavgen --lang=C --wire-protocol=2.0 --output=generated/include/mavlink/v2.0 message_definitions/v1.0/common.xml
```

### Use from cmake

To include the headers in cmake, install them locally, e.g. into the directory `install`:

```
cmake -Bbuild -H. -DCMAKE_INSTALL_PREFIX=install -DMAVLINK_DIALECT=common -DMAVLINK_VERSION=2.0
cmake --build build --target install
```

Then use `find_package` to get the dependency in `CMakeLists.txt`:

```
find_package(MAVLink REQUIRED)

add_executable(my_program my_program.c)

target_link_libraries(my_program PRIVATE MAVLink::mavlink)
```

And pass the local install directory to cmake (adapt to your directory structure):

```
cd ../my_program
cmake -Bbuild -H. -DCMAKE_PREFIX_PATH=../mavlink/install
```

For a full example, check [examples/c](examples/c).

*Note: even though we use `target_link_libraries` in cmake, it doesn't actually "link" to MAVLink as it's just a header-only library.*

### Other instructions

Instructions for using the C libraries are then covered in [Using C MAVLink Libraries (mavgen)](https://mavlink.io/en/mavgen_c/).

> **Note:** [Installing the MAVLink Toolchain](https://mavlink.io/en/getting_started/installation.html) explains how to install MAVLink on other Ubuntu platforms and Windows, while [Generating MAVLink Libraries](https://mavlink.io/en/getting_started/generate_libraries.html) explains how to build MAVLink for the other programming languages [supported by the project](https://mavlink.io/en/#supported_languages).
> The sub-topics of [Using MAVLink Libraries](https://mavlink.io/en/getting_started/use_libraries.html) explain how to use the generated libraries.


## THACO camera stream mode

`CC_STREAM_MODE` is a `PARAM_EXT` UINT8 parameter owned by Companion component 191.
The `CC_STREAM_MODE` enum defines 0 Off, 1 Video, 2 Vision and 3 Depth. Existing
values and all message layouts/CRCs are unchanged. QGC presents Video / Off /
Vision / Depth. Depth publishes a colourized camera depth image on the same
RTSP `/camera` path, using the Video encoder settings and one encoder at a time;
it does not enable obstacle avoidance or Vision inference. Invalid depth is black.
The camera confirms Depth only after a valid depth frame and publisher readiness;
the normal PARAM_EXT apply/readback/save flow still applies. Upgrade Companion
and QGC together before selecting value 3; older Companion schemas reject it.

## THACO RGB bbox following control

`CC_AI_VISION_CONTROL` (42015) retains its three flags and legacy wire slots.
RGB follow uses `gain_d`, `gain_t`, `gain_vel` and `vel_max`:

- `gain_d` and `gain_t`: finite 0..1, defaults 0.5.
- Desired person bbox height / displayed image width: `0.9 / (1 + 3.5 * gain_d)`.
- Relative symmetric tolerance: `0.25 * (1 - gain_t)`; 1 removes the deadband.
- Outside that band, subtract the nearest band edge and divide by the desired ratio.
  Forward/backward velocity is `clamp(gain_vel * error, -vel_max, +vel_max)`.
- `gain_vel` is the direct proportional coefficient in `0..vel_max`, default 0.5.
  `vel_max` is a finite nonnegative speed in m/s, default 2; 3 means 3 m/s.
  Zero coefficient or zero cap disables horizontal translation.
- `bbox_gains_valid=2` applies all four controls atomically, including explicit zeros;
  `0` preserves them. Legacy presence 1 accepts only the old 0..1 triplet, preserves
  the current cap and validates the resulting tuple. Other values preserve controls.
  Invalid numeric controls must never block valid OFF flags.
- Existing numeric values are retained on upgrade; an absent cap defaults to 2.
  The coefficient now has direct P semantics. Update Companion and QGC together.

The retired metric controls and velocity/acceleration feedforward gains have been
removed. Five reserved slots keep their original byte offsets; send zero in them.
They are not configuration parameters and have no runtime control meaning.
Use matching Companion/QGC versions for RGB following.
Offsets: gain_d 14, gain_t 18, gain_vel 22, bbox_gains_valid 26, vel_max 27.
Minimum payload 3, maximum 31 bytes; CRC_EXTRA 211 remains unchanged.
These gains neither request flight mode nor bypass FC admission or target identity checks.

Generate each consumer into its own build directory using the pinned generator;
never edit generated headers. Companion uses the superproject sync helper;
QGC resolves latest main during configure as documented in its custom/README.md.

## Key Links

* [Documentation/Website](https://mavlink.io/en/) (mavlink.io/en/)
* [Discussion/Support](https://mavlink.io/en/#support)
* [Contributing](https://mavlink.io/en/contributing/contributing.html)
* [License](https://mavlink.io/en/#license)

## CC_OA_CONTROL (42018)

Source CC 1/195, GCS 254/190 or 255/190. Payload 199 bytes; MAVLink2.
The enable switch is the first operator control; MAVLink sorts base fields by type.
QGC sends changes once and waits for the echoed request ID; telemetry never triggers writes.
Fresh FC heartbeat gates enabling/tuning; OFF remains admissible. CC persists accepted
OA tuning atomically before success, never the enable flag. Firmware parameters remain FC-owned.
Invalid tuning preserves the old tuple; disabling is still honored.

| Index | oa_values | Range | 2 m/s default |
|---|---|---|---|
| 0 | `speed_m_s` | 0.1..6 | 2.0 |
| 1 | `speed_lead_m_s` | 0.3..6 | 1.0 |
| 2 | `acceleration_m_s2` | 1..15 | 6.0 |
| 3 | `jerk_m_s3` | 1..160 | 40.0 |
| 4 | `body_radius_m` | 0.3..1 | 0.4 |
| 5 | `clearance_m` | 0.7..2 | 0.7 |
| 6 | `tracking_margin_m` | 0..1 | 0.4 |
| 7 | `horizon_m` | 2..19 | 2.5 |
| 8 | `history_frames` | 1..30 | 30 |
| 9 | `obstacle_memory_s` | 0.5..10 | 5.0 |
| 10 | `obstacle_voxel_m` | 0.05..0.3 | 0.1 |
| 11 | `sensor_range_m` | 3..20 | 10.0 |
| 12 | `heading_limit_deg` | 10..85 | 70.0 |
| 13 | `heading_step_deg` | 1..10 | 5.0 |
| 14 | `path_step_m` | 0.05..0.3 | 0.15 |
| 15 | `turn_penalty_m` | 0..5 | 0.6 |
| 16 | `planning_time_s` | 1..4 | 3.0 |
| 17 | `braking_tail_required` | 0..1 | True |
| 18 | `arrival_filter_time_constants` | 2..6 | 6.0 |
| 19 | `reaction_time_s` | 0.15..1 | 0.2 |
| 20 | `tracking_response_s` | 0.1..1 | 0.25 |
| 21 | `minimum_speed_ratio` | 0.3..1 | 1.0 |
| 22 | `time_cost_m_s` | 0..3 | 0.5 |
| 23 | `route_resolution_m` | 0.2..0.5 | 0.25 |
| 24 | `route_lookahead_m` | 0.5..6 | 3.0 |
| 25 | `route_heading_limit_deg` | 5..85 | 25.0 |
| 26 | `route_clearance_weight` | 0..5 | 1.0 |
| 27 | `tracking_recovery_s` | 0.5..3 | 1.5 |
| 28 | `setpoint_lookahead_s` | 0.25..0.75 | 0.3 |
| 29 | `yaw_rate_rad_s` | 0.1..1.5 | 1.0 |
| 30 | `yaw_follow_path` | 0..1 | True |
| 31 | `mission_return_alignment_deg` | 5..45 | 5.0 |
| 32 | `mission_clear_time_s` | 0.5..3 | 1.0 |
| 33 | `mission_return_min_speed_ratio` | 0..1 | 0.9 |

Hardware yaw_follow_path defaults to 1; SITL defaults to 0. Geometry, calibration,
clock and transport are startup-only. Cross-field checks also require body + gap +
reserve ≤3 m, horizon ≤sensor range−1 m and preferred angle ≤search angle.

| Index | fc_values (FC readback only) | SITL 2 m/s preset |
|---|---|---|
| 0 | `SYS_VEHICLE_RESP` | -0.4 |
| 1 | `MPC_XY_VEL_ALL` | -10.0 |
| 2 | `MPC_XY_VEL_MAX` | 2.0 |
| 3 | `MPC_XY_CRUISE` | 2.0 |
| 4 | `MPC_XY_VEL_P_ACC` | 3.0 |
| 5 | `MPC_ACC_HOR_MAX` | 6.0 |
| 6 | `MPC_ACC_HOR` | 6.0 |
| 7 | `MPC_JERK_AUTO` | 20.0 |
| 8 | `MPC_JERK_MAX` | 40.0 |
| 9 | `MPC_TILTMAX_AIR` | 40.0 |
| 10 | `MPC_YAWRAUTO_MAX` | 60.0 |
| 11 | `NAV_ACC_RAD` | 0.4 |
