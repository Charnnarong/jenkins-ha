    [vagrant@centos-active ~]$ sudo systemctl start pcsd.service
    [vagrant@centos-active ~]$ sudo systemctl enable pcsd.service


    [vagrant@centos-passive ~]$ sudo systemctl start pcsd.service
    [vagrant@centos-passive ~]$ sudo systemctl enable pcsd.service


    [vagrant@centos-active ~]$ sudo pcs cluster auth centos-active centos-passive
    Username: hacluster
    Password: password
    centos-active: Authorized
    centos-passive: Authorized


![1](http://i.imgur.com/UjYODRK.png)


<hr/>

    [vagrant@centos-active ~]$ sudo pcs cluster setup --start --name my_cluster centos-active centos-passive
    [vagrant@centos-active ~]$ sudo pcs cluster enable --all

![2](http://i.imgur.com/bjg5DZL.png)

<hr/>

To show cluster status

    [vagrant@centos-active ~]$ sudo pcs cluster status

For simplify this tutorial, we opt to disable "Shoot The Other Node In The Head" (STONITH)

    [vagrant@centos-active ~]$ sudo pcs property set stonith-enabled=false
    

<hr/>

Over view of Active/Passive Apache HTTP Server

![3](http://i.imgur.com/bavKGgG.png)

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

![4](http://i.imgur.com/RVkNPGZ.png)


Find out how much space we can use to create our brand new logical volume from existing volume group.

    [vagrant@centos-active ~]$ sudo vgdisplay | grep Free
    
You are probably see something like this

    Free  PE / Size       10 / 320.00 MiB    

<hr/>

We are going to need storage attatched to it as follow

![5](http://i.imgur.com/jc5WmCp.png)

Instead of attatch /dev/sdb physically, we will do that through iscsi protocal.In short make physical disk appear to server as if it is a physical disk, but it actually located at somewhere else, here called Centos-Storage server that we are going to do next and pause what we have done for now, then we'll be back to this point once we finish with our iscsi service. 

That is effectively means our Active node and Passive node are pointing at the same disk remotely. But only one is taking the disk in control at a time and will never be simultaneously accesses. 

![6](http://i.imgur.com/cmnEyWp.png)

<hr/> 

Create iscsi (Pronounced **eye skuzzy**.)

iSCSI target provides some storage (server),

iSCSI initiator uses this available storage (client).

Overview of iscsi architecture.

![7](http://i.imgur.com/5IgG1pi.jpg)

credit : http://www.tsmtutorials.com/2016/08/iscsi-architecture.html


    [vagrant@centos-storage ~]$ sudo yum install -y targetcli
    [vagrant@centos-storage ~]$ sudo systemctl enable target


    [vagrant@centos-storage ~]$ sudo targetcli
        Warning: Could not load preferences file /root/.targetcli/prefs.bin.
        targetcli shell version 2.1.fb41
        Copyright 2011-2013 by Datera, Inc and others.
        For help on commands, type 'help'.
        
        />

    /> backstores/fileio/ create shareddata /opt/shareddata.img 512M
        Created fileio shareddata with size 268435456
        
create an IQN (Iscsi Qualified Name) called iqn.2016-12.com.centos-storage

    /> iscsi/ create iqn.2016-12.com.centos-storage:t1
        Created target iqn.2016-12.com.centos-storage:t1.
        Created TPG 1.
        Global pref auto_add_default_portal=true
        Created default portal listening on all IPs (0.0.0.0), port 3260.

    /> cd iscsi/iqn.2016-12.com.centos-storage:t1

    /iscsi/iqn.20...os-storage:t1> ls
        o- iqn.2016-12.com.centos-storage:t1 ..................................................................................... [TPGs: 1]
          o- tpg1 ................................................................................................... [no-gen-acls, no-auth]
            o- acls .............................................................................................................. [ACLs: 0]
            o- luns .............................................................................................................. [LUNs: 0]
            o- portals ........................................................................................................ [Portals: 1]
              o- 0.0.0.0:3260 ......................................................................................................... [OK]

    /iscsi/iqn.20...os-storage:t1> cd tpg1/   

    /iscsi/iqn.20...orage:t1/tpg1> portals/ create
        Using default IP port 3260
        Binding to INADDR_ANY (0.0.0.0)
        This NetworkPortal already exists in configFS          
              

    /iscsi/iqn.20...orage:t1/tpg1> luns/ create /backstores/fileio/shareddata
        Created LUN 0.

    /iscsi/iqn.20...orage:t1/tpg1> acls/ create iqn.2016-12.com.centos-storage:client
        Created Node ACL for iqn.2016-12.com.centos-storage:client
        Created mapped LUN 0.        

    /iscsi/iqn.20...orage:t1/tpg1> cd acls/iqn.2016-12.com.centos-storage:client/
    

    /iscsi/iqn.20...torage:client> set auth userid=usr
        Parameter userid is now 'usr'.

    /iscsi/iqn.20...torage:client> set auth password=password1
        Parameter password is now 'password1'.

    /iscsi/iqn.20...torage:client>

    /iscsi/iqn.20...torage:client> cd ../..

    /iscsi/iqn.20...orage:t1/tpg1> ls
        o- tpg1 ..................................................................................................... [no-gen-acls, no-auth]
          o- acls ................................................................................................................ [ACLs: 1]
          | o- iqn.2016-12.com.centos-storage:client ...................................................................... [Mapped LUNs: 1]
          |   o- mapped_lun0 ................................................................................. [lun0 fileio/shareddata (rw)]
          o- luns ................................................................................................................ [LUNs: 1]
          | o- lun0 .............................................................................. [fileio/shareddata (/opt/shareddata.img)]
          o- portals .......................................................................................................... [Portals: 1]
            o- 0.0.0.0:3260 ........................................................................................................... [OK]
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


    /iscsi/iqn.20...orage:t1/tpg1> exit
        Global pref auto_save_on_exit=true
        Last 10 configs saved in /etc/target/backup.
        Configuration saved to /etc/target/saveconfig.json

<hr/>

iSCSI Initiator Configuration

    [vagrant@centos-active ~]$ sudo yum install -y iscsi-initiator-utils

    [vagrant@centos-active ~]$ cd /etc/iscsi

    [vagrant@centos-active iscsi]$ sudo vi initiatorname.iscsi
        InitiatorName=iqn.2016-12.com.centos-storage:client

    [vagrant@centos-active iscsi]$ sudo vi iscsid.conf
        node.session.auth.authmethod = CHAP
        node.session.auth.username = usr
        node.session.auth.password = password1

    [vagrant@centos-active iscsi]$ sudo systemctl start iscsi

    [vagrant@centos-active iscsi]$ sudo iscsiadm --mode discovery --type sendtargets --portal 10.100.199.10
        10.100.199.10:3260,1 iqn.2016-12.com.centos-storage:t1

    [vagrant@centos-active iscsi]$ sudo iscsiadm --mode node --targetname iqn.2016-12.com.centos-storage:t1 --portal 10.100.199.10 --login
        Logging in to [iface: default, target: iqn.2016-12.com.centos-storage:t1, portal: 10.100.199.10,3260] (multiple)
        Login to [iface: default, target: iqn.2016-12.com.centos-storage:t1, portal: 10.100.199.10,3260] successful.

    [root@centos-active html]# lsblk --scsi

    [root@centos-active html]# sudo iscsiadm -m session -P 3

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
    
Rebuild the initramfs boot image to guarantee that the boot image will not try to activate a
volume group controlled by the cluster.

    [root@centos-active vagrant]# dracut -H -f /boot/initramfs-$(uname -r).img $(uname -r)
    [root@centos-passive vagrant]# dracut -H -f /boot/initramfs-$(uname -r).img $(uname -r)
    
Reboots both nodes


**pcs** - pacemaker/corosync configuration system.

It uses to control and configure pacemaker and corosync.
Architecture: http://clusterlabs.org/doc/en-US/Pacemaker/1.1/html/Clusters_from_Scratch/_pacemaker_architecture.html


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
    

**Completed cluster part**
<hr/>

<br/>

###Setup Jenkins
     
      