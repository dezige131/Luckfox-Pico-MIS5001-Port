# MIS5001 for Luckfox Pico (RV1103)

[中文说明](README.zh-CN.md)

Source patch for the Luckfox Pico **SD-card Buildroot IPC** SDK. It adds MIS5001 2560×1440 RAW12 sensor support, wires the RV1103 device tree, selects the sensor IQ file, and configures rkipc for a single H.264 RTSP stream. Choose **1920×1080 / 15 fps** or **1280×720 / 15 fps** when applying the patch. The sensor runs at 2560×1440; the ISP scales the output to the selected stream resolution.

## Hardware and SDK

Tested with a two-lane MIS5001 module on I²C4 at `0x31` (ID `0x1311`), module name `CMK-OT2115-PC1`, lens `30IRC-F16`, and Luckfox SDK revision `aaecc58d9366685751914373b774ec6a7d9b18db`. The patch targets `BoardConfig-SD_CARD-Buildroot-RV1103_Luckfox_Pico-IPC.mk` and `rv1103-luckfox-pico-ipc.dtsi`. Check the I²C address, power GPIOs, clock, lane assignment, and module/lens names for other hardware before building. The board configuration selects 24 MiB CMA.

## Build a firmware image

On a Linux host with the Luckfox SDK and its toolchain/dependencies installed, start from the specified **unmodified** SDK revision. Choose **one** patch:

- `patches/luckfox-sdk.patch`: default 1920×1080 / 15 fps.
- `patches/luckfox-sdk-720p.patch`: alternative 1280×720 / 15 fps.

Assume this repository and `luckfox-pico` are sibling directories. For 720p, replace `luckfox-sdk.patch` with `luckfox-sdk-720p.patch` in both commands:

```sh
cd /path/to/luckfox-pico
git rev-parse HEAD
git apply --check ../mis5001-luckfox-pico/patches/luckfox-sdk.patch
git apply ../mis5001-luckfox-pico/patches/luckfox-sdk.patch
./build.sh info
./build.sh all
```

Select the SD-card Buildroot RV1103 Luckfox Pico IPC board configuration with `./build.sh lunch` first if `./build.sh info` shows a different board. `./build.sh all` builds the kernel/DTB, drivers, media libraries, rkipc, and packages the firmware. The resulting SD-card image is `output/image/sd_update.img`. Write it to a suitable SD card following the [Luckfox SDK flashing instructions](https://wiki.luckfox.com/Luckfox-Pico/Luckfox-Pico-RV1103/Luckfox-Pico-quick-start) (select the **entire device**, not a partition); flashing replaces its contents. Reboot the Pico from that card.

Each patch includes the new `rkipc-mis5001-400w.ini`; no additional source files need copying. Firmware packaging copies the patched MIS5001 IQ **from the SDK source**: the prebuilt `media/isp/out` and `output/out/media_out` IQ files may still have LSC enabled, so do not package either stale copy instead. Re-running `git apply` on the same patched tree will fail; use the original SDK revision for a fresh build.

## Use

Connect over USB networking and play `rtsp://<board-ip>/live/0` in an RTSP client (e.g. `ffplay -rtsp_transport tcp rtsp://<board-ip>/live/0`). The verified USB gadget address was `172.32.0.93`; use the address assigned on your board. If needed, `adb forward tcp:8554 tcp:554` exposes `rtsp://127.0.0.1:8554/live/0` on the host. Audio capture has not been verified on this board.

VLC network caching: the lowest setting observed to play at 1920×1080 @ 15 fps was **200 ms**; at 720p @ 30 fps it was **300 ms**. Test other resolutions, frame rates, and network conditions yourself. These are observed playback settings, not guarantees of stability on every setup.

You can change the frame rate in the SDK file `project/app/rkipc/rkipc/src/rv1103_ipc/rkipc-mis5001-400w.ini`: set `[video.0]` `src_frame_rate_num` and `dst_frame_rate_num` (both denominators default to `1`), adjust `[isp.0.adjustment]` `fps` as needed, and set `gop` for the desired keyframe interval. Rebuild/repackage the firmware; an existing `/userdata/rkipc.ini` on the board may override the factory INI. The current MIS5001 sensor mode supports up to 30 fps. **The 720p patch still defaults to 15 fps**; the 30 fps caching observation refers to a manually adjusted frame-rate setting.

On the board, check `dmesg | grep -i mis5001` and `lsmod | grep mis5001` if no picture appears. A persistent `/userdata/rkipc.ini` from an older firmware may override the new factory configuration: back up and remove it, then reboot to allow rkipc to select `/oem/usr/share/rkipc-mis5001-400w.ini`.

## Notes

- Running at the native 4-megapixel resolution can make the RV1103 run hot; provide adequate cooling.
- The supplied MIS5001 IQ data has lens-shading calibration for 2592×1944, not 2560×1440; the patch disables **LSC only**, leaving other IQ functions enabled. Correct 1440p lens calibration is not included.
- 1080p/15 fps and 720p/15 fps were previously observed streaming on this board; the 720p variant recreates the settings from that record and has not been rebuilt or revalidated as a separate patch. 2560×1440 streaming, HDR, multiple streams, JPEG and long-duration thermal stability are not established. 1080p streaming nearly fills the 24 MiB CMA area; avoid simultaneous full-resolution RAW capture.
- The original 2592×1944 RAW10 sensor mode is retained in the driver but was not validated on this board. A different SDK revision or camera module may require adjustments.
- Both patches modify vendor SDK files; follow the SDK and upstream source licenses when redistributing firmware or derived code.
