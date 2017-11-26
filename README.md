# ceph_on_PI_for_fun
## CEPH Hammer on Raspberry Pi B2

###### Create the SanDisk 32G microSD cards - load raspbian images on them using MAC
```
# diskutil list
/dev/disk3
   #:                       TYPE NAME                    SIZE       IDENTIFIER
   0:     FDisk_partition_scheme                        *31.9 GB    disk3
   1:             Windows_FAT_32 boot                    58.7 MB    disk3s1
   2:                      Linux                         3.2 GB     disk3s2

# diskutil unmountdisk /dev/disk3
Unmount of all volumes on disk3 was successful

# sudo dd bs=60m if=2015-05-05-raspbian-wheezy.img of=/dev/rdisk3
Password:
52+1 records in
52+1 records out
3276800000 bytes transferred in 68.308369 secs (47970696 bytes/sec)
```
The Wheezy Image of Raspbian can be downloaded from - https://downloads.raspberrypi.org/raspbian/images/raspbian-2015-05-07/2015-05-05-raspbian-wheezy.zip.

###Pre-setup of the Raspberry PIs once bootable image is ready:
We need at least 3 Raspberry Pi computers to run CEPH cluster (Monitor + OSD). I used 32G SanDisk microSD card for the Operating system and a 12GB USB stick for the OSD drives.

Once the microSD card is ready with the bootable raspbian image - boot the Raspberry PIs and perform following options using the “raspi-config” utility:

 * Change Default password for “pi” user.
 * Change hostname (I used cephnode1, cephnode2, cephnode3 as hostnames). 
 * Overclock to 1000Mhz for PI2
 * Enable SSH from the advanced options


Next we will disable DHCP and use Static IPs on the wired Ethernet port:
Modify /etc/network/interfaces:

Original file:
```
auto lo
iface lo inet loopback

auto eth0
allow-hotplug eth0
iface eth0 inet manual

auto wlan0
allow-hotplug wlan0
iface wlan0 inet manual
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

auto wlan1
allow-hotplug wlan1
iface wlan1 inet manual
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```
Modified file for Static IP (make sure the static IPs are not part of your Home router DHCP pool):
```
auto lo
iface lo inet loopback

auto eth0
iface eth0 inet static
address 192.168.0.10 # 10, 11, 12 -->3 ceph PIs
netmask 255.255.255.0
network 192.168.0.0
broadcast 192.168.0.255
gateway 192.168.0.1
dns-nameservers 192.168.0.1
mtu 2000 #<--home switch can only support this much 

auto wlan0
allow-hotplug wlan0
iface wlan0 inet manual
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf

auto wlan1
allow-hotplug wlan1
iface wlan1 inet manual
wpa-conf /etc/wpa_supplicant/wpa_supplicant.conf
```
Uninstall DHCP packages:
```
# aptitude -y purge dhcp3-client dhcp3-common isc-dhcp-client isc-dhcp-common
The following packages will be REMOVED:  
  isc-dhcp-client{p} isc-dhcp-common{p} 
0 packages upgraded, 0 newly installed, 2 to remove and 0 not upgraded.
Need to get 0 B of archives. After unpacking 3,216 kB will be freed.
(Reading database ... 77851 files and directories currently installed.)
Removing isc-dhcp-client ...
Purging configuration files for isc-dhcp-client ...
Removing isc-dhcp-common ...
Processing triggers for man-db ...
```
Move the dhcpd init file out of /etc/init.d:
```
# mv /etc/init.d/dhcpcd /root
# reboot
```

###### Kernel Tuning:
```
vm.swappiness=1
vm.min_free_kbytes = 32768
kernel.pid_max = 32768
```
###### Install some software utilities:
```
# aptitude -y install screen htop iotop btrfs-tools lsb-release gdisk
```
###### Add CEPH and Raspbian testing Repos and install binaries:
Add CEPH repo KEY:
```
# curl -O "https://git.ceph.com/?p=ceph.git;a=blob_plain;f=keys/release.asc"
  % Total    % Received % Xferd  Average Speed   Time    Time     Time  Current
                                 Dload  Upload   Total   Spent    Left  Speed
100  1645  100  1645    0     0    463      0  0:00:03  0:00:03 --:--:--   472
# apt-key add release.asc 
OK
```
Add CEPH Repo:
```
# echo deb http://ceph.com/debian-hammer/ $(lsb_release -sc) main | sudo tee /etc/apt/sources.list.d/ceph.list
```
Change Raspbian repo to Testing from wheezy:
```
/etc/apt/sources.list

deb http://mirrordirector.raspbian.org/raspbian/ testing main contrib non-free rpi
# deb http://mirrordirector.raspbian.org/raspbian/ wheezy main contrib non-free rpi
# Uncomment line below then 'apt-get update' to enable 'apt-get source'
#deb-src http://archive.raspbian.org/raspbian/ wheezy main contrib non-free rpi
```
###### Update repos
```
# apt-get update
```
###### Install CEPH packages:
```
# apt-get -y install ceph
```
###### Check Packages:
```
# dpkg --list |grep ceph
ii  ceph                                  0.94.3-1                                armhf        distributed storage and file system
ii  ceph-common                           0.94.3-1                                armhf        common utilities to mount and interact with a ceph storage cluster
ii  libcephfs1                            0.94.3-1                                armhf        Ceph distributed file system client library
ii  python-cephfs                         0.94.3-1                                armhf        Python libraries for the Ceph libcephfs library

# ceph -v
ceph version 0.94.3 (95cefea9fd9ab740263bf8bb4796fd864d9afe2b)

0.94.3 is the latest Hammer release for CEPH
```
###### Create “ceph.conf” configuration file:
```
[global]
fsid = 8e784fd8-a016-411d-bbe8-e2337222e935
public network = 192.168.0.0/24
cluster network = 192.168.0.0/24
auth cluster required = cephx
auth service required = cephx
auth client required = cephx
# Primary purpose of this CEPH cluster is Block storage - RBD...following parameters are tuned for that
osd journal size = 10240
filestore xattr use omap = true
osd pool default size = 3
osd pool default min size = 2
osd crush chooseleaf type = 1
[mon.1]
host = cephnode1
mon addr = 192.168.0.201:6789
mon data = /var/lib/ceph/mon/ceph-cephnode1
[mon.2]
host = cephnode2
mon addr = 192.168.0.202:6789
mon data = /var/lib/ceph/mon/ceph-cephnode2
[mon.3]
host = cephnode3
mon addr = 192.168.0.203:6789
mon data = /var/lib/ceph/mon/ceph-cephnode3

[mon]
  mon compact on start = true
  mon osd down out subtree_limit = host

[osd.0]
    host = cephnode1
    devs = /dev/sda1
    osd addr = 193.168.0.201:6800
    osd data = /var/lib/ceph/osd/ceph-0
[osd.1]
    host = cephnode2
    devs = /dev/sda1
    osd addr = 192.168.0.202:6800
    osd data = /var/lib/ceph/osd/ceph-1
[osd.2]
    host = cephnode3
    devs = /dev/sda1
    osd addr = 192.168.0.203:6800
    osd data = /var/lib/ceph/osd/ceph-2

[osd]
  # Filesystem Optimizations
  osd mkfs type = btrfs

  # Performance tuning
  max open files = 327680
  osd op threads = 12
  filestore op threads = 20
 
  # Recovery tuning
  osd recovery max active = 2
  osd recovery max single start = 2
  osd max backfills = 2
  osd recovery op priority = 2

  # Optimize Filestore Merge and Split
  filestore merge threshold = 40
  filestore split multiple = 8
```


