# Installing KVM on Void Linux  

First we are going to start by testing our computer, this will tell us if its capable of virtualizing.

    egrep -c '(vmx|svm)' /proc/cpuinfo
    
Now we are going to install the basics

    sudo xbps-install virt-manager libvirt qemu
