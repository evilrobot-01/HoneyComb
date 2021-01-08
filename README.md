# HoneyComb

Based on information from https://gist.github.com/meme/c1f1101fac0f58e883ae08872f19b883 and https://dev.to/lizthegrey/first-experiences-with-honeycomb-lx2k-26be.

## Building firmware
Ensure required packages are installed.

    sudo pacman -S acpica dtc
    
Clone the git repository, change to directory, set an alias for the `arch` command and then run within current shell so the alias actually takes effect (seems a script is normally run within a subshell and aliases are therefore not taken into account).

    git clone https://github.com/SolidRun/lx2160a_uefi.git && cd lx2160a_uefi
    alias arch="uname -m" && INITIALIZE=1 . ./runme.sh
    
Once initialised, start the build of the firmware:
