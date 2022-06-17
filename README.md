# HoneyComb

This rough guide provides steps for getting a working Arch Linux ARM install on the HoneyComb LX2K. The steps were documented as I first installed things from an x86 Arch machine and then later updated as I progressed with work on the machine itself. Please also note that I am pretty new to Arch and Linux so some of this might not be the best way to go about things. Any corrections/tips/pointers greatly appreciated! Based on information from https://gist.github.com/meme/c1f1101fac0f58e883ae08872f19b883 and https://dev.to/lizthegrey/first-experiences-with-honeycomb-lx2k-26be.

## Building UEFI firmware
Standard firmware images are available from https://images.solid-run.com, but custom firmware can easily be built using the SolidRun provided Docker image as follows:

    git clone --depth 1 https://github.com/SolidRun/lx2160a_uefi.git && cd lx2160a_uefi
    sudo docker build -t lx2160a_uefi docker/ && sudo docker run -e SOC_SPEED=2200 -e BUS_SPEED=800 -e DDR_SPEED=3200 -e XMP_PROFILE=1 -v "$PWD":/work:Z --rm -i -t lx2160a_uefi build

The resulting image will appear within the `images` subdirectory. Dont forget to clean up the resulting Docker image with `sudo docker rmi lx2160a_uefi`. 
  
Finally flash the firmware to a SD card. Use lsblk to list the disks and check through the output to find the correct target disk. When doing it on my x86 machine it was listed as /dev/sdX, but when doing directly on the HoneyComb it was mmcblk0. An alternative is to use https://www.balena.io/etcher which seems to provide better error notifications. I had an issue where firmware didnt seem to be writing via dd with no error, tried it via etcher and got an error which was resolved by a reboot.

    lsblk
    sudo dd if=images/lx2160acex7_2200_800_3200_8_5_2_sd_ee5c233.img of=/dev/mmcblk0 conv=fsync
    
Insert the SD card into the Honeycomb, connect the USB console cable and then fire up [tio](https://aur.archlinux.org/packages/tio) before finally powering up the Honeycomb to check the firmware loads up. You could also use a monitor connected directly if required, but the linked articles said not to and when I first attempted this I didnt yet have a GPU installed.

    sudo tio /dev/ttyUSB0
    
## Building an Arch Linux ARM USB

The below steps will create a bootable Arch Linux ARM USB drive with the generic Arch Linux ARM build. You will need to use a USB network adapter until a custom kernel is built later in this guide. I did it this way as practice on my main machine, so I have a bootable USB version of Arch to keep handy and to let me kick off a local install on NVMe later. Based on information from https://itsfoss.com/install-arch-raspberry-pi/ and https://gist.github.com/thalamus/561d028ff5b66310fac1224f3d023c12.

On a Linux machine, firstly create a working folder and then pull down the latest generic ARM build:

    mkdir alarm && cd alarm
    wget http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz
    
Then change to root:

    sudo su
    
Insert the USB disk and then list the disks to determine the identifier, in my case it was /dev/sdg:

    lsblk -o NAME,PATH,SIZE
    
Next we will set up the required partitions:

    pacman -S gptfdisk
    gdisk /dev/sdX

Type o to clear out any partitions, then p to check that they have been cleared. To create the boot partition, type n for a new partition, 1 for first partition, enter to accept default first sector and then +260M for last. Then enter ef00 for the EFI system partition.

    o   p   n   1   ENTER   +260M   ef00

To create the root partition, type n for a new partition, 2 for second partition, enter to accept default first sector and enter again for last. Press enter again to accept the default linux filesystem.

    n   2   ENTER   ENTER   ENTER
    
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

    tar -xzvf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
    sync
    
Next we need to populate fstab:

    pacman -S arch-install-scripts
    genfstab -U /mnt >> /mnt/etc/fstab
    cat /mnt/etc/fstab
    
The USB device will require a startup.nsh file that the UEFI shell will run on boot. This is using the EFIStub functionality of the kernel. Replace /dev/sdxn below with the relevant partition identifier. See the UEFI boot entry section further below if you want to create one.

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/sdX2) rw rootfstype=ext4 initrd=initramfs-linux.img arm-smmu.disable_bypass=0 iommu.passthrough=1 amdgpu.pcie_gen_cap=0x4" > /mnt/boot/startup.nsh
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

## Installing Arch Linux ARM on NVMe

We can essentially repeat the steps for setting up a USB drive, but this time installing onto an NVMe disk installed on the HoneyComb.You will still need to use a USB network adapter until a custom kernel is built later in this guide.

If applicable, start up minicom and then boot the HoneyComb, logging in as root with password root.

    sudo minicom -c on -D /dev/ttyUSB0

