# Lenovo-IX2-Debian
Installing Debian on Lenovo IX2 NAS to replace the stock firmware

Since I have a Lenovo IX2 device , but the device is EOL with no security fixed , 
so I decided to find replacement of the stock firmware.

I found information from 
https://kiljan.org/2021/04/22/installing-arch-linux-arm-on-an-iomega-ix2-200/ 
and try the instructions listed , but the device could not boot as expected.

After more research , I found the following URL which listed another installation process.
https://forum.doozan.com/read.php?2,12096
and getting save result which stucks "Starting kernel......"

A few hours later , I booted the device successfully as expected ,
and I found my problem is related to DTS.
https://forum.doozan.com/read.php?2,127264

So if anyone try to hack a IX2 device , 
from my own experience , 
PLEASE MAKE SURE YOU HAVE THE RIGHT DTS FILE .
AND IOMEGA IX2 IS NOT SAME AS LENOVO IX2 .

Here is some experience of installation .

1) Please make sure you have a TTL device which could connect to the device if you lost the network connectivity ( ie: boot failure )
2) Always backup your files before your process , especially the ENV values of the device ( if you do not backup the ENV values , you may not able to restore the stock firmware )
3) If you stuck after loading the images , please make sure you have got the right DTS file with your device.

My installation process
1) Backup the ENV value ( use fw_printenv for SSH connections with existing firmware or use printenv via TTL at interrupted uboot process )
2) Download the updated Debian rootfs file from https://forum.doozan.com/read.php?2,12096 ( Which I use is Debian-5.13.6-kirkwood-tld-1-rootfs-bodhi.tar.bz2 ) , many thanks for Bodhi's great work of this.
3) Prepare a USB drive with at-least 8G and format it as MBR with 2 partitions (should be done with another linux box)
    dd if=/dev/zero of=/dev/sdX bs=1M count=16 (Erase whole USB drive)
    fdisk /dev/sdX ( or other tools which you know how to use )
        o ( make MBR )
        n ( new first partition for /boot )
        p ( Primary )
          ( press enter to accept the first sector as the partition start )
        +256M ( for the BOOT partition )
        n ( the second partition - for the rootfs / )
        p ( Primary type )
          ( press enter to accept the first sector as the partition start )
          ( press enter to accept the end sector as the partition ends with all space )
        w ( write changes to the partition table )
4) Prepare the Debian USB boot media
        mkfs.ext2 /dev/sdX1
        mkfs.ext4 /dev/sdX2
        e2label /dev/sdX1 BOOT
        e2label /dev/sdX2 rootfs
        mount /dev/sdb2 /mnt
        mkdir /mnt/boot
        mount /dev/sdb1 /mnt/boot
        cd /mnt
        bsdtar -xpf ~/Debian-5.13.6-kirkwood-tld-1-rootfs-bodhi.tar.bz2 #( The file location which you have downloaded ) 
        cd /mnt/boot
        cp -a zImage-5.13.6-kirkwood-tld-1  zImage.fdt
        cat dts/kirkwood-lenovo-ix2-ng.dtb >> zImage.fdt #( If you stucks at "Starting kernel......" when booting , beware of this file )
        mv uImage uImage.orig
        mkimage -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 -n Linux-5.13.6-kirkwood-tld-1 -d zImage.fdt  uImage
        mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n initramfs-5.13.6-kirkwood-tld-1 -d initrd.img-5.13.6-kirkwood-tld-1 uInitrd
        cat << 'EOF' >> /mnt/etc/fstab
        LABEL=BOOT    /boot   ext2  defaults,noatime  0  0
        LABEL=rootfs  /       ext2  defaults,noatime  0  0
        EOF
        cd ~
        sync #( Make sure the USB drive have written all files instead of memory cache )
        sync
        sync
        umount -R /mnt
        # Lenovo IX2 is different with IOMEGA IX2 , you need not to change ETH0 to ETH1 
        # and you may config the /mnt/etc/network/interfaces to have a static IP address as you wish.
5) Prepare the UBOOT ENV of the device , the process is assume you connect it via TTL serial , if you connect via SSH , may have a little difference ( fw_setenv vs setenv )
        Connect the TTL : https://kiljan.org/2021/04/22/installing-arch-linux-arm-on-an-iomega-ix2-200/ ( Thanks for great information by Sven Kiljan )
        boot the device ( I suggest to boot the device with no hard drive and no usb , it will boot failure and left you to the uboot console , or you may hit "Enter" during the boot process to interrupt the boot )
        Here are the process of UBOOT:
          printenv ( Copy to a txt file which backup the existing ENV )        
          setenv usb_set_bootargs 'setenv bootargs console=ttyS0,115200 root=LABEL=rootfs rootdelay=10 earlyprintk=serial'
          setenv load_uimage 'ext2load usb 0:1 0x800000 /uImage'
          setenv load_uinitrd 'ext2load usb 0:1 0x2100000 /uInitrd'
          setenv usb_boot 'mw 0x800000 0 1; run load_uimage; run load_uinitrd; bootm 0x800000 0x2100000'
          setenv usb_bootcmd 'run usb_set_bootargs; run usb_boot'
          setenv bootcmd 'usb reset; run usb_bootcmd; usb stop; reset'
          saveenv
6) Power off the device , plug the USB and do not insert any hard drive , and start the device again , if there has no errors , it should boot to the Debian which you installed with the USB drive , you may login the device via ssh with root/root. ( I think you should know to find the IP address with your DHCP or you have already set the static IP address :))

7) Enjoy ~ ( You may copy the files to hard drive and boot from hard drive , but you should modify your own ENV values. )
        
