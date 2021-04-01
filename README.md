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

## Update for keyboard mappings

The trekstor Duo W1 attachable keyboard does implement several event handlers. Event7 is for most keys but Eevent8 is used for the upper row for the multimedia/function keys.

I wanted the function keys instead of the media ones and hat to revise my mapping in the files 70-keyboard.hwdb and 99-HHKB-keyboard.hwdb:

    
     # /etc/udev/hwdb.d/70-keyboard.hwdb
     # for dev/input/event8 LIZHICHIP USB Keyboard Consumer Control
     evdev:input:b0003v1C4Fp0063e0110-e0,1,2,3,4,k71,72,73,74,77,80,82,83,85,86,87,88,89,8A,8B,8C,8E,90,96,98,9B,9C,9E,9F,A1,A3,A4,A5,A6,A7,A8,A9,AB,AC,AD,AE,B1,B2,B5,CE,CF,D0,D1,D2,D4,D8,D9,DB,E0,E1,E4,E5,E6,EA,EB,F0,F1,F4,100,161,162,166,16A,16E,172,174,176,177,178,179,17A,17B,17C,17D,17F,180,182,183,185,188,189,18C,18D,18E,18F,190,191,192,193,195,197,198,199,19A,19C,1A0,1A1,1A2,1A3,1A4,1A5,1A6,1A7,1A8,1A9,1AA,1AB,1AC,1AD,1AE,1AF,1B0,1B1,1B7,1BA,240,241,242,243,244,245,246,247,250,251,r6,C,a20,m4,lsfw
     #b0003v1C4Fp0063*
     #map mute to f1
      KEYBOARD_KEY_C00E2=key_f1
     #map voldown to f2
      KEYBOARD_KEY_C00EA=key_f2
     #map vol_up to f3
      KEYBOARD_KEY_C00E9=key_f3
     # prevsong to f4
      KEYBOARD_KEY_C00B6=Key_f4
     # playpause to f5
      KEYBOARD_KEY_C00CD=key_f5
     # nextsong to F6
      KEYBOARD_KEY_C00B5=key_f6
     # mail to f7
      KEYBOARD_KEY_C018A=key_f7
     # search to f8
      KEYBOARD_KEY_C0221=key_f8



     #/etc/udev/hwdb.d/99-HHKB-keyboard.hwdb
     evdev:input:b0003v1C4Fp0063e0110-e0,1,4,11,14,k71,72,73,74,75,77,79,7A,7B,7C,7D,7E,7F,80,81,82,83,84,85,86,87,88,89,8A,B7,B8,B9,BA,BB,BC,BD,BE,BF,C0,C1,C2,F0,ram4,l0,1,2,sfw
     #b0003v1C4Fp0063*
      KEYBOARD_KEY_7004a=pageup
      KEYBOARD_KEY_7004d=pagedown
      KEYBOARD_KEY_7004b=home
      KEYBOARD_KEY_7004e=end
