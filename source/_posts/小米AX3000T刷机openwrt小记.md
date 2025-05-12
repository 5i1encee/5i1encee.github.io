---
title: 小米 AX3000T 刷机 openwrt 小记
categories:
- Note&Summary
tag:
- 计网
- openwrt
date: 2025-04-21 23:50:11
excerpt: “不务正业”一下
index_img: /img/wallhaven-yq7gox.jpg
banner_img: /img/wallhaven-yq7gox.jpg
---

# 小米 AX3000T 刷机 openwrt 小记

# 前言

这两天在思考如何在寝室局域网之外远程连接寝室设备，以满足我有时候出门不方便带电脑想要用手机远程连接电脑操作的需求。我选定的是 sunshine+moonlight 串流方案，然而该方案本身并不支持远程连接，只适用于局域网连接，所以我需要通过内网穿透或 VPN 构建虚拟局域网来让我的电脑和手机设备处于同一个局域网环境内。先尝试了云服务器之前搭建好的 openVPN 服务，发现限于带宽和性能，连接质量并不理想，于是我想尝试在寝室路由器上使用 wireguard 构建隧道实现内网穿透。

好了，不管怎么说，原厂的可操作性太差了，先刷个 openwrt 耍耍 0v0

# 设备型号

路由器型号：小米 AX3000T

系统 ROM 版本：1.0.98？（好像是 9 几，忘了）

# 实现过程

## 启用 SSH

### 固件降级

为了能够通过 ssh 连接路由器设备控制并操作，需要先降低 ROM 版本换成易于攻击的 1.0.47 版本，从而启用 ssh。

