    [vagrant@centos-active ~]$ sudo systemctl start pcsd.service
    [vagrant@centos-active ~]$ sudo systemctl enable pcsd.service


    [vagrant@centos-passive ~]$ sudo systemctl start pcsd.service
    [vagrant@centos-passive ~]$ sudo systemctl enable pcsd.service


    [vagrant@centos-active ~]$ sudo pcs cluster auth centos-active centos-passive
    Username: hacluster
    Password: password
    centos-active: Authorized
    centos-passive: Authorized


[ Slide 1 ] 

<hr/>

    [vagrant@centos-active ~]$ sudo pcs cluster setup --start --name my_cluster centos-active centos-passive
    [vagrant@centos-active ~]$ sudo pcs cluster enable --all

[ Slide 2 ]

<hr/>

To show cluster status

    [vagrant@centos-active ~]$ sudo pcs cluster status

For simplify this tutorial, we opt to disable "Shoot The Other Node In The Head" (STONITH)

    [vagrant@centos-active ~]$ sudo pcs property set stonith-enabled=false
    

<hr/>

Over view of Active/Passive Apache HTTP Server

[ Slide 3 ]

<hr/>

List Block Devices 

    [vagrant@centos-active ~]$ lsblk
    
Show physical volume 

    [vagrant@centos-active ~]$ sudo pvdisplay
    
Show volume group

    [vagrant@centos-active ~]$ sudo vgdisplay
    
Show logical volume

    [vagrant@centos-active ~]$ sudo lvdisplay
    
All PV , VG and LV together are visualized as

[ Slide 4 ] 


Find out how much space we can use to create our brand new logical volume from existing volume group.

    [vagrant@centos-active ~]$ sudo vgdisplay | grep Free
    
You are probably see something like this

    Free  PE / Size       10 / 320.00 MiB    

<hr/>

We are going to need storage attatched to it as follow

[ Slide 5 ]

Create iscsi (Pronounced **eye skuzzy**.)


Crate new physical volume for Cluster as if we are gonna use it later on.

    [vagrant@centos-active ~]$ sudo lsblk
    NAME                         MAJ:MIN RM  SIZE RO TYPE MOUNTPOINT
    sda                            8:0    0   40G  0 disk
    |-sda1                         8:1    0    1M  0 part
    |-sda2                         8:2    0  500M  0 part /boot
    |-sda3                         8:3    0 39.5G  0 part
      |-VolGroup00-LogVol00      253:0    0 37.7G  0 lvm  /
      |-VolGroup00-LogVol01      253:1    0  1.5G  0 lvm  [SWAP]
      |-VolGroup00-my_cluster_lv 253:2    0  256M  0 lvm
    sdb                            8:16   0  512M  0 disk

    [vagrant@centos-active ~]$ sudo pvcreate /dev/sdb
        Physical volume "/dev/sdb" successfully created

Creat Physical disk

    [vagrant@centos-active ~]$ sudo vgcreate shared_vg /dev/sdb
        Volume group "shared_vg" successfully created
    
Create new logical volume for cluster as if we are gonna use it later on.

    [vagrant@centos-active ~]$ sudo lvcreate -L350M -n shared_lv shared_vg
        Rounding up size to full physical extent 352.00 MiB
        Logical volume "shared_lv" created.
    
Report information about logical volumes 

    [vagrant@centos-active ~]$ sudo lvs
      LV            VG         Attr       LSize   Pool Origin Data%  Meta%  Move Log Cpy%Sync Convert
      LogVol00      VolGroup00 -wi-ao----  37.69g
      LogVol01      VolGroup00 -wi-ao----   1.50g
      my_cluster_lv VolGroup00 -wi-a----- 256.00m
      shared_lv     shared_vg  -wi-a----- 352.00m

Create ext4 filesystem.

    [vagrant@centos-active ~]$ sudo mkfs.ext4 /dev/shared_vg/shared_lv
        
    
Create an ext4 file system on the logical volume my\_cluster\_lv

    [vagrant@centos-active ~]$ sudo mkfs.ext4 /dev/VolGroup00/my_cluster_lv
    
Your output would be similar to the sample below

    mke2fs 1.42.9 (28-Dec-2013)
    Filesystem label=
    OS type: Linux
    Block size=1024 (log=0)
    Fragment size=1024 (log=0)
    Stride=0 blocks, Stripe width=8192 blocks
    90112 inodes, 360448 blocks
    18022 blocks (5.00%) reserved for the super user
    First data block=1
    Maximum filesystem blocks=33947648
    44 block groups
    8192 blocks per group, 8192 fragments per group
    2048 inodes per group
    Superblock backups stored on blocks:
            8193, 24577, 40961, 57345, 73729, 204801, 221185
    
    Allocating group tables: done
    Writing inode tables: done
    Creating journal (8192 blocks): done
    Writing superblocks and filesystem accounting information: done

<hr/>

Install httpd

    [vagrant@centos-active ~]$ sudo yum -y install httpd wget
    [vagrant@centos-passive ~]$ sudo yum -y install httpd wget
    
Set server status.

    [vagrant@centos-active ~]$ sudo su
    
    [vagrant@centos-active ~]$ cat << EOF >  /etc/httpd/conf.d/status.conf
    <Location /server-status>
    SetHandler server-status
    Order deny,allow
    Deny from all
    Allow from 127.0.0.1
    </Location>
    EOF
    
    
    [vagrant@centos-passive ~]$ sudo su
    
    [vagrant@centos-passive ~]$ cat << EOF >  /etc/httpd/conf.d/status.conf
    <Location /server-status>
    SetHandler server-status
    Order deny,allow
    Deny from all
    Allow from 127.0.0.1
    </Location>
    EOF
    