Copy the /etc/ceph/ceph.conf file to all 3 nodes. We are going to run monitor and OSD on all 3 Raspberry Pi nodes.

__Next few steps are required on one node only.__
Create cluster keyring and monitor secret key:
```
# ceph-authtool --create-keyring /tmp/ceph.mon.keyring --gen-key -n mon. --cap mon 'allow *'
creating /tmp/ceph.mon.keyring
```
Generate administrator keyring, create client.admin user and add the user to admin keyring:
```
# ceph-authtool --create-keyring /etc/ceph/ceph.client.admin.keyring --gen-key -n client.admin --set-uid=0 --cap mon 'allow *' --cap osd 'allow *' --cap mds 'allow'
creating /etc/ceph/ceph.client.admin.keyring
```
Add the client.admin key to the ceph.mon.keyring
```
# ceph-authtool /tmp/ceph.mon.keyring --import-keyring /etc/ceph/ceph.client.admin.keyring

importing contents of /etc/ceph/ceph.client.admin.keyring into /tmp/ceph.mon.keyring
```

Generate a monitor map using the hostname(s), host IP address(es) and the FSID. Save it as /tmp/monmap:
```
# monmaptool --create --add cephnode1 192.168.0.201 --fsid 8e784fd8-a016-411d-bbe8-e2337222e935 /tmp/monmap
monmaptool: monmap file /tmp/monmap
monmaptool: set fsid to 8e784fd8-a016-411d-bbe8-e2337222e935
monmaptool: writing epoch 0 to /tmp/monmap (1 monitors)
```
Add the remaining two monitors to the map as well:
```
# monmaptool --add cephnode2 192.168.0.202 --fsid 8e784fd8-a016-411d-bbe8-e2337222e935 /tmp/monmap
monmaptool: monmap file /tmp/monmap
monmaptool: set fsid to 8e784fd8-a016-411d-bbe8-e2337222e935
monmaptool: writing epoch 0 to /tmp/monmap (2 monitors)

# monmaptool --add cephnode3 192.168.0.203 --fsid 8e784fd8-a016-411d-bbe8-e2337222e935 /tmp/monmap
monmaptool: monmap file /tmp/monmap
monmaptool: set fsid to 8e784fd8-a016-411d-bbe8-e2337222e935
monmaptool: writing epoch 0 to /tmp/monmap (3 monitors)
```
###### Create a default data directory on all the monitor hosts:
```
# mkdir /var/lib/ceph/mon/ceph-cephnode(1,2,3)
```
###### Populate the monitor daemon(s) with the monitor map and keyring (copy the keyring files to all 3 monitor nodes before running command on the node):

Generate the correct monmap one more time using the /etc/ceph/ceph.conf file:
```
# monmaptool  --create --generate -c /etc/ceph/ceph.conf /tmp/monmap.right

# ceph-mon --mkfs -i 3 --monmap /tmp/monmap.right --keyring /tmp/ceph.mon.keyring
ceph-mon: set fsid to 8e784fd8-a016-411d-bbe8-e2337222e935
ceph-mon: created monfs at /var/lib/ceph/mon/ceph-cephnode3 for mon.3

# ceph-mon --mkfs -i 2 --monmap /tmp/monmap.right --keyring /tmp/ceph.mon.keyring
ceph-mon: set fsid to 8e784fd8-a016-411d-bbe8-e2337222e935
ceph-mon: created monfs at /var/lib/ceph/mon/ceph-cephnode2 for mon.2

# ceph-mon --mkfs -i 1 --monmap /tmp/monmap.right --keyring /tmp/ceph.mon.keyring
ceph-mon: set fsid to 8e784fd8-a016-411d-bbe8-e2337222e935
ceph-mon: created monfs at /var/lib/ceph/mon/ceph-cephnode1 for mon.1
```

This will create a keyring file and a folder called “store.db” in /var/lib/ceph/mon/ceph-{nodename} folder.

