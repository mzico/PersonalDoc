# Configure static IP in Ubuntu 14.04 

 - Make sure your VM is using 'Broadcast' network type
 - SSH into the VM or log into GUI ( open terminal in this case )
 - `vim -N /etc/network/interfaces`
   ```
   # interfaces(5) file used by ifup(8) and ifdown(8)
    auto lo
    iface lo inet loopback

    auto eth0
    iface eth0 inet static
    address 192.168.0.140
    netmask 255.255.255.0
    gateway 192.168.0.1
    broadcast 192.168.0.255
    dns-domain 8.8.8.8
   ```
     - Address/Gateway/Broadcast address highly dependent on your own network / router config. 
 - `ifdown eth0`
 - `ifup eth0`
