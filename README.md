# Lenovo-IX2-Debian
Installing Debian on Lenovo IX2 NAS to replace the stock firmware

Since I have a Lenovo IX2 device , but the device is EOL with no security fixed , 
so I decided to find replacement of the stock firmware.

I found information from 
https://kiljan.org/2021/04/22/installing-arch-linux-arm-on-an-iomega-ix2-200/ 
and try the instructions listed , but the device could not boot as expected.

After more research , I found the following URL which listed another installation process.
https://forum.doozan.com/read.php?2,12096
and getting same result which stucks "Starting kernel......"

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
3) Prepare a USB drive with at-least 8G and format it as MBR with 2 partitions (should be done with another linux box)<br> 
    dd if=/dev/zero of=/dev/sdX bs=1M count=16 (Erase whole USB drive)<br>
    fdisk /dev/sdX ( or other tools which you know how to use )<br>
        o ( make MBR )<br>
        n ( new first partition for /boot )<br>
        p ( Primary )<br>
          ( press enter to accept the first sector as the partition start )<br>
        +256M ( for the BOOT partition )<br>
        n ( the second partition - for the rootfs / )<br>
        p ( Primary type )<br> 
          ( press enter to accept the first sector as the partition start )<br>
          ( press enter to accept the end sector as the partition ends with all space )<br>
        w ( write changes to the partition table )<br>
4) Prepare the Debian USB boot media<br>  
        mkfs.ext2 /dev/sdX1<br>
        mkfs.ext4 /dev/sdX2<br>
        e2label /dev/sdX1 BOOT<br>
        e2label /dev/sdX2 rootfs<br>
        mount /dev/sdX2 /mnt<br>
        mkdir /mnt/boot<br>
        mount /dev/sdX1 /mnt/boot<br>
        cd /mnt<br> 
        bsdtar -xpf ~/Debian-5.13.6-kirkwood-tld-1-rootfs-bodhi.tar.bz2 #( The file location which you have downloaded )<br> 
        cd /mnt/boot<br>
        cp -a zImage-5.13.6-kirkwood-tld-1  zImage.fdt<br>
        cat dts/kirkwood-lenovo-ix2-ng.dtb >> zImage.fdt #( If you stucks at "Starting kernel......" when booting , beware of this file )<br>
        mv uImage uImage.orig<br>
        mkimage -A arm -O linux -T kernel -C none -a 0x00008000 -e 0x00008000 -n Linux-5.13.6-kirkwood-tld-1 -d zImage.fdt  uImage<br>
        mkimage -A arm -O linux -T ramdisk -C gzip -a 0x00000000 -e 0x00000000 -n initramfs-5.13.6-kirkwood-tld-1 -d initrd.img-5.13.6-kirkwood-tld-1 uInitrd<br>
        cat << 'EOF' >> /mnt/etc/fstab<br>
        LABEL=BOOT    /boot   ext2  defaults,noatime  0  0<br>
        LABEL=rootfs  /       ext4  defaults,noatime  0  0<br>
        EOF<br>  
        cd ~<br>  
        sync #( Make sure the USB drive have written all files instead of memory cache )<br> 
        sync<br> 
        sync<br>  
        umount -R /mnt<br>  
        # Lenovo IX2 is different with IOMEGA IX2 , you need not to change ETH0 to ETH1<br> 
        # and you may config the /mnt/etc/network/interfaces to have a static IP address as you wish<br>   
5) Prepare the UBOOT ENV of the device , the process is assume you connect it via TTL serial , <br>if you connect via SSH , may have a little difference ( fw_setenv vs setenv )<br> 
        Connect the TTL : https://kiljan.org/2021/04/22/installing-arch-linux-arm-on-an-iomega-ix2-200/ <br>( Thanks for great information by Sven Kiljan )<br> 
        boot the device<br>( I suggest to boot the device with no hard drive and no usb , it will boot failure and left you to the uboot console , or you may hit "Enter" during the boot process to interrupt the boot )<br> 
        Here are the process of UBOOT:<br> 
          printenv ( Copy to a txt file which backup the existing ENV )<br> 
          setenv usb_set_bootargs 'setenv bootargs console=ttyS0,115200 root=LABEL=rootfs rootdelay=10 earlyprintk=serial'<br> 
          setenv load_uimage 'ext2load usb 0:1 0x800000 /uImage'<br> 
          setenv load_uinitrd 'ext2load usb 0:1 0x2100000 /uInitrd'<br> 
          setenv usb_boot 'mw 0x800000 0 1; run load_uimage; run load_uinitrd; bootm 0x800000 0x2100000'<br> 
          setenv usb_bootcmd 'run usb_set_bootargs; run usb_boot'<br> 
          setenv bootcmd 'usb reset; run usb_bootcmd; usb stop; reset'<br> 
          saveenv<br> 
6) Power off the device , plug the USB and do not insert any hard drive , <br>and start the device again , if there has no errors , it should boot to the Debian which you installed with the USB drive , <br>you may login the device via ssh with root/root. ( I think you should know to find the IP address with your DHCP or you have already set the static IP address :) , or using the TTL to login to the system )

7) Enjoy ~ ( You may copy the files to hard drive and boot from hard drive , but you should modify your own ENV values. If a hard drive >2TB you should use GPT for it . My drives are 4TB and I want to keep the NAS drive as most simple raid 1 so I just leave the USB as the boot media always. ie: if you installed the boot files to hard drive , raid 1 could only apply with partitions instead of hard drive level which I don't want it to be )
        
