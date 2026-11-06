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


## THACO RGB bbox following control

`CC_AI_VISION_CONTROL` (42015) retains its three flags and legacy wire slots.
RGB follow uses the appended float extensions `gain_d`, `gain_t`, `gain_vel`
(finite 0..1, defaults 0.5) and `uint8_t bbox_gains_valid`:

- Desired person bbox height / displayed image width: `0.9 / (1 + 3.5 * gain_d)`.
- Relative symmetric tolerance: `0.25 + 0.75 * gain_t`.
- Outside that band, subtract the nearest band edge and divide by the desired ratio.
  Forward/backward velocity is `2 m/s * gain_vel * clamp(error, -1, 1)`.
  Positive velocity approaches a smaller bbox; zero gain disables translation.
- `bbox_gains_valid=1` applies all three gains atomically, including zeros;
  `0` preserves them. Invalid presence/gains preserve all gains and must not block OFF.

Legacy distance/threshold (dm), vel_gain/acc_gain and gains_valid retain their
original layout for wire compatibility. Bbox-follow receivers ignore them.
Legacy receivers ignore the appended extensions, so use matching Companion/QGC
versions for RGB following. No field silently acquires a new meaning.
Offsets: gain_d 14, gain_t 18, gain_vel 22, bbox_gains_valid 26.
Minimum payload 3, maximum 27 bytes; CRC_EXTRA 211 remains unchanged.
These gains neither request flight mode nor bypass FC admission or target identity checks.

Generate each consumer into its own build directory using the pinned generator;
never edit generated headers. Companion uses the superproject sync helper;
QGC resolves latest main during configure as documented in its custom/README.md.

## Key Links

* [Documentation/Website](https://mavlink.io/en/) (mavlink.io/en/)
* [Discussion/Support](https://mavlink.io/en/#support)
* [Contributing](https://mavlink.io/en/contributing/contributing.html)
* [License](https://mavlink.io/en/#license)
