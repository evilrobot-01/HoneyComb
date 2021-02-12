# HoneyComb

This rough guide provides steps for getting a working Arch Linux ARM install on the HoneyComb LX2K. The steps were documented as I first installed things from an x86 Arch machine and then later updated as I progressed with work on the machine itself. Please also note that I am pretty new to Arch and Linux so some of this might not be the best way to go about things. Any corrections/tips/pointers greatly appreciated! Based on information from https://gist.github.com/meme/c1f1101fac0f58e883ae08872f19b883 and https://dev.to/lizthegrey/first-experiences-with-honeycomb-lx2k-26be.

## Building UEFI firmware
Ensure required packages are installed.

    sudo pacman -S acpica dtc
    
Clone the git repository and fix the missing `arch` command in the build script (if applicable to your current distro) 

    git clone --depth 1 https://github.com/SolidRun/lx2160a_uefi.git && cd lx2160a_uefi
    sed -i 's/HOST_ARCH=`arch`/HOST_ARCH=`uname -m`/' runme.sh
    
NOTE: there is an issue with building using Python 3.9 so I needed to temporarily install 3.8. I ended up using yay -S python38 and then symlinking python3 to python38 temporarily to get the firmware build done. I spent ages trying to find a neat solution but eventually get fed up. None of the python virtual environment stuff worked for me as the build scripts seem to scan for binaries (whereis python3) rather than determining the running python environment version. I then just removed python38 and re-installed python (3.9.1) afterwards (yay -R python38 && pacman -S python) to ensure everything went back to normal.
    
Next run the script with INITIALIZE to ensure all the required firmware build tools are installed and submodules are fetched from source and then start the build of the firmware, adapting your memory speed as applicable. There is also a patch which needs to be applied to the local source *after* INITIALIZE, which resolves the 'invalid base clock frequency' issue I was having with MMC on boot.

    INITIALIZE=1 . ./runme.sh
    
    # Apply patch
    cd build/tianocore/edk2-platforms && git cherry-pick 0276df7768ac10a3c0596b2882c9fbdedc26d717 && cd ../../..
    
    DDR_SPEED=3200 ./runme.sh

Once built, check the images directory:

    $ ls images
    lx2160acex7_2200_700_3200_8_5_2.img
    
Finally flash the firmware to a SD card. Use fdisk -l to list the disks and check through the output to find the correct target disk. When doing it on my x86 machine it was listed as sdX I believe, but when doing again recently on the HoneyComb it was mmcblk0.

    sudo fdisk -l
    sudo dd if=images/lx2160acex7_2200_700_3200_8_5_2.img of=/dev/sdX conv=fsync
    
Insert the SD card into the Honeycomb, connect the USB console cable and then fire up minicom (ensuring you disable hardware/software flow control under serial port setup and then save for future) before finally powering up the Honeycomb to check the firmware loads up. You could also use monitor connected directly if required, but the linked articles said not to and when I first attempted this I didnt yet have a GPU installed.

    sudo minicom -s -c on -D /dev/ttyUSB0
    
## Building an Arch Linux ARM USB

The below steps will create a bootable Arch Linux ARM USB drive with the generic Arch Linux ARM build. You will need to use a USB network adapter until a custom kernel is built later in this guide. I did it this way as practice on my main machine, so I have a bootable USB version of Arch to keep handy and to let me kick off a local install on NVMe later. Based on information from https://itsfoss.com/install-arch-raspberry-pi/ and https://gist.github.com/thalamus/561d028ff5b66310fac1224f3d023c12.

On a Linux machine, firstly create a working folder and then pull down the latest generic ARM build:

    mkdir alarm && cd alarm
    wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
    
Then change to root:

    sudo su
    
Insert the USB disk and then list the disks to determine the identifier, in my case it was /dev/sdg:

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

    pacman -S dosfstools
    mkfs.vfat /dev/sdX1
    mkfs.ext4 /dev/sdX2
    mount /dev/sdX2 /mnt
    mkdir /mnt/boot
    mount /dev/sdX1 /mnt/boot
    
Next we extract the downloaded build to the mounted partitions and then ensure that any cached writes are flushed to disk. This may take a few moments...

    bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
    sync
    
Next we need to populate fstab:

    pacman -S arch-install-scripts
    genfstab -U /mnt >> /mnt/etc/fstab
    cat root/etc/fstab
    
The USB device will require a startup.nsh file that the UEFI shell will run on boot. This is using the EFIStub functionality of the kernel. Replace /dev/sdxn below with the relevant partition identifier.

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/sdX2) rw rootfstype=ext4 initrd=initramfs-linux.img" > /mnt/boot/startup.nsh
    cat /mnt/boot/startup.nsh
    
Finally unmount and exit from root and then move the USB drive to the HoneyComb.

    umount /mnt/boot /mnt
    exit
    
Start up minicom and then boot the HoneyComb, logging into Arch as root with password root.

    sudo minicom -c on -D /dev/ttyUSB0
    
