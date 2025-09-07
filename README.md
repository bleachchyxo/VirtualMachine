# Installing KVM QUEMU on Artix Linux

    LC_ALL=C lscpu | grep Virtualization

You should see the output as “Virtualization: VT-x” or “Virtualization: AMD-V”. If nothing is displayed, it means your PC can’t be used to install Virtual machines. 
Manufacturers sometimes disable the feature by default settings. To make sure, boot your computer’s BIOS and check. Refer to your computer maker and model manual for how to boot into BIOS.

## Installing KVM

    sudo pacman -S virt-manager qemu libvirt
