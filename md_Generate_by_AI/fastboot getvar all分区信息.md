我帮你逐条解读一下这些 `fastboot getvar all` 输出的含义，这样你就能明白设备的分区布局和刷机限制了。

---

## **1. 基本状态**

```
off-mode-charge: 1
```

* 关机充电功能开关（1 表示支持关机状态下充电）。

```
unlocked: yes
```

* Bootloader 已解锁，可以刷机。

```
secure: no
```

* 安全启动被关闭（对刷机更宽松）。

```
serialno: W4TSNNRWBQCASW9L
product: k6989v2_64
```

* 序列号 & 设备型号（`k6989v2_64` 很可能是联发科 MT6989 平台）。

```
version-baseband: MOLY.NR16.R2.MP3.TC16.PR3.SP.V1.P16
```

* 基带版本（和信号相关）。

```
version-bootloader: k6989v2_64_addbfad_202506221510
```

* Bootloader 的编译版本和日期（2025-06-22）。

---

## **2. Slot（A/B）分区信息**

```
slot-count: 2
current-slot: a
slot-retry-count:a: 7
slot-successful:a: yes
```

* 这是**A/B 分区设备**（有 2 个系统槽：a 和 b）。
* 当前运行的是 **slot a**。
* `slot-successful:a: yes` 表示 slot a 启动成功且可用。

> **重要**：A/B 架构通常**没有单独的 recovery 分区**，而是把 recovery 集成在 boot 或 vendor\_boot 里。

---

## **3. 分区类型和大小**

* 每个分区的格式（`raw`、`f2fs`）和大小（十六进制，字节数）。
* 举例：

```
partition-type:userdata: f2fs
partition-size:userdata: 3740ff8000
```

* 用户数据分区是 `f2fs` 文件系统，大小约 238 GB。

```
partition-type:boot_a: raw
partition-size:boot_a: 4000000
```

* boot\_a 分区大小 0x4000000（约 64 MB）。

```
partition-type:vendor_boot_a: raw
partition-size:vendor_boot_a: 4000000
```

* vendor\_boot\_a 分区也有 64 MB。

```
partition-type:init_boot_a: raw
partition-size:init_boot_a: 800000
```

* init\_boot\_a 分区大小 8 MB（Android 12 之后多了 init\_boot 分区）。

---

## **4. Recovery 分区缺失的原因**

我注意到：

* 没有任何一行 `partition-type:recovery_a` 或 `recovery_b`
* 说明 **设备没有独立的 recovery 分区**。
* A/B 机型的 recovery 一般和 boot/vendor\_boot 绑定。
* 你刷 recovery.img 到 `recovery` 会失败是正常的，因为它根本不存在。

---

## **5. 这台机子的刷法**

* 如果你的 recovery.img 是 TWRP 或其他自定义 recovery，应该：

  1. **临时启动**（不破坏原 boot）：

     ```bash
     fastboot boot recovery.img
     ```
  2. 在 recovery 里刷进 vendor\_boot 或 boot 分区。
* 如果要替换：

  * 刷到 `boot_a` 或 `vendor_boot_a`，视设备布局而定。
  * 建议先用：

    ```bash
    fastboot getvar current-slot
    ```

    再刷当前槽位对应的 boot/vendor\_boot。

---

简单说，你的设备是 **A/B 分区，没有独立 recovery 分区**，recovery.img 要通过 boot/vendor\_boot 来替代或临时启动，直接 `fastboot flash recovery` 永远会报错。

---
## **6.A/B 分区与 vbmeta**

* 采用 A/B 分区的设备（绝大部分新 MTK/高通机型），**系统关键分区（boot、system、vendor 等）都有 \_a / \_b 两个副本**。
* **vbmeta 也有 vbmeta\_a 和 vbmeta\_b**，它们分别对应不同的 slot（即当前活动分区）。
* 系统启动时只会用当前 slot 的 vbmeta 来验证该 slot 下的 boot/system/vendor 等。

👉 所以：
如果你只在 `fastboot flash vbmeta vbmeta.img` 时没有指定 slot，fastboot 通常会默认刷到 **当前 active slot**。
另一边的 slot（比如 b）仍然是原版，切换 slot 后验证又会恢复。
要彻底“全局关闭”，建议：

```bash
fastboot --disable-verity --disable-verification flash vbmeta_a vbmeta.img
fastboot --disable-verity --disable-verification flash vbmeta_b vbmeta.img
```

这样无论切到哪个 slot，AVB 都是关闭的。

3、成功关闭 avb

如果需要还原，指令如下（同样是需要以AB分区为区分）
```
fastboot flash vbmeta vbmeta.img
````

刷机进系统后，可以通过以下方法验证：\
如果为空或者很低版本，可能是关闭状态（但不一定绝对）。
```
adb shell getprop ro.boot.veritymode
```
## **7. vbmeta 和 boot.img/init_boot.img 的刷入顺序**

这两个的关系是 **验证者（vbmeta）** 和 **被验证对象（boot/system 等）**。

* **先关闭 vbmeta → 再刷修补 boot.img**

  * 启动时 vbmeta 已经不再强校验 boot 分区，修补过的 boot.img 可以直接启动。
  * 这是最常见的、最稳妥的做法。
