# Libvirt

By typing the command `ifconfig -a` or `ip -a` on our machine it will output something like this:

    eth0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
            ether 00:21:cc:b5:74:ee  txqueuelen 1000  (Ethernet)
            RX packets 0  bytes 0 (0.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 0  bytes 0 (0.0 B)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
            device interrupt 20  memory 0xf2500000-f2520000

    lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
            inet 127.0.0.1  netmask 255.0.0.0
            inet6 ::1  prefixlen 128  scopeid 0x10<host>
            loop  txqueuelen 1000  (Local Loopback)
            RX packets 0  bytes 0 (0.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 0  bytes 0 (0.0 B)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 192.168.0.103  netmask 255.255.255.0  broadcast 192.168.0.255
            inet6 fe80::120b:a9ff:fe84:f37c  prefixlen 64  scopeid 0x20<link>
            ether 10:0b:a9:84:f3:7c  txqueuelen 1000  (Ethernet)
            RX packets 41991  bytes 53347267 (50.8 MiB)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0  

We are going to start making sure the libvirt deamon is running:

    $ rc-service libvirtd start
    

Now we proceed checking the libvirt networks available with this command:

    $ sudo virsh net-list

This is the command output in my machine:

     Name   State   Autostart   Persistent
    ----------------------------------------
     

As you can see there is no networks setup by default. In order to change that we need to start our default network

    $ doas virsh net-start default

And now we can check again the networks using the `sudo virsh net-list` command and the output should look like this:

     Name      State    Autostart   Persistent
    --------------------------------------------
     default   active   no          yes

You need to understand that the `default` network is where typically machines will attach themselves by default, also by running this command a couple things ocurred in the background.

By entering the commnad `ifconfig -a` or `ip -a` you can see that two new network interfaces got created:

    eth0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
            ether 00:21:cc:b5:74:ee  txqueuelen 1000  (Ethernet)
            RX packets 0  bytes 0 (0.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 0  bytes 0 (0.0 B)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
            device interrupt 20  memory 0xf2500000-f2520000

    lo: flags=73<UP,LOOPBACK,RUNNING>  mtu 65536
            inet 127.0.0.1  netmask 255.0.0.0
            inet6 ::1  prefixlen 128  scopeid 0x10<host>
            loop  txqueuelen 1000  (Local Loopback)
            RX packets 0  bytes 0 (0.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 0  bytes 0 (0.0 B)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    virbr0: flags=4099<UP,BROADCAST,MULTICAST>  mtu 1500
            inet 192.168.122.1  netmask 255.255.255.0  broadcast 192.168.122.255
            ether 52:54:00:4e:60:a8  txqueuelen 1000  (Ethernet)
            RX packets 0  bytes 0 (0.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 0  bytes 0 (0.0 B)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    virbr0-nic: flags=4098<BROADCAST,MULTICAST>  mtu 1500
            ether 52:54:00:4e:60:a8  txqueuelen 1000  (Ethernet)
            RX packets 0  bytes 0 (0.0 B)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 0  bytes 0 (0.0 B)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

    wlan0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
            inet 192.168.0.103  netmask 255.255.255.0  broadcast 192.168.0.255
            inet6 fe80::120b:a9ff:fe84:f37c  prefixlen 64  scopeid 0x20<link>
            ether 10:0b:a9:84:f3:7c  txqueuelen 1000  (Ethernet)
            RX packets 49969  bytes 60457335 (57.6 MiB)
            RX errors 0  dropped 0  overruns 0  frame 0
            TX packets 16831  bytes 3305453 (3.1 MiB)
            TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0

The first one is `virbr0` also known as the `virtual bridge interface` and it works as a virtual switch running inside of the host, can provide internet access by NAT mode and it also provide DHCP.

The second one is `virbr0-nic`, I don't really know what it is but right now isn't that relevant.

        $ sudo virsh list
        
Currently I'm not running any virtual machine as you can see in the ouptut:

         Id   Name   State
        --------------------
         

In order to list all our existing virtual machines you need to enter the following command:

        $ sudo virsh list --all
        
As you can see I only have one machine created named `pc1`
        
         Id   Name   State
        -----------------------
         -    pc1    shut off

In order to check if your machine is using the `defaut network` you can enter this command:

        $  sudo virsh dumpxml <name> | grep -i 'network='
        
In my case:

        $ sudo virsh dumpxml pc1 | grep -i 'network='
        
You can see how it actually uses the `default` network

              <source network='default'/>

Now we are going to start our machine:

        $ sudo virsh start pc1
        
We can see that by entering again the command `sudo virsh list` our machine will appear running

         Id   Name   State
        ----------------------
         1    pc1    running
         
Once our machine is running we can proceed to enter the `ifconfig -a` or `ip -a` commnad again and we will see a new interface created

        vnet0: flags=4163<UP,BROADCAST,RUNNING,MULTICAST>  mtu 1500
                inet6 fe80::fc54:ff:fed1:770  prefixlen 64  scopeid 0x20<link>
                ether fe:54:00:d1:07:70  txqueuelen 1000  (Ethernet)
                RX packets 70  bytes 9550 (9.3 KiB)
                RX errors 0  dropped 0  overruns 0  frame 0
                TX packets 325  bytes 20473 (19.9 KiB)
                TX errors 0  dropped 0 overruns 0  carrier 0  collisions 0
                
The `vnet` interfaces are also called `tap` interfaces, and they're attached to the process running `qemu-kvm` emulator.
