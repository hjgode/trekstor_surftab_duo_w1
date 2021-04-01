# trekstor_surftab_duo_w1

Initial source: https://ianrenton.com/guides/install-linux-on-a-linx-1010b-tablet

## My challenge to install Linux (Ubuntu 20.04) on a TrekStor SurfTab Duo W1

The Windows 10 installation on this TrekStor tablet with a keyboard extension is ugly. The main issue is the suspend resume, where the screen is not lit after a resume and has to be resumed with a press on the volume buttons or the home button.

First I started with a Fedora installation (30.1 live install), this works in general but this runs slow and has many squirks for one coming from a Debian system. I had to enable ssh after every boot manually as it did not keep the enabled.

The I found the above mentioned site about the Ubuntu installation. After unpacking the 20.04.02 iso (the 01 is not available any more), copy it to an USB stick and add the bootia32.efi to the stick. Then booted Ubuntu Live and selected Install to disk (minimal installation). I had done this three time, as the installer broke the USB files. Finally it worked bot the tablet did not show the new installation in the boot options. I needed to prepare the grub installation with grubia32.efi and run the Live USB ubuntu to install grub manually.

### Some installation cmds
     sudo update-grub
     sudo update-grub2

     sudo apt install grub-efi-ia32 grub-efi-ia32-bin
     sudo mkdir -p /boot/efi/EFI/ubuntu/
     sudo cp bootia32.efi /boot/efi/EFI/ubuntu/
     sudo cp /boot/efi/EFI/ubuntu/grubia32.efi /boot/efi/EFI/ubuntu/grubx64.efi

     sudo update-grub && sudo update-grub2 && sudo grub-install

#### Correct the efi boot manager and boot order
Note: these have to be adopted to your installation

     sudo efibootmgr -c --disk /dev/mmcblk0 --part 1
     sudo efibootmgr -o 1,3,4,2,0 --disk /dev/mmcblk0 


The installation was done with the screen rotated incorrectly to inverted (upside down) all the time.

Then more challenges began. The screen rotation is wrong and the touch screen seems to be mirrored for the X-axis.

Fortunately I had ssh access, the keyboard and mouse pad and some scripts that where able to fix the screen rotation and touch screen issue (see rotate-normal.sh and touch-normal.sh).

The problem with the screen rotate is about the accelerometer sensor giving the wrong signal. The simple script:

     #!/bin/sh
     xrandr -o normal

fixes this temporary until next reboot or tablet rotation.

I tried to fix that with an Xorg conf file but this does not work. also automatic calling the script on startup with .xinitrc, .xprofile etc. does not work. Who knows what Ubuntu implemented for Xorg. Finally I created a udev hwdb rule:

     #/etc/udev/hwdb.d/61-sensor-local.hwdb
     #Trekstor Wintab Duo W1 10.1
     sensor:modalias:acpi:BOSC0200*:dmi*
     ACCEL_MOUNT_MATRIX=1, 0, 0; 0, -1, 0; 0, 0, 1

The above fixes the transformation of the sensor values for the X window system.

Now, the touchscreen issue has to be solved permanently. The script 

     #!/bin/sh
     xinput set-prop "pointer:Goodix Capacitive TouchScreen" "Coordinate Transformation Matrix" -1 0 1 0 1 0 0 0 1