Once booted, carry out the following few steps to initialise the pacman keyring, populate the signing keys and update the system packages:

    pacman-key --init
    pacman-key --populate archlinuxarm
    pacman -Syu

Finally install the efibootmgr package to create a UEFI boot entry. Use blkid to identify the root partition (/dev/sda2 in my case) and then efibootmgr to add the boot entry (amend as necessary):

    pacman -S efibootmgr
    blkid
    efibootmgr --disk /dev/Xda --part 1 --create --label "Arch Linux ARM" --loader /Image --unicode 'root=UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX rw initrd=\initramfs-linux.img' --verbose

NOTE: I havent had great success with the efibootmgr yet, mostly due to what seems to be the length of the kernel parameters used later in this guide to get a full working boot. I therefore am relying on the startup.nsh approach on the boot drive, which automatically kicks in when booting the UEFI Shell. I might look at how to generate EFI files next, just to get a clean simple boot without grub etc.
    
## Installing Arch Linux ARM on NVMe

We can essentially repeat the steps for setting up a USB drive, but this time installing onto an NVMe disk installed on the HoneyComb.You will still need to use a USB network adapter until a custom kernel is built later in this guide.

If applicable, start up minicom and then boot the HoneyComb, logging in as root with password root.

    sudo minicom -c on -D /dev/ttyUSB0

Create a working folder and then pull down the latest generic ARM build:

    mkdir alarm && cd alarm
    curl -LO http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz

List the disks to determine the NVMe identifier, in my case it was /dev/nvme0n1:

    fdisk -l

Next we will set up the required partitions:

    fdisk /dev/nvme0n1

Type o to purge any partitions, then p to check that they have been cleared. To create the boot partition, type n for a new partition, p for primary, 1 for first partition, enter to accept default first sector and then +1G for last. Then type t and then c to set the partition as W95 FAT32 (LBA).

    o   p   n   p   1   ENTER   +1G
    t   c

To create the root partition, type n for a new partition, p for primary, 2 for second partition, enter to accept default first sector and enter again for last.

    n   p   2   ENTER   ENTER

Finally write the changes and exit by pressing w.

    w

Next we need to create and mount the file systems (amend device as necessary). I added btrfs to try it out, you can change to ext4 as with USB above if you'd prefer.

    pacman -S dosfstools btrfs-progs
    mkfs.vfat /dev/nvme0n1p1
    mkfs.btrfs /dev/nvme0n1p2
    mount -o compress=lzo /dev/nvme0n1p2 /mnt
    mkdir /mnt/boot
    mount /dev/nvme0n1p1 /mnt/boot

    # Create BTRFS subvolumes based on https://wiki.archlinux.org/index.php/Snapper#Suggested_filesystem_layout and https://www.fastycloud.com/tutorials/kb/install-arch-linux-with-btrfs-snapshotting/
    cd /mnt
    btrfs subvolume create @
    btrfs subvolume create @home
    btrfs subvolume create @snapshots
    btrfs subvolume create @log
    cd ~/alarm

    # Re-mount to subvolumes
    umount /mnt/boot /mnt
    mount -o compress=lzo,subvol=@ /dev/nvme0n1p2 /mnt
    cd /mnt
    mkdir -p {boot,home,.snapshots,var/log}
    mount -o compress=lzo,subvol=@home /dev/nvme0n1p2 /mnt/home
    mount -o compress=lzo,subvol=@snapshots /dev/nvme0n1p2 /mnt/.snapshots
    mount -o compress=lzo,subvol=@log /dev/nvme0n1p2 /mnt/var/log
    mount /dev/nvme0n1p1 /mnt/boot
    cd ~/alarm

Next we extract the downloaded build to the root partition and then ensure that any cached writes are flushed to disk. This may take a few moments...

    bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
    sync

Next we need to set up the boot partition and populate fstab (amend as necessary):

    pacman -S arch-install-scripts
    genfstab -U /mnt >> /mnt/etc/fstab
    cat root/etc/fstab

Next start to configure the system.

    pacstrap -i /mnt base base-devel btrfs-progs
    arch-chroot /mnt
    ln -s /usr/share/zoneinfo/Europe/London /etc/localtime # Replace Region/City with your value
    hwclock --systohc
    nano /etc/locale.gen # Uncomment en_GB.UTF-8 UTF-8 line and save
    locale-gen
    echo "LANG=en_GB.UTF-8" > /etc/locale.conf
    echo "KEYMAP=uk" > /etc/vconsole.conf
    nano /etc/mkinitcpio.conf # Add btrfs to MODULES=()
    mkinitcpio -p linux-aarch64 # re-generate initial ram disk
    exit

Create a startup.nsh file on the NVMe boot partition that the UEFI shell will run on boot. Replace /dev/nvme0n1p2 below with the relevant partition identifier.

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img" > /mnt/boot/startup.nsh
    cat /mnt/boot/startup.nsh

