# NAS
This is how I have setup my nas on a rasperry pi, using samba and borg backup.

### Solution description:
- Raspberry pi to provide a low energy cost server machine.
- Samba share with login to access data across the whole LAN. 
- 1 HDD 1Tb (external) -> Will be replaced soon, take a look on the issues.
- Daily differential backups to cloud using borg-backup.

### 1) Disk preparation:
Delete all previous partitions from the harddrive, or wipe it entirely with `dd`.
<br>Partition deleting with `fdisk`:
```shell
fdisk /dev/sdX
d
w
```
<br>Wiping with random data using `dd`:
```shell
dd if=/dev/urandom of=/dev/sdX bs=4096 status=progress
```

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

### 2) Setting up a samba share:
Install samba:
```shell
apt install samba
```
Create a new user:
```shell
useradd <username>
```
Set a samba password for this user:
```shell
smbpasswd -a <username>
```
Add a new directory for the user:
```shell
mkdir /mnt/nas_drive_1/<username>
```
Set permissions:
```shell
chmod -R 0775 /mnt/nas_drive_1/<username>
sudo chown root:<username> /mnt/nas_drive_1/<username>
```
Open `/etc/samba/smb.conf` in a text editor and add this line:
```shell
[SECURED]
path = /mnt/nas_drive_1/<username>
valid users = <username>
browsable = yes
writable = yes
read only = no
```
Restart samba service:
```shell
systemctl restart smbd.service
```
