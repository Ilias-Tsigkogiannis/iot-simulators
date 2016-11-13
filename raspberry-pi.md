# Raspberry Pi emulator

This tutorial shows how to emulate a Raspberry Pi.

## 1. Download the QEMU emulator

The easiest way to download and install QEMU is to download the 32-bit binaries from https://qemu.weilnetz.de/w32/. Even though my OS is 64-bit, the 64-bit version of QEMU did not work for me (the Raspbian kernel crashes during boot and the system restarts. The same problem happens with any QEMU version that is older than v2.6.0). After you download the latest file version (which is qemu-w32-setup-20161016.exe as I’m writing the article), you can either double click and install it or you can use 7-zip to extract the contents without installing (after you install 7-zip, you can right click on the QEMU exe file and select “Extract”).

## 2. Download Raspbian

You can download the latest Raspbian image from Raspberry Pi’s website. You can either download the full image (https://downloads.raspberrypi.org/raspbian_latest) or the “lite” version (https://downloads.raspberrypi.org/raspbian_lite_latest), which is smaller. The filename that I downloaded is 2016-09-23-raspbian-jessie.img. After you download the image, you should extract it to the same directory as QEMU.

## 3. Download the kernel

You can download the latest precompiled kernel from https://github.com/dhruvvyas90/qemu-rpi-kernel. I used kernel-qemu-4.4.13-jessie. Save this file in the same directory as QEMU.

## 4. Expand the Raspbian image

If you immediately boot into Raspbian without expanding the image, you will notice that there is very little free space. In order to be able to use more space, you need to expand the image. You can determine how much more space you want to allocate to Raspbian. In order to add 5GB to your Raspbian image you can open a command prompt, go to your QEMU directory and type the command:

`qemu-img.exe resize <your_raspbian_image> +5G`

e.g. `qemu-img.exe resize 2016-09-23-raspbian-jessie.img +5G`

## 5. First boot (to fix a setting)

Now you are ready to boot your emulator for the first time. In your command prompt, you can type:

`qemu-system-arm -kernel <your_kernel_image> -cpu arm1176 -m 256 -M versatilepb -serial stdio -append “root=/dev/sda2 panic=1 rootfstype=ext4 rw” -drive “file=<your_rapsbian_image>,index=0,media=disk,format=raw” -redir tcp:2222::22`

e.g. `qemu-system-arm -kernel kernel-qemu-4.4.13-jessie -cpu arm1176 -m 256 -M versatilepb -serial stdio -append “root=/dev/sda2 panic=1 rootfstype=ext4 rw” -drive “file=2016-09-23-raspbian-jessie.img,index=0,media=disk,format=raw” -redir tcp:2222::22`

Notes:

1. If you see an error about missing dll files, then you can download them from https://qemu.weilnetz.de/w32/dll/.
2. You will see a warning saying that the -redir option has been replaced by the `-netdev` option. I am still using `-redir`, since I could not understand how to use -netdev. If you know how, please leave a comment below and I will replace the command.

After the emulator finishes booting, you will see a terminal window saying that you have started in emergency mode, instead of the default mode, due to an error. In order to fix this, you will need to create the file /etc/udev/rules.d/90-qemu.rules using your favorite editor, e.g.

`sudo nano /etc/udev/rules.d/90-qemu.rules`

In that file you need to add the following:
```
KERNEL==”sda”, SYMLINK+=”mmcblk0″
KERNEL==”sda?”, SYMLINK+=”mmcblk0p%n”
KERNEL==”sda2″, SYMLINK+=”root”
```
Note: You are currently using the British locale (en-GB), that’s why some characters do not correspond to the standard American keyboard, e.g. the character ” is typed when you press the key @ (and the opposite). We will fix this in a following step.

After saving the file, you can either close the emulator or issue the reboot command (`sudo shutdown -r now`).

Note: Previous versions of QEMU (before v2.6.0) required changes in /etc/ld.so.preload. However, this is not needed anymore.

During the beginning of the boot process, you will see lots of text. However, at some point the screen will go blank. You will have to wait for some time, but afterwards you will first see the mouse pointer and then you’ll see the GUI. If you are asked for login credentials during the boot process, then you can use the following:
```
username: pi
password: raspberry
```
## 6. Expand the disk

At this point, we need to change the disk size so that it expands to the full size that we allocated in step 4 (i.e. to expand the disk by 5GB). In order to do that you need to do the following steps (I’m pasting the exact commands from http://www.xerxesb.com/2013/06/02/resizing-raspbian-image-for-qemu/):

1. ` sudo fdisk /dev/sda`
2. Print the partition table (“p”). Take note of the starting block of the main partition
3. Delete the main partition (“d”). Should be partition 2.
4. Create (n)ew partition. (P)rimary. Position (2)
5. Start block should be the same as start block from the original partition
6. Size should be the full size of the image file (just press “enter”)
7. Now write the partition table (w)
8. Reboot (shutdown -r now). After reboot,
9. ` sudo resize2fs /dev/sda2`

You can use the following command to verify the new size of the disk:

`df -h`

## 7. Increase the swap file for better performance

As you start using the emulator, you might observe that it is slow (especially if you have an old CPU). One reason for that is that the memory is only 256MB and the swap file is only 100MB. Unfortunately, there is no way to increase memory, but there is a way to increase the swap file. In order to do so, I’m using the instructions from https://www.bitpi.co/2015/02/11/how-to-change-raspberry-pis-swapfile-size-on-rasbian/

First you need to open the file /etc/dphys-swapfile using your favorite editor, e.g.

`sudo nano /etc/dphys-swapfile`

Then you need to find the line

`CONF_SWAPSIZE=100`

and change it to

`CONF_SWAPSIZE=1024`

After saving the file, you need to restart the service that manages the swapfile for Raspbian using the commands:
```
sudo /etc/init.d/dphys-swapfile stop
sudo /etc/init.d/dphys-swapfile start
```
If everything finishes successfully, you can verify the change in the swap file by using the command

`free -m`

The output should look like the following (the important part is that the line next to “swap” says “1023”):
```
             total    used    free    shared    buffers    cached
Mem:           248     213      33         2         12        131
-/+ buffers/cache:     70      177
Swap:         1023      0      1023
```
## 8. Use SSH for better performance

Another way to improve performance is to access your Raspberry Pi using SSH and not using the GUI. In order to enable this, we have mapped port 2222 from your system to port 22 of the emulator. This was done using the option `-redir tcp:2222::22` in the command that starts the emulator (section #4). So, all you need to do is to ssh in port 2222 of your system.

In order to do that, you need an ssh client, such as putty (which you can download from https://the.earth.li/~sgtatham/putty/latest/x86/putty.exe). After you download and run putty, you need to go to the tab “Session” and type pi@127.0.0.1 in the “Host Name (or IP Address)” box, as well as 2222 in the “Port” box. Then you can click on the “Open” button to connect. You can use the following credentials:
```
username: pi
password: raspberry
```
All the following steps can be done either from ssh (faster) or from a terminal window inside the GUI (slower).

## 9. Overclock the CPU for better performance

In a terminal window or ssh type
```
sudo raspi-config
Select option “8 Overclock”
Select “Ok” in the warning message
Select “Turbo 1000MHz ARM”
Select “Ok” in the confirmation message
Exit raspi-config
```
## 10. Change internalization options from British (en-GB) to American (en-US)

As we said earlier, we are currently using the British (en-GB) locale. In order to change it to American (en-US) or any other locale of your preference we need to execute the following commands from a terminal window or ssh:
```
export LANGUAGE=”en_US”
export LC_ALL=”en_US-UTF-8″
export LANG=”en_US-UTF-8″
sudo dpkg-reconfigure locales
Uncheck “en_GB-UTF-8”
Check “en_US-UTF-8”
Press Ok
Select “en_US.UTF-8” as the default locale for the system environment
```
## 11. Change the screen resolution

At this point, your emulator is up and running. However, you might be thinking that the window is too small and the resolution too low. This is because your current resolution is 640×480 at 16-bit color depth. If you want to change it to 800×600 (which will also increase the window size), you need to create the file /etc/X11/xorg.conf. I found the instructions for this at www.linux-mitterteich.de/fileadmin/datafile/papers/2013/qemu_raspiemu_lug_18_sep_2013.pdf

From a terminal window or ssh run your favorite editor, e.g.

`sudo nano /etc/X11/xorg.conf`

In that file you need to type the following contents:
```
Section “Screen”
Identifier “Default Screen”
  SubSection “Display”
    Depth 16
    Modes “800×600” “640×480”
  EndSubSection
EndSection
```
After you save the file you can either close the emulator window or restart using the command
`sudo shutdown -r now`

## Final observations
1. Using the above instructions we can emulate an ARM1176 CPU (which is the same as the one on Raspberry Pi 1) that runs a generic kernel on a Raspbian OS. Technically, this is slightly different than emulating a Raspberry Pi 1, since there are differences in the kernel and the board configuration (e.g. GPIO pins). QEMU does provide support for better Raspberry Pi 1 and Raspberry Pi 2 emulation, but it does not support any network capabilities and I could not make it work (the screen remained black and the system did not boot). Andrew Baumann initially implemented the port for Raspberry Pi 2 at https://github.com/0xabu/ and he also helped me with this tutorial. If you want to try by yourselves (and please do so and let me know, if you make it work), then here are the changes that you need to start with, compared to the above instructions: 
   * In step #3 you need to use a precompiled kernel for Raspberry Pi 1 or Raspberry Pi 2. 
     * One option is to download them https://github.com/raspberrypi/firmware/tree/master/boot
        * kernel.img is the kernel for Raspberry Pi 1
        * kernel7.img is the kernel for Raspberry Pi 2

      * A more difficult option is to install Raspbian on a real SD card, mount it (I’m not sure even if there are tools in Windows to do this) and then copy the kernel from the /boot directory.
    * In order to boot the emulator you need to use the following commands:
      * Raspberry Pi 1: `qemu-system-arm -M raspi -kernel kernel.img -sd 2016-09-23-raspbian-jessie.img -append “rw earlyprintk loglevel=8   console=ttyAMA0 root=/dev/mmcblk0p2” -serial stdio`
      * Raspberry Pi 2: `qemu-system-arm -M raspi2 -kernel kernel7.img -sd 2016-09-23-raspbian-jessie.img -append “rw earlyprintk loglevel=8   console=ttyAMA0 root=/dev/mmcblk0p2” -serial stdio`
2. In order to enable SSH, we used the option `-redir tcp:2222::22` in the qemu-system-arm command. This option is deprecated and replaced by –netdev, that’s why it triggers a warning message in the output. However, I could not understand how the –netdev option works. If you know how, please let me know in your comments, so that I can replace the above command.
3. It is not possible to copy data from the QEMU window, because it is an emulated framebuffer. If you know how to paste data to it, please write it in the comments and I will update the post accordingly.
4. If you have any tips regarding how to improve performance, then please post them in the comments. I tried allocating more than 256MB memory (using the `-m 512` option in qemu-system-arm) as well as > 1 CPU cores (using the -smp 4 option), but neither works.
5. If you know of any other emulators/simulators for popular IoT devices (running on desktop, browser, etc), feel free to paste them in the comments section.