solves the issue temporary until next login or suspend/resume only :-(.

Again no automation or startup for this script worked for me. Even an Xorg conf file does not work:

     Section "InputClass"
	  Identifier	"calibration"
	  MatchProduct	"Goodix Capacitive TouchScreen"
	  Option	"MinX"	"65125"
	  Option	"MaxX"	"0"
	  Option	"MinY"	"785"
	  Option	"MaxY"	"66921"
	  Option	"SwapXY"	"0" # unless it was already set to 1
	  Option	"InvertX"	"0"  # unless it was already set
	  Option	"InvertY"	"0"  # unless it was already set
	  Option "_libinput/cap-keyboard" "False"
	  Option "TransformationMatrix" "0 1 0 -1 0 1 0 0 1"
     EndSection

xinput_calibrator gave:

    DEBUG: Found that 'Goodix Capacitive TouchScreen' is a sysfs name.
    INFO: width=1280, height=800
    DEBUG: Adding click 0 (X=1113, Y=108)
    DEBUG: Adding click 1 (X=155, Y=109)
    DEBUG: Adding click 2 (X=1108, Y=703)
    DEBUG: Adding click 3 (X=155, Y=701)
        --> Making the calibration permanent <--
    DEBUG: Found that 'Goodix Capacitive TouchScreen' is a sysfs name.
    copy the snippet below into '/etc/X11/xorg.conf.d/99-calibration.conf' (/usr/share/X11/xorg.conf.d/ in some distro's)
    Section "InputClass"
        Identifier      "calibration"
        MatchProduct    "Goodix Capacitive TouchScreen"
        Option  "MinX"  "65010"
        Option  "MaxX"  "-218"
        Option  "MinY"  "785"
        Option  "MaxY"  "65610"
        Option  "SwapXY"        "0" # unless it was already set to 1
        Option  "InvertX"       "0"  # unless it was already set
        Option  "InvertY"       "0"  # unless it was already set
    EndSection

with the help of the site https://wiki.archlinux.org/index.php/Talk:Calibrating_Touchscreen I was able to translate the click values into an udev rule.

     #/etc/udev/rules.d/98-touchscreen-cal.rules
     ATTRS{name}=="Goodix Capacitive TouchScreen", ENV{LIBINPUT_CALIBRATION_MATRIX}="-1 0.0 1 0.0 1 0"

# end
Now I have ubuntu running fine on my TrekStor Duo W1 10.1 with Bay Trail proc and ugly sensor and touchscreen.

# update
## Repair EFI grub loader

In case you need to repair the boot loader or accidently get into the grub console: How to repair the EFI Grub loader using the ubuntu usb live media:

The partition table of the TrekStor:

    #(parted) print                                                            
    #Model: MMC NCard (sd/mmc)
    #Disk /dev/mmcblk0: 31,0GB
    #Sector size (logical/physical): 512B/512B
    #Partition Table: gpt
    #Disk Flags: 
    #
    #Number  Start   End     Size    File system  Name                       Flags
    # 1      1049kB  556MB   555MB   ntfs         Basic data partition          hidden, diag
    # 2      556MB   661MB   105MB   fat32        EFI   System Partition          boot, esp
    # 3      661MB   677MB   16,8MB               Microsoft reserved partition  msftres
    # 4      677MB   1751MB  1074MB  ext4
    # 5      1751MB  31,0GB  29,3GB  ext4

Then from the live usb ubuntu terminal:

    sudo mount /dev/mmcblk0p5 /mnt
    sudo mount /dev/mmcblk0p2 /mnt/boot/efi
    for i in /dev /dev/pts /proc /sys /run; do sudo mount -B $i /mnt$i; done
    sudo chroot /mnt
    grub-install /dev/mmcblk0
    update-grub  

That's it.

## Final change? Remapping keys of the hardware keyboard

The TrekStor SurfTab Duo W1 10.1 comes with a attachable keyboard. Unfortunately it has PageUp and PageDown on the FN level, but Home and End is directly usabale. As I often need to scroll up and down using the Page Up/Down keys, I need these keys in the normal level of the keypad. I can remap after login/resume with the script:

    #!/bin/sh
    xmodmap -e "keycode 110 = Prior"
    xmodmap -e "keycode 115 = Next"
    xmodmap -e "keycode 112 = Home"
    xmodmap -e "keycode 117 = End"

But this nasty and I want these to be mapped all the time. I found a good description at https://yulistic.gitlab.io/2017/12/linux-keymapping-with-udev-hwdb/.

For me I had to add 

    sudo vim /etc/udev/hwdb.d/99-HHKB-keyboard.hwdb
    
    # the evdev address product, vendor must be entered in upper case HEX values
    # the key names must be entered without KEY_ and in lower case
    evdev:input:b0003v1c4fp0063*
      KEYBOARD_KEY_7004a=pageup
      KEYBOARD_KEY_7004d=pagedown
      KEYBOARD_KEY_7004b=home
      KEYBOARD_KEY_7004e=end

Please use identication as above.

The above maps Pos1 to PageUp and End to PageDown for the TrekStor keyboard.



