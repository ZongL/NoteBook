# S32G274板 A核系统烧写指南

## 板卡信息

- CPU: NXP S32G274A rev. 2.2.0
- Model: S32G2XX supplier
- U-Boot: 2019.04+gxa1456 (bsp32.0-2019.04)
- OP-TEE: 3.13.0
- BL2: v1.0 (bsp32.0-1.0)
- DRAM: 896 MiB
- eMMC: FSL_SDHC: 0
- 烧写方式: 以太网 fastboot（UDP），板子 IP 为 `172.16.0.2`

## 镜像文件说明（A_0614_userdebug，版本 06.14）

| 文件名 | 说明 |
|--------|------|
| partition-table.img | GPT 分区表 |
| bootloader.img | U-Boot bootloader |
| boot.img | Linux 内核 + ramdisk |
| dtbo.img | 设备树 overlay |
| system.img | 根文件系统（约 276MB） |
| vbmeta.img | AVB 校验信息 |

## 烧写步骤

### 1. 网络连接

用网线将 PC 与板子直连，配置 PC IP 地址：

- PC IP: `172.16.0.1`（同网段，不能是 `172.16.0.2`）
- 子网掩码: `255.255.0.0`
- 板子 IP: `172.16.0.2`（U-Boot 自动配置）

### 2. 板子进入 fastboot 模式

**方式一（eMMC 无分区表时）：** 板子上电后会自动进入 fastboot 模式，串口打印 `Listening for fastboot command on 172.16.0.2` 表示就绪。

**方式二（eMMC 已有分区表时）：** 板子上电后 U-Boot 会显示 `Hit some key to stop autoboot`，此时在串口 **连续按 `h` 键** 可打断启动进入 U-Boot 命令行（OEM 定制，非任意键），然后输入 `fastboot udp` 进入 fastboot 模式。

### 3. 烧写分区表（首次或分区表为空时必须）

如果 eMMC 分区表为空（串口显示 `mbr->signature =0`），必须先写入分区表。

**注意：分区名必须用 `gpt`，用 `partition-table` 会导致 U-Boot Synchronous Abort 崩溃。**

```bash
fastboot -s udp:172.16.0.2 flash gpt partition-table.img
```

串口确认输出：
```
fastboot_mmc_flash_write: updating MBR, Primary and Backup GPT(s)
........ success
```

### 4. 烧写系统镜像

分区表写入成功后，依次烧写各分区。

**注意：分区表采用 A/B 槽位方案，必须加 `--slot all` 参数，否则报 `Failed to identify current slot` 错误。**

```bash
fastboot -s udp:172.16.0.2 --slot all flash bootloader bootloader.img
fastboot -s udp:172.16.0.2 --slot all flash boot boot.img
fastboot -s udp:172.16.0.2 --slot all flash dtbo dtbo.img
fastboot -s udp:172.16.0.2 --slot all flash vbmeta vbmeta.img
fastboot -s udp:172.16.0.2 --slot all flash system system.img
```

如果只更新 A 核 Linux 系统（不动 bootloader），最少烧写：
```bash
fastboot -s udp:172.16.0.2 --slot all flash boot boot.img
fastboot -s udp:172.16.0.2 --slot all flash system system.img
fastboot -s udp:172.16.0.2 --slot all flash vbmeta vbmeta.img
```

### 5. 重启

```bash
fastboot -s udp:172.16.0.2 reboot
```

## 注意事项

- `system.img` 约 276MB，网络传输较慢，耐心等待
- 最大单次下载大小为 112MB（`0x07000000`），如果镜像超过此大小传输失败，需分片或检查 U-Boot 配置
- 如果中途超时断开，重启板子重新进入 fastboot 继续刷写即可
- 验证设备连通性：`fastboot -s udp:172.16.0.2 getvar product`，应返回 `s32-gen1`
- 此板子 U-Boot 的 fastboot **不支持 `oem` 命令**（返回 `unrecognized command`）
- 分区表写入后重启，U-Boot 会显示 `Hit some key to stop autoboot`，可在串口按键打断进入 U-Boot 命令行

## 串口启动日志参考

```
U-Boot 2019.04+

CPU:   NXP S32G274A rev. 2.2.0
Model: S32G2XX 
DRAM:  896 MiB
MMC:   FSL_SDHC: 0
Using internal clock for PCIe0, CRNS
Frequency 100Mhz configured for PCIe0
Configuring PCIe0 as RootComplex(x1)&SGMII [XPCS0 OFF(PCIex1), XPCS1 1G]
PCIe0 disabled
Using internal clock for PCIe1, CRNS
Frequency 100Mhz configured for PCIe1
Configuring PCIe1 as SGMII(x2) [XPCS0 1G, XPCS1 1G]
In:    serial@401C8000
Out:   serial@401C8000
Err:   serial@401C8000
Net:   register phy driver for BCM89883
 PFE: emac0: sgmii emac1: none emac2: sgmii
PFE_EMAC2 Phy power up complete.
PFE_EMAC2 get phy_id 0xae02503a.

Warning: eth_pfeng using MAC address from ROM
eth0: eth_pfeng
mbr->signature =0
mbr->signature =0
Could not find "misc" partition
** Bad device specification mmc 0#misc **
Couldn't find partition mmc 0#misc
Attached to pfe0
Using eth_pfeng device
Listening for fastboot command on 172.16.0.2
```
