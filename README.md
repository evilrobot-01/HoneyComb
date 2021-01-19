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
    mkdir boot
    mount /dev/nvme0n1p1 boot
    mkfs.btrfs /dev/nvme0n1p2
    mkdir root
    mount -o compress=lzo /dev/nvme0n1p2 root

    # Create subvolumes based on https://wiki.archlinux.org/index.php/Snapper#Suggested_filesystem_layout and https://www.fastycloud.com/tutorials/kb/install-arch-linux-with-btrfs-snapshotting/
    cd root
    btrfs subvolume create @
    btrfs subvolume create @home
    btrfs subvolume create @snapshots
    btrfs subvolume create @log
    cd ..

PROBLEM HERE WITH /BOOT MOUNTING AFFECTING PACKAGE UPGRADES LTER

    # Re-mount to subvolumes
    umount root
    mount -o compress=lzo,subvol=@ /dev/nvme0n1p2 root
    cd root
    mkdir -p {home,.snapshots,var/log}
    mount -o compress=lzo,subvol=@home /dev/nvme0n1p2 home
    mount -o compress=lzo,subvol=@snapshots /dev/nvme0n1p2 .snapshots
    mount -o compress=lzo,subvol=@log /dev/nvme0n1p2 var/log
    cd ..

Next we write extract the downloaded build to the root parition and then ensure that any cached writes are flushed to disk. This may take a few moments...

    bsdtar -xpf ArchLinuxARM-aarch64-latest.tar.gz -C root
    sync

Next we need to set up the boot partition and populate fstab (amend as necessary):

    mv root/boot/* boot
    pacman -S arch-install-scripts
    genfstab -U root >> root/etc/fstab
    genfstab -U boot >> root/etc/fstab ERROR HERE?
    cat root/etc/fstab
    

Next start to configure the system.

    arch-chroot root
    ln -s /usr/share/zoneinfo/Europe/London /etc/localtime # Replace Region/City with your value
    hwclock --systohc
    nano /etc/locale.gen # Uncomment en_GB.UTF-8 UTF-8 line and save
    locale-gen
    echo "LANG=en_GB.UTF-8" > /etc/locale.conf
    nano /etc/mkinitcpio.conf # Add btrfs to MODULES=()
    mount /dev/nvme0n1p1 boot
    mkinitcpio -p linux-aarch64 # re-generate initial ram disk
    umount boot
    exit

Create a startup.nsh file that the UEFI shell will run on boot. Replace /dev/nvme0n1p2 below with the relevant partition identifier.

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img" >> boot/startup.nsh
    cat boot/startup.nsh

Finally unmount and exit from root and then power off. Remove the USB drive.

    umount boot root/home root/.snapshots root/var/log root
    poweroff



Once booted, initialise the pacman keyring, populate the signing keys and update the system packages:

    pacman-key --init
    pacman-key --populate archlinuxarm
    pacman -S btrfs-progs
    #pacman -Syu
    reboot











## Building Kernel

Based on information at https://gist.github.com/thalamus/561d028ff5b66310fac1224f3d023c12 and https://wiki.archlinux.org/index.php/Kernel/Traditional_compilation

Ensure all the packages required for building a kernel are installed.

    pacman -S base-devel git xmlto kmod inetutils bc libelf cpio perl tar xz python

Pull down the latest kernel source from SolidRun's GitHub, ensure the kernel tree is clean, create the kernel configuration based on the running stock Arch kernel, customise the config to add required modules and then finally start the kernel compilation.>

    # Preparation
    git clone --depth 1 -b linux-5.10.y-cex7 https://github.com/SolidRun/linux-stable linux-source-5.10 && cd linux-source-5.10
    make mrproper
    # Configuration
    zcat /proc/config.gz > .config
    #curl -o .config https://raw.githubusercontent.com/archlinux/svntogit-packages/26f78450f0fc7cd12bc5a041e1248739a0ce11bf/trunk/config
    #curl -o .config https://raw.githubusercontent.com/archlinuxarm/PKGBUILDs/master/core/linux-aarch64-rc/config
    #make nconfig
    #make olddefconfig
    sed -ri '/CONFIG_PHYLIB/s/=.+/=y/g' .config             # Ensure phylib is enabled by default for onboard networking
    sed -ri '/CONFIG_NR_CPUS/s/=.+/=16/g' .config           # Config limited to 8 cores by default for some reason
    
    
    # Enable HoneyComb specific modules
    echo "CONFIG_NET_SWITCHDEV=y" >> .config
    echo "CONFIG_FSL_MC_BUS=y" >> .config
    echo "CONFIG_FSL_MC_UAPI_SUPPORT=y" >> .config
    echo "CONFIG_FSL_XGMAC_MDIO=y" >> .config
    echo "CONFIG_FSL_DPAA2_ETH=m" >> .config
    echo "CONFIG_FSL_DPAA2_PTP_CLOCK=m" >> .config
    echo "CONFIG_FSL_DPAA2_QDMA=m" >> .config
    echo "CONFIG_STAGING=y" >> .config
    echo "CONFIG_FSL_DPAA2=y" >> .config
    echo "CONFIG_FSL_DPAA2_ETHSW=m" >> .config
    echo "CONFIG_ARM_SMMU_DISABLE_BYPASS_BY_DEFAULT=y" >> .config
    echo "CONFIG_FSL_MC_DPIO=m" >> .config
    echo "CONFIG_ARM_GIC_V3_ITS_FSL_MC=y" >> .config
    echo "CONFIG_CRYPTO_DEV_FSL_DPAA2_CAAM=m" >> .config
    echo "CONFIG_ARCH_LAYERSCAPE=y" >> .config
    echo "CONFIG_PCIE_MOBIVEIL=y" >> .config
    echo "CONFIG_PCIE_MOBIVEIL_HOST=y" >> .config
    echo "CONFIG_PCIE_LAYERSCAPE_GEN4=y" >> .config
    echo "CONFIG_PCI_QUIRKS=y" >> .config

    # Enable AMD GPU


    # Compilation
    make -j$(nproc) Image Image.gz modules
    # Install modules
    make modules_install

Next, copy the kernel to the boot partition and then generate the initial RAM disk:

    cp -v arch/arm64/boot/Image /boot/Image510
    cp -v arch/arm64/boot/Image.gz /boot/Image510.gz
    mkinitcpio -k 5.10.5-ARCH+ -g /boot/initramfs-linux510.img

Finally update boot loader (startup.nsh for now) to load new kernel, along with a few parameters to work around current issues:

Image510 root=UUID=0e2373e4-c270-4a31-af5f-8d83dcc815bc rw rootfstype=ext4 initrd=initramfs-linux510.img arm-smmu.disable_bypass=0 amdgpu.pcie_gen_cap=0x4

echo "Image510 root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux510.img arm-smmu.disable_bypass=0 amdgpu.pcie_gen_cap=0x4" > /boot/startup.nsh