Finally unmount and exit from root and then power off. Remove the USB drive.

    umount /mnt/boot /mnt/home /mnt/.snapshots /mnt/var/log /mnt
    poweroff

Start up again and once booted, initialise the pacman keyring, populate the signing keys and update the system packages:

    pacman-key --init
    pacman-key --populate archlinuxarm
    pacman -Syu
    reboot
    
Once rebooted, change the default root password and then set up a new user with sudo privileges:

    passwd
    useradd -m -G wheel -s /bin/bash username
    visudo   # Uncomment the line "%wheel   ALL=(ALL)   ALL" and then enter :wq to save and quit
    passwd username
    
Test that you can login as new new user on another tty (CTRL-ALT-F2) and if successful, delete the default alarm user:

    userdel alarm
    
I still havent gotten around to changing the default host name!

## Building Kernel

Based on information at https://gist.github.com/thalamus/561d028ff5b66310fac1224f3d023c12 and https://wiki.archlinux.org/index.php/Kernel/Traditional_compilation

Ensure all the packages required for building a kernel are installed.

    pacman -S base-devel git xmlto kmod inetutils bc libelf cpio perl tar xz python

Pull down the latest kernel source from SolidRun's GitHub, ensure the kernel tree is clean, create the kernel configuration based on the default config, customise the config to add missing/required modules and then finally start the kernel compilation. I just accepted all the defaults if prompted. NOTE: the generic Arch ARM image has a kernel parameter limiting the number of CPUs to 8 (https://github.com/archlinuxarm/PKGBUILDs/blob/d883ab288f620dfd4967ae5e923faa45efc621dd/core/linux-aarch64/config#L376) so the first kernel build wont be full throttle. Using the default config corrects this, but you can always set it manually to 16 if you wish.

    # Preparation
    git clone --depth 1 -b linux-5.10.y-cex7 https://github.com/SolidRun/linux-stable linux-source-5.10 && cd linux-source-5.10
    make mrproper
    # Configuration - I just used the default config and then addded a few Arch specific settings, along with some missing settings from SolidRun discord.
    make defconfig
    sed -ri '/CONFIG_LOCALVERSION=/s/=.+/="-ARCH"/g' .config
    sed -i '/CONFIG_LOCALVERSION_AUTO=/s/.*/# CONFIG_LOCALVERSION_AUTO is not set/' .config
    sed -i '/CONFIG_HID_PID/s/.*/CONFIG_HID_PID=y/' .config
    sed -i '/CONFIG_USB_HIDDEV/s/.*/CONFIG_USB_HIDDEV=y/' .config
    sed -ri '/CONFIG_NLS_DEFAULT=/s/=.+/="utf8"/g' .config
    sed -i '/CONFIG_NLS_ASCII/s/.*/CONFIG_NLS_ASCII=y/' .config
    sed -ri '/CONFIG_NLS_ISO8859_1/s/=.+/=m/g' .config
    # Enable HoneyComb specific
    sed -i '/CONFIG_FSL_MC_UAPI_SUPPORT/s/.*/CONFIG_FSL_MC_UAPI_SUPPORT=y/' .config
    sed -i '/CONFIG_FSL_DPAA2_QDMA/s/.*/CONFIG_FSL_DPAA2_QDMA=m/' .config
    sed -i '/CONFIG_STAGING/s/.*/CONFIG_STAGING=y/' .config
    echo "CONFIG_FSL_DPAA2=y" >> .config
    echo "CONFIG_FSL_DPAA2_ETHSW=m" >> .config
    # Silent boot
    sed -i '/CONFIG_LOGO=/s/.*/# CONFIG_LOGO is not set/' .config
    # A few peripherals specifc to my setup
    sed -i '/CONFIG_DRM_AMDGPU/s/.*/CONFIG_DRM_AMDGPU=m/' .config
    sed -i '/CONFIG_HID_MAGICMOUSE/s/.*/CONFIG_HID_MAGICMOUSE=m/' .config
    sed -i '/CONFIG_SND_USB_AUDIO/s/.*/CONFIG_SND_USB_AUDIO=m/' .config
    
    # Compilation
    make -j$(nproc) ARCH=arm64 Image Image.gz modules
    # Install modules
    sudo make modules_install

Next, copy the kernel to the boot partition and then generate the initial RAM disk. 

    sudo cp -v arch/arm64/boot/Image /boot
    sudo cp -v arch/arm64/boot/Image.gz /boot
    sudo mkinitcpio -k 5.10.12-ARCH+ -g /boot/initramfs-linux.img

Finally update "boot loader" (startup.nsh for now) to load new kernel, along with a few parameters to work around current known issues. The below is based on my NVMe BTRFS setup, but you could use something like the USB version above for a more traditional setup:

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img amdgpu.pcie_gen_cap=0x4 quiet" > /boot/startup.nsh

If everything is working, update /etc/pacman.conf to ignore kernel package updates until all of the SolidRun patches are in the mainline kernel.

    IgnorePkg   = linux linux-aarch64
