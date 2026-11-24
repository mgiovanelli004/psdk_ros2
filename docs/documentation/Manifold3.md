# DJI Manifold 3 (Matrice 4 series, Matrice 400)

> **Status:** builds against PSDK 3.18.0 for `PSDK_PLATFORM=manifold3`. It has
> **not been tested on a real Manifold 3 or aircraft yet**. Please report
> results in the issue tracker.

On the Matrice 4E/4T, Matrice 4D/4TD, Matrice 400 and newer models, the PSDK
advanced functions (flight control, perception data, liveview, ...) are only
available from the DJI Manifold 3 onboard computer.

## What is different on Manifold 3

| Topic | M300/M350/M30 (manifold2 HAL) | Manifold 3 |
|---|---|---|
| Link to the aircraft | UART + USB bulk or UART + network | **USB bulk only** (`use_only_usb_bulk_device`) |
| USB bulk endpoints | from `link_config.json` | fixed in the DJI HAL: `/dev/usb-ffs/bulk2` (liveview), `/dev/usb-ffs/bulk6` (perception, media), `/dev/usb-ffs/bulk3` |
| Mount position type | `EXTENSION_PORT` | `DJI_MOUNT_POSITION_TYPE_MANIFOLD3_ONBOARD` |
| Max FC subscription rate | 400 Hz (M300/M350) | **50 Hz** for every topic |
| `HARD_SYNC` IMU topic | yes (M300/M350 400 Hz, M30 50 Hz) | **no**: `psdk_ros2/imu` is composed from attitude, raw angular rate and raw acceleration |
| Subscription timestamps | aircraft time on M300/M350 | PSDK local reception time |
| OS | any | Ubuntu 20.04, JetPack 5.1.3, no root: use the `box-*` container tools |

## Build

```bash
colcon build --packages-up-to psdk_wrapper \
  --cmake-args -DPSDK_PLATFORM=manifold3 -DPSDK_VERSION=3.18.0
```

* `PSDK_VERSION` (default `3.18.0`): Payload-SDK git tag. Manifold 3 needs >= 3.12.0.
* `PSDK_PLATFORM` (default `manifold2`): `manifold3` compiles
  `samples/sample_c++/platform/linux/manifold3/hal/hal_usb_bulk.c` instead of the
  manifold2 one (the UART/network HALs are still compiled but not registered).
* Offline builds: `-DFETCHCONTENT_SOURCE_DIR_PAYLOAD_SDK=/path/to/Payload-SDK`
  skips the download.
* FFmpeg **4.x** is required by the PSDK liveview decoder (Ubuntu 22.04 ships 4.4).

## Run

```bash
ros2 launch psdk_wrapper wrapper.launch.py \
  link_config_file_path:=$(ros2 pkg prefix psdk_wrapper)/share/psdk_wrapper/cfg/link_config_manifold3.json \
  psdk_params_file_path:=$(ros2 pkg prefix psdk_wrapper)/share/psdk_wrapper/cfg/psdk_params_manifold3.yaml
```

Inside a Manifold 3 `box-*` container the FunctionFS endpoints and the host
network must be passed through, and no other PSDK program (host or DPK) may
run at the same time:

```bash
box-run --name psdk --network host \
  --functionfs /dev/usb-ffs/bulk2 --functionfs /dev/usb-ffs/bulk3 \
  --functionfs /dev/usb-ffs/bulk6 \
  <image>:<tag>
```

DJI's documentation lists `bulk2` and `bulk3` as mandatory. The Manifold 3 HAL
in PSDK 3.18 also opens `bulk6` for perception images and media download, so
pass it through as well if it exists (`ls -d /dev/usb-ffs/bulk*`).

## New parameters and services

* `imu_source` (`auto` | `hard_sync` | `composed`, default `auto`): see
  `cfg/psdk_params.yaml`. With `composed`, `data_frequency.attitude`,
  `data_frequency.angular_velocity` and `data_frequency.acceleration` must be
  > 0; `psdk_ros2/imu` is published at the `angular_velocity` rate.
  To verify on the aircraft: at rest the norm of `linear_acceleration`
  should be about 9.81 (the PSDK header documents `ACCELERATION_RAW` in m/s²).
* Frequencies above the aircraft limit are reduced automatically (50 Hz on
  everything except M300/M350).
* `psdk_ros2/set_control_in_atti_mode` (`std_srvs/SetBool`) and
  `psdk_ros2/get_control_in_atti_mode` (`std_srvs/Trigger`): wrap
  `DjiFlightController_Set/GetControlInAttitudeModeEnabled()`, documented by DJI
  for the Matrice 4E/4T only. When the aircraft is in ATTI mode (no GNSS and no
  vision positioning), DJI only executes attitude commands: use
  `psdk_ros2/flight_control_setpoint_rollpitch_yawrate_thrust`; velocity and
  position setpoints are ignored.

## Known limitations

* Perception images are only streamed while the motors are running (or in the
  DJI Assistant 2 simulator), at most two directions at a time.
* Liveview: on Manifold 3 the FPV and main camera streams cannot be subscribed
  at the same time, and I-frame requests are not supported (PSDK samples).
* PPS time synchronisation needs a GNSS/RTK clock source; it is not expected to
  work indoors.
