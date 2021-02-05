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
    
Insert the SD card into the Honeycomb, connect the console cable and then fire up minicom (ensuring you disable flow control under serial port setup and then save for future) before finally powering up the Honeycomb.

    sudo minicom -s -c on -D /dev/ttyUSB0
    
## Building an Arch Linux USB

The below steps will create an Arch Linux USB drive with the generic Arch Linux ARM build. You will need to use a USB network adapter until a custom kernel is built later in this guide. Based on information from https://itsfoss.com/install-arch-raspberry-pi/ and https://gist.github.com/thalamus/561d028ff5b66310fac1224f3d023c12.

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
    
Next we write extract the downloaded build to the mounted partitions and then ensure that any cached writes are flushed to disk. This may take a few moments...

    bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
    sync
    
Next we need to populate fstab:

    pacman -S arch-install-scripts
    genfstab -U /mnt >> /mnt/etc/fstab
    cat root/etc/fstab
    
The USB device will require a startup.nsh file that the UEFI shell will run on boot. Replace /dev/sdxn below with the relevant partition identifier.

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/sdX2) rw rootfstype=ext4 initrd=initramfs-linux.img" > /mnt/boot/startup.nsh
    cat /mnt/boot/startup.nsh
    
Finally unmount and exit from root and then move the USB drive to the HoneyComb.

    umount /mnt/boot /mnt
    exit
    
Start up minicom and then boot the HoneyComb, logging in as root with password root.

    sudo minicom -c on -D /dev/ttyUSB0
    
Once booted, initialise the pacman keyring, populate the signing keys and update the system packages:

    pacman-key --init
    pacman-key --populate archlinuxarm
    pacman -Syu

Finally install the efibootmgr package to create a UEFI boot entry. Use blkid to identify the root partition (/dev/sda2 in my case) and then efibootmgr to add the boot entry (amend as necessary):

    pacman -S efibootmgr
    blkid
    efibootmgr --disk /dev/Xda --part 1 --create --label "Arch Linux ARM" --loader /Image --unicode 'root=UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX rw initrd=\initramfs-linux.img' --verbose

    
## Installing Arch Linux ARM on NVMe

We can essentially repeat the steps for setting up a USB drive, but this time installing onto an NVMe disk installed on the HoneyComb.You will still need to use a USB network adapter until a custom kernel is built later in this guide.

Start up minicom and then boot the HoneyComb, logging in as root with password root.

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

Next we need to create and mount the file systems (amend device as necessary).

    pacman -S dosfstools btrfs-progs
    mkfs.vfat /dev/nvme0n1p1
    mkfs.btrfs /dev/nvme0n1p2
    mount -o compress=lzo /dev/nvme0n1p2 /mnt
    mkdir /mnt/boot
    mount /dev/nvme0n1p1 /mnt/boot

    # Create subvolumes based on https://wiki.archlinux.org/index.php/Snapper#Suggested_filesystem_layout and https://www.fastycloud.com/tutorials/kb/install-arch-linux-with-btrfs-snapshotting/
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

Next we write extract the downloaded build to the root parition and then ensure that any cached writes are flushed to disk. This may take a few moments...

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

Create a startup.nsh file that the UEFI shell will run on boot. Replace /dev/nvme0n1p2 below with the relevant partition identifier.

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img" > /mnt/boot/startup.nsh
    cat /mnt/boot/startup.nsh

Finally unmount and exit from root and then power off. Remove the USB drive.

    umount /mnt/boot /mnt/home /mnt/.snapshots /mnt/var/log /mnt
    poweroff

Once booted, initialise the pacman keyring, populate the signing keys and update the system packages:

    pacman-key --init
    pacman-key --populate archlinuxarm
    pacman -Syu
    reboot
    
Once rebooted, change the root password and then set up a new user:

    passwd
    useradd -m -G wheel -s /bin/bash username
    visudo   # Uncomment the line "%wheel   ALL=(ALL)   ALL" and then enter :wq to save and quit
    passwd username
    
Test that you can login as new new user on another tty (CTRL-ALT-F2) and if successful, delete the default alarm user:

    userdel alarm

## Building Kernel

Based on information at https://gist.github.com/thalamus/561d028ff5b66310fac1224f3d023c12 and https://wiki.archlinux.org/index.php/Kernel/Traditional_compilation

Ensure all the packages required for building a kernel are installed.

    pacman -S base-devel git xmlto kmod inetutils bc libelf cpio perl tar xz python

Pull down the latest kernel source from SolidRun's GitHub, ensure the kernel tree is clean, create the kernel configuration based on the running stock Arch kernel, customise the config to add required modules and then finally start the kernel compilation.>

    # Preparation
    git clone --depth 1 -b linux-5.10.y-cex7 https://github.com/SolidRun/linux-stable linux-source-5.10 && cd linux-source-5.10
    make mrproper
    # Configuration
    make defconfig
    sed -ri '/CONFIG_LOCALVERSION=/s/=.+/="-ARCH"/g' .config
    sed -i '/CONFIG_LOCALVERSION_AUTO=/s/.*/# CONFIG_LOCALVERSION_AUTO is not set/' .config
    sed -i '/CONFIG_FSL_MC_UAPI_SUPPORT/s/.*/CONFIG_FSL_MC_UAPI_SUPPORT=y/' .config
    sed -ri '/CONFIG_NLS_DEFAULT=/s/=.+/="utf8"/g' .config
    sed -i '/CONFIG_NLS_ASCII/s/.*/CONFIG_NLS_ASCII=y/' .config
    sed -ri '/CONFIG_NLS_ISO8859_1/s/=.+/=m/g' .config
    
    sed -i '/CONFIG_DRM_AMDGPU/s/.*/CONFIG_DRM_AMDGPU=m/' .config
    sed -i '/CONFIG_MOUSE_APPLETOUCH/s/.*/CONFIG_MOUSE_APPLETOUCH=m/' .config
    sed -i '/CONFIG_SND_USB_AUDIO/s/.*/CONFIG_SND_USB_AUDIO=m/' .config
    
    # Enable HoneyComb specific modules?
            echo "CONFIG_FSL_DPAA2_QDMA=m" >> .config
            echo "CONFIG_STAGING=y" >> .config
            echo "CONFIG_FSL_DPAA2=y" >> .config
            echo "CONFIG_FSL_DPAA2_ETHSW=m" >> .config
    
    # Compilation
    make -j$(nproc) ARCH=arm64 Image Image.gz modules
    # Install modules
    sudo make modules_install

Next, copy the kernel to the boot partition and then generate the initial RAM disk:

    sudo cp -v arch/arm64/boot/Image /boot/Image510
    sudo cp -v arch/arm64/boot/Image.gz /boot/Image510.gz
    sudo mkinitcpio -k 5.10.5-ARCH+ -g /boot/initramfs-linux510.img

Finally update boot loader (startup.nsh for now) to load new kernel, along with a few parameters to work around current issues:

    echo "Image510 root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux510.img arm-smmu.disable_bypass=0 amdgpu.pcie_gen_cap=0x4 quiet logo.nologo" > /boot/startup.nsh

