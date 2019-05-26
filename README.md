# datatransfer
DTN and data transfer

**Table of Contents:**
1. [Mount Salk USB drive](#usb-mount)
1. [Copy data from USB to zpool](#cp-data)
1. [Add route](#add-route)
1. [Setup FDT](#setup-fdt) <br>


#### 1. Mount Salk USB drive
{: id="usb-mount"}

1. Create zfs slice for storing the data
   ```bash
   zpool list
   zfs create pool1/pragma/salk
   zfs list
   ```
1. Using Salk drive via USB port

   1. Check that no current USB drive is attached
      ```bash
      cd /var/log
      grep usb messages
      fdisk -l
      ```
   
   1. Attach USB drive and check what device is used
      ```bash
      grep usb messages
      fdisk -l /dev/sds
      ```
   
   1.  Create  mount point
       ```bash
       ls /media
       mkdir /media/usb-drive
       ```
 
    1. Install fuse and ntfs-3g RPMs needed for mounting USB drive (partitioned on windows)

       Install yum-utils (to get yumdownloader) and fuse form local repo and ntfs-3g from epel.
       ```bash
       cd /pool1/pragma/salk/
       mkdir downloads
       cd downloads/

       yum list yum-utils
       install yum-utils
   
       yum list fuse
       yum install fuse
   
       yum --enablerepo=epel list ntfs-3g
       yumdownloader --resolve --enablerepo=epel ntfs-3g
       rpm -i ntfs-3g-2017.3.23-6.el7.x86_64.rpm
       yum list ntfs-3g
       ```
   
    1. Mount USB drive

       ```bash
       modprobe fuse
       lsmod | grep fuse

       fdisk -l
       ls /media/usb-drive/
       mount -t ntfs-3g /dev/sds2 /media/usb-drive/
       ls /media/usb-drive/
       ```
#### 2. Copy data from USB to zpool
{: id="cp-data"}

Once the USB dive is mounted, copy data from the USB disk using its original layout .
The Disk contents for one day sampling :
```text
20180620/
   File_and_preprocessing_info/  approx 20Mb, info and matlab processing scripts, power point presentation
   Sample_1/  ~900MB raw data
   Sample_2/  ~1Tb   raw data
   Sample_3/  ~900Mb raw data
```

Create data storage on the zfs slice:
```bash
 cd /pool1/pragma/salk/
 mkdir 20180620
 cd /pool1/pragma/salk/20180620
 ```

For each sample directory recursively copy data
```bash
date >> out
cp -r /media/usb-drive/Sample_1/ .
date >> out
```

Collected local data transfer times (from out):

|  | Sample 2 | Sample 3 |
|--|--|--|
| Start | Thu Jun 21 17:32:47 AKDT 2018   | Fri Jun 22 16:55:35 AKDT 2018 |
| End | Fri Jun 22 01:15       |          Sat Jun 23 00:02:40 AKDT 2018 |



#### 3. Add Route
{: id="add-route"}

TBA

#### 4. Setup FDT
{: id="setup-fdt"}

TBA


