# Getting armbian 23 working

The archived images supposedly boot fine
https://rsync.armbian.com/oldarchive/jetson-nano/archive/ 	
Armbian_23.8.1_Jetson-nano_bookworm_current_6.1.50.img.xz 

But it didnt boot for me. Might be because I'm trying to boot from usb not sd card. Boot logs showed it couldnt find the boot drive beccause usb driver wasnt being brought up.

Mount and edit grub:
``` bash
sudo mkdir -p /mnt/armbian
lsblk
sudo mount /dev/sda2 /mnt/armbian
sudo vim /mnt/armbian/boot/grub/grub.cfg
```

Add `modprobe.blacklist=onboard_usb_hub` to the kernel command line params.

eg:
`linux /boot/vmlinuz-6.1.50-current-media root=UUID=e9bb94bf-b334-4855-837c-feaa7e517849 ro console=ttyS0,115200n8 console=tty0 rw no_console_suspend consoleblank=0 fsck.fix=yes fsck.repair=yes net.ifnames=0 splash plymouth.ignore-serial-consoles modprobe.blacklist=onboard_usb_hub plymouth.enable=0 splash=0`

``` bash
sync
sudo umount /mnt/armbian
sudo eject /dev/sda
```


After this it boots fine. Set up via uart. Hdmi was working when I used the minimal image, showed a shell on my monitor.