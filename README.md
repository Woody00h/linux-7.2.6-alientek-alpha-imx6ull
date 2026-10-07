# Linux 7.2.6：ALIENTEK ALPHA V2.4 使用指南

更新：2026-10-07。目标硬件：i.MX6ULL、512MB DDR、8GB eMMC、ALPHA V2.4 底板。
本文件记录本项目的编译、部署及启动方式；上游说明见 [README](README)。

## 1. 已验证范围

内核和板级 DTB 已成功启动，ENET2 网络及 NFS 根文件系统可用；新 U-Boot 2026.07 已能引导此内核。NFS 已从 192.168.5.233 迁至 192.168.5.226。

设备树历史名称仍为 `imx6ull-alientek-alpha-v27-emmc`，其 model 和配置实际针对 V2.4；不要因文件名误换板型。UART 控制台为 ttymxc0，115200 波特率；ENET2 使用 PHY 地址 1。

## 2. Ubuntu 主机准备

源码目录：`/home/woody/alpha/woody/linux-7.2.6`，Windows 挂载为 `W:\alpha\woody\linux-7.2.6`。

```bash
sudo apt update
sudo apt install build-essential gcc-arm-linux-gnueabihf binutils-arm-linux-gnueabihf \
    flex bison bc libssl-dev libelf-dev libncurses-dev
arm-linux-gnueabihf-gcc --version
```

旧环境 GCC 4.9.4 已验证无法通过配置检查；当时内核要求至少 GCC 8.1.0。使用已成功构建的新 Ubuntu 环境。

## 3. 配置与编译

首次配置，或明确需要恢复默认配置时执行下列第一条 make。已有自定义配置时先备份 `out/.config`，不要每次增量编译都重新 defconfig。

```bash
cd ~/alpha/woody/linux-7.2.6
make O=out ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- imx_v6_v7_defconfig
make O=out ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- -j"$(nproc)" zImage dtbs modules
```

仅修改板级 DTS 后：

```bash
make O=out ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- \
    nxp/imx/imx6ull-alientek-alpha-v27-emmc.dtb
```

产物：

| 文件 | 路径 |
|---|---|
| 内核 | `out/arch/arm/boot/zImage` |
| DTB | `out/arch/arm/boot/dts/nxp/imx/imx6ull-alientek-alpha-v27-emmc.dtb` |
| 配置 | `out/.config` |

NFS 根启动需要网络、FEC/PHY、IP 自动配置及 NFS 根支持内建进内核，而非仅编译成模块：

```bash
grep -E '^CONFIG_(FEC|PHYLIB|INET|IP_PNP|NFS_FS|NFS_V3|ROOT_NFS)=' out/.config
```

如根文件系统需要本次内核的模块，确认根目录后安装，不能沿用旧 4.1.15 模块：

```bash
# 替换成真实的 NFS 根目录；不要指向主机自身的 /
ROOTFS=/实际的NFS根目录
sudo make O=out ARCH=arm CROSS_COMPILE=arm-linux-gnueabihf- \
    INSTALL_MOD_PATH="$ROOTFS" modules_install
```

## 4. 部署至 TFTP 并使用 NFS 启动

先确认服务器实际 TFTP 根目录，例如查看 `/etc/default/tftpd-hpa`。下面变量必须换成真实路径：

```bash
TFTP_ROOT=/实际的TFTP根目录
sudo install -m 0644 out/arch/arm/boot/zImage "$TFTP_ROOT/zImage"
sudo install -m 0644 out/arch/arm/boot/dts/nxp/imx/imx6ull-alientek-alpha-v27-emmc.dtb \
    "$TFTP_ROOT/imx6ull-alientek-alpha-v27-emmc.dtb"
```

以下在开发板 U-Boot 中执行。MAC 为本板历史测试值，多板同时联网须使用各自唯一地址：

```text
setenv ethaddr 00:04:9f:04:d2:35
setenv ipaddr 192.168.5.2
setenv serverip 192.168.5.226
ping 192.168.5.226
```

若现有 bootargs 已能挂载 NFS，应保留；需重新设置时用实际导出目录替换 `/实际的NFS根目录`：

```text
setenv bootargs 'console=ttymxc0,115200 root=/dev/nfs nfsroot=192.168.5.226:/实际的NFS根目录,vers=3,proto=tcp rw ip=192.168.5.2:192.168.5.226:192.168.5.1:255.255.255.0::eth0:off'
setenv bootcmd 'if tftpboot 80800000 zImage; then if tftpboot 83000000 imx6ull-alientek-alpha-v27-emmc.dtb; then bootz 80800000 - 83000000; fi; fi'
run bootcmd
```

确认配置正确后可在 U-Boot 执行 `saveenv`。新 U-Boot 的环境存储跟随启动设备，SD 与 eMMC 的环境各自独立。TFTP 的 serverip 不会自动改变 bootargs 中的 NFS 地址。新服务器的实际 NFS 目录未在记录中明确，不能照抄旧服务器目录。

## 5. 将内核与 DTB 保存到 eMMC FAT（可选，待独立验证）

内核和 DTB 是文件，不要套用 U-Boot 的原始扇区写入方法，也不要写入 eMMC boot0。
已确认 U-Boot 中 mmc1 是 eMMC；用户区第一个软件分区是 32MiB FAT。先用 `fatls mmc 1:1` 检查内容，并将同名旧文件备份到主机或 SD，确认剩余空间足够。

下面逐组执行，只有下载成功、长度正确才进行对应 fatwrite：

```text
mmc dev 1 0
fatls mmc 1:1
tftpboot 80800000 zImage
fatwrite mmc 1:1 80800000 zImage ${filesize}
tftpboot 83000000 imx6ull-alientek-alpha-v27-emmc.dtb
fatwrite mmc 1:1 83000000 imx6ull-alientek-alpha-v27-emmc.dtb ${filesize}
```

首次可手动测试从 eMMC 加载内核和 DTB，仍保留已验证的 NFS bootargs：

```text
if fatload mmc 1:1 80800000 zImage; then if fatload mmc 1:1 83000000 imx6ull-alientek-alpha-v27-emmc.dtb; then bootz 80800000 - 83000000; fi; fi
```

此方式仅将内核与 DTB 本地化，根文件系统仍依赖 NFS。完全离线启动还需另行部署 rootfs 并验证 root= 参数；本文不将其标为已完成。

## 6. 排查要点

- TFTP 失败：检查 ENET2 网线、MAC、IP、serverip、服务器服务及防火墙。
- NFS 挂载失败：检查 bootargs、实际导出路径、导出权限、NFS v3 支持和内核内建配置。
- PHY：地址为 1；Linux MDIO 复位低电平 100000us，释放后等待 200000us。U-Boot 的同类属性以毫秒计，不能混用。
- 修改 DTS 后必须重新编译并复制新 DTB，不能只更新 zImage。
- 上游源码中的 README、Documentation 保留，本文仅描述本板工作流。