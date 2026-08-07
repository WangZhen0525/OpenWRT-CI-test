# 斐讯 N1 OpenWrt 固件编译与刷机说明

## 固件方案

斐讯 N1 使用 Amlogic S905D，不是 M2 的 Qualcomm IPQ60xx 平台。此仓库先从 `VIKINGYFY/immortalwrt` 编译 `armsr/armv8` 通用 rootfs，再由 `ophub/amlogic-s9xxx-openwrt` 打包成 N1 可启动的 `s905d .img.gz` 镜像。

默认设置：

- 管理地址：`192.168.3.2`
- 用户名：`root`
- 初始密码：无（首次登录后立即设置）
- 主机名：`OWRT`
- OpenClash、TTYD、DDNS、WireGuard 和常用排错工具已编入
- IPv6 和 OpenClash 的订阅/DNS 配置仍需在首次启动后按实际网络配置
- N1 不作为无线 AP，因此不编译 hostapd/wpad

> 如果原来的 M2 仍是 `192.168.3.2`，首次启动 N1 前必须先断开 M2，避免地址冲突。

## 在 GitHub Actions 编译

1. 打开仓库的 **Actions**。
2. 选择 **WRT-TEST**，点击 **Run workflow**。
3. 参数选择：
   - `CONFIG`：`N1`
   - `SOURCE`：`VIKINGYFY/immortalwrt`
   - `BRANCH`：`main`
   - `TEST`：`false`
4. 等待编译和二次打包完成。
5. 从本次 Release 下载文件名中含有 `s905d` 且后缀为 `.img.gz` 的镜像。不要刷 `rootfs.tar.gz`，它只是中间产物。

流水线会检查 armv8 target、打包状态和最终 `s905d` 镜像；缺少任一项会直接失败，不会发布“看起来成功但不能刷”的包。

## 刷机前准备

- 质量可靠的 U 盘，建议 8–32 GB。
- Windows 电脑和 balenaEtcher 或 Rufus。
- N1 原装或稳定的 12V 电源。
- 确认 N1 已具备从 USB 启动条件。全新原厂 Android 若从未开启 USB 启动，应先用 N1 常用的降级/盒子助手流程开启；不要直接覆盖 eMMC。
- 强烈建议先保留原 Android 恢复包，并在 USB 启动 OpenWrt 后执行 `openwrt-ddbr` 备份 eMMC。

## 写入 U 盘并首次启动

1. 用 balenaEtcher 直接选择 `.img.gz` 写入 U 盘，不必手动解压。
2. N1 断电，将 U 盘插入靠近 HDMI 的 USB 口，网线接到主路由 LAN。
3. 如果设备已开启 USB 启动，通电后等待约 2–5 分钟。
4. 将电脑临时设为与 `192.168.3.2/24` 同网段，然后访问 `http://192.168.3.2`。
5. 无法启动时，可在 N1 的 Android ADB 已开启且同网段的前提下执行：
   ```powershell
   adb connect N1的IP地址
   adb shell reboot update
   ```
   若 ADB 未授权或拒绝连接，先完成 N1 的 USB 启动解锁流程，不要反复写 eMMC。

## 先从 U 盘验证

在写入 eMMC 前至少确认：

- LuCI 和 SSH 均可登录。
- `系统 → TTYD 终端` 中 `ip addr` 能看到有线网卡。
- N1 能访问主路由 `192.168.3.1` 和互联网。
- 重启一次后仍能从 U 盘正常启动。
- OpenClash 尚未导入配置时，不要先把全屋 DHCP 网关切到 N1。

建议先执行：

```sh
openwrt-ddbr
```

按提示选择 `b`，将原 eMMC 系统备份到外部介质。

## 安装到 eMMC

确认 USB 运行稳定后：

1. 进入 `系统 → 晶晨宝盒（Amlogic Service）→ 安装 OpenWrt`。
2. 设备选择 **Phicomm N1 / s905d**。
3. 再次核对目标是 N1 的 eMMC，然后开始安装。
4. 安装完成后执行关机，拔掉 U 盘，再重新上电。
5. 首次从 eMMC 启动后立即设置 root 密码并生成 OpenWrt 配置备份。

不要在掉电风险、镜像型号不明或 USB 启动尚未验证时写 eMMC。

## 按现有家庭旁路由方式配置

你的现有拓扑可继续使用：

- 小米主路由：`192.168.3.1`，继续提供 DHCP。
- N1：`192.168.3.2`。
- N1 自身的网关和 DNS：`192.168.3.1`。
- 需要代理的客户端由 DHCP 或手动设置使用 `192.168.3.2` 作为网关/DNS。
- NAS、IoT 等需要稳定端口映射的设备继续使用 `192.168.3.1` 作为网关，避免回程经过旁路由。

OpenClash 建议继续沿用 M2 的规则模式、Fake-IP、关闭 IPv6和开机自启设置。导入配置时不要保留 `nameserver: system`，以免产生 DNS 循环；确认配置文件存在并通过检查后，再修改客户端网关。
