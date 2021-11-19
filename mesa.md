# Patching Mesa

Run the following commands to download the mesa package definition along with additional patch:

    sudo pacman -S svn
    svn export https://github.com/archlinuxarm/PKGBUILDs/trunk/extra/mesa && cd mesa
    wget https://raw.githubusercontent.com/void-linux/void-packages/master/srcpkgs/mesa/patches/0001-radeonsi-On-Aarch64-force-persistent-buffers-to-GTT.patch

Edit the PKGBUILD and add the following line into the source section to add the patch:

    0001-radeonsi-On-Aarch64-force-persistent-buffers-to-GTT.patch
        
And then add the following to the end of prepare() to apply the patch:

    patch -p1 -i ../0001-radeonsi-On-Aarch64-force-persistent-buffers-to-GTT.patch
  
Finally re-generate the integrate checks and then build/install the package:
       
    makepkg -g >> PKGBUILD
    makepkg --ignorearch --skippgpcheck -si