###### Mark that the monitor is created and ready to be started (on all three hosts):
```
# touch /var/lib/ceph/mon/ceph-cephnode<1,2,3>/done
```
###### Start the monitor processes on all 3 nodes:
```
# /etc/init.d/ceph start mon.[1,2,3]

# /etc/init.d/ceph start mon.1
# /etc/init.d/ceph start mon.2
# /etc/init.d/ceph start mon.3
```
###### Run Gather keys on one node to check (not required as we already created admin keys):
```
# /usr/sbin/ceph-create-keys --cluster ceph -i 1
INFO:ceph-create-keys:Key exists already: /etc/ceph/ceph.client.admin.keyring
INFO:ceph-create-keys:Talking to monitor...
INFO:ceph-create-keys:Talking to monitor...
INFO:ceph-create-keys:Talking to monitor...
```
###### Other commands to check monitor:
```
# ceph auth list
installed auth entries:

client.admin
    key: xxxxxxxxxx/xxxxxxxxxp9Q/xxxxxxxxxSFzCEtw==
    auid: 0
    caps: [mds] allow
    caps: [mon] allow *
    caps: [osd] allow *
client.bootstrap-mds
    key: xxxxxxxxxxxxxxxxxxMZFhkxF/Sg==
    caps: [mon] allow profile bootstrap-mds
client.bootstrap-osd
    key: xxxxxxxxxxxnF01B+IJhh4c/rp/g==
    caps: [mon] allow profile bootstrap-osd
client.bootstrap-rgw
    key: xxxxxxxxxxxxxxnHbtngziuKelag==
    caps: [mon] allow profile bootstrap-rgw

# ceph mon dump
dumped monmap epoch 1
epoch 1
fsid 8e784fd8-a016-411d-bbe8-e2337222e935
last_changed 2015-10-09 02:50:20.086273
created 2015-10-09 02:50:20.086273
0: 192.168.0.201:6789/0 mon.1
1: 192.168.0.202:6789/0 mon.2
2: 192.168.0.203:6789/0 mon.3

# ceph -s
    cluster 8e784fd8-a016-411d-bbe8-e2337222e935
     health HEALTH_ERR
            64 pgs stuck inactive
            64 pgs stuck unclean
            no osds
     monmap e1: 3 mons at {1=192.168.0.201:6789/0,2=192.168.0.202:6789/0,3=192.168.0.203:6789/0}
            election epoch 6, quorum 0,1,2 1,2,3
     osdmap e1: 0 osds: 0 up, 0 in
      pgmap v2: 64 pgs, 1 pools, 0 bytes data, 0 objects
            0 kB used, 0 kB / 0 kB avail
                  64 creating
                  
```
###### Now we will create the 3 OSDs using the 128GB USB stick in each Raspberry Pi:
```
# ceph-disk-prepare --zap-disk /dev/sda

***************************************************************
Found invalid GPT and valid MBR; converting MBR to GPT format.
***************************************************************

Warning! Main partition table overlaps the first partition by 2 blocks!
Try reducing the partition table size by 8 entries.
(Use the 's' item on the experts' menu.)

Warning! Secondary partition table overlaps the last partition by
33 blocks!
You will need to delete this partition or resize it in another utility.
GPT data structures destroyed! You may now partition the disk using fdisk or
other utilities.
Creating new GPT entries.
The operation has completed successfully.
Information: Moved requested sector from 34 to 2048 in
order to align on 2048-sector boundaries.
The operation has completed successfully.
Information: Moved requested sector from 2097153 to 2099200 in
order to align on 2048-sector boundaries.
The operation has completed successfully.

WARNING! - Btrfs Btrfs v0.19 IS EXPERIMENTAL
WARNING! - see http://btrfs.wiki.kernel.org before using

fs created label (null) on /dev/sda1
    nodesize 32768 leafsize 32768 sectorsize 4096 size 114.69GB
Btrfs Btrfs v0.19
The operation has completed successfully.

# ceph-disk list
Problem opening /dev/mmcblk for reading! Error is 2.
The specified file does not exist!
Problem opening /dev/mmcblk for reading! Error is 2.
The specified file does not exist!
/dev/mmcblk0 :
 /dev/mmcblk0p1 other, vfat, mounted on /boot
 /dev/mmcblk0p2 other, ext4, mounted on /
/dev/sda :
 /dev/sda1 ceph data, prepared, cluster ceph, journal /dev/sda2
 /dev/sda2 ceph journal, for /dev/sda1
```

