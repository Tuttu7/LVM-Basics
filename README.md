#### I have a additional hard disk /dev/xvdb in the system. To create a new partition in the new hard disk 

```
fdisk -l

fdisk /dev/xvdb

n - To create a new partition

w - To write changes to the file 

mkfs.ext4 /dev/xvdb - To format the new partition with ext4 filesystem
```

#### For checking if any physical volumes is present in the server 

```
[root@ip-172-31-45-192 ~]# pvscan
  No matching physical volumes found
```

#### Creating a new physical volume in the newly created partition 

```
[root@ip-172-31-45-192 ~]# pvcreate /dev/sdb1
WARNING: ext4 signature detected on /dev/sdb1 at offset 1080. Wipe it? [y/n]: y
  Wiping ext4 signature on /dev/sdb1.
  Physical volume "/dev/sdb1" successfully created.
  
[root@ip-172-31-45-192 ~]# pvscan
  PV /dev/sdb1                      lvm2 [<5.59 GiB]
  Total: 1 [<5.59 GiB] / in use: 0 [0   ] / in no VG: 1 [<5.59 GiB]
```

#### For checking if any volume group is present in the server 

```
[root@ip-172-31-45-192 ~]# vgscan
  Reading volume groups from cache.
```

#### Creating a new volume group
```
[root@ip-172-31-45-192 ~]# vgcreate vg00  /dev/sdb1
Volume group "vg00" successfully created
[root@ip-172-31-45-192 ~]# vgdisplay
  --- Volume group ---
  VG Name               vg00
  System ID             
  Format                lvm2
  Metadata Areas        1
  Metadata Sequence No  1
  VG Access             read/write
  VG Status             resizable
  MAX LV                0
  Cur LV                0
  Open LV               0
  Max PV                0
  Cur PV                1
  Act PV                1
  VG Size               <5.59 GiB
  PE Size               4.00 MiB
  Total PE              1430
  Alloc PE / Size       0 / 0   
  Free  PE / Size       1430 / <5.59 GiB
  VG UUID               m4PyRm-Wzc0-ssTh-hXMF-6fkw-HVUZ-N6kTEF
  
[root@ip-172-31-45-192 ~]# vgscan
Reading volume groups from cache.
Found volume group "vg00" using metadata type lvm2
```
#### Creating two LVs named vol_projects (4 GB ) and vol_backups (remaining space), which we can use later to store projects and  backups respectively. The -n option is used to indicate a name for the LV, whereas -L sets a fixed size and -l (lowercase L) is used to indicate a percentage of the remaining space in the container VG.

```
[root@ip-172-31-45-192 ~]# lvcreate -n vol_projects -L 4G vg00
Logical volume "vol_projects" created.
[root@ip-172-31-45-192 ~]# lvcreate -n vol_backups -l 100%free vg00
  Logical volume "vol_backups" created.
[root@ip-172-31-45-192 ~]# lvscan
  ACTIVE            '/dev/vg00/vol_projects' [4.00 GiB] inherit
  ACTIVE            '/dev/vg00/vol_backups' [<1.59 GiB] inherit
[root@ip-172-31-45-192 ~]# lvdisplay
  --- Logical volume ---
  LV Path                /dev/vg00/vol_projects
  LV Name                vol_projects
  VG Name                vg00
  LV UUID                BMNpPB-p7bj-NkCe-5Ike-2OO9-Lrjo-BmqlYN
  LV Write Access        read/write
  LV Creation host, time ip-172-31-45-192.ap-south-1.compute.internal, 2021-11-03 06:55:53 +0000
  LV Status              available
  # open                 0
  LV Size                4.00 GiB
  Current LE             1024
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
   
  --- Logical volume ---
  LV Path                /dev/vg00/vol_backups
  LV Name                vol_backups
  VG Name                vg00
  LV UUID                NeJHEG-BNqs-OeJI-23YK-uJFq-mufd-x11bMF
  LV Write Access        read/write
  LV Creation host, time ip-172-31-45-192.ap-south-1.compute.internal, 2021-11-03 06:56:36 +0000
  LV Status              available
  # open                 0
  LV Size                <1.59 GiB
  Current LE             406
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:1
```

#### To view information about a single LV, use lvdisplay with the VG and LV as parameters, as follows :

```
[root@ip-172-31-45-192 ~]# lvdisplay vg00/vol_projects
  --- Logical volume ---
  LV Path                /dev/vg00/vol_projects
  LV Name                vol_projects
  VG Name                vg00
  LV UUID                BMNpPB-p7bj-NkCe-5Ike-2OO9-Lrjo-BmqlYN
  LV Write Access        read/write
  LV Creation host, time ip-172-31-45-192.ap-south-1.compute.internal, 2021-11-03 06:55:53 +0000
  LV Status              available
  # open                 0
  LV Size                4.00 GiB
  Current LE             1024
  Segments               1
  Allocation             inherit
  Read ahead sectors     auto
  - currently set to     256
  Block device           253:0
  ```
  
#### Formating nelwy created logical volumes with ext4 filesystem since it allows us both to increase and reduce the size of each LV ( as opposed to xfs that only allows to increase the size )

```
# mkfs.ext4 /dev/vg00/vol_projects
# mkfs.ext4 /dev/vg00/vol_backups
```

#### Resizing Logical Volumes and Extending Volume Groups

```
[root@ip-172-31-45-192 ~]# lvreduce -L -2.5G -r /dev/mapper/vg00-vol_projects
fsck from util-linux 2.30.2
/dev/mapper/vg00-vol_projects: clean, 11/262144 files, 53326/1048576 blocks
resize2fs 1.42.9 (28-Dec-2013)
Resizing the filesystem on /dev/mapper/vg00-vol_projects to 393216 (4k) blocks.
The filesystem on /dev/mapper/vg00-vol_projects is now 393216 blocks long.

  Size of logical volume vg00/vol_projects changed from 4.00 GiB (1024 e
  
[root@ip-172-31-45-192 ~]# lvextend -l +100%FREE -r /dev/mapper/vg00-vol_backups
fsck from util-linux 2.30.2
/dev/mapper/vg00-vol_backups: clean, 11/104000 files, 15948/415744 blocks
  Size of logical volume vg00/vol_backups changed from <1.59 GiB (406 extents) to <4.09 GiB (1046 extents).
  Logical volume vg00/vol_backups successfully resized.
resize2fs 1.42.9 (28-Dec-2013)
Resizing the filesystem on /dev/mapper/vg00-vol_backups to 1071104 (4k) blocks.
The filesystem on /dev/mapper/vg00-vol_backups is now 1071104 blocks long.
```
#### -r option is equalvaelnt to --resizefs

#### Mounting Logical Volumes on Boot. To better identify a logical volume we will need to find out what its UUID (a non-changing attribute that uniquely identifies a formatted storage device) is. To do that, use blkid followed by the path to each device:


```
[root@ip-172-31-45-192 /]# blkid /dev/mapper/vg00-vol_backups
/dev/mapper/vg00-vol_backups: UUID="f8342778-628b-4650-a2cd-ee9301d21c7a" TYPE="ext4"
[root@ip-172-31-45-192 /]# blkid /dev/mapper/vg00-vol_projects
/dev/mapper/vg00-vol_projects: UUID="f2224199-f1e8-4316-80c6-3ca0ee6eb015" TYPE="ext4"

Insert the corresponding entries in /etc/fstab. Then :

mount -a
```




  
  
  
  




  
