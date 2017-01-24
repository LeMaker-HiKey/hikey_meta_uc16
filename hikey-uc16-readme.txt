            Hikey Ubuntu Core 16 Guide


Prepare enviroment
------------------

install ubuntu xenial 16.04 and install below packages:

$ apt-get install snapcraft parted kpartx \
  dosfstools squashfs-tools android-tools-fsutils \
  gcc-aarch64-linux-gnu


Get source of hikey snaps
-------------------------

 - Export PATH to build UC16 image

$ export UC16_PATH=/path/snap

 - Get hikey meta:

$ cd $UC16_PATH
$ git clone git//github.com/LeMaker-HiKey/hikey_snappy_meta.git

 - Get hikey kernel snap:

$ cd $UC16_PATH
$ git clone git//github.com/LeMaker-HiKey/hikey_linux_uc16.git

 - Get hikey gadget snap:

$ cd $UC16_PATH
$ git clone git//github.com/LeMaker-HiKey/hikey_gadget_snap_uc16.git


Build and install u-boot to hikey
---------------------------------

Because ubuntu core only supports u-boot on arm platform, but hikey now uses UEFI as its default 
bootloader, we have to switch hikey to u-boot.

 - Get mainline u-boot source:

$ cd $UC16_PATH
$ git clone git//github.com/LeMaker-HiKey/hikey_u-boot.git u-boot
$ cd u-boot
$ git checkout -b hikey_snappy.test origin/hikey_snappy

 - Get other source code:

$ cd $UC16_PATH
$ git clone https://github.com/96boards/edk2.git
$ git clone https://github.com/96boards/arm-trusted-firmware.git
$ git clone https://github.com/96boards/burn-boot.git
$ git clone git://github.com/96boards/l-loader.git

And follow u-boot/board/hisilicon/hikey/README to compile and burn to hikey board.

Note: before generate l-load.bin. please modify generate_ptable.sh, replace "system" with
"writable" in the following line.

 sgdisk -n -E -t 9:8300 -u 9:FC56E345-2E8E-49AE-B2F8-5B9D263FE377 -c 9:"system" ${TEMP_FILE}

After all are done successfully, we have a hikey board running u-boot.


Build hikey kernel snap
-----------------------

 - Build the kernel snap:

$ cd $UC16_PATH
$ cd hikey_linux_uc16
$ git checkout -b snappy_v4.4-c28 origin/snappy_v4.4-c28
$ snapcraft --target-arch arm64 snap
$ ls hikey-kernel_4.4.0_arm64.snap
$ cp hikey-kernel_4.4.0_arm64.snap ../


Build hikey gadget snap
-----------------------

 - Build the gadget snap:

$ cd $UC16_PATH
$ rm -rf hikey_gadget_snap_uc16/.git
$ snapcraft snap hikey_gadget_snap_uc16

If we want to change uboot.env, please use mkenvimage to re-generate uboot.env
after modify hikey_gadget_snap_uc16/uboot.env.in:

$ ./u-boot/tools/mkenvimage -s 0x20000 -o hikey_gadget_snap_uc16/uboot.env \
  -r hikey_gadget_snap_uc16/uboot.env.in


Create a model assertion
------------------------

 - Create a json file:

$ cd $UC16_PATH
$ emacs hikey.json
write the content as below into the hikey.json
$ cat hikey.json
{
  "type": "model",
  "authority-id": "BgrPuBQAD1Q5FznHMaoJpRp9vOQ7dQZw",
  "brand-id": "BgrPuBQAD1Q5FznHMaoJpRp9vOQ7dQZw",
  "series": "16",
  "model": "hikey",
  "architecture": "arm64",
  "gadget": "hikey-snappy-gadget",
  "kernel": "hikey-kernel",  
  "timestamp": "2016-12-21T13:41:20+00:00"
}
About the explanation for the content, please refer to:
http://docs.ubuntu.com/core/en/guides/build-device/board-enablement#the-model-assertion

 - Sign the model assertion

$ cat hikey.json | snap sign -k whkey &> hikey.model
Please refer to the above URL to be familiar with the sign process.


Build Ubuntu Core image
------------------

$ cd $UC16_PATH
$ sudo ubuntu-image -c stable --image-size 4G --extra-snaps \
  ./hikey-snappy-gadget_16.04-1_arm64.snap --extra-snaps \
  ./hikey-kernel_4.4.0_arm64.snap  -o hikey-uc16.img hikey.model


Burn Ubuntu Core image to SD card
----------------------------

Insert a sd card to PC linux and burn UC16 image to SD card:

$ sudo dd if=hikey-uc16.img of=/dev/sdx bs=32M

Please replace /dev/sdx with your SD card device file.


Burn Ubuntu Core rootfs to EMMC FLASH
--------------------------------

 - Generate writable image from Ubuntu Core image:

$ cd $UC16_PATH
$ mkdir writable
$ sudo kpartx -av hikey-uc16.img
$ sudo mount /dev/mapper/loop0p2 writable
$ sudo make_ext4fs -L writable -l 1500M -s writable.img writable/
$ sudo umount writable
$ sudo kpartx -d hikey-uc16.img

Note:
 1. loop0p2 is the loop device of the second partition of hikey-uc16.img, it should
 be replaced with your loop device of kpartx mapping.


 - Burn writable image to EMMC FLASH:

$ cd $UC16_PATH
$ sudo ./burn-boot/hisi-idt.py -d /dev/ttyUSB1 --img1=l-loader/l-loader.bin
$ sudo fastboot flash ptable l-loader/ptable-linux-8g.img
$ sudo fastboot flash writable writable.img