As seen above ceph-disk-prepare has created two partitions - one for data and one for OSD journal.
```
# mkdir /var/lib/ceph/osd/ceph-0
# mount /dev/sda1 /var/lib/ceph/osd/ceph-0
# ceph osd create
0
# ceph-osd -i 0 -c /etc/ceph/ceph.conf --mkfs --mkjournal
2015-10-09 03:36:37.192111 760a4400 -1 journal FileJournal::_open: disabling aio for non-block journal.  Use journal_force_aio to force use of aio anyway
2015-10-09 03:36:37.386428 760a4400 -1 journal FileJournal::_open: disabling aio for non-block journal.  Use journal_force_aio to force use of aio anyway
2015-10-09 03:36:37.389218 760a4400 -1 filestore(/var/lib/ceph/osd/ceph-0) could not find 23c2fcde/osd_superblock/0//-1 in index: (2) No such file or directory
2015-10-09 03:36:37.508850 760a4400 -1 created object store /var/lib/ceph/osd/ceph-0 journal /var/lib/ceph/osd/ceph-0/journal for osd.1 fsid 8e784fd8-a016-411d-bbe8-e2337222e935

# ceph-disk list
Problem opening /dev/mmcblk for reading! Error is 2.
The specified file does not exist!
Problem opening /dev/mmcblk for reading! Error is 2.
The specified file does not exist!
/dev/mmcblk0 :
 /dev/mmcblk0p1 other, vfat, mounted on /boot
 /dev/mmcblk0p2 other, ext4, mounted on /
/dev/sda :
 /dev/sda1 ceph data, prepared, cluster ceph, osd.0, journal /dev/sda2
 /dev/sda2 ceph journal, for /dev/sda1
```
###### Create the OSD on each node:
```
# ceph osd tree
ID WEIGHT TYPE NAME          UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1      0 root default                                         
-2      0     host cephnode1 
                                  
# ceph osd create
0

# ceph auth get-or-create osd.0 mon 'allow rwx' osd 'allow *' -o /var/lib/ceph/osd/ceph-0/keyring

# /etc/init.d/ceph start osd.0
=== osd.0 === 
Mounting Btrfs on cephnode1:/var/lib/ceph/osd/ceph-0
Scanning for Btrfs filesystems
create-or-move updating item name 'osd.0' weight 0.11 at location {host=cephnode1,root=default} to crush map
Starting Ceph osd.0 on cephnode1...
starting osd.0 at :/0 osd_data /var/lib/ceph/osd/ceph-0 /var/lib/ceph/osd/ceph-0/journal
```
__Node 2:__
```
# mkdir /var/lib/ceph/osd/ceph-1

# mount /dev/sda1 /var/lib/ceph/osd/ceph-1

# ceph-osd -i 1 -c /etc/ceph/ceph.conf --mkfs --mkjournal
2015-10-09 03:54:58.315471 760943c0 -1 journal FileJournal::_open: disabling aio for non-block journal.  Use journal_force_aio to force use of aio anyway
2015-10-09 03:54:58.503126 760943c0 -1 journal FileJournal::_open: disabling aio for non-block journal.  Use journal_force_aio to force use of aio anyway
2015-10-09 03:54:58.505834 760943c0 -1 filestore(/var/lib/ceph/osd/ceph-1) could not find 23c2fcde/osd_superblock/0//-1 in index: (2) No such file or directory
2015-10-09 03:54:58.623734 760943c0 -1 created object store /var/lib/ceph/osd/ceph-1 journal /var/lib/ceph/osd/ceph-1/journal for osd.1 fsid 8e784fd8-a016-411d-bbe8-e2337222e935

# ls -ltr /var/lib/ceph/osd/ceph-1/
total 20
lrwxrwxrwx 1 root root 58 Oct  9 04:26 journal -> /dev/disk/by-partuuid/6ad1a703-6c8f-456a-80f3-2de3a96d9bc9
-rw-r--r-- 1 root root 37 Oct  9 04:26 ceph_fsid
-rw-r--r-- 1 root root 37 Oct  9 04:26 fsid
-rw-r--r-- 1 root root 21 Oct  9 04:26 magic
-rw-r--r-- 1 root root 37 Oct  9 04:26 journal_uuid
```
###### Create the Key
```
# ceph auth get-or-create osd.1 mon 'allow rwx' osd 'allow *' -o /var/lib/ceph/osd/ceph-1/keyring

# ls -ltr /var/lib/ceph/osd/ceph-1/
total 24
lrwxrwxrwx 1 root root 58 Oct  9 04:26 journal -> /dev/disk/by-partuuid/6ad1a703-6c8f-456a-80f3-2de3a96d9bc9
-rw-r--r-- 1 root root 37 Oct  9 04:26 ceph_fsid
-rw-r--r-- 1 root root 37 Oct  9 04:26 fsid
-rw-r--r-- 1 root root 21 Oct  9 04:26 magic
-rw-r--r-- 1 root root 37 Oct  9 04:26 journal_uuid
-rw-r--r-- 1 root root 56 Oct  9 04:30 keyring


# ceph osd tree
ID WEIGHT  TYPE NAME          UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 0.10999 root default                                         
-2 0.10999     host cephnode1                                   
 0 0.10999         osd.0           up  1.00000          1.00000 
```
Create the OSD:
```
# ceph osd create
1

# ceph osd tree
ID WEIGHT  TYPE NAME          UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 0.10999 root default                                         
-2 0.10999     host cephnode1                                   
 0 0.10999         osd.0           up  1.00000          1.00000 
 1       0 osd.1                 down        0          1.00000 
```
Start the OSD:
```
# /etc/init.d/ceph start osd.1
=== osd.1 === 
Mounting Btrfs on cephnode2:/var/lib/ceph/osd/ceph-1
Scanning for Btrfs filesystems
create-or-move updated item name 'osd.1' weight 0.11 at location {host=cephnode2,root=default} to crush map
Starting Ceph osd.1 on cephnode2...
starting osd.1 at :/0 osd_data /var/lib/ceph/osd/ceph-1 /var/lib/ceph/osd/ceph-1/journal



# ceph osd tree
ID WEIGHT  TYPE NAME          UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 0.21997 root default                                         
-2 0.10999     host cephnode1                                   
 0 0.10999         osd.0           up  1.00000          1.00000 
-3 0.10999     host cephnode2                                   
 1 0.10999         osd.1           up  1.00000          1.00000 
```
__Node3:__
```
# mkdir /var/lib/ceph/osd/ceph-2

# mount /dev/sda1 /var/lib/ceph/osd/ceph-2/

# df -h
Filesystem      Size  Used Avail Use% Mounted on
rootfs           30G  2.6G   26G  10% /
/dev/root        30G  2.6G   26G  10% /
devtmpfs        460M     0  460M   0% /dev
tmpfs            93M  244K   93M   1% /run
tmpfs           5.0M     0  5.0M   0% /run/lock
tmpfs           186M     0  186M   0% /run/shm
/dev/mmcblk0p1   56M   19M   37M  34% /boot
/dev/sda1       115G   33M  114G   1% /var/lib/ceph/osd/ceph-2

# ls -ltr /var/lib/ceph/osd/ceph-2/
total 20
lrwxrwxrwx 1 root root 58 Oct  9 03:18 journal -> /dev/disk/by-partuuid/bdd65d5a-78f1-4287-b11a-969f10abfc2c
-rw-r--r-- 1 root root 37 Oct  9 03:18 ceph_fsid
-rw-r--r-- 1 root root 37 Oct  9 03:18 fsid
-rw-r--r-- 1 root root 37 Oct  9 03:18 journal_uuid
-rw-r--r-- 1 root root 21 Oct  9 03:18 magic

# ceph-osd -i 2 -c /etc/ceph/ceph.conf --mkfs --mkjournal
SG_IO: bad/missing sense data, sb[]:  f0 00 05 00 00 00 00 14 00 00 00 00 20 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
2015-10-09 04:47:45.360205 7608a400 -1 journal check: ondisk fsid 00000000-0000-0000-0000-000000000000 doesn't match expected 0decf0c9-9a41-48be-9caf-05026b353eb1, invalid (someone else's?) journal
SG_IO: bad/missing sense data, sb[]:  f0 00 05 00 00 00 00 14 00 00 00 00 20 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
SG_IO: bad/missing sense data, sb[]:  f0 00 05 00 00 00 00 14 00 00 00 00 20 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
SG_IO: bad/missing sense data, sb[]:  f0 00 05 00 00 00 00 14 00 00 00 00 20 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00 00
2015-10-09 04:47:45.747023 7608a400 -1 filestore(/var/lib/ceph/osd/ceph-2) could not find 23c2fcde/osd_superblock/0//-1 in index: (2) No such file or directory
2015-10-09 04:47:45.844732 7608a400 -1 created object store /var/lib/ceph/osd/ceph-2 journal /var/lib/ceph/osd/ceph-2/journal for osd.2 fsid 8e784fd8-a016-411d-bbe8-e2337222e935

# ls -ltr /var/lib/ceph/osd/ceph-2/
total 36
lrwxrwxrwx 1 root root 58 Oct  9 03:18 journal -> /dev/disk/by-partuuid/bdd65d5a-78f1-4287-b11a-969f10abfc2c
-rw-r--r-- 1 root root 37 Oct  9 03:18 ceph_fsid
-rw-r--r-- 1 root root 37 Oct  9 03:18 fsid
-rw-r--r-- 1 root root 37 Oct  9 03:18 journal_uuid
-rw-r--r-- 1 root root 21 Oct  9 03:18 magic
-rw-r--r-- 1 root root  4 Oct  9 04:47 store_version
-rw-r--r-- 1 root root 53 Oct  9 04:47 superblock
drwxr-xr-x 1 root root 26 Oct  9 04:47 snap_1
drwxr-xr-x 1 root root 42 Oct  9 04:47 snap_2
drwxr-xr-x 1 root root 42 Oct  9 04:47 current
-rw-r--r-- 1 root root  2 Oct  9 04:47 whoami
-rw-r--r-- 1 root root  6 Oct  9 04:47 ready

# ceph auth get-or-create osd.2 mon 'allow rwx' osd 'allow *' -o /var/lib/ceph/osd/ceph-2/keyring

# ls -ltr /var/lib/ceph/osd/ceph-2/
total 40
lrwxrwxrwx 1 root root 58 Oct  9 03:18 journal -> /dev/disk/by-partuuid/bdd65d5a-78f1-4287-b11a-969f10abfc2c
-rw-r--r-- 1 root root 37 Oct  9 03:18 ceph_fsid
-rw-r--r-- 1 root root 37 Oct  9 03:18 fsid
-rw-r--r-- 1 root root 37 Oct  9 03:18 journal_uuid
-rw-r--r-- 1 root root 21 Oct  9 03:18 magic
-rw-r--r-- 1 root root  4 Oct  9 04:47 store_version
-rw-r--r-- 1 root root 53 Oct  9 04:47 superblock
drwxr-xr-x 1 root root 26 Oct  9 04:47 snap_1
drwxr-xr-x 1 root root 42 Oct  9 04:47 snap_2
drwxr-xr-x 1 root root 42 Oct  9 04:47 current
-rw-r--r-- 1 root root  2 Oct  9 04:47 whoami
-rw-r--r-- 1 root root  6 Oct  9 04:47 ready
-rw-r--r-- 1 root root 56 Oct  9 04:48 keyring

# ceph osd create
2

# ceph osd tree
ID WEIGHT  TYPE NAME          UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-1 0.21997 root default                                         
-2 0.10999     host cephnode1                                   
 0 0.10999         osd.0           up  1.00000          1.00000 
-3 0.10999     host cephnode2                                   
 1 0.10999         osd.1           up  1.00000          1.00000 
 2       0 osd.2                 down        0          1.00000 

# ceph osd crush add 2 0.10999 host=cephnode3
add item id 2 name 'osd.2' weight 0.10999 at location {host=cephnode3} to crush map

# ceph osd tree
ID WEIGHT  TYPE NAME          UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-4 0.10999 host cephnode3                                       
 2 0.10999     osd.2             down        0          1.00000 
-1 0.21997 root default                                         
-2 0.10999     host cephnode1                                   
 0 0.10999         osd.0           up  1.00000          1.00000 
-3 0.10999     host cephnode2                                   
 1 0.10999         osd.1           up  1.00000          1.00000 

# /etc/init.d/ceph start osd.2
=== osd.2 === 
Mounting Btrfs on cephnode3:/var/lib/ceph/osd/ceph-2
Scanning for Btrfs filesystems
create-or-move updated item name 'osd.2' weight 0.11 at location {host=cephnode3,root=default} to crush map
Starting Ceph osd.2 on cephnode3...
starting osd.2 at :/0 osd_data /var/lib/ceph/osd/ceph-2 /var/lib/ceph/osd/ceph-2/journal

# ceph osd tree
ID WEIGHT  TYPE NAME          UP/DOWN REWEIGHT PRIMARY-AFFINITY 
-4 0.10999 host cephnode3                                       
 2 0.10999     osd.2               up  1.00000          1.00000 
-1 0.21997 root default                                         
-2 0.10999     host cephnode1                                   
 0 0.10999         osd.0           up  1.00000          1.00000 
-3 0.10999     host cephnode2                                   
 1 0.10999         osd.1           up  1.00000          1.00000 
```

