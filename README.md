# My NAS setup:
This document explains the steps I took to setup my NAS on a raspbery pi.

### Solution description:
- Raspberry pi because of low energy consumption.
- Samba share with login to access data across the LAN. 
- 2x HDDs 500Gb, synced every 1 hour to provide data redundancy.
- No Raid1 because to be able to recover accidentally deleted files.
- Every 12 Hour encrypted differential backups to cloud using borg-backup.

### 1) Disk preparation:
Delete all previous partitions from the harddrive, or wipe it entirely with `dd`.
<br><strong>Partition deleting with `fdisk`</strong>:
```shell
fdisk /dev/sdX
d
w
```
<strong>Wiping with random data using `dd`</strong>:
```shell
dd if=/dev/urandom of=/dev/sdX bs=4096 status=progress
```

### 2) Partitioning:
Create a new full size partition:
```shell
fdisk /dev/sdX
n
Partirion type: p (primary)
First sector: enter
Last sector: enter
w
```
Create `ext4` filesystem on previously created partition:
```shell
mkfs.ext4 /dev/sdX
```
Create a new mount point:
```shell
mkdir /mnt/nas_drive_1
```
Mount the harddrive partition to the previously created mount point:
```shell
mount /dev/sdaX /mnt/nas_drive_1/
```
Add fstab entry to automount:
```shell
nano /etc/fstab
```
Append this line ↓
```shell
/dev/sda1 /mnt/nas_drive_1 ext4 defaults 0 0
```
Reboot and verify if the drive got auto mounted.
```shell
mount -l | grep /dev/sdaX
```
↑ should result in ↓
```shell
/dev/sdaX on /mnt/nas_drive_1 type ext4 (rw,relatime)
```

### 3) Setting up a samba share:
Install samba:
```shell
apt install samba
```
Create a new user:
```shell
useradd <username>
```
Set passwords for this user:
```shell
passwd -a <username>
smbpasswd -a <username>
```
Add a new directory for the user:
```shell
mkdir /mnt/nas_drive_1/<username>
```
Set permissions:
```shell
chown <username>:<username> /mnt/nas_drive_1/<username>
chmod 770 /mnt/nas_drive_1/<username>
```
Open `/etc/samba/smb.conf` in a text editor and add this line:
```shell
[<share-name>]
path = /mnt/nas_drive_1/<username>
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