1.0.47 固件链接：[小米官方 CDN](https://cdn.cnbj1.fds.api.mi-img.com/xiaoqiang/rom/rd03/miwifi_rd03_firmware_ef0ee_1.0.47.bin)

小米官方刷机工具：[https://www1.miwifi.com/miwifi_download.html](https://www1.miwifi.com/miwifi_download.html)

下载编译好的固件后先打开刷机工具，用网线连接电脑和路由器 lan 口，选择要上传的文件的路径，点击下一步

![](/img/小米AX3000T刷机openwrt小记/JsDKbvdoVomz1bxsbYscOWv4nab.png)

选择网卡，需要区分 VMware 设备的虚拟网卡等其他网卡，必要的时候可以暂时禁用其他网卡，必须选择物理网卡，一般名称带有“以太网”。而后点击下一步

![](/img/小米AX3000T刷机openwrt小记/MSVEbD2NHoTV7ExS90EcsFZVnpb.png)

根据提示先给路由器断电，找一根卡针或者牙签按住路由器上的 reset 键，然后接通电源，等到橙色灯闪烁之后再松开 reset 键。一开始下方会出现“可以开始刷机操作”的提示，路由器橙色灯闪烁，然后路由器发送请求，电脑返回文件，工具自动刷入 ROM，完成之后变蓝灯闪烁，刷机成功，断电重启路由器。

我遇到的问题是路由器的请求迟迟没有出现，指示灯一直在橙灯长亮、橙灯闪烁之间循环，后来发现需要先暂时关闭 windows 的防火墙，否则电脑不会接收路由器的请求。

![](/img/小米AX3000T刷机openwrt小记/BcSQb7k5So2YtSx0i04cCuH4nrc.png)

### 修改路由器配置启用 ssh

![](/img/小米AX3000T刷机openwrt小记/Zxk4bIl24oM7mJx1XA1c0zKBnye.png)

需使用 cmd 依次执行以下命令，`<stok>` 处填入访问路由器管理页面的 `stok` 字段内容，使用 PowerShell 解析参数错误会报错。每次都要返回 `{"code":0}` 才算成功。

```
curl -X POST http://192.168.31.1/cgi-bin/luci/;stok=<stok>/api/misystem/arn_switch -d "open=1&model=1&level=%0Anvram%20set%20ssh_en%3D1%0A"
curl -X POST http://192.168.31.1/cgi-bin/luci/;stok=<stok>/api/misystem/arn_switch -d "open=1&model=1&level=%0Anvram%20commit%0A"
curl -X POST http://192.168.31.1/cgi-bin/luci/;stok=<stok>/api/misystem/arn_switch -d "open=1&model=1&level=%0Ased%20-i%20's%2Fchannel%3D.*%2Fchannel%3D%22debug%22%2Fg'%20%2Fetc%2Finit.d%2Fdropbear%0A"
curl -X POST http://192.168.31.1/cgi-bin/luci/;stok=<stok>/api/misystem/arn_switch -d "open=1&model=1&level=%0A%2Fetc%2Finit.d%2Fdropbear%20start%0A"
```

### ssh 登入

而后尝试直接 ssh 登录，报错，需要用 ssh-rsa 协商

![](/img/小米AX3000T刷机openwrt小记/VuDJbZgDfosQQOxjhBhcfWZenqb.png)

改用如下命令

```shell
ssh -o HostKeyAlgorithms=+ssh-rsa -o PubkeyAcceptedKeyTypes=+ssh-rsa root@192.168.31.1
```

登录密码需要用路由管理界面 SN 的全部内容放到 https://miwifi.dev/ssh 处理获取，注意 Ctrl+v 无法使用，右键粘贴

![](/img/小米AX3000T刷机openwrt小记/WqFGbsLi1oqKktxWV99cESHVnOh.png)

![](/img/小米AX3000T刷机openwrt小记/MsNabZPz6o2VSSx54wecGYHnnDe.png)

## 刷入 U-Boot

U-Boot 全称 Universal Boot Loader，是一个开源的引导加载程序，广泛应用于嵌入式系统中。它的主要作用是在系统启动时初始化硬件设备，建立内存空间的映射图，加载操作系统内核到内存中并执行，为操作系统的启动提供必要的准备。

### 备份原固件

为了避免后续出现意外导致难以修复，先备份固件以防万一。`cat /proc/mtd` 查看所有分区

![](/img/小米AX3000T刷机openwrt小记/OKGzbOTXsoNZllxPVNMc3lMrnpc.png)

然后使用如下命令依次打包 1~12 共 12 个分区

{% note warning %}

**注意：**

请不要偷懒想要一次性完成下面所有操作，由于路由器的 ROM 通常较小，一次性大量的备份操作会让 ROM “爆掉”，最好每一个都单独备份，且备份完成后就下载并删除。这是来自 zkz098 的提醒。

其中 mtd0 似乎是众多分区的汇总，无需重复备份，而我在好奇心的驱使下尝试在完成其他所有分区的备份后对其进行备份，导致路由器崩溃重启......

那么上述问题一般而言的应对方法是：重启路由器后重复“修改路由器配置启用 ssh”到“ssh 登录”步骤

{% endnote %}

```bash
dd if=/dev/mtd1 of=/tmp/BL2.bin
dd if=/dev/mtd2 of=/tmp/Nvram.bin
dd if=/dev/mtd3 of=/tmp/Bdate.bin
dd if=/dev/mtd4 of=/tmp/Factory.bin
dd if=/dev/mtd5 of=/tmp/FIP.bin
dd if=/dev/mtd6 of=/tmp/crash.bin
dd if=/dev/mtd7 of=/tmp/crash_log.bin
dd if=/dev/mtd8 of=/tmp/ubi.bin
dd if=/dev/mtd9 of=/tmp/ubi1.bin
dd if=/dev/mtd10 of=/tmp/overlay.bin
dd if=/dev/mtd11 of=/tmp/date.bin
dd if=/dev/mtd12 of=/tmp/KF.bin
```

单句执行效果如下

![](/img/小米AX3000T刷机openwrt小记/VIsgbXhqlogbaNxWxO1cX18snUg.png)

随后将打包好的 bin 文件传回电脑保存，路由器原本自带 netcat 但功能不完整，不好用，所以为了方便起见使用 WinSCP 对文件进行传输管理。

使用 WinSCP 建立连接，注意协议需选择 SCP，其他无法连接，密码即之前获取的密码。

![](/img/小米AX3000T刷机openwrt小记/Zba8brDVDoiCuzxNqlDcO0r7nEg.png)

![](/img/小米AX3000T刷机openwrt小记/WfW4bONy7oDyaJxmsR1ctn3ynls.png)

而后发生的就是我把路由器玩崩了，重新 ssh 登录时提示 SHA256 认证信息改变，该设备不受信任

![](/img/小米AX3000T刷机openwrt小记/WhJVb6SmTob9WwxVvwPcJfQ2nxe.png)

这是因为之前连接的时候保存了指纹信息，现在对不上了，用 `ssh-keygen -R 192.168.31.1` 清除原本保存着的 192.168.31.1 相关指纹即可。

![](/img/小米AX3000T刷机openwrt小记/HqXBbUnoJoOF9TxGUVccZA5Mntb.png)

### 刷入 U-Boot 固件

**确保备份完成后**，用 WinSCP 把固件文件传输到路由器上，执行以下命令将固件写入 FIP 分区

```bash
mtd write mt7981_ax3000t-fip-fixed-parts-multi-layout.bin FIP
```

![](/img/小米AX3000T刷机openwrt小记/V71ubshtDoNGQAxpnBBc4DvFnbf.png)

这步结束即完成了 U-Boot 固件的刷入

## 刷入 openwrt

接下来要用 U-Boot 加载 openwrt 的固件刷入

### 配置网络连接

目前网线仍然连接着电脑与路由器，由于 uboot 不具备 DHCP 能力，所以需要手动配置一下网卡来访问 uboot 的界面

将 IP 分配模式设为手动，选一个不会冲突的 IPv4 地址，掩码、网关如下，DNS 服务器选择国内能用的且不建议加密

![](/img/小米AX3000T刷机openwrt小记/GYCPbgH4VoFirSxHqgec5zzOnsc.png)

### 刷入 openwrt 固件

访问 192.168.1.1 的 U-Boot 页面，选择要上传的本地的 openwrt 固件，并选择相应的 `mtd layout`（QWRT 选 QWRT，immortalwrt 选 immortalwrt-w112，这里我选择的固件是 QWRT），而后点击 Upload。

![](/img/小米AX3000T刷机openwrt小记/EATQbRa6HokI28xlenxctdQanPd.png)

上传完成后再点击 Update，在这个过程中路由器会变为橙灯，随后橙灯快速闪烁，再变为蓝灯，此时刷入成功。

![](/img/小米AX3000T刷机openwrt小记/G8bsbfzEgoUhXlxL4XQcBAVgnDg.png)

如果观察到路由器的橙灯慢速闪烁 (约 5 秒一次)，此时请等待半分钟，如果仍然保持此状态，可以按照前文重新进入 UBoot 了。这个闪烁意味着固件刷入失败，UBoot 会提示 UPDATE FAILED，建议更换固件并选择正确的 mtd layout 后重试

## 登录 openwrt

QWRT 的默认管理页面地址为 192.168.1.1，默认管理员账号密码 `root`、`password`

![](/img/小米AX3000T刷机openwrt小记/Sf8EbAJejoFXMXxsEANcuxDhn1W.png)

完成，帅~

# 如何刷回原厂？

来自 zkz098 的实践经验：

> 先恢复原厂 UBoot 后使用小米救砖工具刷入固件
> 找到前文的备份文件中的 `FIP.bin` ，使用 SSH 工具上传到 /tmp 目录下，随后：`mtd write FIP.bin FIP` 等待命令完成 (15-30 秒左右) 后断电重启，此时按照前文的降级方法刷入原厂 1.0.47 固件 (可能需要多次刷入)
> 如果刷入成功，橙灯会常亮一会，然后橙灯闪烁，此时可以按照正常新路由器的方法，连接到无密码默认 SSID 中进入开始使用

# 参考资料：

zkz098 的文章，所需固件可以从下方文章中获取，或者网上寻找，或者自行编译，这篇是主要参考，很详细：

[https://www.kaitaku.xyz/misc/ax3000t-openwrt/](https://www.kaitaku.xyz/misc/ax3000t-openwrt/)

固件降级示范：

[https://www.bilibili.com/video/BV1jZ421875V/?vd_source=ae44e9df2e6bb265e83888153930e885](https://www.bilibili.com/video/BV1jZ421875V/?vd_source=ae44e9df2e6bb265e83888153930e885)
