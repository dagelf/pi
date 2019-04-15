# Raspberry Pi 3 B+ LAN / pxe boot with NFS root (if you don't have an SD card or reader)

I didn't have a way to write an SD card, so I booted up my Pi 3 B+ over LAN and wrote the SD card from the Pi itself. Unlike some reports, I didn't have to do anything on or to my Pi out of the box. (You do have to set a flag if your Pi contains a bootable SD card, or else it will just boot from the card instead of the LAN.)

I used my Ubuntu 18.04 laptop as the server, and connected the Pi to the LAN port on my laptop, while my laptop was on Wi-Fi, and shared the laptop internet with the Pi via the LAN socket. My Wi-Fi IP address was in the 192.168.1.x range so I set up the LAN port as 192.168.10.x. The LAN device name was eno1. (In other linuxes it's often eth0, eth1 or so.)

These steps should also work with a Pi as a server. (Or from one Pi to many, via a network switch - but in that case it would need some expanding to serve different disk images or filesystems to the different Pi's!)

Visit [raspberrypi.org](https://www.raspberrypi.org/downloads/raspbian/) and download the latest image, or:

    $ wget http://director.downloads.raspberrypi.org/raspbian_full/images/raspbian_full-2018-11-15/2018-11-13-raspbian-stretch-full.zip
    ...
    $ unzip 2018-11-13-raspbian-stretch-full.zip

Or: (untested)

    $ wget -O pi.zip https://downloads.raspberrypi.org/raspbian_full_latest
    $ mkdir _pi
    $ unzip pi.zip -d _pi/
    $ mv _pi/*.img pi.img
    $ rmdir _pi 
    $ ls -l pi.img
    $ # rm pi.zip

Then some set-up:

    $ sudo su # this is just easier than adding sudo in front of all of the below
    # mkdir /nfs
    # mkdir /nfs/pi1
    # kpartx -av pi.img   # or 2018-11-13-raspbian-stretch-lite.img
    add map loop30p1 (253:0): 0 89854 linear 7:30 8192
    add map loop30p2 (253:1): 0 3547136 linear 7:30 98304

    # echo /dev/mapper/loop*  # check the loop device names
    # if /dev/mapper/loop30p1 doesn't exist: 
    # mknod /dev/loop30p1 b 253 0 # might not be needed, numbers from kpartx output
    # mknod /dev/loop30p2 b 253 1 # might not be needed, numbers from kpartx output
    # mkdir /mnt/pi
    # mount /dev/loopXXp2 /mnt/pi      # or /dev/mapper/loopXXp2
    # mount /dev/loopXXp1 /mnt/pi/boot # or /dev/mapper/loopXXp1
    # rsync -xav /mnt/pi/ /nfs/pi1     # trailing / needed else it copies the top directory into the target
    # cd /nfs/pi1
    
You are now in the pi NFS filesystem, get it ready to boot:

    # vi /nfs/pi1/boot/cmdline.txt
    dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=/dev/nfs nfsroot=192.168.10.1:/nfs/pi1,tcp,v3 rootfstype=nfs rw ip=dhcp rootwait elevator=deadline 

Remove everything except the first line that starts with /proc in fstab, add /sys:

    # vi /nfs/pi1/etc/fstab
    proc            /proc           proc    defaults          0       0
    sysfs 		/sys 			sysfs	defaults 		  0 	  0
    
Make it start ssh on boot: (This is pi specific, just like the config.txt file)

    # touch /nfs/pi1/boot/ssh
    
The default login and password is pi and raspberry    
    
If you want to add your ssh key: (optional)

    # mkdir /nfs/pi1/home/pi/.ssh
    # cat /home/XXXX/.ssh/id_rsa.pub >> /nfs/pi1/home/pi/.ssh/authorized_keys
    # chmod -R 600 /nfs/pi1/home/pi/.ssh
   
Change the default user to your default user on the server: (optional)
    
    U=newuser 
    sed s@:pi@:$Ug -i /nfs/pi1/etc/passwd
    sed s@:pi@:$Ug -i /nfs/pi1/etc/shadow
    sed s@:pi@:$Ug -i /nfs/pi1/etc/groups
    sed s@pi:@$U:g -i /nfs/pi1/etc/passwd
    sed s@pi:@$U:g -i /nfs/pi1/etc/shadow
    sed s@pi:@$U:g -i /nfs/pi1/etc/groups
    mv /nfs/pi1/home/pi /nfs/pi1/home/$U
    
If you're on a Linux that uses NetworkManager and systemd-resolved (eg. Debian or Ubuntu) and you want full control over your ethernet interface, which you will need if you want to run dnsmasq yourself, do this before the next steps: 

    # ip link || ifconfig # to find your ethernet interface name - unlikely that it will be eno1 or eth1

    # vi /etc/udev/rules.d/00-net.rules 
    
    # Interfaces that shouldn't be managed by NetworkManager 
    ACTION=="add", SUBSYSTEM=="net", KERNEL=="eno1", ENV{NM_UNMANAGED}="1"
    ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth1", ENV{NM_UNMANAGED}="1"
    
    :w
    :q
    
    # vi /etc/NetworkManager/NetworkManager.conf
    [main]
    plugins=ifupdown,keyfile
    dns=none  # add this

    [ifupdown]
    managed=false

    [device]
    wifi.scan-rand-mac-address=no
 
    :w
    :q 

    # echo nameserver 1.1.1.1 > /etc/resolv.conf

    # systemctl restart network
    # systemctl restart NetworkManager
    # systemctl disable systemd-resolved.service
    # systemctl stop systemd-resolved.service
    # For some reason Ubuntu sets dnsmasq up as a service when you explicitly install it, undo it!
    # systemctl disable dnsmasq.service
    # systemctl stop dnsmasq.service
    # rm /etc/resolv.conf  # this is a symlink to a dynamic file, which gets automatically overwritten
    # echo nameserver 1.1.1.1 > /etc/resolv.conf # use cloudflare DNS, or change to your own
    # I have also seen newer firefoxes take over port :53, to disable this open about:config and
    # look for network.dns.offline-localhost and set it to false
    
Configure an IP addres on your network interface - your interface name will likely be different:

    # ip link || ifconfig # to find your ethernet interface name, you hopefully still have this from above... :-) 
    # ifconfig eno1 192.168.10.1 up

If you want to give your Pi internet access through the server: (this is all you need to turn your server into a router)

    # echo 1 > /proc/sys/net/ipv4/ip_forward
    # iptables -P FORWARD ACCEPT                    # docker changes this policy for you, on of its many evils
    # iptables -t nat -A POSTROUTING -j MASQUERADE

See https://www.google.com/search?q=iptables if you want to know more

Enable NFS on your server: (and test it)
   
    # apt-get install nfs-kernel-server
    # vi /etc/exports

    # /etc/exports: the access control list for filesystems which may be exported
    #               to NFS clients.  See exports(5).
    #
    # Example for NFSv2 and NFSv3:
    # /srv/homes       hostname1(rw,sync,no_subtree_check) hostname2(ro,sync,no_subtree_check)
    #
    # Example for NFSv4:
    # /srv/nfs4        gss/krb5i(rw,sync,fsid=0,crossmnt,no_subtree_check)
    # /srv/nfs4/homes  gss/krb5i(rw,sync,no_subtree_check)
    #
    /nfs/pi1 192.168.10.0/24(rw,sync,no_subtree_check,no_root_squash)

    :w
    :q
    # exportfs -ra
    # systemctl restart nfs-kernel-service || systemctl stop nfs-mountd.service # the latter is on latest Ubuntu
    
    # systemctl status nfs-server.service || systemctl status nfs-mountd.service 
    ● nfs-server.service - NFS server and services
    Loaded: loaded (/lib/systemd/system/nfs-server.service; enabled; vendor preset: enabled)
    Active: active (exited) since Tue 2018-12-18 12:23:02 GMT; 10h ago
    Main PID: 9364 (code=exited, status=0/SUCCESS)
    Tasks: 0 (limit: 4915)
    CGroup: /system.slice/nfs-server.service

    Dec 18 12:23:02 server systemd[1]: Starting NFS server and services...
    Dec 18 12:23:02 server systemd[1]: Started NFS server and services.

    # showmount -e
    Export list for server:
    /nfs/pi1 192.168.10.0/24

    # cat /proc/fs/nfs/exports  # this only shows what is actually used, so need to boot the pi first
    # Version 1.1
    # Path Client(Flags) # IPs
    /nfs/pi1    192.168.10.0/24(rw,no_root_squash,sync,wdelay,no_subtree_check,uuid=3b6ace17:e18e4834:96f5c0eb:5cbe2796,sec=1)

    # note that you can't run docker etc. on a client nfs path, if you need something like that you will need to have some sort of coordinated service running on the server
Now serve DHCP on your lan port and get ready to boot up your Pi:

    # dnsmasq -d -i eno1 -F 192.168.10.1,192.168.10.199 --enable-tftp --tftp-root=/nfs/pi1/boot --pxe-service=0,"Raspberry Pi Boot" 

Plug in the power on your Pi and you should see the DHCP responses and files being served to your Pi. On the latest version of raspbian I couldn't SSH in on the first boot, but only on the second. I waited at least 3 minutes on the first boot, then just power cycled it, it took about 2 minutes to boot up but then I could ssh in. 

You can then SSH into your Pi once it's booted up by doing an SSH into the IP address shown by dnsmasq:

    $ ssh 192.168.10.37 -p22 -lpi 
    Password: raspberry
  
    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.

    pi@raspberrypi:~ $ 

To write my SD card image to an actual SD card, I had to, on the server:

    $ sudo cp $HOME/Downloads/pi.img /nfs/pi1/mnt

To make the file visible to the Pi at /mnt, and then on the Pi:

    $ sudo su
    # dd if=/mnt/pi.img of=/dev/mmcblk0 bs=4k &
    # while sudo kill -SIGUSR1 %1; do sleep 1; done # to monitor progress
    
See also

    $ sudo raspi-config
    
### Thanks to / References

This document is the result of about an hour or two of "hacking", and probably around 20 reboots and some Googling to get it working. Turned out to be nothing major, just a few typos. So one or two of the above steps might not even be necessary... I have attempted to contribute back fixes. 
* https://www.raspberrypi.org/documentation/hardware/raspberrypi/bootmodes/net_tutorial.md
* http://nfs.sourceforge.net/nfs-howto/ar01s07.html
* https://hackernoon.com/raspberry-pi-headless-install-462ccabd75d0?gi=768e5160971a
* https://raspberrypi.stackexchange.com/questions/48350/nfsroot-boot-fails-nfs-server-reports-the-request
    
**PS** Don't set up server intensive tasks on your Pi to run from your SD Card, rather use NFS like here, or attach a removable hard drive or SSD - according to reports on various forums, SD Cards last from days to weeks, to maybe a year if you are lucky, if you write to them often.



    
                          
    
  

    
