# RAX3000Me 新版固件编译与刷机教程

适用于 2025 年新款 RAX3000Me（25位 SN、GD5F1GM7 闪存、AN8855 交换机）

## 硬件信息

| 组件 | 旧版 | 新版 |
|------|------|------|
| SN | 15位 | **25位** |
| 闪存 | FM25S01A | **GD5F1GM7UEYIGR** |
| 交换机 | MT7531 | **AN8855** |
| SoC | MT7981B | MT7981B |
| 内存 | DDR3 512MB | DDR3 512MB |

## NAND 分区布局

```
偏移地址      大小      分区名
0x000000      1MB       bl2
0x100000      512KB     orig-env
0x180000      2MB       factory
0x380000      2MB       fip        ← 关键！
0x580000      114MB     ubi
```

## 快速开始

### 1. 下载固件

从 [Releases](https://github.com/FASUZO/rax3000me-firmware/releases) 下载最新固件。

### 2. 准备工具

- USB转TTL（3.3V，推荐 FT232RL）
- [mtk_uartboot](https://github.com/981213/mtk_uartboot/releases)
- [Tftpd64](https://pjo2.github.io/tftpd64/)
- 网线

### 3. TTL 刷机

```powershell
# 路由器断电，运行命令后上电
mtk_uartboot.exe -s COM5 -p mt7981-ddr3-bl2-ramboot.bin -a -f immortalwrt-*-nand-ddr3-bl31-uboot.fip --brom-load-baudrate 921600 --bl2-load-baudrate 1500000
```

### 4. 刷入固件

1. 进入 U-Boot 菜单
2. 选择 **5. Load production system via TFTP then write to NAND**
3. 输入文件名：`squashfs-sysupgrade.itb`
4. 电脑 IP 设为 192.168.1.2，启动 Tftpd64

### 5. 启动系统

浏览器访问 192.168.1.1

## 固件特性

### 已优化
- 移除不必要的服务，减小固件体积
- 包含 WiFi 固件（kmod-mt7915e-firmware）
- 添加性能工具（irqbalance、sqm-scripts、htop、iperf3）
- Argon 主题

### 性能优化（刷机后执行）

```bash
# 降低 WiFi 功率降温
uci set wireless.radio0.txpower='17'
uci set wireless.radio1.txpower='20'
uci commit wireless
wifi reload

# 启用硬件加速
uci set network.globals.packet_steering='1'
uci commit network
/etc/init.d/network restart

# CPU 性能模式
echo 'performance' > /sys/devices/system/cpu/cpu0/cpufreq/scaling_governor
echo 'performance' > /sys/devices/system/cpu/cpu1/cpufreq/scaling_governor
```

## 自行编译

### GitHub Actions（推荐）

1. Fork 本仓库
2. 进入 Actions 页面
3. 运行 Build workflow
4. 下载 Artifacts

### 本地编译

```bash
# 安装依赖
sudo apt-get install -y gcc-aarch64-linux-gnu build-essential libncurses-dev python3 rsync git

# 克隆源码
git clone --depth 1 https://github.com/immortalwrt/immortalwrt.git
cd immortalwrt

# 更新 feeds
./scripts/feeds update -a
./scripts/feeds install -a

# 配置
cat > .config << 'EOF'
CONFIG_TARGET_mediatek=y
CONFIG_TARGET_mediatek_mt7981=y
CONFIG_TARGET_MULTI_PROFILE=y
CONFIG_TARGET_PER_DEVICE_ROOTFS=y
CONFIG_PACKAGE_luci=y
CONFIG_PACKAGE_luci-base=y
CONFIG_PACKAGE_luci-compat=y
CONFIG_PACKAGE_luci-app-firewall=y
CONFIG_PACKAGE_luci-app-opkg=y
CONFIG_PACKAGE_kmod-mt7915e-firmware=y
EOF
make defconfig

# 编译
make -j$(nproc)
```

### 编译 RAM Boot BL2

```bash
git clone --depth 1 -b mtksoc https://github.com/mtk-openwrt/arm-trusted-firmware.git
cd arm-trusted-firmware

# 启用 UART 下载支持
sed -i 's/depends on _INTERNAL/# depends on _INTERNAL/' plat/mediatek/apsoc_common/bl2/Config-uart_dl.in
sed -i 's/default n/default y/' plat/mediatek/apsoc_common/bl2/Config-uart_dl.in

# 编译
docker run --rm -v $(pwd):/build/atf ubuntu:22.04 bash -c "
  apt-get update && apt-get install -y gcc-aarch64-linux-gnu make python3 device-tree-compiler bc swig &&
  cd /build/atf &&
  make CROSS_COMPILE=aarch64-linux-gnu- PLAT=mt7981 BOOT_DEVICE=ram DRAM_USE_DDR4=0 BOARD_BGA=1 RAM_BOOT_UART_DL=1 -j\$(nproc)
"
```

## 常见问题

| 问题 | 原因 | 解决方案 |
|------|------|----------|
| BL2 超时 | preloader 不支持 RAM Boot | 自编译 BL2 |
| DRAM 校准失败 | 未指定 BGA 封装 | 加 `BOARD_BGA=1` |
| FIP 加载失败 | 偏移地址错误 | FIP 在 0x380000 |
| UBI 空间不足 | 直接写 raw 到 UBI | 用 U-Boot 菜单选项 5 |
| WiFi 不工作 | EEPROM 缺失 | 上传 EEPROM 文件 |
| WiFi 温度过高 | 发射功率过大 | 降低 txpower |

## 参考资源

- [mtk_uartboot](https://github.com/981213/mtk_uartboot)
- [ImmortalWrt](https://github.com/immortalwrt/immortalwrt)
- [ATF 源码](https://github.com/mtk-openwrt/arm-trusted-firmware)
- [Daniel-Hwang 教程](https://github.com/Daniel-Hwang/RAX3000Me)
- [rmoyulong 固件](https://github.com/rmoyulong/cmcc-rax3000me)

## License

MIT
