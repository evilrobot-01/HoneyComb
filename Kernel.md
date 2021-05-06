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
    sed -i '/CONFIG_HIDRAW/s/.*/CONFIG_HIDRAW=y/' .config
    sed -i '/CONFIG_HID_PID/s/.*/CONFIG_HID_PID=y/' .config
    sed -i '/CONFIG_USB_HIDDEV/s/.*/CONFIG_USB_HIDDEV=y/' .config
    sed -ri '/CONFIG_NLS_DEFAULT=/s/=.+/="utf8"/g' .config
    sed -i '/CONFIG_NLS_ASCII/s/.*/CONFIG_NLS_ASCII=y/' .config
    sed -ri '/CONFIG_NLS_ISO8859_1/s/=.+/=m/g' .config
    sed -i '/CONFIG_TMPFS_POSIX_ACL/s/.*/CONFIG_TMPFS_POSIX_ACL=y/' .config
    # Enable HoneyComb specific
    sed -i '/CONFIG_FSL_MC_UAPI_SUPPORT/s/.*/CONFIG_FSL_MC_UAPI_SUPPORT=y/' .config
    sed -i '/CONFIG_FSL_DPAA2_QDMA/s/.*/CONFIG_FSL_DPAA2_QDMA=m/' .config
        # sed -i '/CONFIG_STAGING/s/.*/CONFIG_STAGING=y/' .config
    echo "CONFIG_FSL_DPAA2=y" >> .config
    echo "CONFIG_FSL_DPAA2_ETHSW=m" >> .config
    # Silent boot
    sed -i '/CONFIG_LOGO=/s/.*/# CONFIG_LOGO is not set/' .config
    # A few peripherals specifc to my setup
    sed -i '/CONFIG_DRM_AMDGPU/s/.*/CONFIG_DRM_AMDGPU=m/' .config
    sed -i '/CONFIG_FB_SIMPLE/s/.*/CONFIG_FB_SIMPLE=y/' .config
    sed -i '/CONFIG_HID_MAGICMOUSE/s/.*/CONFIG_HID_MAGICMOUSE=m/' .config
    sed -i '/CONFIG_SND_HDA_INTEL/s/.*/CONFIG_SND_HDA_INTEL=m/' .config
    sed -i '/CONFIG_SND_USB_AUDIO/s/.*/CONFIG_SND_USB_AUDIO=m/' .config
    
    # Compilation
    make -j$(nproc) ARCH=arm64 Image Image.gz modules
    # Install modules
    sudo make modules_install

Next, copy the kernel to the boot partition and then generate the initial RAM disk. 

    sudo cp -v arch/arm64/boot/Image /boot
    sudo cp -v arch/arm64/boot/Image.gz /boot
    sudo mkinitcpio -k 5.10.23-ARCH+ -g /boot/initramfs-linux.img

Finally update "boot loader" (startup.nsh for now) to load new kernel, along with a few parameters to work around current known issues. The below is based on my NVMe BTRFS setup, but you could use something like the USB version above for a more traditional setup:

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img amdgpu.pcie_gen_cap=0x4 quiet" > /boot/startup.nsh
    # sudo efibootmgr --disk /dev/nvme0n1 -e 3 --create --label "Arch Linux ARM via efibootmgr" --loader /Image --UCS-2 'root=UUID=a51cd699-f746-4722-95e9-be86cd8eab43 rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img amdgpu.pcie_gen_cap=0x4 quiet udev.log_level=3 vt.global_cursor_default=0' --verbose

If everything is working, update /etc/pacman.conf to ignore kernel package updates until all of the SolidRun patches are in the mainline kernel.

    IgnorePkg   = linux-aarch64