###### Fix the CRUSH map (if required):
```
# ceph -s
    cluster 8e784fd8-a016-411d-bbe8-e2337222e935
     health HEALTH_WARN
            128 pgs degraded
            128 pgs stuck degraded
            128 pgs stuck unclean
            128 pgs stuck undersized
            128 pgs undersized
            recovery 66/198 objects degraded (33.333%)
     monmap e1: 3 mons at {1=192.168.0.201:6789/0,2=192.168.0.202:6789/0,3=192.168.0.203:6789/0}
            election epoch 44, quorum 0,1,2 1,2,3
     osdmap e123: 3 osds: 3 up, 3 in
      pgmap v368: 128 pgs, 4 pools, 264 MB data, 66 objects
            634 MB used, 340 GB / 344 GB avail
            66/198 objects degraded (33.333%)
                 128 active+undersized+degraded
```

The cluster stays in “stale+undersized+degraded+peered” state forever. This means there is something wrong with the cluster “CRUSH” map and needs to be fixed.

__CRUSHTOOL:__
Get the current “CRUSH” Map:
```
# ceph osd getcrushmap -o /tmp/crush.map 
got crush map from osdmap epoch 133
```
This dumps a “binary” crush map in the output file location specified in /tmp. We need to make it ‘human readable’ or convert to text.

