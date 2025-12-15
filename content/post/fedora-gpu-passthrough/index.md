---
title: Fedora qemu/kvm gpu 直通(passthrough) 筆記
description: 來自舊文章移植
slug: fedora-gpu-passthrough
date: 2025-11-28 00:00:00+0800
categories:
    - 筆記
tags:
    - linux
    - 筆記
    - 折騰
    - kvm
---

> 該文章來源: https://hackmd.io/@U9m9FarsR0SQGVuSmTJNiw/HkBjGlwuq  
> 日期: 2022/6/3

原本打算用 Windows 的 docker 開遊戲伺服器就好，不過總會覺得效能很差，所以想用原生 Linux 開伺服器。  
由於要用同一台電腦開 Windows 玩遊戲，所以雙開(dual boot)就不適合我的需求。  
於是乎，用 vm 並且 gpu passthrough 成了唯一選項(大概)，但也是最難搞的一個，因為 Linux 發行版、版本、硬體的不同，網路上各種不同方法，許多都因為是很久以前版本的方法所以不管用，更不用說大多數都英文(雖然大部分都看得懂)，導致我弄了整整快三天了吧...

這篇作為筆記，記錄我實作的過程，因為發行版、硬體的不同，過程會有所不同，僅供參考。

## 使用硬體

- CPU: `intel i7-10700kf`
- GPU1: `nvidia rtx3060 ti` (作為 guest, Windows)
- GPU2: `nvidia gt710` (作為 host, Linux)
- Mother: `msi z590 torpedo`

## BIOS

開啟 intel VT-D 功能，我的 bios 預設是關閉的，位於 Overclocking\CPU Features，需打開(Enabled)

![](/cdn-assets/fedora-gpu-passthrough/9wvuHwP.webp)

## 安裝 nvidia 驅動

要先有 nvidia 的驅動，否則進 vm 沒多久會崩潰
Fedora 安裝方式見 https://rpmfusion.org/Howto/NVIDIA

## 安裝虛擬套件

```bash
sudo yum -y install @virtualization
sudo systemctl start libvirtd
sudo systemctl enable libvirtd
```

若不想每次開 vm 都要輸入密碼的話:

```bash
sudo usermod -aG libvirt $USER
```

## 取得 pci id

```bash
lspci -nn | grep NVIDIA
```

找到需要直通的 gpu，後面的 [] 裡面是等下需要用到的 pci id，VGA 跟 Audio 都需要用到，例如我的 01:00.0, 01:00.1，其 id 為 10de:2489, 10de:228b

![](/cdn-assets/fedora-gpu-passthrough/MAN6EXf.webp)

## grep 設定

```bash
sudo nano /etc/default/grub
```

在 GRUB_CMDLINE_LINUX="... 後面加入 `intel_iommu=on iommu=pt vfio-pci.ids=...,... rd.driver.pre=vfio_pci` (...,... 為 pci id)

![](/cdn-assets/fedora-gpu-passthrough/OGxy0oG.webp)

儲存設定:

```bash
sudo grub2-mkconfig -o /boot/efi/EFI/fedora/grub.cfg
```

## 設定載入 vfio-pci 模組

```bash
sudo nano /etc/dracut.conf.d/10-vfio.conf
```

加入以下並儲存: `add_drivers+=" vfio_pci vfio vfio_iommu_type1 vfio_virqfd "`

```bash
sudo dracut --force /boot/initramfs-$(uname -r).img
```

重啟電腦

## 檢查是否設定成功

### 檢查 iommu 是否啟動

```bash
dmesg | grep -i -e DMAR -e IOMMU
```

看是否有 "IOMMU enabled"

### 檢查 vfio 是否啟用

```bash
dmesg | grep -i vfio
```

查看是否有出現需要的id

![](/cdn-assets/fedora-gpu-passthrough/Sg74lLI.webp)

此外，可以透過以下指令檢查該 gpu 是否使用 vfio-pci (Kernel driver in use: vfio-pci)

```bash
lspci -nnk -d (id)
```

![](/cdn-assets/fedora-gpu-passthrough/ihulRB4.webp)

## 設定 vm

基本上在 virt-manager 上新增 gpu, audio 的 pci 就完成 passthrough 了

![](/cdn-assets/fedora-gpu-passthrough/j7YfkNU.webp)
![](/cdn-assets/fedora-gpu-passthrough/YUCu9yl.webp)

### 如果常用到一半藍屏的話

可以試試

```bash
sudo nano /etc/modprobe.d/kvm.conf
```

增加 `options kvm ignore_msrs=1`

## 個人遇到的問題

(個人推測)  
由於我要直通的卡插在第一個 pcie 的槽，開機時會預設使用一下這張卡，然後再切另一張。

### 設定開機後重設 gpu

主要遇到問題是要直通的卡(pci 設備)需要重設，否則虛擬機(vm)開啟時會崩潰，然後就開不起來。

建立一個腳本，記得 `chmod +x pcie_hot_reset.sh`

```
#!/bin/bash

dev=$1

if [ -z "$dev" ]; then
    echo "Error: no device specified"
    exit 1
fi

if [ ! -e "/sys/bus/pci/devices/$dev" ]; then
    dev="0000:$dev"
fi

if [ ! -e "/sys/bus/pci/devices/$dev" ]; then
    echo "Error: device $dev not found"
    exit 1
fi

echo "Removing $dev..."

echo 1 > "/sys/bus/pci/devices/$dev/remove"

echo "Rescanning bus..."

echo 1 > "/sys/bus/pci/rescan"
```

然後我要設定開機時自動使用該腳本

```bash
sudo nano /etc/systemd/system/gpu_reset@.service
```

內容: (ExecStart 自行修改)

```
[Unit]
Description=GPU reset at startup.

[Service]
Type=simple
ExecStart=/bin/bash /home/user/pcie_hot_reset.sh %i

[Install]
WantedBy=multi-user.target
```

```bash
sudo chmod 644 /etc/systemd/system/gpu_reset@.service
sudo systemctl enable gpu_reset@01:00.0.service gpu_reset@01:00.1.service
```

### 設定 X11 預設顯示 GPU

我的 X11 會預設使用要直通的卡，導致當那張卡有接螢幕時，會連 Linux 登入畫面都無法顯示，所以我用以下設定預設顯示 GPU 為 gt710 那張

```bash
sudo nano /etc/X11/xorg.conf
```

加入

```
Section "Device"
    Identifier     "Device1"
    Driver         "nvidia"
    VendorName     "NVIDIA Corporation"
    BusID          "PCI:3:0:0"
EndSection
```

## 參考

- pcie hot reset: https://alexforencich.com/wiki/en/pcie/hot-reset-linux
- How to Reset/Cycle Power to a PCIe Device?: https://unix.stackexchange.com/questions/73908/how-to-reset-cycle-power-to-a-pcie-device
- PCI passthrough via OVMF: https://wiki.archlinux.org/title/PCI_passthrough_via_OVMF
- Single GPU Passthrough Guide - Fedora 34: https://www.reddit.com/r/linux_gaming/comments/o73gwf/single_gpu_passthrough_guide_fedora_34/
- Run a Script on Startup in Linux: https://www.baeldung.com/linux/run-script-on-startup
- Unknown PCI header type ‘127’ for device ‘0000:01:00.0’: https://forum.level1techs.com/t/unknown-pci-header-type-127-for-device-000000-0/180125
