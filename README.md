## WIP OpenWRT Firmware for Askey RT4230W (RAC2V1K)

First and foremost, all information, sources and binaries are provided AS IS. I take no responsibility if you cause any damage to your hardware by following this guide. Proceed at your OWN RISK.

This is a general guide. Prior knowledge and understanding of bootloaders, TFTP, UART and networking is required.

Currently, only a pre-built image is available. Over the course of the next few days, I will be uploading the many different patches needed to build your own image. Many thanks to **efsg** and **lmore377** on the OpenWRT forums. This has been made possible with the invaluable information and resources they've provided.

**The pre-built image is for Revision 6 boards only.** You can check which revision of the board you have by running the command ``cat /proc/device-tree/model`` in shell on the device, either through SSH or serial connection. To obtain shell and root access, you will need to restore configuration files provided by **efsg** in his post [here](https://forum.openwrt.org/t/askey-rac2v1k-support/15830/17). I recommend you read the entire topic and the README file he provided in the archive.

Before you proceed, a working UART serial connection to the board must be established. Information on the location of the UART interface and pinout is available on the active OpenWRT topic [here](https://forum.openwrt.org/t/askey-rac2v1k-support/15830). I have not yet figured out a way to enter TFTP recovery mode using any of the exposed buttons.

------
Using PuTTY or any other terminal emulator with serial capibilities, you will need to stop the autoboot sequence. To do this, press **SPACE** repeatedly after you have plugged in the power connection to the board. If you've succeeded, you will be at the U-Boot IPQ prompt.

```
...
eth0, eth1
Hit space key to stop autoboot:  0
(IPQ) #
```
From here, you will perform the following steps in order: TFTP the OpenWRT image into memory, erase the UBI rootfs partition and write the OpenWRT image from memory to NAND. If you do not feel confident proceeding, **STOP** here. No modifications have been made at this point.

Connect an ethernet cable between your machine and one of the LAN ports on the board. Set a fixed IP address, Gateway and Subnet mask for the connection. In this guide, I will be using 192.168.0.100 as the IP, 192.168.0.1 as Gateway and 255.255.255.0 as the Subnet mask.

You will need to set up a TFTP server. If you're running Windows, [Ttfpd32](http://tftpd32.jounin.net/tftpd32_download.html) is a good choice. Once you have your TFTP server up and running, you need to place the OpenWRT image into the directory being served. Rename the image to **C0A80001.img**. In case of Tftp32, you will also need to change the interface that it is listening on. Select the interface with the fixed IP address you chose.

At this point, you will be working entirely within the U-Boot IPQ prompt. You need to set-up the networking enviornment variables. To do this, enter the two commands below. Make sure to change the values if you chose a different fixed IP address and Gateway.

```
...
(IPQ) # setenv ipaddr 192.168.0.1
(IPQ) # setenv serverip 192.168.0.100
(IPQ) #
```

------
The next step is to have the bootloader load the contents of **C0A80001.img** from your TFTP server into memory. You do this by entering the `tftpboot` command. Don't let the name of the command fool you, it does not perform any booting. If you've set your networking and TFTP server up correctly, you should see output similar to the one below.

```
...
(IPQ) # tftpboot
Mac0 unit failed
*** Warning: no boot file name; using 'C0A80001.img'
Using eth1 device
TFTP from server 192.168.0.100; our IP address is 192.168.0.1
Filename 'C0A80001.img'.
Load address: 0x44000000
Loading: #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         #################################################################
         ############################
done
Bytes transferred = 7077888 (6c0000 hex)
(IPQ) #
```

If you do not see similar output, do **NOT** proceed. Check your wiring and networking settings. Then try again.

------
Once the OpenWRT image is loaded into memory, proceed by erasing the NAND rootfs partition. This partition is located at offset 0x2400000 with size of 0x8000000. To erase it, you will need to pass the offset and size to the `nand erase` command. Triple check your arguments before pressing enter, you don't want to erase any other partitions. Again, you should see output same to the one below.

```
...
(IPQ) # nand erase 0x2400000 0x8000000

NAND erase: device 0 offset 0x2400000, size 0x8000000
Erasing at 0xa3e0000 -- 100% complete.
OK
(IPQ) #
```

------
In the final step, you will write the contents of the image in memory to the NAND rootfs partition. To do this, use the `nand write` command with the address of the image in memory, offset of the partition and size of the image. The address of the image is provided as **Load address** in the output of the `tftpboot` command, but should always be 0x44000000. The offset of the rootfs partition is 0x2400000. The size of the OpenWRT image can be seen next to **Bytes transferred** in the output of `tfpboot` command (use the hex value), in this case 0x6c0000.


```
...
(IPQ) # nand write 0x44000000 0x2400000 0x6c0000

NAND write: device 0 offset 0x2400000, size 0x6c0000
 7077888 bytes written: OK
(IPQ) # 
```

If your output is the same, you have succesfully written the OpenWRT image to the NAND rootfs partition. Proceed by power cycling the board. If all went well, you will see OpenWRT kernel booting up. You can now remove your fixed IP, Gateway and Subnet mask from the network connection. DHCP built into OpenWRT will auto-assign an IP for you. LuCI is built into the image and should be accessible via a web browser.