Get Text format (decompile):
```
# crushtool -d /tmp/crush.map >/tmp/crush.txt
```


Now Edit the text CRUSH map

FROM (old):
```
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable straw_calc_version 1

# devices
device 0 osd.0
device 1 osd.1
device 2 osd.2

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host cephnode1 {
    id -2        # do not change unnecessarily
    # weight 0.110
    alg straw
    hash 0    # rjenkins1
    item osd.0 weight 0.110
}
host cephnode2 {
    id -3        # do not change unnecessarily
    # weight 0.110
    alg straw
    hash 0    # rjenkins1
    item osd.1 weight 0.110
}
```
```diff
- root default {
-    id -1        # do not change unnecessarily
-    # weight 0.220
-    alg straw
-    hash 0    # rjenkins1
-    item cephnode1 weight 0.110
-    item cephnode2 weight 0.110
-}
```
```
host cephnode3 {
    id -4        # do not change unnecessarily
    # weight 0.110
    alg straw
    hash 0    # rjenkins1
    item osd.2 weight 0.110
}

# rules
rule replicated_ruleset {
    ruleset 0
    type replicated
    min_size 1
    max_size 10
    step take default
    step chooseleaf firstn 0 type host
    step emit
}

# end crush map
```
 Section in RED does not look quite right...fix it!!!

TO (new):
```
# begin crush map
tunable choose_local_tries 0
tunable choose_local_fallback_tries 0
tunable choose_total_tries 50
tunable chooseleaf_descend_once 1
tunable straw_calc_version 1

# devices
device 0 osd.0
device 1 osd.1
device 2 osd.2

# types
type 0 osd
type 1 host
type 2 chassis
type 3 rack
type 4 row
type 5 pdu
type 6 pod
type 7 room
type 8 datacenter
type 9 region
type 10 root

# buckets
host cephnode1 {
    id -2        # do not change unnecessarily
    # weight 0.110
    alg straw
    hash 0    # rjenkins1
    item osd.0 weight 0.110
}
host cephnode2 {
    id -3        # do not change unnecessarily
    # weight 0.110
    alg straw
    hash 0    # rjenkins1
    item osd.1 weight 0.110
}
host cephnode3 {
    id -4        # do not change unnecessarily
    # weight 0.110
    alg straw
    hash 0    # rjenkins1
    item osd.2 weight 0.110
}
```
```diff
+ root default {
+    id -1        # do not change unnecessarily
+    # weight 0.330
+    alg straw
+    hash 0    # rjenkins1
+    item cephnode1 weight 0.110
+    item cephnode2 weight 0.110
+    item cephnode3 weight 0.110
+}
```
```
# rules
rule replicated_ruleset {
    ruleset 0
    type replicated
    min_size 1
    max_size 10
    step take default
    step chooseleaf firstn 0 type host
    step emit
}

# end crush map
```
The fixed section in GREEN

Compile the new CRUSH map:
# crushtool -c /tmp/crush.txt -o /tmp/new_crush.map

Test the new compiled CRUSH map and make sure it fixed the PGs:
# crushtool -i /tmp/new_crush.map --test --output-csv
Check the output CSV files in current folder (folder from where crushtool was run) and make sure there were no “errors” reported.

Now we load the new compiled MAP into the ceph cluster.:
If no errors reported in CSV file - then we will load the crush map into the cluster:

# ceph osd setcrushmap -i /tmp/new_crush.map 
set crush map

