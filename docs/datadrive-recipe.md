# datadrive-recipe.md

## Description

Manual steps required to install, configure and prepares a local scratch data drive

## Defaults

Guessed variables and values for completing the task at hand.

```shell
datadrive_mount_point: /data1

# local users
datadrive_users_home : {{ datadrive_mount_point }}/home
datadrive_users_home_default_permissions : 0755
datadrive_users_home_default_owner : root
datadrive_users_home_default_owner : root

# ldap users
datadrive_ldap_users_home : {{ datadrive_mount_point }}/home/users
datadrive_ldap_users_home_default_permissions : 0775
datadrive_ldap_users_home_default_owner : root
datadrive_ldap_users_home_default_group : aceusers #
```

## Requirements

### gparted

```shell
sudo apt update
sudo apt install -y gparted
```



## Installation

Attach drive to SATA1 if possible and power.

```shell
ssh -X admin@workstation-001
sudo fdisk -l
```

Output example

```shell
Disk /dev/sda: 232.9 GiB, 250059350016 bytes, 488397168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 013EF090-0958-431A-A4BA-E320065E5171

Device         Start       End   Sectors   Size Type
/dev/sda1       2048   1050623   1048576   512M EFI System
/dev/sda2    1050624 455106559 454055936 216.5G Linux filesystem
/dev/sda3  455106560 488396799  33290240  15.9G Linux swap


Disk /dev/sdb: 3.7 TiB, 4000787030016 bytes, 7814037168 sectors
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
```

So our data drive is the very large hdd, in this case **sdb**

### Label, partition and format our drive

#### Set the partition table type

```shell
sudo gparted /dev/sdb
```

Choose **Device** > **Create Partition Table...**

Ensure **Set new partition table type** dropdown menu is set to **gpt**.

Choose **Apply**

#### Create on large partition

* Ensure that the **Create as**: option is set to **Primary Partition**
* Choose **Partition** then **New**.


* Set the **Partition name:** to be **data1**.
* Ensure the **File system:** is equal to **ext4**.
* Set the **Label:** to be **data1**.
* Choose the **Add** button. 

#### Apply your changes

* Choose **Edit** then **Apply All Operations** and then select the **Apply** button.

* Wait for the operations to complete and look for the message:

  ```shell
  All operations successfully completed
  ```

* If everything went well choose the **Close** button.

* Then exit **gparted** using **GParted** then **Quit**

## Getting the drive UUID

```shell
ls -al /dev/disk/by-uuid
```

Output example:

```shell
total 0
drwxr-xr-x 2 root root 120 Sep 29 10:44 .
drwxr-xr-x 8 root root 160 Sep 29 10:44 ..
lrwxrwxrwx 1 root root  10 Sep 28 16:56 2b1c5b12-8186-4076-8d84-1367fe4e152a -> ../../sda3
lrwxrwxrwx 1 root root  10 Sep 29 10:45 9802d0f9-c23a-4d51-bc8c-0467e60b0c43 -> ../../sdb1
lrwxrwxrwx 1 root root  10 Sep 28 16:56 9D6D-DA95 -> ../../sda1
lrwxrwxrwx 1 root root  10 Sep 28 16:56 e2568e93-27be-4b65-90ec-fc0e25fcea7d -> ../../sda2
```

In this case we know that our drive was originally mounted to **/dev/sdb** and the only partition showing here is **sdb1** so this is the UUID we want.

## /etc/fstab

Open /etc/fstab for editing

```shell
sudo nano /etc/fstab
```

Add (append) something like the following to the end of the file and then save your edits:

```shell
# / was on /dev/sdb1 during installation
UUID=9802d0f9-c23a-4d51-bc8c-0467e60b0c43 /data1               ext4    errors=remount-ro 0       0
```

When completed the file will look something like the following. In our example new lines have been inserted for better readability.:

```shell
# /etc/fstab: static file system information.
#
# Use 'blkid' to print the universally unique identifier for a
# device; this may be used with UUID= as a more robust way to name devices
# that works even if disks are added and removed. See fstab(5).
#
# <file system> <mount point>   <type>  <options>       <dump>  <pass>

# / was on /dev/sda2 during installation
UUID=e2568e93-27be-4b65-90ec-fc0e25fcea7d /               ext4    errors=remount-ro 0       1

# /boot/efi was on /dev/sda1 during installation
UUID=9D6D-DA95  /boot/efi       vfat    umask=0077      0       1

# swap was on /dev/sda3 during installation
UUID=2b1c5b12-8186-4076-8d84-1367fe4e152a none            swap    sw              0       0

# / was on /dev/sdb1 during installation
UUID=9802d0f9-c23a-4d51-bc8c-0467e60b0c43 /data1               ext4    errors=remount-ro 0       0
```



## Create the mount point

```shell
sudo mkdir /data1
```

## Mount the drive as a test

```shell
sudo mount /data1
mount | grep data1
```

Example  output if successfully mounted

```shell
 /dev/sdb1 on /data1 type ext4 (rw,relatime,errors=remount-ro,data=ordered)
```

## Reboot test

Next reboot the system to ensure that the drive will automatically mount on system boot.

```shell
sudo reboot
```

Wait a bit and then reconnect to the system. This time X forwarding is not required as we no longer need  to use X windows for **gparted** or any other GUI program interfaces.

Run our mount test again to ensure the disk partition is mounted

```shell
mount | grep data1
```

Example  output if successfully mounted

```shell
 /dev/sdb1 on /data1 type ext4 (rw,relatime,errors=remount-ro,data=ordered)
```

## data disk home directory

Next we will create a home directory on the data disk so that users are able to create directories for themselves.

```shell
sudo mkdir -p /data1/home/users
```

ensure for the correct permissions

For the local users `/data1/home`

```shell
sudo chown root:root /data1/home
sudo chmod 0755 /data1/home
```

Confirm changes

```shell
ls -al /data1 | grep home
```

Example output confirming our desired permissions for /data1/home

```shell
drwxr-xr-x  3 root root  4096 Sep 29 11:01 home
```

For the LDAP users `/data1/home/users`

```shell
sudo chown root:aceusers /data1/home/users
sudo chmod 0770 /data1/home/users
```

Confirm changes

```shell
ls -al /data1/home | grep users
```

Example output confirming our desired permissions for /data1/home/users

```shell
drwxr-xr-x 2 root aceusers 4096 Sep 29 11:01 users
```

## Data restore

If the users previous data drive contains data you can install the drive and copy the data to the new more robust hdd.

* mount old drive to **/data2**
* Copy, rsync or move data as required.

### Restoring data

Using rsync

as root

```shell
sudo rsync -a /data2/cmoreau/ /data1/home/users/cmoreau/
```

as regular user

```shell
rsync -a /home/my_home/ /media/backup/my_home/
```

details

```shell
From the rsync manpage:

 -a, --archive
              This  is  equivalent  to  -rlptgoD.  It  is a quick way of saying you want
              recursion and want to preserve almost everything (with -H being a  notable
              omission).    The   only  exception  to  the  above  equivalence  is  when
              --files-from is specified, in which case -r is not implied.

              Note that -a does not preserve hardlinks, because finding  multiply-linked
              files is expensive.  You must separately specify -H.
```

Using cp

```shell
sudo cp -rp /home/my_home /media/backup/my_home
```

details

```shell
From cp manpage:

 -p     same as --preserve=mode,ownership,timestamps

 --preserve[=ATTR_LIST]
          preserve the specified attributes (default: mode,ownership,timestamps),
          if possible additional attributes: context, links, xattr, all
```



