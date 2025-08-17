# 刷机代码 & img 镜像作用及原理

## 1. AB 卡槽机制

- Android 设备采用 A/B 分区（双卡槽），支持无缝系统更新。
- 刷入 `init_boot_a`（LKM 模块）只影响 A 卡槽，B 卡槽仍用原始内核。
- 切换到 GKI 内核时需同时刷 A/B 卡槽的 `boot` 和 `init_boot` 分区。
- 刷入 GKI 后，可能还需 LKM 模块保证驱动兼容性。
- 修补后刷入 `boot.img` 或 `init_boot.img` 属于 LKM 工作模式，只有将 kernelsu 或 magiskboot 打包进内核压缩包并刷入，才是 GKI 工作模式。
- 刷机时需处理好 AVB 验证，避免启动失败。

---

## 2. 安卓/一加卡刷包各包作用

### 2.1 重要文件

| 文件名              | 大小      | 作用说明                                                         |
|---------------------|-----------|------------------------------------------------------------------|
| `boot.img`          | 67MB      | 系统启动所需的内核和 ramdisk，系统核心文件                        |
| `vbmeta.img`        | 8KB       | AVB 验证信息，验证系统完整性                                      |
| `system.img`        | ~816MB    | Android 系统核心部分，包含大部分应用和系统文件                    |
| `vendor.img`        | ~1.2GB    | 硬件相关文件和驱动，厂商定制化软件和驱动                          |
| `recovery.img`      | 变种      | 恢复模式镜像，设备恢复和刷机功能                                  |
| `preloader_raw.img` | 1.9MB     | 联发科设备预加载程序，硬件初始化                                  |
| `vendor_boot.img`   | 64MB      | Vendor 相关启动镜像，只在 Vendor 部分加载                         |
| `dtbo.img`          | 1.1MB     | 设备树文件，定义硬件架构和驱动                                    |
| `modem.img`         | 71MB      | 基带固件，支持蜂窝网络通信                                        |
| `init_boot.img`     | 8MB       | 设备初始化所需启动参数                                            |

### 2.2 其他文件

| 文件名                  | 大小      | 作用说明                                         |
|-------------------------|-----------|--------------------------------------------------|
| `cdt_engineering.img`   | 1MB       | 工程模式固件                                     |
| `audio_dsp.img`         | 12MB      | 音频 DSP 固件，处理音频信号                      |
| `mcf_ota.img`           | 32MB      | OTA 更新相关文件                                 |
| `scp.img`               | 8MB       | 安全控制处理器固件，负责加密和身份验证           |
| `tee.img`               | 1.6MB     | TEE 固件，硬件级安全操作                         |
| `vcp.img`               | 2MB       | 视频编解码器或图形处理相关固件                   |
| `odm.img`               | ~1.5GB    | 设备特有定制化文件，厂商（ODM）提供              |
| `system_dlkm.img`       | 7MB       | 系统动态加载内核模块                             |
| `product.img`           | ~1.1GB    | 应用和系统功能定制文件                           |
| `spmfw.img`             | 36KB      | 系统功率管理固件                                 |

### 2.3 其他小文件

- `my_carrier.img` / `my_engineering.img` / `my_heytap.img` 等（约 335KB）：运营商定制或工程功能相关镜像。
- `gz.img`（3.7MB）：可能为系统压缩文件（GZIP）。
- `vcp.img`、`ccu.img` 等（小于 10MB）：硬件驱动、固件加载相关文件。

---

## 3. 总结

- **核心系统文件**：`boot.img`, `system.img`, `vendor.img`, `vbmeta.img`
- **硬件驱动及初始化文件**：`modem.img`, `dtbo.img`, `preloader_raw.img`
- **安全与恢复相关文件**：`recovery.img`, `tee.img`, `scp.img`
- **定制与 OTA 更新文件**：`mcf_ota.img`, `product.img`, `odm.img`

大多数文件用于设备硬件、启动、系统和安全功能。通过刷入修改后的 `vbmeta.img` 可关闭 AVB 验证，常见于刷机操作。