Now “watch” the cluster get fixed:
2015-10-12 01:15:36.429315 mon.0 [INF] HEALTH_WARN; 128 pgs degraded; 128 pgs stuck degraded; 128 pgs stuck unclean; 128 pgs stuck undersized; 128 pgs undersized; recovery 66/198 objects degraded (33.333%)
2015-10-12 01:31:49.980976 mon.1 [INF] from='client.? 192.168.0.201:0/1008795' entity='client.admin' cmd=[{"prefix": "osd setcrushmap"}]: dispatch
2015-10-12 01:31:49.992538 mon.0 [INF] from='client.74197 :/0' entity='client.admin' cmd=[{"prefix": "osd setcrushmap"}]: dispatch
2015-10-12 01:31:51.819164 mon.0 [INF] from='client.74197 :/0' entity='client.admin' cmd='[{"prefix": "osd setcrushmap"}]': finished
2015-10-12 01:31:51.832785 mon.0 [INF] osdmap e134: 3 osds: 3 up, 3 in
2015-10-12 01:31:51.879429 mon.0 [INF] pgmap v390: 128 pgs: 128 active+undersized+degraded; 264 MB data, 634 MB used, 340 GB / 344 GB avail; 66/198 objects degraded (33.333%)
2015-10-12 01:31:52.933439 mon.0 [INF] osdmap e135: 3 osds: 3 up, 3 in
2015-10-12 01:31:52.989034 mon.0 [INF] pgmap v391: 128 pgs: 128 active+undersized+degraded; 264 MB data, 634 MB used, 340 GB / 344 GB avail; 66/198 objects degraded (33.333%)
2015-10-12 01:31:57.257171 mon.0 [INF] pgmap v392: 128 pgs: 92 active+undersized+degraded, 5 active+degraded, 31 active+clean; 264 MB data, 635 MB used, 340 GB / 344 GB avail; 78/198 objects degraded (39.394%); 1530 kB/s, 0 objects/s recovering
2015-10-12 01:31:58.391323 mon.0 [INF] pgmap v393: 128 pgs: 27 active+degraded, 101 active+clean; 264 MB data, 636 MB used, 340 GB / 344 GB avail; 95/198 objects degraded (47.980%); 1535 kB/s, 0 objects/s recovering
2015-10-12 01:31:55.559625 osd.0 [INF] 9.1 scrub starts
2015-10-12 01:31:55.566834 osd.0 [INF] 9.1 scrub ok
2015-10-12 01:31:57.608938 osd.0 [INF] 9.2 scrub starts
2015-10-12 01:31:57.616512 osd.0 [INF] 9.2 scrub ok
2015-10-12 01:32:02.207728 mon.0 [INF] pgmap v394: 128 pgs: 26 active+degraded, 102 active+clean; 264 MB data, 636 MB used, 340 GB / 344 GB avail; 87/198 objects degraded (43.939%); 3289 kB/s, 0 objects/s recovering
2015-10-12 01:32:03.326769 mon.0 [INF] pgmap v395: 128 pgs: 23 active+degraded, 105 active+clean; 264 MB data, 636 MB used, 340 GB / 344 GB avail; 77/198 objects degraded (38.889%); 8258 kB/s, 2 objects/s recovering
2015-10-12 01:31:58.092963 osd.1 [INF] 9.0 scrub starts
2015-10-12 01:31:58.099615 osd.1 [INF] 9.0 scrub ok
2015-10-12 01:31:59.616379 osd.2 [INF] 9.3 scrub starts
2015-10-12 01:32:00.023651 osd.2 [INF] 9.3 scrub ok
2015-10-12 01:32:07.193534 mon.0 [INF] pgmap v396: 128 pgs: 22 active+degraded, 106 active+clean; 264 MB data, 637 MB used, 340 GB / 344 GB avail; 73/198 objects degraded (36.869%); 6547 kB/s, 1 objects/s recovering
2015-10-12 01:32:08.297791 mon.0 [INF] pgmap v397: 128 pgs: 19 active+degraded, 109 active+clean; 264 MB data, 661 MB used, 340 GB / 344 GB avail; 63/198 objects degraded (31.818%); 6489 kB/s, 1 objects/s recovering
2015-10-12 01:32:11.678214 osd.0 [INF] 9.7 scrub starts
2015-10-12 01:32:11.685271 osd.0 [INF] 9.7 scrub ok
2015-10-12 01:32:12.207372 mon.0 [INF] pgmap v398: 128 pgs: 17 active+degraded, 111 active+clean; 264 MB data, 661 MB used, 340 GB / 344 GB avail; 57/198 objects degraded (28.788%); 7371 kB/s, 1 objects/s recovering
2015-10-12 01:32:13.298026 mon.0 [INF] pgmap v399: 128 pgs: 14 active+degraded, 114 active+clean; 264 MB data, 661 MB used, 340 GB / 344 GB avail; 48/198 objects degraded (24.242%); 6619 kB/s, 1 objects/s recovering
2015-10-12 01:32:07.390507 osd.1 [INF] 9.4 scrub starts
2015-10-12 01:32:07.397477 osd.1 [INF] 9.4 scrub ok
2015-10-12 01:32:08.096283 osd.1 [INF] 9.5 scrub starts
2015-10-12 01:32:08.102076 osd.1 [INF] 9.5 scrub ok
2015-10-12 01:32:10.780613 osd.1 [INF] 9.9 scrub starts
2015-10-12 01:32:10.795690 osd.1 [INF] 9.9 scrub ok
2015-10-12 01:32:17.194088 mon.0 [INF] pgmap v400: 128 pgs: 13 active+degraded, 114 active+clean, 1 active+recovering+degraded; 264 MB data, 661 MB used, 340 GB / 344 GB avail; 46/198 objects degraded (23.232%); 5733 kB/s, 1 objects/s recovering
2015-10-12 01:32:18.282792 mon.0 [INF] pgmap v401: 128 pgs: 11 active+degraded, 116 active+clean, 1 active+recovering+degraded; 264 MB data, 717 MB used, 340 GB / 344 GB avail; 42/198 objects degraded (21.212%); 4106 kB/s, 1 objects/s recovering
2015-10-12 01:32:13.094476 osd.0 [INF] 9.b scrub starts
2015-10-12 01:32:13.101666 osd.0 [INF] 9.b scrub ok
2015-10-12 01:32:14.306653 osd.0 [INF] 9.c scrub starts
2015-10-12 01:32:14.313794 osd.0 [INF] 9.c scrub ok
2015-10-12 01:32:17.586963 osd.0 [INF] 9.e scrub starts
2015-10-12 01:32:17.593028 osd.0 [INF] 9.e scrub ok
2015

