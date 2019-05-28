# datatransfer
DTN and data transfer

**Table of Contents:**
1. [Mount Salk USB drive](#mount-salk-usb-drive)
1. [Copy data from USB](#copy-data-from-usb)
1. [Add firewall](#add-firewall)
1. [Add route](#add-route)
1. [Setup FDT](#setup-fdt) 
   1. [Prerequisites](#prerequisites)
   1. [Install FDT](#install-fdt)
   1. [Test setup](#test-setup)


### Mount Salk USB drive

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
   
   1. Load fuse module

      First, there should be no fuse module, as the fuse RPM was just installed. After modprobe command
      fuse is loaded:
      ```txt
      # lsmod | grep fuse
      # modprobe fuse
      # lsmod | grep fuse
      fuse                   91874  1 
      ```
       
   1. Mount USB drive

      Check what drive systems recognizes as USB
      ```
      fdisk -l | grep Disk | grep dev | grep -v fdisk
      Disk /dev/sda: 120.0 GB, 120034123776 bytes, 234441648 sectors
      Disk /dev/sdb: 120.0 GB, 120034123776 bytes, 234441648 sectors
      Disk /dev/md127: 2147 MB, 2147483648 bytes, 4194304 sectors
      Disk /dev/md126: 68.7 GB, 68719476736 bytes, 134217728 sectors
      Disk /dev/md125: 2147 MB, 2147483648 bytes, 4194304 sectors
      Disk /dev/sde: 10000.8 GB, 10000831348736 bytes, 19532873728 sectors
      Disk /dev/sdf: 10000.8 GB, 10000831348736 bytes, 19532873728 sectors
      ...
      Disk /dev/sdp: 10000.8 GB, 10000831348736 bytes, 19532873728 sectors
      Disk /dev/md124: 46.9 GB, 46912241664 bytes, 91625472 sectors
      Disk /dev/sds: 8001.6 GB, 8001563221504 bytes, 15628053167 sectors
      ```
      
      From the output above it is /dev/sds. Mount via:
      ```txt
      # ls /media/usb-drive/
      # mount -t ntfs-3g /dev/sds2 /media/usb-drive/
      # ls /media/usb-drive/
      Autorun.inf                  Sample_1  Sample_3  SeagateExpansion.ico  Start_Here_Win.exe
      File_and_preprocessing_info  Sample_2  Seagate   Start_Here_Mac.app    Warranty.pdf
      ```
      
### Copy data from USB

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


### Add firewall

on frontend  create a script `/root/code/add-pragmadata-firewall` with the wollowing contents:
```bash
#!/bin/bash

# add rules on frontend for the pragmadata-1-1 host:
# - acceppt conencitons for ports 5000-5020 
# - acceppt conencitons for ssh
# - acceppt conencitons for tcp on established on host for prototocls (initiated on the host, like wget)
# - reject all other connecitons 

/opt/rocks/bin/rocks add firewall host=pragmadata-1-1 network=optistorage service=all protocol=tcp action=ACCEPT chain=INPUT flags="--dport 5000:5020 -m state --state NEW,RELATED" rulename=A20-OPTISTORAGE-TCP
/opt/rocks/bin/rocks add firewall host=pragmadata-1-1 network=optistorage service=all protocol=udp action=ACCEPT chain=INPUT flags="--dport 5000:5020 -m state --state NEW,RELATED" rulename=A20-OPTISTORAGE-UDP
/opt/rocks/bin/rocks add firewall host=pragmadata-1-1 network=optistorage service=ssh protocol=tcp action=ACCEPT chain=INPUT rulename=A21-OPTISTORAGE-SSH
/opt/rocks/bin/rocks add firewall host=pragmadata-1-1 network=optistorage service=all protocol=tcp action=ACCEPT chain=INPUT flags="-m state --state RELATED,ESTABLISHED" rulename=A30-RELATED-OPTISTORAGE
/opt/rocks/bin/rocks add firewall host=pragmadata-1-1 network=optistorage service=0:65535 protocol=tcp action=DROP chain=INPUT rulename=R100-OPTISTORAGE-DROPALL-TCP
/opt/rocks/bin/rocks add firewall host=pragmadata-1-1 network=optistorage service=0:65535 protocol=udp action=DROP chain=INPUT rulename=R100-OPTISTORAGE-DROPALL-UDP

/opt/rocks/bin/rocks sync host firewall pragmadata-1-1
```

Execute the script to enable the new rules for pragmadata-1-1.

### Add Route

On pragmadata-1-1 do

```bash
ip route add default via 67.58.50.193
ip route show
```

The IP above is a gateway for the optistorage network. See network settings with
```bash
rocks list network
```

Check connectivity
```bash
traceroute www.ucsd.edu
tcpdump -i bond0 host fiji.rocksclusters.org
```

### Setup FDT

On pragmadata-1-1 do

#### Prerequisites
   
   Install latest available (for the OS ) java

   ```txt
   # yum install java-1.8.0-openjdk.x86_64
   # which java
   /usr/bin/java
   # java -version
   openjdk version "1.8.0_161"
   OpenJDK Runtime Environment (build 1.8.0_161-b14)
   OpenJDK 64-Bit Server VM (build 25.161-b14, mixed mode)
   ```

#### Install FDT

   Previously tried FDT version 0.24.  All available FDT  distributions are at the [FDT releases page][fdtrel].
   In 2018 tried to use FDT v. 0.24. As of 2019, download latest availalbe 0.26.1
   
   ```bash
   cd /pool1/pragma/salk/downloads
   wget https://github.com/fast-data-transfer/fdt/releases/download/0.26.1/fdt.jar
   
   java -jar fdt.jar -version
   2019-05-26 13:20:14 INFO lia.util.net.copy.FDT printVersion FDT 0.26.1-201708081830
   2019-05-26 13:20:14 INFO lia.util.net.copy.FDT printVersion Contact: support-fdt@monalisa.cern.ch
   mkdir /opt/fdt
   cp fdt.jar /opt/fdt
   ```
   
#### Test setup
   
Use the same version of FDT on server and client.

1. test pragmadata-1-1 as a server

   On pragmadata-1-1 start FDT server. Options:
   - `-S` disables the standalone mode, when specified the FDT server will stop after the last client finishes. 
   - `-noupdates` disables the  new releases updates check
   
   ```bash
   java -jar /opt/fdt/fdt.jar  -p 5000 -S
   ```
   
   Trimmed output:
   ```txt
   FDT [ 0.26.1-201708081830 ] STARTED ... 

   2019-05-26 14:44:23 INFO lia.util.net.common.Config <init> Using lia.util.net.copy.PosixFSFileChannelProviderFactory as FileChannelProviderFactory
   2019-05-26 14:44:23 INFO lia.util.net.common.Config <init> FDT started in server mode
   2019-05-26 14:44:23 INFO lia.util.net.copy.FDT main FDT uses *blocking* I/O mode.
   READY
   2019-05-26 14:44:23 INFO lia.util.net.copy.FDTServer doWork FDTServer start listening on port: 5000
   ```

   On the client pc-163 start  the client. Options:
   - `-c` means connect to the specified host. If this parameter is missing the FDT will become server
   - `-v` verbose

   ```bash
   java -jar ./fdt.jar -v -c 67.58.50.197 -p 5000  -d /pool1/pragma/salk/downloads file2transfer 
   ```
   
   Trimmed output:
   ```txt
   May 26, 2019 3:47:32 PM lia.util.net.copy.FDT initLogging
   INFO:  LogLevel: FINE
   May 26, 2019 3:47:32 PM lia.util.net.common.Utils initLocalProps
   INFO: Using local properties file: /root/.fdt/fdt.properties
   May 26, 2019 3:47:32 PM lia.util.net.common.Utils initLocalProps
   INFO: No local properties defined
   May 26, 2019 3:47:32 PM lia.util.net.copy.FDT main
   INFO: 

   FDT [ 0.26.1-201708081830 ] STARTED ... 
   ...
   May 26, 2019 3:47:40 PM lia.util.net.copy.FDT doWork
   INFO: 
    [ Sun May 26 15:47:40 PDT 2019 ]  FDT Session finished OK.
   ```

1. test pragmadata-1-1 as a client

   Run server on  pc-163
   ```bash
   java -jar ./fdt.jar  -p 5000 -S -noupdates
   ```
   
   Send file from pragmadata-1-1
   ```
   java -jar /opt/fdt/fdt.jar -v -c 67.58.51.163 -p 5000 -d /tmp howto-fuse
   ```
   Transfer result OK:
   ```txt
   May 26, 2019 3:11:59 PM lia.util.net.copy.FDT initLogging
   INFO:  LogLevel: FINE
   May 26, 2019 3:11:59 PM lia.util.net.common.Utils initLocalProps
   INFO: Using local properties file: /root/.fdt/fdt.properties
   May 26, 2019 3:11:59 PM lia.util.net.common.Utils initLocalProps
   INFO: No local properties defined
   May 26, 2019 3:11:59 PM lia.util.net.copy.FDT main
   INFO: 

   FDT [ 0.26.1-201708081830 ] STARTED ... 
   ...
   INFO:  [ Sun May 26 15:12:06 AKDT 2019 ]  - GracefulStopper hook finished!
   May 26, 2019 3:12:06 PM lia.util.net.copy.FDT doWork
   INFO: 
    [ Sun May 26 15:12:06 AKDT 2019 ]  FDT Session finished OK.
   ```



[fdtrel]: https://github.com/fast-data-transfer/fdt/releases/
