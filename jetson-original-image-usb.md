# Creating jetson usb stick

How to boot jetson if sd card reader is dead.

With sd, my jetson's boot always fails, hanging on splash. uart output shows "Card did not respond to voltage select!" no matter what card I use. This board use to work.
Attempting a normal recovery flash also fails with sd related errors. I think the sd reader is dead. So I need to create a bootable usb or ssd.

### install sdkmanager
I downloaded docker image, loaded it and renamed it to sdkmanager:latest 

check its working:
```bash
docker run -it --rm sdkmanager:latest --ver
```

### install the jetpack sdk
you only need to install this the first time

generate command

```bash
docker run -it --rm   --privileged   -v /dev/bus/usb:/dev/bus/usb   -v /dev:/dev   --network host   -v $HOME/nvidia-sdkm:/home/nvidia/nvidia   sdkmanager:latest   --cli --query interactive --archived-versions --stay-logged-in true
```

 I modified the command a bit, removed --host since we dont want host sdk

```bash
docker run -it --rm   --privileged   -v /dev/bus/usb:/dev/bus/usb   -v /dev:/dev   --network host   -v $HOME/nvidia-sdkm:/home/nvidia/nvidia   sdkmanager:latest   --cli   --action install   --login-type devzone   --product Jetson   --target-os Linux   --version 4.6.5  --target JETSON_NANO_TARGETS   --flash   --archived-versions true   --license accept   --stay-logged-in true   --exit-on-finish
```
Choose sdk components you want.
Cancel and dont flash (you can probably modify the command to avoid it asking to flash or needing to plug in jetson in recovery idk)

### flash qspi in recovery mode
you only need to do this the first time. after that boot loader should be good to go.

turn off jetson. plug in micro usb to laptop. short recovery pins (3,4 of j40 on a02 board) with jumper. plug in barrel connector to jetson, remove short.

```bash
cd ~/nvidia-sdkm/nvidia_sdk/JetPack_4.6.5_Linux_JETSON_NANO_TARGETS/Linux_for_Tegra/
```

```bash
sudo ./flash.sh jetson-nano-qspi internal
```


### plug in usb

```bash
lsblk
```

should see something like:
```text
sda      28.6G
└─sda1   28.6G  /media/user/XXXX
```

unmount it:

```bash
sudo umount /dev/sda1 # i needed sudo umount -l /media/user/Sandisk32
```

if theres multiple partitions:

```bash
sudo umount /dev/sda*
```


### create paritions table

```bash
sudo parted /dev/sda --script mklabel gpt
sudo parted /dev/sda --script mkpart APP ext4 1MiB 100%
```

### format

```bash
sudo mkfs.ext4 /dev/sda1 -L rootfs
```

```bash
lsblk
```

should see something like:
```text
sda      28.7G
└─sda1   28.7G
```

confirm filesystem

```bash
sudo blkid /dev/sda1
```

### mount for rootfs copy

```bash
sudo mkdir -p /mnt/jetson-usb
sudo mount /dev/sda1 /mnt/jetson-usb
ls /mnt/jetson-usb
```


```bash
cd ~/nvidia-sdkm/nvidia_sdk/JetPack_4.6.5_Linux_JETSON_NANO_TARGETS/Linux_for_Tegra/
```

if you've already done this before, you shouldn't need to apply binaries again.
to check if binaries ahve already been applied:

```bash
ls rootfs/usr/lib/aarch64-linux-gnu/tegra
ls rootfs/etc/nv_tegra_release
```

if they havent, apply them

```bash
sudo ./apply_binaries.sh
```

### copy rootfs to usb and setup boot config


```bash
sudo rsync -axHAWX --numeric-ids rootfs/ /mnt/jetson-usb
```

you might instead want `--info=progress2` and `--exclude=/proc` i didnt use them though.

```bash
sudo blkid /dev/sda1 # use this partuuid
```

```bash
sudo vim /mnt/jetson-usb/boot/extlinux/extlinux.conf
```

change this line
```text
APPEND ${cbootargs} quiet
```

replace the uuid below with the partuuid from blkid output. root=UUID wont work because cbootargs sets root=PARTUUID which overides it.
```text
APPEND ${cbootargs} root=PARTUUID=63e19e2e-4fc9-47a9-a0b3-5e338e0d672b rootwait rootfstype=ext4 quiet
```


```bash
sync # takes a few mins
sudo umount /mnt/jetson-usb
```

### add user
my boot attempt at this point worked, but the oem config wizard errored and i couldnt login to desktop. I needed to add user manually:

remount the usb and chroot into it
```bash
sudo mount /dev/sda1 /mnt/jetson-usb

sudo mount --bind /dev  /mnt/jetson-usb/dev
sudo mount --bind /proc /mnt/jetson-usb/proc
sudo mount --bind /sys  /mnt/jetson-usb/sys

sudo chroot /mnt/jetson-usb
```
add user (choose a password, leave all details blank is fine)
```bash
adduser jetson
```
add to sudo
```bash
usermod -aG sudo jetson
```

optional steps i didnt follow so idk if useful.
```
echo jetson > /etc/hostname
cat /etc/hosts # ensure "127.0.1.1 jetson" exists
# disable oem config
systemctl disable oem-config
rm -rf /var/lib/oem-config
```

finally exit and unmount
```bash
exit

sync
sudo umount /mnt/jetson-usb/dev
sudo umount /mnt/jetson-usb/proc
sudo umount /mnt/jetson-usb/sys
sudo umount /mnt/jetson-usb
```

plug in the usb to the jetson, and it should boot now from usb