AND VOILA...the error % starts going down...and slowly we get HEALTH_OK!!!

# ceph -s
    cluster 8e784fd8-a016-411d-bbe8-e2337222e935
     health HEALTH_OK
     monmap e1: 3 mons at {1=192.168.0.201:6789/0,2=192.168.0.202:6789/0,3=192.168.0.203:6789/0}
            election epoch 44, quorum 0,1,2 1,2,3
     osdmap e135: 3 osds: 3 up, 3 in
      pgmap v462: 128 pgs, 4 pools, 264 MB data, 66 objects
            902 MB used, 340 GB / 344 GB avail
                 128 active+clean

Run a RADOS bench “benchmark” test (to check the speed...not really...its just to make sure everything works):

rados bench -p cinder 50 -t 64 write
 Maintaining 64 concurrent writes of 4194304 bytes for up to 50 seconds or 0 objects
 Object prefix: benchmark_data_cephnode1_9504
   sec Cur ops   started  finished  avg MB/s  cur MB/s  last lat   avg lat
     0       0         0         0         0         0         -         0
     1      25        25         0         0         0         -         0
     2      25        25         0         0         0         -         0
     3      25        25         0         0         0         -         0
     4      26        26         0         0         0         -         0
     5      27        27         0         0         0         -         0
     6      31        31         0         0         0         -         0
     7      33        33         0         0         0         -         0
     8      35        35         0         0         0         -         0
     9      35        35         0         0         0         -         0
    10      35        35         0         0         0         -         0
    11      38        38         0         0         0         -         0
    12      40        40         0         0         0         -         0
    13      45        45         0         0         0         -         0
    14      45        45         0         0         0         -         0
    15      46        46         0         0         0         -         0
    16      47        47         0         0         0         -         0
    17      48        48         0         0         0         -         0
    18      53        53         0         0         0         -         0
    19      55        55         0         0         0         -         0
2015-10-12 01:51:32.948934min lat: 9999 max lat: 0 avg lat: 0
   sec Cur ops   started  finished  avg MB/s  cur MB/s  last lat   avg lat
    20      55        55         0         0         0         -         0
    21      55        55         0         0         0         -         0
    22      60        60         0         0         0         -         0
    23      62        62         0         0         0         -         0
    24      63        64         1    0.1665  0.166667   23.5925   23.5925
    25      63        65         2  0.319688         4   24.5966   24.0945
    26      63        66         3  0.461099         4   25.1594   24.4495
    27      63        67         4  0.592041         4    26.355   24.9259
    28      63        72         9   1.28454        20    27.918   26.4207
    29      63        73        10   1.37808         4   28.1545   26.5941
    30      63        75        12   1.59852         8   29.9215   27.1245
    31      63        75        12   1.54698         0         -   27.1245
    32      63        76        13   1.62353         2    31.761   27.4812
    33      63        76        13   1.57433         0         -   27.4812
    34      63        79        16   1.88067         6   33.7933   28.6271
    35      63        84        21   2.39774        20   34.9465   30.0125
    36      63        85        22   2.44217         4   35.5441    30.264
    37      63        86        23   2.48421         4   36.2024   30.5222
    38      63        88        25    2.6292         8   37.7433   31.0744
    39      63        88        25   2.56182         0         -   31.0744
2015-10-12 01:51:52.962611min lat: 23.5925 max lat: 39.8838 avg lat: 31.6124
   sec Cur ops   started  finished  avg MB/s  cur MB/s  last lat   avg lat
    40      63        90        27   2.69763         4   36.7915   31.6124
    41      63        94        31   3.02168        16   35.2597   32.0906
    42      63        94        31   2.94974         0         -   32.0906
    43      63        95        32   2.97412         2   37.2814   32.2528
    44      63        97        34   3.08819         8   36.8831   32.5262
    45      63        99        36   3.19718         8   37.2577   32.7953
    46      63       102        39   3.38836        12   34.9474   32.9873
    47      63       105        42    3.5714        12   34.4683   33.1233
    48      63       106        43   3.58027         4   35.6819   33.1828
    49      63       107        44   3.58876         4   35.4957   33.2354
    50      63       107        44     3.517         0         -   33.2354
    51      35       108        73   5.72066        58   22.9508   31.8742
    52      33       108        75   5.76441         8   22.3088   31.6324
    53      33       108        75    5.6557         0         -   31.6324
    54      33       108        75   5.55101         0         -   31.6324
    55      31       108        77   5.59547   2.66667   22.6091   31.4219
    56      24       108        84   5.99519        28   21.1266   30.6068
    57      20       108        88   6.17054        16   19.9303   30.1456
    58      20       108        88   6.06414         0         -   30.1456
    59      20       108        88    5.9614         0         -   30.1456
2015-10-12 01:52:12.974881min lat: 19.9303 max lat: 39.8838 avg lat: 30.1456
   sec Cur ops   started  finished  avg MB/s  cur MB/s  last lat   avg lat
    60      20       108        88   5.86204         0         -   30.1456
    61      17       108        91   5.96245         3   20.9452    29.861
    62      14       108        94   6.05968        12   20.7667   29.5737
    63      10       108        98    6.2173        16   19.1789   29.1746
    64      10       108        98   6.12019         0         -   29.1746
    65      10       108        98   6.02607         0         -   29.1746
    66       7       108       101   6.11643         4   20.1544   28.9187
    67       5       108       103   6.14445         8   20.7899    28.761
    68       5       108       103   6.05413         0         -    28.761
 Total time run:         68.932936
Total writes made:      108
Write size:             4194304
Bandwidth (MB/sec):     6.267 

Stddev Bandwidth:       8.849
Max bandwidth (MB/sec): 58
Min bandwidth (MB/sec): 0
Average Latency:        28.433
Stddev Latency:         6.30008
Max latency:            39.8838
Min latency:            19.1789