Create a working folder and then pull down the latest generic ARM build:

    mkdir alarm && cd alarm
    curl -LO http://os.archlinuxarm.org/os/ArchLinuxARM-aarch64-latest.tar.gz

List the disks to determine the NVMe identifier, in my case it was /dev/nvme0n1:

    lsblk -o NAME,PATH,SIZE

Next we will set up the required partitions:

    pacman -S gptfdisk
    gdisk /dev/nvme0n1

Type o to clear out any partitions, then p to check that they have been cleared. To create the boot partition, type n for a new partition, 1 for first partition, enter to accept default first sector and then +1G for last. Then enter ef00 for the EFI system partition.

    o   p   n   1   ENTER   +1G   ef00

To create the root partition, type n for a new partition, 2 for second partition, enter to accept default first sector and enter again for last.

    n   2   ENTER   ENTER   ENTER

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

    tar -xzvf ArchLinuxARM-aarch64-latest.tar.gz -C /mnt
    sync

Next we need to set up the boot partition and populate fstab (amend as necessary):

    pacman -S arch-install-scripts
    genfstab -U /mnt >> /mnt/etc/fstab
    cat /mnt/etc/fstab

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

Create a startup.nsh file on the NVMe boot partition that the UEFI shell will run on boot. Replace /dev/nvme0n1p2 below with the relevant partition identifier. See the UEFI boot entry section further below if you want to create one.

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img" > /mnt/boot/startup.nsh
    cat /mnt/boot/startup.nsh

Finally unmount and exit from root and then power off. Remove the USB drive.

    umount /mnt/boot /mnt/home /mnt/.snapshots /mnt/var/log /mnt
    poweroff

Start up again and once booted, initialise the pacman keyring, populate the signing keys and update the system packages:

    pacman-key --init
    pacman-key --populate archlinuxarm
    pacman -Syu # NOTE: dont update linux-aarch64 to 5.11.*
    reboot
    
Once rebooted, change the default root password and then set up a new user with sudo privileges:

    passwd
    useradd -m -G wheel -s /bin/bash username
    visudo   # Uncomment the line "%wheel   ALL=(ALL)   ALL" and then enter :wq to save and quit
    passwd username
    
Test that you can login as new new user on another tty (CTRL-ALT-F2) and if successful, delete the default alarm user:

    userdel alarm
    
Finally set a host name and reboot when required.

    hostnamectl set-hostname "your hostname"

## Building Kernel

Information can now be found [here](Kernel.md).
    
## Creating UEFI Boot Entry

I havent had success with creating a UEFI boot entry directly from Arch yet, but I have gotten close. The ideal scenario would be to use efibootmgr, but a workaround is to use bcfg from within the UEFI shell. Both options detailed below.

### UEFI Shell

Firstly generate a boot options file from Arch, which will hold all the required kernel parameters and which will be used to make entry creation within the shell easier. The file must be encoded using UCS-2:

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img amdgpu.pcie_gen_cap=0x4 arm-smmu.disable_bypass=0 iommu.passthrough=1 quiet udev.log_level=3 vt.global_cursor_default=0" | iconv -t ucs2 -o /boot/options.txt
    
Boot into the UEFI Shell and then enter the following to create the entry, which will set it as the first boot entry.

    bcfg boot add 0 FS0:\Image "Arch Linux ARM"
    bcfg boot -opt 0x0 FS0:\options.txt

You can use the following to list all entries verbosely

    bcfg boot dump -v -b

NOTE: you cannot update an existing entry as the -opt command appends information to the supplied boot entry rather than replacing. You can remove an entry with the following:

    bcfg boot rm #
    
And finally reset to restart and boot straight into Arch

    reset

### efitbootmgr

efibootmgr currently seems to generate the drive identifier starting with PciRoot(0x0)/Pci(0x0,0x0)/NVMe ... whereas the UEFI firmware generates PcieRoot(0x2)/Pci(0x0,0x0)/Pci(0x0,0x0)/NVMe... As a result the entry just seems to disappear after adding, presumably as it is invalid. The ideal steps would be as below.

Install the efibootmgr package to create a UEFI boot entry. Use blkid to identify the root partition (/dev/sda2 in my case) and then efibootmgr to add the boot entry (amend as necessary):

    pacman -S efibootmgr
    blkid
    efibootmgr --disk /dev/Xda -e 3 --create --label "Arch Linux ARM" --loader /Image --UCS-2 'root=UUID=XXXXXXXX-XXXX-XXXX-XXXX-XXXXXXXXXXXX rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img amdgpu.pcie_gen_cap=0x4 arm-smmu.disable_bypass=0 iommu.passthrough=1 quiet udev.log_level=3 vt.global_cursor_default=0' --verbose
