# Building Kernel

Based on information at https://gist.github.com/thalamus/561d028ff5b66310fac1224f3d023c12 and https://wiki.archlinux.org/index.php/Kernel/Traditional_compilation

Ensure all the packages required for building a kernel are installed.

    pacman -S base-devel git xmlto kmod inetutils bc libelf cpio perl tar xz python wget

Pull down the latest kernel source from SolidRun's GitHub, ensure the kernel tree is clean, create the kernel configuration based on the default Arch Linux ARM config, merge in the required config for the HoneyComb and then finally start the kernel compilation. NOTE: the generic Arch ARM image has a kernel parameter limiting the number of CPUs to 8 (https://github.com/archlinuxarm/PKGBUILDs/blob/master/core/linux-aarch64/config#L393) so the first kernel build wont be full throttle. This is corrected in the .config.HoneyComb kernel fragments file which is merged in below.

    # Preparation
    git clone -b linux-5.18.y-cex7 https://github.com/SolidRun/linux-stable && cd linux-stable
    git remote add kernel-org https://git.kernel.org/pub/scm/linux/kernel/git/stable/linux.git
    git pull --rebase kernel-org linux-5.18.y

    # Ensure clean
    make mrproper
    
    # Use default Arch Linux ARM config, along with additional options required for HoneyComb
    wget https://raw.githubusercontent.com/archlinuxarm/PKGBUILDs/master/core/linux-aarch64/config -O .config
    wget https://raw.githubusercontent.com/frank-bell/HoneyComb/main/.config.HoneyComb -O .config.HoneyComb
    ./scripts/kconfig/merge_config.sh .config .config.HoneyComb
        
    # Compilation
    make -j$(nproc) ARCH=arm64 Image Image.gz modules
    # Install modules
    sudo make modules_install

Next, copy the kernel to the boot partition and then generate the initial RAM disk. 

    sudo cp -v arch/arm64/boot/Image /boot
    sudo cp -v arch/arm64/boot/Image.gz /boot
    sudo mkinitcpio -k 5.18.5-ARCH+ -g /boot/initramfs-linux.img

Finally update "boot loader" (startup.nsh for now) to load new kernel, along with a few parameters to work around current known issues. The below is based on my NVMe BTRFS setup, but you could use something like the USB version in the main README for a more traditional setup:

    echo "Image root=UUID=$(blkid -s UUID -o value /dev/nvme0n1p2) rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img amdgpu.pcie_gen_cap=0x4 quiet" > /boot/startup.nsh

The following ideal approach is not currently working as at last test, need to look into it again.

    # sudo efibootmgr --disk /dev/nvme0n1 -e 3 --create --label "Arch Linux ARM via efibootmgr" --loader /Image --UCS-2 'root=UUID=a51cd699-f746-4722-95e9-be86cd8eab43 rootflags=subvol=@ rw rootfstype=btrfs initrd=initramfs-linux.img amdgpu.pcie_gen_cap=0x4 quiet udev.log_level=3 vt.global_cursor_default=0' --verbose

If everything is working, update /etc/pacman.conf to ignore kernel package updates until all of the SolidRun patches are in the mainline kernel.

    IgnorePkg   = linux-aarch64
