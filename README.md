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
    
Insert the SD card into the Honeycomb, connect the console cable and then fire up minicom (ensuring you disable flow control under serial port setup) before finally powering up the Honeycomb.

    sudo minicom -s -c on -D /dev/ttyUSB0
    
## Building an Arch Linux USB

Based on information from https://itsfoss.com/install-arch-raspberry-pi/ and https://gist.github.com/thalamus/561d028ff5b66310fac1224f3d023c12.

Firstly create a working folder and then pull down the latest generic ARM build:

    mkdir alarm && cd alarm
    wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
    
Then change to root:

    sudo su
    
Then list the disks to determine the identifier, in my case it was /dev/sdg:

    fdisk -l
    
Next we will set up the required partitions:

    fdisk /dev/sdX

Type o to purge any partitions, then p to check that they have been cleared. To create the boot partition, type n for a new partition, p for primary, 1 for first partition, enter to accept default first sector and then +260M for last. Then type t and then c to set the partition as W95 FAT32 (LBA).

    o   p   n   p   1   ENTER   +260M
    t   c

To create the root partition, type n for a new partition, p for primary, 2 for second partition, enter to accept default first sector and enter again for last.

    n   p   2   ENTER   ENTER
    
Finally write the changes and exit by pressing w.

    w
    
Next we need to create and mount the file systems (amend device as necessary).

    mkfs.vfat /dev/sdX1
    mkdir boot
    mount /dev/sdX1 boot
    mkfs.ext4 /dev/sdX2
    mkdir root
    mount /dev/sdX2 root
    https://itsfoss.com/install-arch-raspberry-pi/
    
Next we write extract the downloaded build to the root parition and then ensure that any cached writes are flushed to disk. This may take a few moments...

    bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C root
    sync
    
Next we need to set up the boot partition and populate fstab (amend as necessary):

    mv root/boot/* boot
    echo "UUID=$(blkid -s UUID -o value /dev/sdX2) / ext4 defaults 0 0" >> root/etc/fstab
	echo "UUID=$(blkid -s UUID -o value /dev/sdX1) /boot vfat defaults 0 0" >> root/etc/fstab
	cat root/etc/fstab
    
The USB device will require a startup.nsh file that the UEFI shell will run on boot. Replace /dev/sdxn below with the partition identifier.

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/sdX2) rw rootfstype=ext4 initrd=initramfs-linux.img" >> boot/startup.nsh
    cat boot/startup.nsh
    
Finally unmount and exit from root

    umount boot root
    exit
    
Start up minicom (ensuring you disable flow control under serial port setup) and then boot the HoneyComb, logging in as root with password root.

    sudo minicom -s -c on -D /dev/ttyUSB0
    
Once booted, initialise the pacman keyring, populate the signing keys and update the system packages:

    pacman-key --init
    pacman-key --populate archlinuxarm
    pacman -Syu

Finally install the efibootmgr package to create a UEFI boot entry. Use blkid to identify the root partition (/dev/sda2 in my case) and then efibootmgr to add the boot entry (amend as necessary):

    pacman -S efibootmgr
    blkid
    efibootmgr --disk /dev/Xda --part 1 --create --label "Arch Linux ARM" --loader /Image --unicode 'root=UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX rw initrd=\initramfs-linux.img' --verbose

    
