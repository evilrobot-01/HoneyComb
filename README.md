# HoneyComb

Based on information from https://gist.github.com/meme/c1f1101fac0f58e883ae08872f19b883 and https://dev.to/lizthegrey/first-experiences-with-honeycomb-lx2k-26be.

## Building firmware
Ensure required packages are installed.

    sudo pacman -S acpica dtc python-virtualenv
    
NOTE: there is an issue with building using Python 3.9 so I needed to downgrade to 3.8. Use the following and then select the version.

    yay -S downgrade && downgrade python
    
You can check the installed version using:

    python -V

Clone the git repository, change to directory, and then run within a subshell in order set an alias for the missing `arch` command.

    git clone https://github.com/SolidRun/lx2160a_uefi.git && cd lx2160a_uefi
    bash -c 'alias arch="uname -m" && INITIALIZE=1 . ./runme.sh'
    
Once initialised, start the build of the firmware, adapting your memory speed as applicable:

    DDR_SPEED=3200 ./runme.sh

Once built, check the images directory:

    [fb@home lx2160a_uefi]$ ls images
    lx2160acex7_2200_700_3200_8_5_2.img
    
Finally flash the firmware to a SD card. Use fdisk -l to list the disks and check through the output to find the correct disk.

    sudo fdisk -l
    sudo dd if=images/lx2160acex7_2200_700_3200_8_5_2.img of=/dev/sdX conv=fsync
