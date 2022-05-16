KVM
https://phoenixnap.com/kb/ubuntu-install-kvm

OpenRC services
https://wiki.gentoo.org/wiki/OpenRC_to_systemd_Cheatsheet

SSH
https://phoenixnap.com/kb/ssh-to-connect-to-remote-server-linux-or-windows


    Error starting domain: Requested operation is not valid: network 'default' is not active

    Traceback (most recent call last):
    File "/usr/share/virt-manager/virtManager/asyncjob.py", line 75, in cb_wrapper
    callback(asyncjob, *args, **kwargs)
    File "/usr/share/virt-manager/virtManager/asyncjob.py", line 111, in tmpcb
    callback(*args, **kwargs)
    File "/usr/share/virt-manager/virtManager/libvirtobject.py", line 66, in newfn
    ret = fn(self, *args, **kwargs)
    File "/usr/share/virt-manager/virtManager/domain.py", line 1400, in startup
    self._backend.create()
    File "/usr/lib/python3/dist-packages/libvirt.py", line 1080, in create
    if ret == -1: raise libvirtError ('virDomainCreate() failed', dom=self)
    libvirt.libvirtError: Requested operation is not valid: network 'default' is not active

Type:

    sudo virsh net-start default 
