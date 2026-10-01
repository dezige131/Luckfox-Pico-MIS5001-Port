# Luckfox Pico（RV1103）接入 MIS5001

[English](README.md)

本仓库提供适用于 **Luckfox Pico RV1103、SD 卡 Buildroot IPC 固件**的 SDK 源码补丁。补丁包含 MIS5001 驱动的 2560×1440 RAW12 模式、摄像头设备树连接、IQ 文件选择及 rkipc 配置。编译出的固件提供单路 H.264 RTSP 视频，可选择 **1920×1080 / 15 fps** 或 **1280×720 / 15 fps**。传感器输入仍为 2560×1440，由 ISP 缩放到所选的串流分辨率。

## 适用条件

- 双通道 CSI-2 MIS5001，I²C4 地址 `0x31`，芯片 ID `0x1311`；测试模组 `CMK-OT2115-PC1`、镜头 `30IRC-F16`。
- Luckfox SDK 基线提交：`aaecc58d9366685751914373b774ec6a7d9b18db`。目标板配置：`BoardConfig-SD_CARD-Buildroot-RV1103_Luckfox_Pico-IPC.mk`。
- 其他模组或板型请先核对设备树中的 I²C、时钟、供电 GPIO、CSI 通道及模组名称；补丁不适用于直接给已有固件单独替换 `.ko`。

## 编译与烧录

在已安装 Luckfox SDK 编译依赖和交叉工具链的 **Linux 主机**上，使用上述提交的未修改 SDK。**只选一份补丁**：

- `patches/luckfox-sdk.patch`：默认 1920×1080 / 15 fps；
- `patches/luckfox-sdk-720p.patch`：低分辨率 1280×720 / 15 fps。

以下假定本仓库和 `luckfox-pico` 是同级目录。要编译 720p 版，将以下两条 `git apply` 命令中的 `luckfox-sdk.patch` 都改成 `luckfox-sdk-720p.patch`：

```sh
cd /path/to/luckfox-pico
git rev-parse HEAD
git apply --check ../mis5001-luckfox-pico/patches/luckfox-sdk.patch
git apply ../mis5001-luckfox-pico/patches/luckfox-sdk.patch
./build.sh info
./build.sh all
```

如 `./build.sh info` 显示的不是 SD 卡 Buildroot RV1103 Luckfox Pico IPC 板型，先执行 `./build.sh lunch` 选择该配置，再执行 `./build.sh all`。`all` 会编译内核和设备树、驱动、媒体组件、rkipc，并打包固件；SD 卡镜像位于 **`output/image/sd_update.img`**。按 [Luckfox 官方烧录说明](https://wiki.luckfox.com/Luckfox-Pico/Luckfox-Pico-RV1103/Luckfox-Pico-quick-start) 将镜像写入目标 SD 卡（选择整张卡而非分区），插卡启动。烧录将覆盖目标卡数据。

两份补丁都包含新增的 `rkipc-mis5001-400w.ini`，不需要额外复制文件。固件打包时会直接从 SDK 源码目录复制已关闭 LSC 的 MIS5001 IQ；`media/isp/out` 和 `output/out/media_out` 可能仍是启用 LSC 的旧文件，不要直接拿它们打包。已经打过补丁的 SDK 不要重复执行 `git apply`。

## 推流和检查

启动后通过 USB 网卡访问 `rtsp://<板子IP>/live/0`，例如：

```sh
ffplay -rtsp_transport tcp rtsp://172.32.0.93/live/0
```

`172.32.0.93` 是测试时的地址，实际以设备地址为准。也可运行 `adb forward tcp:8554 tcp:554`，在主机打开 `rtsp://127.0.0.1:8554/live/0`。音频采集尚未在此板上验证。

VLC 网络缓存：1920×1080 @ 15 fps 时实测最低可设为 **200 ms**；720p @ 30 fps 时实测最低可设为 **300 ms**。其他分辨率、帧率及网络条件请自行验证。这些值是当时的播放测试结果，不保证所有设备或网络环境下都能稳定播放。

帧率可以自行调整：在 SDK 的 `project/app/rkipc/rkipc/src/rv1103_ipc/rkipc-mis5001-400w.ini` 中修改 `[video.0]` 的 `src_frame_rate_num`、`dst_frame_rate_num`（分母默认均为 `1`）；按需要同步调整 `[isp.0.adjustment]` 中的 `fps`，并根据期望的关键帧间隔调整 `gop`。修改后重新编译/打包固件；板上已有的 `/userdata/rkipc.ini` 也可能覆盖出厂 INI。当前 MIS5001 传感器模式最高 30 fps。**720p 补丁默认仍是 15 fps**，上面的 30 fps 缓存值对应手动调整帧率后的播放测试。

如果没有图像，在板上查看 `dmesg | grep -i mis5001` 和 `lsmod | grep mis5001`。如果以前的固件在 `/userdata/rkipc.ini` 留有配置，它会覆盖新固件的出厂配置；先备份并移走旧文件，重启后会使用 `/oem/usr/share/rkipc-mis5001-400w.ini`。

## 已知限制

- 以 400 万像素原生分辨率运行时，RV1103 温度会比较高，请做好散热后再使用。
- SDK 附带的 MIS5001 IQ 文件仅有 2592×1944 的镜头阴影校准，与 2560×1440 模式不匹配。补丁只关闭 **LSC**，保留其他 IQ 功能；本项目没有提供 1440p 的镜头标定数据。
- 此前在本板观察到 1080p/15fps 和 720p/15fps 均可推流；720p 补丁是据此复现的配置，尚未作为独立补丁重新编译和验证。未验证 2560×1440 推流、HDR、副码流、JPEG 及长期热稳定性。1080p 推流占用接近 24 MiB 的 CMA，不建议同时抓取全分辨率 RAW。
- 驱动保留原来的 2592×1944 RAW10 模式，但本板未验证。其他 SDK 版本或硬件需自行适配。
- 补丁修改了原厂 SDK 文件；再分发固件或衍生源码时请遵循对应的许可证。
