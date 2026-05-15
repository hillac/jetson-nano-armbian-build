# Getting armbian 26 working on jetson nano 4gb

First, since I'm booting from usb, follow instruction in the jetson-original-image-usb.md to flash flash qspi (only if this nano has never booted a JetPack >=4.5 image from nvidia, otherwise this can be skipped). This only needs to be done once and never again.

Get armbian image from https://github.com/hillac/build-armbian/releases/tag/26.2.7

Flash to drive

```bash
xzcat Armbian-unofficial_26.2.7_Jetson-nano_trixie_current_6.18.29.img.xz | sudo dd of=/dev/sda bs=1M status=progress conv=fsync
```

Mount drive
``` bash
sudo mkdir -p /mnt/armbian
lsblk
sudo mount /dev/sda2 /mnt/armbian
```

Decompile DTB

``` bash
dtc -I dtb -O dts -o 6.18.dts /mnt/armbian/boot/armbian-dtb-6.18.29-current-arm64
```

Find the venc block in 6.18.dts
``` dts
venc {
        clocks = <0x03 0xe4 0x03 0x34>;
        resets = <0x0d 0x0b 0x03 0x14 0x03 0x34>;
        #power-domain-cells = <0x00>;
        phandle = <0x0f>;
};
```

Edit it to remove reset dependency
``` dts
venc {
        clocks = <0x03 0xe4 0x03 0x34>;
        resets = <0x03 0x14 0x03 0x34>;
        #power-domain-cells = <0x00>;
        phandle = <0x0f>;
};
```

Recompile

``` bash
dtc -I dts -O dtb -o armbian-dtb-6.18.29-current-arm64.patched 6.18.dts
```

Backup old dtb and copy patched one in

``` bash
sudo cp -a /mnt/armbian/boot/armbian-dtb-6.18.29-current-arm64 /mnt/armbian/boot/armbian-dtb-6.18.29-current-arm64.bak

sudo cp armbian-dtb-6.18.29-current-arm64.patched /mnt/armbian/boot/armbian-dtb-6.18.29-current-arm64
```

Edit grub linux kernel args. This is optional, but I wanted to silence some pci errors from wifi card and add rootwait.

``` bash
sudo vim /mnt/armbian/boot/grub/grub.cfg
```

Change config to: `ro efi=noruntime console=ttyS0,115200n8 rootwait pci=noaer`

eg: `linux /boot/vmlinuz-6.18.29-current-arm64 root=UUID=569c1acb-992e-4935-9c18-606e7abddcba ro efi=noruntime console=ttyS0,115200n8 rootwait pci=noaer`
(Obviously use the correct dtb file and root uuid)

Unoumt and eject:
``` bash
sync
sudo umount /mnt/armbian
sudo eject /dev/sda
```

This should now boot. HDMI does not work. Set up over uart.
Default login is login: root password: 1234

## Set up ethernet so I can connect over ssh
I thought eth would be ready to go when booting but for some reason I had to manually configure. Next time I'll try set this up as part of the drive setup so I dont need uart.

``` bash
nmcli con show
sudo nmcli con mod "Wired connection 1"   connection.interface-name end0   ipv4.addresses 192.168.50.2/24   ipv4.method manual   ipv6.method disabled   connection.autoconnect yes
# check
ip -br addr show end0
nmcli device status
```
