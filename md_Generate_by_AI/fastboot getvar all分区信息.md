# fastboot getvar all 分区信息解读与刷机建议

---

## 1. 基本状态

| 项目                | 输出示例                                    | 说明                           |
|---------------------|---------------------------------------------|--------------------------------|
| off-mode-charge     | 1                                           | 关机充电功能（1=支持）         |
| unlocked            | yes                                         | Bootloader 已解锁              |
| secure              | no                                          | 安全启动关闭                   |
| serialno            | W4TSNNRWBQCASW9L                            | 设备序列号                     |
| product             | k6989v2_64                                  | 设备型号（MTK 平台）           |
| version-baseband    | MOLY.NR16.R2.MP3.TC16.PR3.SP.V1.P16         | 基带版本                       |
| version-bootloader  | k6989v2_64_addbfad_202506221510             | Bootloader 编译版本与日期      |

---

## 2. Slot（A/B）分区信息

| 项目                  | 输出示例         | 说明                       |
|-----------------------|------------------|----------------------------|
| slot-count            | 2                | A/B 分区架构               |
| current-slot          | a                | 当前运行槽位               |
| slot-successful:a     | yes              | slot a 启动成功            |
| slot-retry-count:a    | 7                | slot a 启动重试次数        |

> **说明**：A/B 架构通常没有独立 recovery 分区，recovery 集成在 boot 或 vendor_boot。

---

## 3. 分区类型和大小（示例）

| 分区名             | 类型   | 大小（十六进制） | 约等于（十进制） |
|--------------------|--------|------------------|------------------|
| userdata           | f2fs   | 3740ff8000       | 约 238 GB        |
| boot_a             | raw    | 4000000          | 约 64 MB         |
| vendor_boot_a      | raw    | 4000000          | 约 64 MB         |
| init_boot_a        | raw    | 800000           | 约 8 MB          |

---

## 4. Recovery 分区缺失说明

- 没有 `recovery_a` 或 `recovery_b` 分区，说明设备没有独立 recovery 分区。
- A/B 机型的 recovery 集成在 boot/vendor_boot。
- 刷 recovery.img 到 recovery 会失败，需通过 boot/vendor_boot 替代或临时启动。

---

## 5. 刷入自定义 Recovery 的方法（未验证）

1. **临时启动**（不破坏原 boot）：
   ```bash
   fastboot boot recovery.img
   ```
2. 在 recovery 里刷进 vendor_boot 或 boot 分区。
3. 建议先查询当前槽位：
   ```bash
   fastboot getvar current-slot
   ```
   再刷当前槽位对应的 boot/vendor_boot。

---

## 6. A/B 分区与 vbmeta

| 分区名      | 说明                       |
|-------------|----------------------------|
| boot_a/b    | 系统启动分区（A/B 副本）   |
| system_a/b  | 系统分区（A/B 副本）       |
| vendor_a/b  | 驱动分区（A/B 副本）       |
| vbmeta_a/b  | 验证分区（A/B 副本）       |

- 系统启动时只验证当前 slot 的 vbmeta。
- **关闭 AVB 验证建议：**
  ```bash
  fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta.img
  fastboot --disable-verity --disable-verification flash vbmeta_b vbmeta.img
  ```
- 还原 AVB 验证：
  ```bash
  fastboot flash vbmeta vbmeta.img
  ```
- 验证 AVB 状态：
  ```bash
  adb shell getprop ro.boot.veritymode
  ```

---



## 7. 槽位查询与切换

| 操作           | 命令示例                       | 说明                       |
|----------------|--------------------------------|----------------------------|
| 查询槽位       | fastboot getvar current-slot   | 查看当前活动槽             |
| 切换到 B 槽    | fastboot set_active b          | 切换到 slot b              |
| 指定槽刷入     | fastboot flash boot_a boot.img | 刷入 A 槽 boot.img         |
|                | fastboot flash boot_b boot.img | 刷入 B 槽 boot.img         |
| 只刷当前槽     | fastboot flash boot boot.img   | 风险大，另一槽可能失败     |

---

## 9. 实战建议

- 做实验/刷不稳定内核时，可只刷当前槽，出问题可切回另一槽救机。
- 长期使用建议 A、B 两个槽都刷一致，避免 OTA 更新或切换槽后启动失败。没特殊需求还是就稳定使用一个版本，不要老是更新系统。