<hr/>


Create a webpage.

    [vagrant@centos-active ~]$ sudo mount /dev/shared_vg/shared_lv /var/www/

    [vagrant@centos-active ~]$ sudo mkdir /var/www/html
    [vagrant@centos-active ~]$ sudo mkdir /var/www/cgi-bin
    [vagrant@centos-active ~]$ sudo mkdir /var/www/error
    

    [vagrant@centos-active ~]$ sudo restorecon -R /var/www

    [vagrant@centos-active ~]$ sudo su
    [root@centos-active vagrant]# cat <<-END >/var/www/html/index.html
    <html>
    <body>Welcome from iscsi</body>
    </html>
    END
    
Umount share directory.

    [root@centos-active vagrant]# umount /var/www
    

    [root@centos-active vagrant]# vi /etc/logrotate.d/httpd

Then replace the line "bin/systemctl ..." to

    /var/log/httpd/*log {
    missingok
    notifempty
    sharedscripts
    delaycompress
    postrotate
        /usr/sbin/httpd -f /etc/httpd/conf/httpd.conf -c "PidFile /var/run/httpd.pid" -k graceful > /dev/null 2>/dev/null || true
    endscript
    }    
    
Exclusive Activation of a Volume Group in a Cluster

All volume groups managed by the cluster manager must be excluded from the volume_list entry

    [root@centos-active vagrant]# lvmconf --enable-halvm --services --startstopservices
        Warning: Stopping lvm2-lvmetad.service, but it can still be activated by:
          lvm2-lvmetad.socket
        Removed symlink /etc/systemd/system/sysinit.target.wants/lvm2-lvmetad.socket.
    
    [root@centos-passive vagrant]# lvmconf --enable-halvm --services --startstopservices
        Warning: Stopping lvm2-lvmetad.service, but it can still be activated by:
          lvm2-lvmetad.socket
        Removed symlink /etc/systemd/system/sysinit.target.wants/lvm2-lvmetad.socket.

Modified volume\_list to volume\_list = [ "VolGroup00" ]
        
    [root@centos-active vagrant]# vgs --noheadings -o vg_name
    [root@centos-active vagrant]#  vi /etc/lvm/lvm.conf


    volume_list = [ "VolGroup00" ]

Modified volume\_list to volume\_list = [ "VolGroup00" ]
        
    [root@centos-passive vagrant]# vgs --noheadings -o vg_name
    [root@centos-passive vagrant]#  vi /etc/lvm/lvm.conf


    volume_list = [ "VolGroup00" ]
    
Rebuild the i ni tramfs boot image to guarantee that the boot image will not try to activate a
volume group controlled by the cluster.

    [root@centos-active vagrant]# dracut -H -f /boot/initramfs-$(uname -r).img $(uname -r)
    [root@centos-passive vagrant]# dracut -H -f /boot/initramfs-$(uname -r).img $(uname -r)
    
Reboots both nodes

    [vagrant@centos-active ~]$ sudo pcs cluster start
    [vagrant@centos-active ~]$ sudo pcs cluster start --all

    [root@centos-active vagrant]# pcs resource show
        NO resources configured

    [root@centos-active vagrant]# pcs resource create shared_lvm LVM volgrpname=shared_vg exclusive=true --group apachegroup
    

    [root@centos-active vagrant]# pcs resource show
     Resource Group: apachegroup
         shared_lvm (ocf::heartbeat:LVM):   Started centos-active   

    [root@centos-active vagrant]# pcs resource create shared_fs Filesystem device="/dev/shared_vg/shared_lv" directory="/var/www" fstype="ext4" --group apachegroup
   
 
    [root@centos-active vagrant]# pcs resource create Website apache configfile="/etc/httpd/conf/httpd.conf" statusurl="http://127.0.0.1/server-status" --group apachegroup
    
     [root@centos-active vagrant]# pcs resource create VirtualIP IPaddr2 ip=10.100.199.25 cidr_netmask=24 --group apachegroup
     

     [root@centos-active vagrant]# pcs status
    Cluster name: my_cluster
    Last updated: Wed Dec  7 14:48:19 2016          Last change: Wed Dec  7 14:48:10 2016 by root via cibadmin on centos-active
    Stack: corosync
    Current DC: centos-active (version 1.1.13-10.el7_2.4-44eb2dd) - partition with quorum
    2 nodes and 4 resources configured
    
    Online: [ centos-active centos-passive ]
    
    Full list of resources:
    
     Resource Group: apachegroup
         shared_lvm (ocf::heartbeat:LVM):   Started centos-active
         shared_fs  (ocf::heartbeat:Filesystem):    Started centos-active
         Website    (ocf::heartbeat:apache):        Started centos-active
         VirtualIP  (ocf::heartbeat:IPaddr2):       Started centos-active
    
    PCSD Status:
      centos-active: Online
      centos-passive: Online
    
    Daemon Status:
      corosync: active/enabled
      pacemaker: active/enabled
      pcsd: active/enabled
      

Test your cluster web on 10.100.199.25
If that's worked then test cluster by issue command

    [root@centos-active vagrant]# pcs cluster standby centos-active

and test your cluster again.
    
    [root@centos-active vagrant]# pcs status
    [root@centos-active vagrant]# pcs cluster unstandby centos-active
    
        

     
      