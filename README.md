# Raspberry Pi 3 B+ LAN / pxe boot with NFS root (if you don't have an SD card or reader)

I didn't have a way to write an SD card, so I booted up my Pi 3 B+ over LAN and wrote the SD card from the Pi itself. I used my laptop as the server, and connected the Pi to the LAN port on my laptop, while my laptop was on Wi-Fi.

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
    # mkdir /nfs/client1
    # kpartx -av pi.img   # or 2018-11-13-raspbian-stretch-lite.img
    add map loop30p1 (253:0): 0 89854 linear 7:30 8192
    add map loop30p2 (253:1): 0 3547136 linear 7:30 98304

    # mknod /dev/loop30p1 b 253 0 # might not be needed
    # mknod /dev/loop30p2 b 253 1 # might not be needed
    # mkdir /mnt/pi
    # mount /dev/loopXXp2 /mnt/pi
    # mount /dev/loopXXp1 /mnt/pi/boot
    # rsync -xav /mnt/pi /nfs/client1
    # cd /nfs/client1
    
You are now in the pi NFS filesystem, get it ready to boot:

    # vi boot/cmdline.txt
    dwc_otg.lpm_enable=0 console=serial0,115200 console=tty1 root=/dev/nfs nfsroot=192.168.10.1:/nfs/client1,tcp,v3 rootfstype=nfs rw ip=dhcp rootwait elevator=deadline 

Remove everything except the first line that starts with /proc in fstab, add /sys:

    # vi etc/fstab
    proc            /proc           proc    defaults          0       0
    sysfs 			/sys 			sysfs	defaults 		  0 	  0
    
Make it start ssh on boot:

    # touch boot/ssh
    
The default login and password is pi and raspberry    
    
If you want to add your ssh key: (optional)

    # mkdir home/pi/.ssh
    # cat /home/XXXX/.ssh/id_rsa.pub >> home/pi/.ssh/authorized_keys
    # chmod -R 600 home/pi/.ssh
   
To give access to everyone who has access to the server: (optional)

    # cat /home/XXXX/.ssh/authorized_keys >> home/pi/.ssh/authorized_keys

Change the default user to your default user on the server: (optional)
    
    # U=newuser 
    # sed s@:pi@:$Ug -i etc/passwd
    # sed s@:pi@:$Ug -i etc/shadow
    # sed s@:pi@:$Ug -i etc/groups
    # mv home/pi home/$U
    
If you're on a Linux that uses NetworkManager and systemd-resolved (eg. Debian or Ubuntu) and you want full control over your ethenet socket, so this before: 

    # vi /etc/udev/rules.d/00-net.rules 
    
    # Interfaces that shouldn't be managed by NetworkManager 
    ACTION=="add", SUBSYSTEM=="net", KERNEL=="eno1", ENV{NM_UNMANAGED}="1"
    ACTION=="add", SUBSYSTEM=="net", KERNEL=="eth1", ENV{NM_UNMANAGED}="1"
    
    :w
    :q
    
    # vi /etc/NetworkManager/NetworkManager.conf
    [main]
    plugins=ifupdown,keyfile
    dns=default  # add this

    [ifupdown]
    managed=false

    [device]
    wifi.scan-rand-mac-address=no
 
    :w
    :q 
    
    # systemctl restart network
    # systemctl restart NetworkManager
    # systemctl disable systemd-resolved.service
    # systemctl stop systemd-resolved.service
    # rm /etc/resolv.conf  # this is a symlink to a dynamic file, will automatically be recreated 
    
Configure an IP addres on your network interface: 

    # ifconfig eno1 192.168.10.1 up

If you want to give your Pi internet access through the server:

    # echo 1 > /proc/sys/net/ipv4/ip_forward
    # iptables -P FORWARD ACCEPT
    # iptables -t nat -A POSTROUTING -j MASQUERADE

Enable NFS on your server: (and test it)
   
    # apt-get install nfs-kernel-server
    # exportfs -ra
    # systemctl restart nfs-kernel-service

   
Now serve DHCP on your lan port and get ready to boot up your Pi:

    # dnsmasq -d -i eno1 -F 192.168.10.1,192.168.10.199 --enable-tftp --tftp-root=/nfs/client1/boot --pxe-service=0,"Raspberry Pi Boot" 

Plug in the power on your Pi and you should see the DHCP responses and files being served to your Pi. 

You can then SSH into your Pi once it's booted up by doing an SSH into the IP address shown by dnsmasq:

    $ ssh 192.168.10.37 -p22 -lpi 
    Password: raspberry
  
    The programs included with the Debian GNU/Linux system are free software;
    the exact distribution terms for each program are described in the
    individual files in /usr/share/doc/*/copyright.

    Debian GNU/Linux comes with ABSOLUTELY NO WARRANTY, to the extent
    permitted by applicable law.

    pi@raspberrypi:~ $ 

    
                          
    
  

    