# My NAS setup:

- 2 HDDs (Software Raid 1) for data redundancy
- LAN share with Samba
- User home directories with password protection


## 1) Create partitions:
Read disk names with `lsblk`
<br>Before creating partitions you may need to create GPT partition tables first:
```bash
root@nas:~# fdisk /dev/sda

Command (m for help): g
```

Now create the partitions on each drive:
```bash
root@nas:~# fdisk /dev/sda

Command (m for help): n
Partition number (1-128, default 1): 
First sector (34-976773134, default 2048): 
Last sector, +/-sectors or +/-size{K,M,G,T,P} (2048-976773134, default 976773134): 

Created a new partition 1 of type 'Linux filesystem' and of size 465.8 GiB.

Command (m for help): t
Selected partition 1
Partition type (type L to list all types): 29
Changed type of partition 'Linux filesystem' to 'Linux RAID'.

Command (m for help): w
The partition table has been altered.
Calling ioctl() to re-read partition table.
Syncing disks.
```
Now you should have the following drive schema (read from fdisk):

<strong>First drive</strong>
```bash
Disk /dev/sda: 465.8 GiB, 500107862016 bytes, 976773168 sectors
Disk model: 2115            
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 1078D1CF-7AF8-4F75-B4B6-D83D14BB9419

Device     Start       End   Sectors   Size Type
/dev/sda1   2048 976773134 976771087 465.8G Linux RAID
```

<br><strong>Second drive</strong>
```bash
Disk /dev/sdb: 465.8 GiB, 500107862016 bytes, 976773168 sectors
Disk model: 2115            
Units: sectors of 1 * 512 = 512 bytes
Sector size (logical/physical): 512 bytes / 512 bytes
I/O size (minimum/optimal): 512 bytes / 512 bytes
Disklabel type: gpt
Disk identifier: 06D2213F-6889-B94F-9652-7588EE8EED04

Device     Start       End   Sectors   Size Type
/dev/sdb1   2048 976773134 976771087 465.8G Linux RAID
```


## 2) Setup Software Raid 1:
Install mdadm with apt:
```bash
apt install mdadm -y
```
Tell mdadm to create a software raid 1:
```bash
mdadm --create /dev/md0 --level=1 --raid-devices=2 /dev/sda1 /dev/sdb1
Continue creating array? Y
```
Ensure mdadm started its raid syncing (first sync will take some time):
```bash
cat /proc/mdstat
or
tail -f /proc/mdstat
```

## 3) Create filesystem and mount:
Create ext4 filesystem on raid volume:
```bash
mkfs.ext4 /dev/md0
```
Create a mountpoint:
```bash
mkdir /mnt/raid_1_drive
```
Mount the raid1 volume at the previously created mountpoint:
```bash
mount /dev/md0 /mnt/raid_1_drive
```
Create a new entry in `/etc/fstab`:
```bash
/dev/md0 /mnt/raid_1_drive ext4 defaults 0 0
```


## 3) Setting up samba share:
Install samba with apt:
```shell
apt install samba -y
```
Create a new user:
```shell
useradd <username>
```
Set a password for the user:
```shell
passwd <username>
smbpasswd -a <username>
```
Add a new share directory for the user:
```shell
mkdir /mnt/raid_1_drive/<username>
```
Set permissions:
```shell
chown <username>:<username> /mnt/raid_1_drive/<username>
chmod 770 /mnt/raid_1_drive/<username>
```
Open `/etc/samba/smb.conf` and tell samba about the new share:
```shell
[<share-name>]
path = /mnt/nas/drive_1/<username>
read only = no
create mode = 0777
directory mode = 0777
writable = yes
valid users = <username>
```
Restart samba service:
```shell
systemctl restart smbd.service
```


## 4) Maintenance:
Add a cronjob to do run daily updates at midnight `crontab -e`:
```bash
0 0 * * * apt update && apt upgrade > /dev/null
```

## 5) How to repair broken Raid:
### Replace broken harddrive:
First identify which drive failed:
```bash
mdadm --detail /dev/md0
```
(Its helpful to place stickers on the hardware with the label of the disk)

Remove the broken harddrive from the raid:
```bash
mdadm --manage /dev/md0 --fail /dev/sda1
mdadm --manage /dev/md0 --remove /dev/sda1
```
Power down your machine and replace the broken drive (sticker with a name can be usefull)
```bash
shutdown -h now
```
Now format the disk properly (See 1: partitioning):
<br>When done you can add the new drive to the raid:
```bash
mdadm --manage /dev/md0 --add /dev/sda1
```
Now the raid will start its sync process ... View the state with:
```bash
tail -f /proc/mdstat
```
