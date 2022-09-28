# Virtual Machine Guide Setup

## Check for virtualization support on your machine
  
    egrep -c '(vmx|svm)' /proc/cpuinfo
    
If the command returns 0 it means your processor isn't capable of running KVM, any other number means you can proceed with the installation.

## Install KVM packages

    sudo apt update

and install the KVM essentials:

    sudo apt install qemu-kvm libvirt-daemon-system libvirt-clients bridge-utils

## Authorize users
    
    sudo adduser username libvirt
    
Replace `username` for your actual username
    
    sudo adduser username] kvm
    
Replace `username` for your actual username

In case having trouble with the `adduser` command use:

    sudo /usr/sbin/adduser username libvirt
    sudo /usr/sbin/adduser username kvm

## Verify the installation

    virsh list --all

When I first ran the command I got an error:
    
    error: failed to connect to the hypervisor
    error: Cannot recv data: Connection reset by peer
    
Truns out I just need to run the  command as root.
